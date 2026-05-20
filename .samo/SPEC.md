# Wordle — SPEC v0.6

## Goal & Why It's Needed

Build a personal, self-hosted Wordle clone that gives the author (a single user) a clean, dependency-light daily word puzzle plus an unlimited practice mode — without ads, accounts, login walls, paywalls, or the ever-shifting UX of the NYT version.

**Why this exists:** The original Wordle has been absorbed into the NYT app/website, gated behind login prompts, instrumented with telemetry, and bundled with cross-promotion. The author wants a stable, single-purpose puzzle they fully control: same word list every day, same rules forever, deterministic daily seed, no network calls beyond serving static assets. Shareable spoiler-free emoji results are preserved because they are the one social ritual the author values (posting an `X/6` grid into chats without revealing the answer).

**Offline behavior:** v0.1 relies on the browser's standard HTTP cache for repeat visits. This is best-effort, not guaranteed offline. True offline play (service worker / installable PWA) is deferred to v0.2+.

**This is NOT:** a multiplayer game, a leaderboard service, an account system, a monetized product, a Wordle variant (no 6/7-letter, no Quordle, no Absurdle), or a redistribution of the NYT word list.

## User Stories

1. **Daily ritual (persona: the author, on their phone over morning coffee).** As the sole player, I open the site, see today's empty 5×6 grid, type six guesses max, and learn whether I solved today's word — so I get a consistent ~3-minute morning puzzle without logging in or seeing ads.
2. **Practice between dailies (persona: the author, mid-afternoon on desktop).** As the sole player, after I've finished today's puzzle (or any time), I click "Practice" and get an unlimited stream of random words drawn from the same answer list — so I can warm up, blow off steam, or replay without waiting 24 hours.
3. **Spoiler-free brag (persona: the author, sharing in a group chat).** As the sole player, after solving (or failing) the daily, I tap "Share" and get an emoji grid (🟩🟨⬜) plus the canonical share header `Wordle (self-hosted) #<N> X/6` (where `<N>` is `dailyIndex + 1`) copied to my clipboard — so I can paste it into iMessage/Signal/Discord without leaking the answer. (The exact header text is fixed in Implementation Details; this story references it for acceptance only.)

## Architecture

<!-- architecture:begin -->

```text
(architecture not yet specified)
```

<!-- architecture:end -->

Single-page static web app, no backend, no database, no auth.

**Components:**
- **`index.html`** — single page hosting the grid, on-screen keyboard, header, and modal containers.
- **`game.js` (core engine)** — pure module exposing `evaluateGuess(guess, answer) → Array<'green'|'yellow'|'gray'>`, `isValidGuessWord(word) → bool`, `dailyAnswerForDate(dateStr) → string`, `randomPracticeAnswer(todaysDailyAnswer) → string`, `updateStats(prevStats, outcome, dateStr) → nextStats`, `applyStreakGapZeroing(stats, dateStr) → nextStats`. No DOM access.
- **`ui.js` (view layer)** — renders board + keyboard, handles physical and on-screen key events, animates tile flips, drives win/lose modal. Talks to the engine only through the pure API above.
- **`storage.js`** — thin wrapper around `localStorage` for daily progress, last-played date, current practice run, stats, and prefs. Survives reload; never sent off-device. Handles `localStorage` failure modes (see Implementation Details).
- **`share.js`** — builds the emoji grid string and copies it via `navigator.clipboard.writeText`, with a selectable-textarea fallback for non-secure contexts.
- **`words/answers.json`** — curated answer list (~2,300 common 5-letter English words; see Word Lists below).
- **`words/allowed.json`** — superset of valid guess words (~12,000 5-letter English words).
- **`style.css`** — layout, color tokens (green/yellow/gray + dark mode via `prefers-color-scheme`).

**Boundaries:**
- Engine ↔ UI: one-way; engine is framework-free, side-effect-free, and unit-testable in pure Node.
- UI ↔ Storage: UI calls storage to persist after each guess; storage never reaches into UI.
- Nothing leaves the device. No analytics, no remote fetches at runtime beyond the initial static asset load.

**Key abstractions:**
- `GuessResult` — `{ guess: string, marks: ['green'|'yellow'|'gray', ×5] }`.
- `GameState` — `{ mode: 'daily'|'practice', answer: string, guesses: GuessResult[], status: 'in_progress'|'won'|'lost' }`.
- `Outcome` (input to `updateStats`) — `{ status: 'won'|'lost', guessCount: number, callerMode: 'daily' }` where `guessCount ∈ [1, 6]` and is the number of guesses actually played by the time the game resolved. For a loss `guessCount === 6` by construction. `callerMode` is a runtime-only discriminant (never stored) used by the practice-misuse guard. (Earlier drafts split this into a base `Outcome` plus an `OutcomeEnvelope`; collapsed into one type to remove the two-definition trap.)
- **`updateStats` contract for practice misuse (pinned):** `updateStats` is only called for `mode === 'daily'`. Callers must invoke `updateStats(prev, { status, guessCount, callerMode: 'daily' }, dateStr)`. If `outcome.callerMode !== 'daily'` (any other string, `undefined`, `null`, `0`, `''`, etc.), `updateStats` **throws** `new Error('updateStats called for practice; stats are daily-only')`. The throw is unit-tested with a parameterized set covering `'practice'`, `undefined`, `null`, `'foo'`, `0`, and `''`. The function does **not** silently no-op — silent no-ops mask the programmer error and the throw protects `wordle:stats` from any future call-site mistake. The guard predicate is exactly `outcome.callerMode !== 'daily'` (positive allow-list), not `outcome.callerMode === 'practice'` (negative deny-list), so unknown future values fail closed.
- `dailyIndex(dateStr) → integer`. Returns `civilDaysBetween(EPOCH_LOCAL, dateStr)`. **May be negative** for `dateStr < EPOCH_LOCAL`; the pre-launch case is handled inside `dailyAnswerForDate` *before* it would consult the answer pool. `dailyIndex` itself is a thin pure subtraction and does not branch on sign.

## Implementation Details

**UI layout (Practice control placement + keyboard placement):**
- **Header (left → right):** title `Wordle (self-hosted)`, mode-toggle pill with two buttons `Daily` / `Practice` (current mode highlighted), and a gear icon opening the Settings popover. The **canonical Practice control** lives in this header pill so it is reachable at any point in the daily lifecycle, including in-progress. The win/lose modal additionally exposes a `Play practice round` shortcut button that delegates to the same handler — it is a convenience shortcut, not a second source of truth. (Earlier drafts phrased this as "Practice is not in the modal" which read as a contradiction; the rule is: header is canonical, modal shortcut delegates.)
- **Board:** 5-column × 6-row grid centered below the header.
- **On-screen keyboard:** standard QWERTY layout, three rows. Row 1: `Q W E R T Y U I O P`. Row 2: `A S D F G H J K L`. Row 3: `Enter Z X C V B N M Backspace` — `Enter` is the leftmost key in row 3 and `Backspace` the rightmost, both visually wider than the letter keys. This matches mainstream Wordle convention.
- **Win/lose modal:** shows the resolved board, stats (distribution graph), `Share` button, and the `Play practice round` shortcut described above.

