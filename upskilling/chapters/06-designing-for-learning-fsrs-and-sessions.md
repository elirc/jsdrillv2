# 06: Designing for learning: FSRS and sessions

> Files: `src/lib/fsrs.ts`, `src/lib/sessionBuilder.ts`, `src/lib/progress.ts`, `src/app/api/attempts/route.ts`, `src/components/drill/useDrillSession.ts`, `src/app/app/drill/page.tsx`, `src/components/drill/Answers.tsx`

Reps is a learning product, so its core logic encodes claims about how people learn. A senior engineer on a product like this has to understand the domain (here, memory research) well enough to turn it into code, and to know which parts are proven and which are judgement calls.

## FSRS in one paragraph

FSRS (Free Spaced Repetition Scheduler, via the `ts-fsrs` package) models each card with a **stability** (roughly: the number of days until recall probability falls to the target) and a **difficulty**. After each review you give it a rating: `Again` (1), `Hard` (2), `Good` (3) or `Easy` (4). It updates stability and difficulty and returns the next due date. Reps configures it in `src/lib/fsrs.ts`:

```ts
const scheduler = fsrs(generatorParameters({
  request_retention: 0.9,   // schedule reviews for when recall ≈ 90%
  enable_fuzz: true,        // jitter intervals so reviews don't clump on one day
}));
```

A 90% retention target is the standard trade-off. Higher means many more reviews for little extra retention. Lower means you forget more often between reviews.

## Deriving a rating from behaviour

Flashcard apps ask the user to rate themselves. Reps mostly *doesn't*, because it can observe correctness and time. `ratingFor(item, grade, timeSpent, reps)`:

```ts
if (grade.selfGraded) { /* explain-it: map the learner's 1-4 straight through */ }

if (!grade.correct) {
  return grade.score >= 0.75 ? Rating.Hard : Rating.Again;
}

const budget = item.estSeconds || 45;
if (timeSpent > budget * 1.6) return Rating.Hard;              // right, but not fluent
if (timeSpent < budget * 0.5 && reps > 0) return Rating.Easy;  // fast, and not a first exposure
return Rating.Good;
```

Decisions worth noticing:

- **Correctness first, speed second.** Speed only breaks ties among correct answers. A fast wrong answer is still `Again`.
- **A near miss is not a blank.** An incorrect answer with `score >= 0.75` (3 of 4 blanks, say) is `Hard`, not `Again`. The README table says "Wrong → Again", which is a simplification. The code is more precise, and when prose and code disagree, the code is what runs.
- **`Easy` requires `reps > 0`.** On first exposure you might be fast because the answer looked obvious, not because you remember it. Only a *review* can be Easy.
- **`estSeconds` is per kind by default** (`base(…, defaultSecs)` in `builder.ts`: `tf` 25s, `mcq` 40s, `code` 240s). So "slow" means slow *for that kind of question*.
- **Explain-it items are self-graded.** Only the learner knows whether they'd have satisfied an interviewer. `gradeSelf()` maps Blanked/Shaky/Solid/Nailed (1 to 4) to `score = (r - 1) / 3`, and `ratingFor` maps it back with `Math.round(score * 3) + 1`. The round trip exists so self-graded items flow through the same `Grade` shape as every other kind.

## The second attempt (c027fca)

For `mcq`, `predict-output` and `fill-blank` (`RETRYABLE_KINDS` in `Answers.tsx`), a first miss does not reveal the answer. The wrong choice is eliminated (correct blanks are locked) and the learner tries once more.

The scoring rule is in two places. The client computes it for display, and the server enforces it in `POST /api/attempts`:

```ts
if (attempt === 2) {
  verdict.attempt = 2;
  if (verdict.correct) verdict.score = Math.min(verdict.score, 0.5);
}
// …
const rating = verdict.attempt === 2 && verdict.correct
  ? Rating.Hard
  : ratingFor(item, verdict, paceSeconds, card.reps);
```

