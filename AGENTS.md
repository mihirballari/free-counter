# AGENTS.md: free-counter

Rules for every AI agent (Claude Code, Codex, anything else) working in this repo.
`CLAUDE.md` is a symlink to this file. If a spec in `docs/prompts/` conflicts with
this file, this file wins. Log the conflict in `docs/DECISIONS.md`.

---

## 1. What this project is

free-counter is a free, open-source, Hi-Lo card counting trainer that runs as a
static site on GitHub Pages. MIT license.

**Goals**
- Fast, minimal, keyboard-first running count drill.
- Local history and analytics so a user can see progress over time.
- An EV/hour calculator backed by precomputed simulation tables.

**Non-goals (do not build)**
- Accounts, login, sync, servers, databases.
- Basic strategy drills, deviation drills, full blackjack game, betting game.
- Forums, messaging, videos, leaderboards, ranks, achievements, themes beyond light/dark.
- Any real-money feature.
- Anything that copies another product's name, logo, art, copy or code.

---

## 2. Repo layout

```
/
├── index.html          app shell (trainer branch)
├── style.css           all styles (trainer branch)
├── app.js              DOM, rendering, events (trainer branch)
├── core.js             pure logic, no DOM (trainer branch)
├── ev.js               EV math, pure (added later, EV phase)
├── tests/              node --test files
├── sim/                offline simulator, Node only (sim branch)
│   ├── run.js
│   ├── engine/         rules, shoe, strategy, counting
│   ├── tests/
│   └── VALIDATION.md
├── data/games/*.json   sim output consumed by the app (sim branch)
├── docs/
│   ├── prompts/        specs per branch, read-only for agents
│   ├── DECISIONS.md    append-only log of judgment calls
│   └── QUESTIONS.md    open questions for the human
├── AGENTS.md
├── CLAUDE.md -> AGENTS.md
├── README.md
└── LICENSE
```

---

## 3. Branch and file ownership

Work happens in git worktrees, one branch per agent. Never touch files you do not own.

| Branch    | Owns                                                         | Must not touch          |
|-----------|--------------------------------------------------------------|-------------------------|
| `trainer` | index.html, style.css, app.js, core.js, tests/, README.md    | sim/, data/             |
| `sim`     | sim/, data/games/                                            | everything at repo root |
| `ev`      | ev.js, tests/ev.test.js, the EV section of app.js/index.html | sim/, data/             |
| `main`    | human only. Agents never commit to main.                     |                         |