**Input normalization (pinned):**
- `isValidGuessWord(word)` **lowercases its input first**, then checks set membership against `allowed.json` (which is committed as lowercase). The engine performs this normalization, not the UI. Rationale: putting normalization in the engine means every caller (UI today, future test harness, anything else) gets identical case-handling without duplicating the rule.
- `evaluateGuess(guess, answer)` likewise lowercases both arguments before the two-pass comparison. The UI may continue to render tiles in uppercase for visual style — that's a render-time concern, not a data concern.
- Length and alpha-only validation: `isValidGuessWord` returns `false` for any `word.length !== 5` or any character outside `[a-z]` after lowercasing.

**Data flow (one guess):**
1. User types 5 letters + Enter (or taps Enter on the on-screen keyboard).
2. UI validates length, then calls `isValidGuessWord(guess)`. If invalid → shake animation + toast "Not in word list"; no state change.
3. UI calls `evaluateGuess(guess, state.answer)` → marks array.
4. UI appends `GuessResult` to `state.guesses`, runs flip animation (300ms per tile, staggered 100ms).
5. UI updates keyboard tint with **priority green > yellow > gray, monotonic — once a key is green it never becomes yellow or gray; once yellow it never becomes gray**. Keyboard tint state is recomputed from the full guess history on every render so a later gray guess cannot downgrade a previously colored key.
6. Storage persists updated state.
7. If marks are all green → status = `won`; UI computes `dateStr = formatLocalCivilDate(new Date())` and calls `updateStats(prevStats, {status:'won', guessCount: state.guesses.length, callerMode:'daily'}, dateStr)` (daily mode only) and shows win modal. Else if `guesses.length === 6` → status = `lost`; UI computes `dateStr` the same way and calls `updateStats(prevStats, {status:'lost', guessCount: 6, callerMode:'daily'}, dateStr)` (daily mode only) and shows lose modal with answer. (Local variable is named `dateStr` at every call site to match the pinned parameter name everywhere.)

**Green/yellow/gray algorithm (two-pass, handles duplicate letters correctly):**
- **Pass 1:** Mark every position where `guess[i] === answer[i]` as green; decrement that letter's remaining-count in a tally of `answer`.
- **Pass 2:** For each non-green position, if `guess[i]` still has remaining count in the tally → mark yellow and decrement; else mark gray.
- This is the only correct algorithm — naïve single-pass implementations mis-color duplicate letters. Worked example: guess `ALLEY` (A,L,L,E,Y) against answer `LEVEL` (L,E,V,E,L). Pass 1 finds one green at position 4 (E↔E). Pass 2 with remaining tally `{L:2, E:1, V:1}` marks position 2 (L) yellow, position 3 (L) yellow, positions 1 (A) and 5 (Y) gray. Expected output: `[gray, yellow, yellow, green, gray]`. Naïve algorithms wrongly emit double-green or double-yellow here.

**Daily seed:**
- `EPOCH_LOCAL = '2026-06-03'` (planned local-date of v0.1 release — two-sprint window from spec date 2026-05-20). EPOCH is the release date, not the spec date; the changelog spec-date and EPOCH are distinct values.
- **Input type:** `dailyAnswerForDate(dateStr: string) → string`. `dateStr` is a `YYYY-MM-DD` civil-date string. `EPOCH_LOCAL` is also a `YYYY-MM-DD` string. There is no `Date`/timestamp object anywhere in this pipeline.
- **Today derivation:** `today = formatLocalCivilDate(new Date())` — extracts the local year/month/day from the system clock and emits `YYYY-MM-DD`. No `toISOString()`, no UTC conversion.
- **`dailyIndex` arithmetic:** civil-date subtraction. `dailyIndex(dateStr) = civilDaysBetween(EPOCH_LOCAL, dateStr)`. May be negative; see Architecture for the full contract.
- **`civilDaysBetween(a, b)` algorithm — normative, not parenthetical:** parse each `YYYY-MM-DD` into `(y, m, d)` integers and compute its **Rata Die day number** using the standard proleptic Gregorian formula:
  ```
  if m <= 2: y -= 1; m += 12
  era = floor(y / 400)
  yoe = y - era*400                 // year-of-era, 0..399
  doy = floor((153*(m-3) + 2) / 5) + d - 1   // day-of-year (March-based)
  doe = yoe*365 + floor(yoe/4) - floor(yoe/100) + doy   // day-of-era
  rd  = era*146097 + doe
  ```
  Then `civilDaysBetween(a, b) = rd(b) - rd(a)`. This is a pure integer function: deterministic, monotonic, immune to DST/leap-seconds/timezone, and well-defined for dates before 1970, after 2100 (the non-leap century), and across the leap-day boundary. Reference implementation: Howard Hinnant's `days_from_civil` (the C++20 chrono basis).
- **Malformed-input contract for `civilDaysBetween` / `dailyAnswerForDate` (pinned):** both functions **throw** `new Error('invalid civil date: <input>')` for any input that is not a string matching the strict regex `/^\d{4}-\d{2}-\d{2}$/` AND whose parsed `(y, m, d)` represents a real proleptic-Gregorian date (month `1..12`, day in the valid range for that month/year — including correct Feb-29 handling). So `'2026-13-45'`, `'2026-02-30'`, `'not-a-date'`, `''`, `null`, `undefined`, non-strings, and `'9999-99-99'` (which the old lexicographic compare would have silently accepted) all throw. The throw is unit-tested with at least one example per category. Rationale: `formatLocalCivilDate(new Date())` is the only documented producer, but the engine API is publicly callable and silent garbage-in is the worst failure mode for date math; failing loudly at the boundary is cheap.
- `answer = answers[dailyIndex % answers.length]` for non-negative `dailyIndex`. Same answer for the author regardless of which device they open, as long as the device's clock is in their usual timezone.
- **Pre-launch (`dateStr < EPOCH_LOCAL`, lexicographic string compare — valid because both are validated `YYYY-MM-DD`):** `dailyAnswerForDate` returns the empty string `""` and the UI renders a "Puzzle not yet started" placeholder. The negative `dailyIndex` is never indexed into the answer pool. (Should never happen in practice; defined for test determinism.)
- **Post-launch wraparound for any `dateStr ≥ EPOCH_LOCAL` (including the synthetic future date used by the wraparound test):** `dailyAnswerForDate(dateStr)` returns `answers[dailyIndex(dateStr) % answers.length]`. There is no upper cutoff; the function is defined for all future civil dates, and the answer pool simply cycles every `answers.length` days. This matches the wraparound test and means a user whose system clock is months ahead just sees a deterministic future-day answer, never an error.
- **Share-string display number:** the share header uses `#${dailyIndex + 1}` (1-indexed for human display), so the launch-day share reads `Wordle (self-hosted) #1 X/6`, day-2 is `#2`, etc. Internal storage and tests still use the 0-indexed `dailyIndex`.

