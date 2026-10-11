Read AGENTS.md. Build v1 of the trainer per this spec. Work in this worktree only.

FILES: index.html, style.css, app.js, core.js, tests/core.test.js, README.md

SETUP SCREEN
- Decks: 1, 2, 6, 8 (default 6). Penetration: 50 to 90% slider (default 75).
- Check frequency: every 1, 2, 3, 4, 5, 10 hands, or end of shoe only.
- Also ask true count: off / whole / half precision. TC = RC / decks remaining,
  where decks remaining = cards remaining / 52, estimated to nearest half deck.
- Speed: slow / medium / fast (card reveal interval 600 / 350 / 150 ms).
- Mode: manual (Enter deals next hand) or auto (next hand after speed-based delay).
- Settings persist between visits.

DEALING (core.js)
- Build the shoe, shuffle, place cut card at penetration.
- Each round: player 2 cards, dealer 2 cards (hole card revealed at end of round).
- Player draws using a simple rule (hit hard < 17 vs dealer 7 to A, hit < 12
  vs 2 to 6, hit soft < 18). Dealer hits soft 17. Busted player: dealer stops.
  Purpose is a realistic number of cards per round, not correct strategy.
- Hi-Lo: 2 to 6 = +1, 7 to 9 = 0, 10 to A = -1. Running count across the shoe.
- At the cut card: finish the round, prompt the final check, then new shoe with RC reset.

SESSION SCREEN
- Dealer row on top, player row below, cards animate in at the selected speed.
- Left rail: current streak, session accuracy (x/y checks).
- On a check: numeric input autofocused (accepts minus sign), Enter submits.
  If the TC check is on, ask it second, after the RC.
- Feedback: Correct / Incorrect, your answer vs actual, signed error.
- Esc or X ends the session and opens the report.
- Gear menu: sound on/off (WebAudio tick, no files), hand totals on/off.

RECORDING (per check)
{ t, expectedRC, answerRC, expectedTC?, answerTC?, msToAnswer, cardsSinceLastCheck, decksRemaining }

SESSION RECORD
{ id, startedAt, endedAt, config, hands, cardsSeen, checks: [...], correct, accuracy, avgMs }

REPORT SCREEN
Hands, cards seen, checks, correct, accuracy, avg response time, duration,
error histogram (signed error -3..+3, clamped). Buttons: Train again, Analytics.

ANALYTICS SCREEN
- Range toggles: 7D / 30D / 90D / All.
- KPIs: accuracy, total checks, sessions, current streak, best streak
  (streaks span sessions), avg response ms, cards/min.
- Hand-written inline SVG line chart: per-session accuracy + cumulative accuracy.
- Second small chart: avg response ms per session.
- Error histogram across the range.
- Recent sessions list (date, config summary, accuracy, checks, duration),
  click to open that session's report.
- Export JSON / Import JSON (merge by id) / Clear data (with confirm).

TESTS (node --test) must cover: shoe composition and size, shuffle uniformity
(chi-square on first card over 100k shuffles, loose bound), Hi-Lo values,
RC across a full shoe sums to 0, cut card behavior, decks-remaining
estimate, TC rounding for both precisions, streak logic across sessions,
import merge dedupe.

ACCEPTANCE CHECKLIST: print it at the end with each item ticked or explained.
Do not add features not listed. Do not add dependencies.