Shared files (AGENTS.md, docs/prompts/*, LICENSE) are edited by the human on main only.
You may append to docs/DECISIONS.md and docs/QUESTIONS.md on your own branch.

---

## 4. Hard constraints

- Vanilla HTML, CSS, JS. ES modules (`<script type="module">`). No frameworks.
- Zero runtime dependencies. No build step. No bundler. No TypeScript compile.
- Dev-only tooling is allowed in sim/ (still zero npm deps preferred).
- Shipped app (index.html + style.css + all .js at root) under 60 KB uncompressed.
  data/*.json is excluded but each game file must stay under 50 KB.
- No network calls at runtime except fetching `data/games/*.json` from the same origin.
- No analytics, trackers, fonts from CDNs, or external images.
- Cards drawn with CSS and Unicode suits only.
- Randomness: `crypto.getRandomValues` in the app. Seeded PRNG (xoshiro256**) in sim/.
- Must work offline after first load (no service worker required, just no external fetches).
- Must work in current Chrome, Firefox, Safari, and at 375px width.

---

## 5. Architecture

### 5.1 core.js (pure)
- No `document`, `window`, `localStorage`, timers, or `Date.now()` inside logic.
  Time and randomness are passed in as arguments so tests are deterministic.
- Every function that shuffles or draws takes an `rng` argument: `() => number in [0,1)`.
- Exports plain functions and plain objects. No classes required. No global state.
- Suggested exports (names may vary, keep them obvious):
  `createShoe(decks)`, `shuffle(cards, rng)`, `cutIndex(decks, pen)`,
  `hiLo(card)`, `dealRound(state, rng)`, `decksRemaining(cardsLeft, step)`,
  `trueCount(rc, decksRem, precision)`, `isCheckDue(state, config)`,
  `gradeCheck(expected, answer)`, `summarizeSession(session)`,
  `updateStreaks(streaks, correct)`, `mergeImport(existing, incoming)`.

### 5.2 app.js (UI)
- Owns DOM, events, timers, storage, audio.
- One state object, one `render()` per screen. No virtual DOM, no templating library.
- Screens: `setup`, `session`, `report`, `analytics`. Switch via hash routing
  (`#/setup`, `#/session`, `#/report/<id>`, `#/analytics`) so the back button works.
- All storage access goes through one module-level `store` with `load()`, `save()`,
  wrapped in try/catch. The app must still work (without history) if storage throws.

### 5.3 Storage schema (key `fc:v1`)

```js
{
  version: 1,
  settings: {
    decks: 6, pen: 0.75, checkEvery: 5,     // 1,2,3,4,5,10, or 0 = end of shoe
    tcCheck: "off",                          // "off" | "whole" | "half"
    speed: "medium", mode: "manual",         // speed: slow|medium|fast, mode: manual|auto
    sound: true, handTotals: false
  },
  streaks: { current: 0, best: 0 },          // span sessions
  sessions: [Session]                        // newest last, cap 1000, drop oldest
}

Session = {
  id, startedAt, endedAt,                    // ISO strings
  config,                                    // copy of settings at start
  hands, cardsSeen,
  checks: [Check],
  correct, accuracy, avgMs
}

Check = {
  t,                                         // ms since session start
  expectedRC, answerRC,
  expectedTC, answerTC,                      // null when tcCheck is off
  msToAnswer, cardsSinceLastCheck, decksRemaining
}
```

Schema changes: bump `version`, write a `migrate(old)` in core.js, add a test. Never
silently drop user data. Export/import uses the same shape. Import merges by session id.

---

## 6. Domain rules (single source of truth)

- **Hi-Lo tags**: 2,3,4,5,6 = +1. 7,8,9 = 0. 10,J,Q,K,A = -1.
- **Running count** starts at 0 for each new shoe. A full shoe sums to 0.
- **Cut card**: placed at `round(decks * 52 * pen)`. When reached mid-round, finish the
  round, then the shoe ends. If check frequency is "end of shoe", that is the only check.
- **Decks remaining**: `cardsLeft / 52`, estimated to the nearest 0.5, minimum 0.5.
- **True count**: `RC / decksRemaining`.
  - Drill grading, whole precision: correct if the answer equals the truncated
    value or the rounded value (round half away from zero).
  - Drill grading, half precision: same rule at 0.5 steps.
  - Sim bucketing: floor by default, rounding as an option. Bucket by TC at the start
    of each round, before any card is dealt. Clamp buckets to [-5, +10].
- **Round dealing** (trainer only, for realistic card flow, not strategy):
  player 2 cards, dealer 2 cards, hole card revealed at the end.
  Player hits hard < 17 vs dealer 7-A, hard < 12 vs 2-6, soft < 18. Stands otherwise.
  Dealer hits soft 17. If the player busts, the dealer reveals and stops.
- **Speeds**: card reveal interval slow 600 ms, medium 350 ms, fast 150 ms.
  Auto mode waits 2x the interval between rounds.

---

## 7. Sim output schema (contract between sim and EV tab)

Do not change field names without a version bump. All edges and variances are per unit
bet, as fractions (0.0044 means +0.44%).

```json
{
  "schema": 1,
  "id": "6d-h17-das-ls-32-p75",
  "label": "6 decks, H17, DAS, LS, 3:2, 75% pen",
  "rules": { "decks": 6, "pen": 0.75, "h17": true, "das": true,
             "surrender": "late", "double": "any2", "bj": 1.5,
             "maxHands": 4, "rsa": false, "hsa": false },
  "counting": { "system": "hilo", "tc": "floor", "deckStep": 0.5 },
  "deviations": "i18f4",
  "rounds": 200000000,
  "seed": "0x...",
  "houseEdgeOTT": -0.0055,
  "buckets": [
    { "tc": -5, "freq": 0.0, "edge": 0.0, "var": 0.0, "se": 0.0 }
  ],
  "simVersion": "x.y.z",
  "generatedAt": "ISO"
}
```

`buckets` covers every integer TC from -5 to +10, `freq` sums to 1 within 1e-9.

---

## 8. Code style

- Modern JS (ES2022). `const` by default. Strict equality. No `var`.
- Small functions, early returns, no clever one-liners that hide logic.
- JSDoc types on every exported function in core.js and ev.js.
- Names describe domain: `runningCount`, `decksRemaining`, not `rc2`, `tmp`.
- No commented-out code. No TODOs without a matching line in docs/QUESTIONS.md.
- CSS: custom properties on `:root` for every color and spacing value. Dark mode via
  `prefers-color-scheme` plus a manual toggle. No `!important`.
- Respect `prefers-reduced-motion`: disable card animations, keep timing.

---

## 9. UI and accessibility rules

- **Keyboard map** (must all work, shown in a `?` help overlay):
  `Enter` deal / submit / continue, `Esc` end session, digits and `-` type the count,
  `S` toggle sound, `T` toggle hand totals, `?` help.
- Count input is autofocused on every check and accepts negative numbers.
- Feedback is announced via an `aria-live="polite"` region.
- Cards have accessible labels (`aria-label="Queen of hearts"`).
- No layout shift when cards are dealt. The table has fixed-height rows.
- Contrast at least WCAG AA. Hit targets at least 44px on mobile.
- The setup screen loads last-used settings. One press of Enter starts a session.

---

## 10. Testing

Run `node --test` before every commit. All tests must pass.

Required coverage in tests/:
- Shoe has `decks * 52` cards with correct rank counts.
- Shuffle uniformity: chi-square on first-card rank over 100k shuffles, loose bound.
- Hi-Lo tags for every rank. Full-shoe running count sums to 0.
- Cut card position and end-of-shoe behavior.
- Decks remaining estimate at boundaries (e.g. 26, 27, 78 cards left).
- TC grading for both precisions, including negatives and both accepted answers.
- Check scheduling for every frequency including end of shoe.
- Streaks across sessions (correct, wrong, resets, best).
- Import merge: dedupe by id, preserve newer, reject malformed input without crashing.
- Schema migration (once version 2 exists).

Tests use a seeded rng passed into core functions. No `Math.random` in tests.

---

## 11. Browser verification (Playwright MCP)

Unit tests are not enough. Before saying a UI task is done:

1. Serve the repo: `npx serve .` or `python3 -m http.server 8000`.
2. With Playwright MCP, run this walkthrough at 1280x800 and again at 375x812:
   - Load setup, change decks and check frequency, start a session with Enter.
   - Deal until a count check appears. Submit the correct running count, confirm
     "Correct". Deal to the next check, submit a wrong value, confirm "Incorrect"
     with the actual RC and signed error shown.
   - If TC check is on, confirm the TC prompt appears after the RC.
   - Press Esc, confirm the report shows the right totals.
   - Open analytics, switch 7D / 30D / All, confirm charts and recent list render.
   - Export JSON, clear data, import the file, confirm history is restored.
   - Reload the page, confirm settings and history persisted.
3. Check the browser console. Zero errors, zero warnings from our code.
4. Take screenshots of setup, session, report, analytics at both sizes and save them
   to `docs/screenshots/` (gitignored unless the human asks to commit them).

Getting the correct RC in the walkthrough: read it from a debug hook exposed only when
the URL has `?debug=1` (e.g. `window.__fc.state.runningCount`). The hook must not exist
without that flag.

---

## 12. Sim rules (sim branch only)

- Node 20+, no npm dependencies. Parallelize with worker_threads, one worker per core.
- Seeded xoshiro256** per worker, seeds derived from a master seed. Same seed and
  config must produce identical output.
- Print progress and ETA. Support `--rounds`, `--config`, `--pen`, `--seed`, `--out`.
- Develop and validate with 20M-round runs. Full runs (200M+) are started by the human.
- Validation is mandatory and lives in sim/VALIDATION.md:
  - Off-the-top house edge per config vs a published reference (cite it), within 0.05%.
  - Edge slope for 6D H17 roughly +0.5% per +1 TC between TC 0 and +5.
  - `freq` sums to 1. SE reported per bucket.
  - Rules engine unit tests on hand-built decks: splits, resplits, doubles, DAS,
    surrender, dealer soft 17, blackjack vs blackjack, split aces.
- Output must match section 7 exactly.

---

## 13. Git workflow

- Never commit to main. Never force push. Never rewrite history on a shared branch.
- Small commits with prefixes: `feat:`, `fix:`, `test:`, `refactor:`, `docs:`, `chore:`.
- Each commit leaves tests passing.
- Do not commit: node_modules, screenshots, sim scratch output, .DS_Store, editor files.
- Keep a `.gitignore` covering the above.

---

## 14. Review mode

When asked to review another branch:
- Do not modify code. Output a numbered list: file:line, what is wrong, why, and a
  repro or reasoning. Separate "definite bugs" from "possible issues" and "spec gaps".
- Check against this file and the branch's spec in docs/prompts/.
- For the sim, independently recompute at least one headline number with your own
  minimal script and report the difference.

When asked to fix review findings:
- Apply the ones you agree with. Push back with reasoning on the ones you do not.
- Rerun tests and the Playwright walkthrough. Commit.

---

## 15. When unsure

- If the human is present, ask one specific question.
- If not, pick the most conservative reasonable option, log it in docs/DECISIONS.md
  (date, decision, alternatives, why), and keep going.
- Never invent features to fill gaps. Never add dependencies to save time.

---

## 16. Definition of done

A task is done only when all of these are true:
- [ ] Matches the spec in docs/prompts/ for your branch and this file.
- [ ] `node --test` passes.
- [ ] Playwright walkthrough (section 11) passes at both sizes with a clean console.
- [ ] Size budget (section 4) holds. Report the actual byte count.
- [ ] No files outside your ownership (section 3) changed.
- [ ] README updated if user-facing behavior changed.
- [ ] Committed on your branch with clear messages.
- [ ] Acceptance checklist printed, each item ticked or explained.