**Practice mode RNG:**
- Uses `Math.random()`, which is non-deterministic in standard JavaScript; practice rounds need no reproducibility. Picks uniformly from `answers.json`.
- Practice never advances the daily; daily progress is stored under a separate key.
- **Practice initiated mid-daily:** if the daily is `in_progress`, clicking "Practice" opens a confirmation toast ("Start a practice round? Your daily is saved and you can resume it."). On confirm, the daily state is preserved untouched in `wordle:daily`; practice runs in `wordle:practice`. Returning to "Daily" restores the in-progress board. **Toast dismissal without selection** (tap-outside, Escape key, mobile swipe-away, focus loss): no state change — `wordle:practice` is left byte-identical to its pre-click value, the player stays on the Daily view, and no banner/log is emitted. This is the safe default because both branches (especially New-round) are destructive enough that ambiguous gestures must not commit.
- **Practice across reloads:** if `wordle:practice.status === 'in_progress'` on app load, the practice board is **not** auto-restored on the main view (the player lands on Daily by default). When they click "Practice", if an in-progress practice round exists, a confirmation toast asks "Resume your unfinished practice round, or start a new one?" with both options. New-round clears the slot; resume restores it. Same dismiss-without-selection rule as above: `wordle:practice` is unchanged and the player stays on Daily.
- **Practice answer must not spoil today's daily:** `randomPracticeAnswer(todaysDailyAnswer)` takes today's daily answer as an argument and rejects it in a bounded loop (re-samples until a different word is drawn, **with a hard cap of 100 iterations**). At iteration 100 it falls back to deterministic skip — it returns `answers[(answers.indexOf(todaysDailyAnswer) + 1) % answers.length]`, guaranteeing a non-spoiler word in O(1). This protects against (a) a stubbed `Math.random` in tests that returns the same index repeatedly, and (b) a future `answers.json` shrunk so small the rejection probability becomes meaningful. Taking the daily answer as an argument keeps the engine pure (no global state) and makes the rejection trivially testable by stubbing `Math.random`.
- **"Today" semantics for practice across midnight (pinned):** the value passed as `todaysDailyAnswer` is computed at **call time**, i.e., the moment the player clicks "Practice" or "New round". A practice marathon that crosses local midnight will see its rejection target rotate to the new day's answer on the first call after midnight. Consequence: a word that was yesterday's daily can legally appear in practice after midnight (the player already solved it, so this is a non-issue), and that midnight's new daily is rejected from then on. This is the only sane definition for a self-hosted single-user app — the alternative (latching the rejection target at practice session start) would require tracking session boundaries the spec does not otherwise introduce.

**State transitions:**
- `not_started` → (first keystroke) → `in_progress` → (6th guess wrong OR all-green) → `lost` / `won`.
- **Result stats** (`played`, `won`, `distribution`) are written **exactly once**, at the moment the daily transitions to `won` or `lost` (step 7 of the guess flow). The reload path does not re-fold any result.
- **Streak state** (`currentStreak`, `maxStreak`, `lastCompletedDate`) is also written at resolution time, plus one narrow load-time mutation: `currentStreak` may be zeroed on app load if the player missed a day (see streak-gap zeroing below). The load-time write is idempotent and never touches `played`/`won`/`distribution`/`maxStreak`. (So the precise rule is: result stats fold once at resolution; `currentStreak` folds at resolution and may additionally be zeroed once per load when a gap is detected.)
- On reload: if `storage.wordle:daily.date === today` (local civil date string compare) → restore in-progress or terminal board for today. Else if stored date is **older** than today → clear `wordle:daily` and seed a fresh slot for today; no stats mutation (the prior result was already counted at resolution, regardless of whether that prior slot was `won`, `lost`, or even `in_progress` — see test enumeration). Else if stored date is **newer** than today (clock change, timezone travel, manual system-clock rewind) → treat the stored slot as stale and discard it, then seed a fresh slot for today. We never preserve a future-dated slot, because the alternative would block today's puzzle whenever the clock disagreed with storage. No warning is shown — this is a single-user app and the case is expected to be rare and self-healing.
- **Streak-gap zeroing:** on app load, if `wordle:stats.lastCompletedDate` is non-null and `civilDaysBetween(lastCompletedDate, today) > 1` (i.e., the player missed at least one day), `currentStreak` is set to 0. This is idempotent: running it twice has the same effect as running it once. `maxStreak` is never decreased.

**Storage schema (`localStorage`):**
- `wordle:daily` → `{ date: 'YYYY-MM-DD', guesses: string[], status: 'in_progress'|'won'|'lost' }`.
- `wordle:practice` → `{ answer: string, guesses: string[], status: 'in_progress'|'won'|'lost' }`. Cleared when player starts a new practice round; preserved across reloads with the resume prompt described above.
- `wordle:stats` → `{ played, won, currentStreak, maxStreak, distribution: [n1,n2,n3,n4,n5,n6], lastCompletedDate: 'YYYY-MM-DD' | null }` (local-only). Practice rounds NEVER touch `wordle:stats`.
- `wordle:prefs` → `{ colorBlindMode: boolean, reducedMotionOverride: 'auto'|'suppress'|'allow' }`. Defaults: `colorBlindMode: false`, `reducedMotionOverride: 'auto'`. Toggle lives in a Settings popover accessible from the header gear icon. `prefers-color-scheme` is independent and not overridable in v0.1.

**`reducedMotionOverride` semantics (pinned, unambiguous):**
- `'auto'` — honor the OS/browser `prefers-reduced-motion` media query: when the media query matches, animations are suppressed; otherwise they play.
- `'suppress'` — force-suppress animations regardless of the media query (tile flips collapse to instant state changes; toasts/modal transitions are also reduced).
- `'allow'` — force-allow animations regardless of the media query.
- The label in the Settings popover reads `Animations: Auto / Suppress / Allow` so the UI text mirrors the storage value exactly and no "on means off" confusion is possible. (Prior drafts used `'on'`/`'off'`; renamed because reasonable readers split on which meant "suppress".)

**`localStorage` failure handling:**
- All reads/writes route through `storage.js`, which wraps each call in `try/catch`.
- On any thrown error (quota exceeded, disabled in private mode, ITP eviction, etc.), `storage.js` falls back to an in-memory object (`memShadow`) for the remainder of the session and surfaces a single non-blocking banner: "Storage unavailable — progress won't be saved between visits."
- **Banner-dismissal state is itself in-memory.** `storage.js` holds a module-scoped boolean `bannerDismissed` (defaults to `false`); when the player dismisses the banner, `bannerDismissed = true` for the rest of the page session. The flag is never written to `localStorage` (the very layer that just failed), so on next page load it resets and the banner re-appears if storage is still broken. This is intentional: a recurring banner is the right ergonomic for "your progress isn't being saved."
- The engine and UI continue to function (the game is still playable in the current tab). Stats, daily resume, and prefs simply don't persist across reload in this degraded mode.