(`paceSeconds` is the reported `timeSpent`, or the item's `estSeconds` when the client did not report one, so a missing time is rated as "on pace" rather than as a lightning-fast 0 seconds.)

Why: **a near miss is worth more than a peek and less than knowing it.** Retrieving the answer yourself, even on a second try, strengthens memory more than reading it. But it isn't fluency, so it's scheduled `Hard` and comes back soon. "Just show me" records the *first* response as a miss (`giveUp()` in `useDrillSession.ts` uses the `firstResponse` ref). Interview mode never retries, because a real interview doesn't give you a second go.

`multi`, `truefalse`, `order` and `code` are deliberately not retryable. A true/false retry is a free guaranteed win. Multi and order already give partial credit. Code has its own iterate-until-green loop.

## Building a session

`buildSession(spec)` in `sessionBuilder.ts` does this for the default `mixed` mode:

1. **Overdue reviews first, up to 60%.** `reviewBudget = Math.ceil(size * 0.6)`, taking the most overdue cards first (sorted by `due`). Since 2026-09-23, `buildSession()` reads the published items and the user's cards once per request and filters in memory. Cards for unpublished items are dropped at that point, so a removed item is never scheduled again.
2. **Fill with new items** from unlocked levels. Module- and level-scoped sessions skip the level gate: *"the learner asked for it explicitly."*
3. **Still short? Fill** with seen, not-yet-due items, shuffled.
4. **Interleave by track** so consecutive items rarely share a technology.

`weak` mode uses the whole budget for reviews and never adds new material. `interview` mode is shaped differently (below).

### Interleaving: desirable difficulty

`interleaveByTrack()` picks, each time, the first remaining item whose track differs from the previous one. Blocked practice (ten React questions in a row) *feels* easier and produces worse retention. The learner stays in one mental context and never has to work out *which* knowledge applies. Interleaving makes each retrieval harder, and that effort is what makes it stick. This is a **desirable difficulty**: the session feels harder and teaches more.

### Spreading new material across modules

`spreadAcrossModules()` groups new items by module, **sorts each module's queue by difficulty (easier first, so a new module opens gently)**, shuffles the module order, then round-robins. Without this, a fresh learner's first session could be five questions from one module, with the hardest trap question first.

### Primers

`enrich()` works out `touchedModules`: modules with at least one card for this user. For any other module it sets `moduleKeyIdeas`, and the drill shows those key ideas above the first question. In `c027fca` a `primedModules` set on the drill page stopped the primer repeating within a session. Today the page derives it instead: `firstFromModule` is true only if no earlier item in the session came from the same module, and `showPrimer` also skips `module`-mode sessions, which are always entered from a page that already lists the key ideas. **Pre-questions work better when the learner has a scaffold.** Being asked about something you have never heard of just teaches you to guess.

### Interview mode

`pickInterviewSet()` draws from levels up to one above what's unlocked (`(unlocked.get(i.trackId) ?? 1) + 1`, capped at 4), round-robins across tracks, and allows at most two items per module. Coverage matters more than depth here, because that is what a screening interview is like.

## Measuring progress

`masteryScore()` in `progress.ts`:

```ts
const retention = Math.min(1, card.stability / MASTERY_STABILITY_DAYS); // 21 days
const accuracy  = card.totalCount > 0 ? card.correctCount / card.totalCount : 0;
total += retention * 0.7 + accuracy * 0.3;
// … averaged over ALL items in scope, unseen ones counting 0
```

Retention dominates, because the goal is long-term memory. Accuracy is there so that *"a lucky guess"* doesn't read as mastery. Dividing by all items, not just the seen ones, means coverage is built in: you can't reach 100% on a track by mastering three items.

Keep **mastery** (a display metric, driven by stability) separate from **unlocking** (a gate, driven by coverage and accuracy; bug 5c). They answer different questions: "how durable is what you know?" versus "are you ready for the next material?". Using one number for both is what caused bug 5c.

## What is judgement, not science

The research supports the principles: spacing, retrieval practice, interleaving, pre-questions with scaffolds. It does *not* pick the constants. 60/40, 0.7/0.3, the 1.6× slow threshold, the 0.75 near-miss score, the 80%/70% unlock gate and the 21-day mastery bar are all **tunable judgement calls**. They are named constants or clear literals, not buried magic numbers, so they can be tuned with data later. A senior engineer says which of their numbers are measured and which are guesses.

## Try it yourself

1. Write a table-driven test for `ratingFor` with `estSeconds = 40`: wrong with score 0 → Again; wrong with score 0.75 → Hard; right at 70s → Hard; right at 15s with reps 0 → Good; right at 15s with reps 2 → Easy.
2. Call `buildSession({ mode: "mixed", size: 12 })` on a fresh DB from a scratch `tsx` script. Print `trackId` in order and count adjacent repeats. Then replace `interleaveByTrack` with identity and count again.
3. Change `reviewBudget` to `size`. What happens to a learner with 50 overdue cards? Argue for or against a cap.
4. `maybeUnlockNextLevel` is only called when `verdict.correct` is true (`attempts/route.ts`). Construct a sequence of answers where this matters, or prove it can't.
5. Find the field the database forgets. Goal: explain why `cardFromDb` in `src/lib/fsrs.ts` sets `learning_steps: 0` on every read, and when that matters. **Check:** `grep -n "learning_steps" -r src` prints only the line in `cardFromDb`; `CardRow` and the `user_cards` table have no such column, so a card's position inside its learning steps is reset on every review, and the `as unknown as Card` cast hides it from the compiler. (Added 2026-10-06, after running ts-fsrs 5.2.3 on its own: with the default two learning steps this app uses, keeping, zeroing or dropping the field gave identical due dates for Again/Hard/Good sequences; with four learning steps (`1m, 10m, 1h, 4h`) a zeroed card stayed at the 60-minute step through repeated Good ratings instead of graduating. The bug is latent until someone configures longer steps.) Write down the schema change, the two mapping changes, and what value existing rows should get.

> **Junior vs senior**
>
> **Junior:** "Right answer → Good, wrong → Again. Sessions pick 12 random items." One "progress" number drives both the display and the gate.
>
> **Senior:** "Correctness first, then partial credit, then speed, and `Easy` only on review. Mix reviews with new work, interleave by track, open new modules gently with a primer. Keep retention (mastery) separate from readiness (unlock). And say which constants come from research and which are judgement."