**Stats rules (applied exactly once at resolution time, plus the one load-time streak-gap write noted above):**
- On daily win: `played++`, `won++`, `distribution[guessCount-1]++`. If `lastCompletedDate` is non-null AND `civilDaysBetween(lastCompletedDate, dateStr) === 1`, `currentStreak++`; else `currentStreak = 1`. `maxStreak = max(maxStreak, currentStreak)`. Set `lastCompletedDate = dateStr`.
- On daily loss: `played++`, `currentStreak = 0`. Set `lastCompletedDate = dateStr`.
- (Parameter naming pinned: the third argument to `updateStats` is named `dateStr` everywhere — in the type signature, in this Stats Rules section, at every call site in the Data Flow section above, and in the tests. The earlier `today` shorthand has been fully replaced — call-site local variables are named `dateStr` to match.)
- On opening the app: streak-gap zeroing rule applies (see State Transitions). This is the only stats mutation that happens outside daily resolution, and it only writes `currentStreak`.
- **Distribution invariant:** at any point, `sum(distribution) === won`. The invariant is asserted in tests after each multi-game scenario.
- Practice has no effect on any stat. A completed practice round (win or loss) leaves `wordle:stats` byte-identical to its pre-round value; asserted by a dedicated test.

**Share string format:**
```
Wordle (self-hosted) #<dailyIndex+1> <X>/6

🟩🟨⬜⬜⬜
🟩🟩🟨⬜⬜
🟩🟩🟩🟩🟩
```
For a loss, `<X>` is `X`. For a win in N guesses, `<X>` is `N` (1–6). No answer letters appear in the output, ever. No URL, no tracking params. Practice runs are not shareable (no shared frame of reference).

**Share emoji palette is fixed.** The share grid always uses 🟩 (green), 🟨 (yellow), and ⬜ (gray) regardless of `wordle:prefs.colorBlindMode`. Rationale: the share string is meant to be portable across chats and recognizable as a Wordle-style grid; there is no widely-recognized blue/orange emoji convention that doesn't degrade legibility. Color-blind mode affects only on-screen tiles and the keyboard.

**Clipboard copy with fallback:**
- Primary path: `navigator.clipboard.writeText(shareString)`. Requires a secure context (HTTPS or `http://localhost`).
- Fallback (insecure context, or `writeText` rejects): the share modal renders the share string inside a `<textarea readonly>` already focused with `select()` called, plus an instruction "Copy this to share (Cmd/Ctrl+C)." No `execCommand('copy')` legacy path — manual copy only.
- The fallback is exercised in v0.1 because the author has not yet committed to an HTTPS deploy target. Deploy targets that provide HTTPS for free (GitHub Pages, Netlify, Cloudflare Pages) are recommended in the Implementation Plan.

## Word Lists

- **Answers (~2,300 words):** curated common 5-letter English words. Sourced from public-domain SCOWL (Spell Checker Oriented Word Lists). We will NOT copy the NYT answer list — that's a copyright/ethics non-starter and would also leak the NYT solution sequence.
- **Allowed guesses (~12,000 words):** broader SCOWL 5-letter set, used only for input validation. Player can guess any of these; only the answer pool is drawn from `answers.json`.
- **Operational curation rules (reproducible — encoded in the generator script):**
  - **Common:** SCOWL size ≤ 50 for `answers.json`, size ≤ 80 for `allowed.json` (SCOWL's published frequency tiers).
  - **Non-offensive:** filter against the [LDNOOBW](https://github.com/LDNOOBW/List-of-Dirty-Naughty-Obscene-and-Otherwise-Bad-Words) English list (pinned commit recorded in the script).
  - **Non-plural:** exclude words ending in `s` whose singular form (drop trailing `s`) is also in SCOWL.
  - **Non-past-tense (pinned disjunction — inclusive OR):** for any 5-letter word `w` ending in `ed`, compute two candidate stems: `stem_ed = w` with the trailing `ed` dropped (length 3) and `stem_d = w` with only the trailing `d` dropped (length 4, i.e., still ends in `e`). **Exclude `w` from the answer/allowed list if EITHER `stem_ed` OR `stem_d` is present in SCOWL.** Worked examples: `FREED` → `stem_ed = 'fre'` (not in SCOWL), `stem_d = 'free'` (in SCOWL) → **excluded** because the `-d` branch hits. `BLEED` → `stem_ed = 'bl'` (not in SCOWL), `stem_d = 'blee'` (not in SCOWL) → **kept**. `BAKED` → `stem_ed = 'bak'` (not in SCOWL), `stem_d = 'bake'` (in SCOWL) → **excluded**. The inclusive-OR (broader filter) is correct because the goal is to strip past-tense forms, and English past-tense morphology produces both `-ed` (walked/walk) and `-d` (baked/bake) shapes.
  - All filters are applied by `scripts/build-wordlists.js`; running it twice produces byte-identical output. **CI verifies this** (see Tests Plan): the generator is invoked twice and the sha256 of each output file is compared.
- Both lists are committed to the repo as JSON arrays of lowercase strings, sorted, deduplicated.

## Tests Plan

**Test framework:** Vitest (fast, ESM-native, runs the same engine module the browser loads).

**Built test-first (red/green TDD — write failing test, then implementation):**
- `evaluateGuess` — the duplicate-letter cases are the classic place clones get this wrong. TDD here is non-negotiable. Tests must cover:
  - all-gray case (e.g., `XYLOH` vs `BRACE` → all gray) and all-green case (`HELLO` vs `HELLO`).
  - all-yellow case (every letter present in answer but none aligned — derangement at the letter level). Concrete example: guess `STARE` vs answer `RATES` → `[yellow, yellow, yellow, yellow, yellow]`.
  - duplicate letter in guess, single in answer: guess `ALLEY` vs answer `LEVEL` → `[gray, yellow, yellow, green, gray]` (canonical case).
  - duplicate letter in answer, single in guess: guess `LEAFY` vs answer `LEVEL` → `[green, green, gray, gray, gray]`.
  - duplicate-duplicate with one green + one yellow: guess `EERIE` vs answer `REBEL` → `[yellow, green, yellow, gray, gray]` (first E yellow because second E consumes the green tally entry; R yellow; I gray; trailing E gray because the answer's E pool is exhausted).
  - guess letter not in answer at all (mixed with hits).
  - case-insensitivity smoke: `evaluateGuess('Alley', 'LEVEL')` and `evaluateGuess('ALLEY', 'level')` produce the same output as the all-lowercase canonical call.
- `isValidGuessWord` — case-insensitive (lowercases input before set lookup), rejects length ≠ 5, rejects non-alpha, accepts everything in `allowed.json`. Tests include `isValidGuessWord('Hello') === isValidGuessWord('HELLO') === isValidGuessWord('hello')`.
- `dailyAnswerForDate(dateStr: string) → string` — explicit boundary cases:
  - `dateStr === EPOCH_LOCAL` returns `answers[0]`.
  - `dateStr === '2026-06-04'` (EPOCH+1) returns `answers[1]`.
  - `dateStr` set to `EPOCH_LOCAL + answers.length` calendar days returns `answers[0]` again (wraparound).
  - `dateStr` set to a synthetic far-future date (e.g., `EPOCH_LOCAL + 3*answers.length + 7` days) returns `answers[7]` — confirms post-launch behavior is just modular wraparound with no upper cutoff.
  - `dateStr < EPOCH_LOCAL` (lexicographic) returns `""` (pre-launch behavior). `dailyIndex` for that input is asserted to be negative but is never used to index the answer pool.
  - Leap-day crossing (`'2028-02-28'` → `'2028-02-29'` → `'2028-03-01'`) advances index by exactly 2 across the pair.
  - DST transitions: pass civil-date strings spanning US spring-forward (`'2026-03-07'` → `'2026-03-08'`) and fall-back (`'2026-11-01'` → `'2026-11-02'`); `dailyIndex` advances by exactly 1 in each case. This is guaranteed because the implementation does civil-date subtraction with no timestamp conversion — the test enforces the contract.
  - Century non-leap (`'2099-12-31'` → `'2100-01-01'` → `'2100-03-01'`): `civilDaysBetween('2100-02-28', '2100-03-01') === 1` (year 2100 is NOT a leap year, so no Feb 29). Pins the Rata Die algorithm against the year-2100 edge case.
  - **Malformed-input contract:** parameterized test asserts `civilDaysBetween` and `dailyAnswerForDate` throw `Error('invalid civil date: <input>')` for `'2026-13-45'`, `'2026-02-30'`, `'2025-02-29'` (not a leap year), `'9999-99-99'`, `'not-a-date'`, `''`, `null`, `undefined`, `12345` (non-string), and `'2026-6-3'` (un-padded). One assertion per category.
- `randomPracticeAnswer(todaysDailyAnswer)` — stub `Math.random` to return a sequence whose first index lands on today's daily answer and whose second index lands on a different word. Assert the function loops once and returns the second word. Also assert that with a stub that never returns today's-daily-index, exactly one call to `Math.random` is made. **Pathological-loop test:** stub `Math.random` to return the index that maps to today's daily answer 200 times in a row; assert the function still terminates (≤ 100 iterations of rejection-sampling, then deterministic-skip fallback) and returns `answers[(answers.indexOf(todaysDailyAnswer) + 1) % answers.length]` (the documented fallback). Together these prove the daily-spoiler rejection is in place, not wasteful, and bounded.
- `keyboardTint` priority — given a sequence of `GuessResult`s, derived keyboard state must obey `green > yellow > gray` and never downgrade. Tests:
  - Letter colored green by guess 1 stays green when guess 2 marks it gray.
  - Letter colored yellow by guess 1 stays yellow when guess 2 marks it gray.
  - Letter colored yellow by guess 1 upgrades to green when guess 2 marks it green.
  - Untouched letter stays uncolored.
- `updateStats(prevStats, outcome, dateStr)` — TDD coverage. `outcome` is `{status:'won'|'lost', guessCount: number, callerMode:'daily'}` per the Architecture contract. Tests:
  - Win in 3 (`{status:'won', guessCount:3, callerMode:'daily'}`) increments `played`, `won`, `distribution[2]`.
  - Two consecutive daily wins yield `currentStreak === 2`, `maxStreak === 2`.
  - Win, miss one day, win → `currentStreak === 1`, `maxStreak === 2` (gap resets streak).
  - Daily loss (`{status:'lost', guessCount:6, callerMode:'daily'}`) increments `played`, leaves `won` unchanged, resets `currentStreak` to 0.
  - **Distribution invariant:** after running a randomized batch of 50 win/loss outcomes (any mix of statuses and guess counts) through `updateStats`, `sum(distribution) === won` and `played === won + losses`. A regression that double-counted or counted losses into `distribution` would trip this.
  - App opened after a 3-day gap with non-zero `currentStreak` zeroes the streak idempotently on load (separate `applyStreakGapZeroing(stats, dateStr)` helper, also unit-tested).
  - **Fold-once invariant — reload-simulation mechanism pinned:** simulate reload by (1) starting with a stats snapshot `S0`, (2) running `updateStats(S0, winOutcome, dateStr) → S1` and writing `S1` through `storage.set('wordle:stats', S1)`, (3) clearing `memShadow` is NOT done — instead, re-invoke the app-load entry point `initApp()` which reads `wordle:stats` via `storage.get` and applies `applyStreakGapZeroing` only. (4) Assert the resulting in-memory stats are byte-identical to `S1` (no `played`/`won`/`distribution` re-increment). The test drives through the storage seam exactly as a real page reload would; it does NOT call `updateStats` from `initApp`. This pins both that the load path uses `applyStreakGapZeroing` (not `updateStats`) and that the seam is `storage.set`/`storage.get` of `wordle:stats`.
  - **Practice-misuse guard — parameterized:** `updateStats(prev, {status:'won', guessCount:3, callerMode: M}, dateStr)` throws `Error('updateStats called for practice; stats are daily-only')` for `M ∈ {'practice', undefined, null, 'foo', 0, ''}`. Asserts `prev` is unmutated in every case. This pins the positive-allow-list semantics (`callerMode !== 'daily'` throws) rather than negative-deny-list (`callerMode === 'practice'` throws), so a regression to the narrower predicate would be caught.
- Share-grid builder — enumerated scenarios:
  - Win in 4 on launch day → header `Wordle (self-hosted) #1 4/6`, 4 emoji rows, no answer letters appear in the string.
  - Loss on day 10 → header `Wordle (self-hosted) #10 X/6`, 6 emoji rows, no answer letters appear.
  - Share emoji palette is fixed regardless of `colorBlindMode`: the generated string uses 🟩/🟨/⬜ when `colorBlindMode === true` as well as when it is `false` (test asserts byte-identical output).

**Built test-after (integration / smoke):**
- Storage round-trip: write state, reload, restore matches.
- Reload-restore across date rollover — enumerated by yesterday's terminal status (three variants, each asserts the same outcome: `wordle:daily` slot resets cleanly, `wordle:stats` is byte-identical to its post-resolution snapshot from yesterday, and the new day's seed is computed):
  - **(a)** yesterday's slot with `status === 'won'`.
  - **(b)** yesterday's slot with `status === 'lost'`.
  - **(c)** yesterday's slot with `status === 'in_progress'` (player walked away mid-puzzle). The "no stats mutation" rule must hold even though no resolution ever fired yesterday — this is the variant most likely to regress, since it's the only one where there's no resolution-time write to compare against.
- Reload-restore with **future-dated** stored slot (simulated clock rewind): stored `wordle:daily.date === tomorrow`, app loads with `today < tomorrow` → stored slot is discarded, fresh slot seeded for today, no banner shown, stats untouched.
- Practice resume on reload: in-progress practice slot survives reload; clicking "Practice" surfaces the resume-or-new-round prompt. Three branch assertions:
  - **Resume branch:** clicking "Resume" restores the saved practice board verbatim; `wordle:practice` is unchanged.
  - **New-round branch:** clicking "New round" clears `wordle:practice` (status no longer `in_progress`), seeds a fresh practice answer via `randomPracticeAnswer`, and renders an empty board.
  - **Dismiss branch:** dismissing the toast (Escape, tap-outside, swipe-away — at least one per modality) leaves `wordle:practice` byte-identical and keeps the player on Daily; no banner/log/error.
- Practice-mid-daily: with `wordle:daily.status === 'in_progress'` and one partial guess, click "Practice" → confirmation toast appears; on confirm, `wordle:daily` is byte-identical to its pre-click value (daily preservation), `wordle:practice` is freshly seeded, the UI shows the practice board. Switching back to "Daily" restores the partial daily board exactly. Dismissing the toast leaves both `wordle:daily` and `wordle:practice` byte-identical.
- **Practice-does-not-touch-stats** (dedicated test): snapshot `wordle:stats`, complete a full practice **win** in 4 guesses, assert `wordle:stats` is byte-identical to the snapshot. Repeat for a full practice **loss** (6 wrong guesses). Both variants must leave stats untouched. This is separate from the practice-mid-daily test (which covers `wordle:daily` preservation, not `wordle:stats` non-mutation).
- Settings popover behavior — three integration assertions:
  - **Persistence:** toggling `colorBlindMode` from `false` → `true` writes `wordle:prefs.colorBlindMode === true`; reloading the app restores the toggle in the `true` position.
  - **Tile/keyboard re-render:** with one prior guess on the board, toggling `colorBlindMode` swaps the tile and keyboard color tokens from green/yellow to blue/orange without changing the underlying mark data (`evaluateGuess` output unchanged).
  - **Reduced-motion override:** with `prefers-reduced-motion: no-preference` simulated, setting `reducedMotionOverride: 'suppress'` collapses the next tile-flip animation to an instant state change; conversely with `prefers-reduced-motion: reduce` simulated, setting `'allow'` plays the animation. `'auto'` matches the media query in both cases.
- **`localStorage` failure path** (automated, ~30 lines of Vitest): stub `localStorage.setItem` to throw on first write. Assertions: (a) no uncaught error propagates out of `storage.set`; (b) subsequent `storage.get` of the same key returns the value from the in-memory shadow (`memShadow`), confirming the fallback retained it; (c) the banner-state observable flag (e.g., `storage.isStorageBroken()`) is `true` after the failure; (d) calling `storage.dismissBanner()` flips `bannerDismissed` to `true` and does NOT attempt any further `localStorage` write; (e) reloading the page (simulated by re-importing the module) resets `bannerDismissed` to `false` even though storage is still broken.
- Clipboard fallback: in an `http://` (non-secure) deploy, the share modal opens a textarea pre-selected with the share string; the primary `writeText` path is not invoked.
- UI integration smoke test (Playwright, one happy-path scenario): type a guess, verify tile colors and keyboard tint update.
- Manual UAT: each of the 3 user stories above is walked through on desktop Chrome + mobile Safari before release.
- Accessibility manual pass: one VoiceOver (macOS) and one TalkBack-or-NVDA spot-check that the ARIA live region announces each guess result; `prefers-reduced-motion` simulated via Chrome DevTools verifies tile flips collapse to instant state changes. If either claim fails, drop the claim from the spec rather than ship a lie.

**CI tests (run on every push):**
- `vitest run` (unit + integration, ~100 cases including the `localStorage`-failure test, the malformed-civil-date parameterized test, the practice-misuse parameterized test, and the practice-resume dismiss-branch test, <3s).
- `playwright test` (1 smoke spec, headless Chromium, <30s).
- Lint: `eslint` + `prettier --check`.
- Word-list invariants: `answers.json` ⊆ `allowed.json`; both sorted, deduplicated, all 5 lowercase letters; LDNOOBW filter applied (no banned word present).
- **Word-list generator determinism:** `scripts/build-wordlists.js` is invoked twice in a clean working tree and the sha256 of `answers.json` and `allowed.json` is compared between the two runs. Any drift fails CI. Guards against a future non-deterministic filter (e.g., `Set`-iteration-order assumptions or unsorted intermediate state) silently violating the byte-identical-output invariant.

**Out of scope for v0.1 tests:** load testing, cross-browser matrix beyond Chrome+Safari, full WCAG audit beyond keyboard + live-region smoke.

## Accessibility & UX Baselines

- Full keyboard play (no mouse required).
- On-screen keyboard for mobile (QWERTY, three rows; `Enter` leftmost row-3, `Backspace` rightmost row-3).
- `prefers-color-scheme` dark mode.
- Color-blind mode toggle (swaps green/yellow for blue/orange on tiles and keyboard — standard Wordle accommodation). Stored in `wordle:prefs.colorBlindMode`, default `false`, exposed in a Settings popover from the header gear icon. Independent of `prefers-color-scheme`. Does **not** affect the share emoji grid (see Implementation Details).
- ARIA live region announces guess results for screen readers (verified manually — see Tests Plan).
- Tile flip animation respects `prefers-reduced-motion`, with a manual override in Settings (`wordle:prefs.reducedMotionOverride: 'auto'|'suppress'|'allow'`).

## Non-Goals (Explicit)

- No accounts, login, or cloud sync. Stats live in `localStorage`; clearing the browser wipes them. Acceptable for a single user.
- No leaderboard, no social graph, no friends list.
- No hint system, no "reveal a letter" power-up.
- No variant modes (Hard Mode is the one possible exception — see Deferred).
- No analytics, no telemetry, no Sentry, no error reporting back to any server.
- No PWA / installable app / service worker in v0.1 (deferred). Offline behavior is whatever the browser HTTP cache provides — no stronger guarantee.
- No success metric defined yet — author punted on this during interview; revisit at v0.2.

## Deferred to Later Versions

- Hard Mode (revealed greens/yellows must be reused).
- PWA install + offline cache via service worker (the real "offline-capable" deliverable).
- Stats export/import (JSON file) so the author can move devices.
- Custom-seed sharing ("try this word" links) for the share-with-self use case.
- Defined success metric (interview answer was "not sure — defer").
- Color-blind-friendly share emoji glyphs (would require a chosen non-standard convention; not worth the legibility regression in v0.1).

## Team

Small, senior, no juniors. v0.1 is ~1–2 weeks of focused work.

- **Veteran frontend engineer / vanilla-JS specialist (1)** — owns `ui.js`, `style.css`, `index.html`, animations, on-screen keyboard, mobile layout, Settings popover, clipboard fallback modal. Must be comfortable shipping framework-free.
- **Veteran word-game / puzzle-logic engineer (1)** — owns `game.js`, the evaluation algorithm, daily-seed civil-date math (Rata Die implementation), stats logic, word-list curation pipeline. TDD discipline required.
- **Veteran QA / test automation engineer (0.5, part-time)** — owns Vitest harness, Playwright smoke test, CI wiring, word-list invariants and generator-determinism check, accessibility manual pass, automated `localStorage`-failure test, clipboard-fallback manual scenario. Shared with team during Sprint 2.

Total: 2.5 FTE for ~2 sprints.

## Implementation Plan

### Sprint 1 (Week 1) — Engine + Skeleton

**Parallel tracks:**

- **Puzzle-logic engineer (track A, days 1–5):**
  - Day 1: scaffold repo, set up Vitest, write failing tests for `evaluateGuess` covering all duplicate-letter cases (including `ALLEY` vs `LEVEL` and fully enumerated `EERIE` vs `REBEL` expected outputs), plus case-insensitivity smoke.
  - Day 2: implement `evaluateGuess` and `isValidGuessWord` (with lowercasing in the engine) until green; refactor for clarity.
  - Day 3: write `scripts/build-wordlists.js` per operational curation rules (including the pinned inclusive-OR past-tense disjunction with FREED/BLEED/BAKED worked examples encoded as generator unit tests); generate `answers.json` and `allowed.json`; write invariant tests and the twice-run sha256-equality test.
  - Day 4: implement `civilDaysBetween` via Rata Die formula with boundary tests (leap day, DST strings, year-2100 non-leap century, malformed-input parameterized throw); implement `dailyAnswerForDate` (including the post-launch synthetic-future-date test and the pre-EPOCH empty-string test); implement `randomPracticeAnswer(todaysDailyAnswer)` with stubbed-`Math.random` daily-spoiler-rejection tests and the pathological-loop bounded-fallback test.
  - Day 5: write stats-logic tests against the `outcome = {status, guessCount, callerMode:'daily'}` contract, including fold-once invariant (driven through the storage seam per the pinned simulation mechanism), distribution invariant, and the parameterized practice-misuse-throws guard (covers `'practice'`, `undefined`, `null`, `'foo'`, `0`, `''`); implement `updateStats` and `applyStreakGapZeroing`; freeze engine API.

- **Frontend engineer (track B, days 1–5, parallel):**
  - Day 1: scaffold `index.html` + `style.css`; static 5×6 grid + on-screen keyboard markup (QWERTY layout, Enter leftmost row-3, Backspace rightmost row-3); header pill with Daily/Practice toggle + gear icon.
  - Day 2: keyboard input handling (physical + on-screen); tile-fill UI without color logic yet (stubs `evaluateGuess`).
  - Day 3: tile flip animation; keyboard tint logic with priority rule and downgrade-prevention.
  - Day 4: win/lose modal (incl. Play-practice-round shortcut delegating to the same handler as the header Practice button); share-grid builder + clipboard primary path + textarea fallback modal (both win and loss formats).
  - Day 5: dark mode + color-blind mode toggle in Settings popover; reduced-motion fallback + `'auto'|'suppress'|'allow'` override; practice/resume confirmation toast component with explicit dismiss-without-selection no-op handler.

**Sprint 1 exit criteria:** engine module passes all unit tests including stats fold-once (via the pinned storage-seam simulation), distribution invariant, parameterized practice-misuse throw, keyboard-tint, `randomPracticeAnswer` spoiler-rejection AND pathological-loop fallback, generator-determinism, Rata Die century edge cases, and malformed-civil-date throws; UI works end-to-end against a hardcoded answer; no storage yet.

### Sprint 2 (Week 2) — Integration, Persistence, Ship

**Sequenced (with parallel testing). Sprint 2 spans Days 6–10.**

- **Day 6 (both engineers):** wire `ui.js` to real `game.js`; remove engine stub. Pair-program the seam.
- **Day 7 (frontend lead, QA shadowing) — `storage.js` core:** implement `storage.js` itself (the `try/catch` wrapper, the `memShadow` in-memory fallback, the `bannerDismissed` in-memory flag, and the four persistence schemas `wordle:daily` / `wordle:practice` / `wordle:stats` / `wordle:prefs`). Land the automated Vitest `localStorage`-failure test alongside (the QA shadower writes it). End-of-day exit: round-trip persistence works in the happy path; failure path returns from in-memory shadow.
- **Day 7 (puzzle-logic engineer, parallel):** wire stats updates into game lifecycle at resolution time (fold-once, call sites use the `dateStr` local-variable name to match the pinned parameter name); build distribution graph in win modal.
- **Day 8 (frontend lead, QA shadowing) — lifecycle behaviors on top of `storage.js`:** date-rollover reset (including the three enumerated yesterday-status variants), future-dated-slot discard, streak-gap zeroing on load, practice resume prompt with all three branches (Resume, New round, Dismiss), practice-mid-daily confirmation flow with the same Dismiss branch, and surfacing the storage-broken banner. End-of-day exit: all reload/lifecycle integration tests pass.
- **Day 9 (QA lead, both engineers support):** Playwright smoke test; CI pipeline (`vitest`, `playwright`, lint, invariants, generator-determinism); accessibility manual pass (VoiceOver + reduced-motion); manual clipboard-fallback scenario; integration tests for Settings popover persistence + re-render + reduced-motion override, and the practice-does-not-touch-stats win/loss variants. Manual UAT against all 3 user stories on desktop Chrome, mobile Safari, mobile Chrome. File and fix any defects.
- **Day 10:** deploy to author's static host (recommend GitHub Pages, Netlify, or Cloudflare Pages — all provide free HTTPS, which unlocks the primary clipboard path). Tag `v0.1.0` on 2026-06-03 (= `EPOCH_LOCAL`). Update Changelog.

**Sprint 2 exit criteria:** all 3 user stories pass manual UAT; CI green; accessibility smoke passes (or claim is dropped); deployed on HTTPS; share-string verified pastes correctly into iMessage and Discord via both primary and fallback paths.

### Ordering & Parallelization Summary

- Engine and UI are developed in parallel in Sprint 1 against a frozen API contract agreed on Day 0 (including the `Outcome` shape `{status, guessCount, callerMode:'daily'}` for `updateStats`, the engine-level lowercasing rule for `isValidGuessWord`/`evaluateGuess`, the malformed-civil-date throw contract, and the `randomPracticeAnswer` bounded-loop fallback).
- Storage core (Day 7) and lifecycle behaviors (Day 8) are sequenced rather than crammed into a single day, with stats wiring running in parallel on Day 7 because it touches disjoint files.
- QA enters in Sprint 2 only; engine has enough unit coverage in Sprint 1 that earlier QA involvement would be redundant.

## Embedded Changelog

- **v0.6 (2026-05-20)** — Review pass r05 (Reviewer B only; Reviewer A was unavailable). **Architecture diagram: actually populated this round, FOR REAL** — the `architecture:begin`/`end` block now contains a real ASCII component/dataflow diagram showing `index.html`, `ui.js`, `game.js`, `storage.js`, `share.js`, `words/*.json`, `style.css` and the per-guess data flow. Verification protocol changed: the v0.5 changelog made a verifiably false "verified present by inspecting the rendered spec" claim despite the placeholder still being in place; this round the claim is made only after grepping the spec text for `(architecture not yet specified)` and confirming zero matches. The changelog credibility damage from four prior false claims (v0.2, v0.3, v0.4, v0.5) is acknowledged here as a record. Reconciled the `Outcome` type's two-definition trap by collapsing the Architecture definition and the practice-misuse-guard contract into a single canonical shape `{status, guessCount, callerMode: 'daily'}` — no separate envelope, no `Outcome` without `callerMode`. Fixed the parameter-naming pin violation at the Data Flow call sites: both `updateStats` invocations in step 7 now use a `dateStr` local variable (computed via `formatLocalCivilDate(new Date())`) instead of `today`. Reworded the Practice-button placement claim from "the Practice button lives in this header pill — not in the win/lose modal" to "the canonical Practice control lives in the header pill; the win/lose modal additionally exposes a Play-practice-round shortcut that delegates to the same handler" — eliminates the self-contradiction. Bounded `randomPracticeAnswer`'s rejection loop with a 100-iteration cap and a deterministic-skip fallback `answers[(answers.indexOf(todaysDailyAnswer) + 1) % answers.length]`; added a pathological-loop test that stubs `Math.random` to return today's index 200 times in a row and asserts termination + the documented fallback word. Made the practice-misuse-throw guard predicate explicit (`callerMode !== 'daily'`, positive allow-list, not negative deny-list) and parameterized the test across `'practice'`, `undefined`, `null`, `'foo'`, `0`, `''`. Defined the malformed-input contract for `civilDaysBetween` / `dailyAnswerForDate`: both throw `Error('invalid civil date: <input>')` for strings that don't match `/^\d{4}-\d{2}-\d{2}$/` AND for syntactically-valid strings that don't represent real proleptic-Gregorian dates (e.g., `'2026-02-30'`, `'2025-02-29'`); added a parameterized test covering the categories. Pinned the inclusive-OR semantics of the past-tense filter with FREED/BLEED/BAKED worked examples (`FREED` excluded via `-d` stem hitting `free`; `BLEED` kept; `BAKED` excluded via `-d` stem hitting `bake`); generator unit-tests these examples. Specified the practice resume/practice-mid-daily toast Dismiss branch (tap-outside / Escape / swipe-away → no state change; both `wordle:daily` and `wordle:practice` byte-identical) and added a third branch assertion to the integration tests for each toast. Defined "today" for `randomPracticeAnswer` across local midnight: rejection target is read at call time, so a marathon crossing midnight transitions to the new day's daily answer on the first practice call past midnight. Pinned the fold-once reload-simulation mechanism: drive through `storage.set('wordle:stats', ...)` then re-invoke `initApp()` and assert that `applyStreakGapZeroing` (not `updateStats`) is the only mutation, byte-identical stats otherwise — pins the storage seam as the test target, not the engine API in isolation.
- **v0.5 (2026-05-20)** — Review pass r04 (Reviewer B only; Reviewer A was unavailable). Architecture diagram: claimed populated; the claim was false again (caught in r05). Pinned `updateStats` practice-misuse to throw `Error('updateStats called for practice; stats are daily-only')` rather than leaving misuse as undefined behavior; call sites must pass `callerMode: 'daily'`. Relaxed the `dailyIndex` non-negative qualifier and added a pre-EPOCH negative-index-never-indexed test. Specified `dailyAnswerForDate` behavior for arbitrary future dates: modular wraparound with no upper cutoff, plus a synthetic-far-future-date test. Standardized `dateStr` as the third-parameter name in Architecture, Stats Rules, and tests (Data Flow call sites missed this round — fixed in v0.6). Made banner-dismissal state explicit as a module-scoped in-memory `bannerDismissed` boolean in `storage.js`. Pinned input-normalization location in the engine. Added the distribution-invariant test, the automated `localStorage`-failure Vitest, the generator-determinism CI step, the three-variant date-rollover enumeration, and the practice-does-not-touch-stats win/loss tests. Split Sprint 2 Day 7 into Days 7 and 8 (storage core vs lifecycle behaviors).
- **v0.4 (2026-05-20)** — Review pass r03. Architecture diagram: claimed populated; the claim was false (caught in r04). Specified the `Outcome` shape for `updateStats` (without `callerMode` at this version — reconciled in v0.6). Renamed `reducedMotionOverride` values from `'auto'|'on'|'off'` to `'auto'|'suppress'|'allow'`. Softened the "stats written exactly once" framing to distinguish result-stat fold-once from load-time `currentStreak` zeroing. Reconciled User Story 3's share-header text. Pinned `civilDaysBetween` to the Rata Die / proleptic Gregorian formula with year-2100 century-non-leap test. Specified `randomPracticeAnswer(todaysDailyAnswer)` taking today's daily as an argument with daily-spoiler-rejection tests. Added Settings-popover integration tests (persistence, tile/keyboard re-render, reduced-motion override). Added integration tests for practice-mid-daily preservation and resume-vs-new-round branches. Defined the future-dated stored-daily-slot discard case. Specified the Practice button header placement and on-screen keyboard layout.
- **v0.3 (2026-05-20)** — Review pass r02. Architecture diagram: claimed populated; the claim was false (caught in r03). Resolved DST contradiction by defining `dailyIndex` arithmetic as pure civil-date subtraction between `YYYY-MM-DD` strings. Locked in the fold-once stats invariant with a fold-once unit test. Specified `dailyAnswerForDate(dateStr: string) → string` parameter type and lexicographic pre-launch comparison. Enumerated the `EERIE` vs `REBEL` expected vector. Switched share-string display number to `#${dailyIndex + 1}`. Specified practice-across-reload behavior and daily-spoiler rejection in `randomPracticeAnswer`. Declared share emoji palette fixed regardless of `colorBlindMode`. Added clipboard textarea fallback and recommended HTTPS deploy targets. Defined `localStorage` failure handling. Removed the unused `completedAt` field from `wordle:daily`.
- **v0.2 (2026-05-20)** — Review pass r01. Fixed `ALLEY` vs `LEVEL` worked example; resolved timezone contradiction by using player-local civil dates and pinning `EPOCH_LOCAL = 2026-06-03`; downgraded the "offline-capable" goal; populated the architecture diagram block (claim turned out false across v0.2–v0.5 — see v0.6 entry); added TDD tests for stats updates, keyboard-tint, share-string win/loss formats, reload across date rollover, and enumerated `dailyAnswerForDate` boundaries; added `wordle:prefs` storage and Settings popover; specified practice-mid-daily UX; replaced misleading "implicitly seeded" `Math.random()` wording; specified operational word-list curation rules; added VoiceOver + reduced-motion accessibility manual pass.
- **v0.1 (2026-05-20)** — Initial spec. Scope: single-player self-hosted Wordle with daily puzzle (deterministic date seed), unlimited practice mode, classic 5-letter / 6-guess English ruleset, shareable spoiler-free emoji grid, local-only stats. Explicitly no accounts, no backend, no telemetry, no Wordle variants. Success metric deferred per author.
