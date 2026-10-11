Read AGENTS.md. Build the offline EV table generator in sim/. It is a dev tool,
not shipped to the browser. Node only, no dependencies, use worker_threads.

GOAL: for each game config, output data/games/<id>.json with, per true count
bucket t from <= -5 to >= +10:
  freq_t (fraction of rounds), edge_t (player EV per unit at flat bet,
  basic strategy plus the chosen deviation set), var_t (per-round variance
  per unit), and standard error on edge_t.
Also include off-the-top house edge, total rounds simulated, and the config.

RULES ENGINE: decks, penetration, H17/S17, DAS, surrender (none/late),
double any 2 / 9-11 / 10-11, BJ 3:2 / 6:5, max split hands, resplit aces,
hit split aces, US hole card. Insurance taken per index when deviations on.

STRATEGY: generate basic strategy per rule set via exact or composition-
dependent-off-the-top computation, OR embed published tables for supported
rule sets. Document which. Deviation sets: none, Illustrious 18 + Fab 4.

COUNTING: Hi-Lo, TC = RC / decks remaining, deck estimation to nearest half
deck, TC floored (also support rounding as an option). Bucket by TC at the
start of each round, before cards are dealt.

CONFIGS TO GENERATE: 6D H17 DAS LS 3:2, 6D S17 DAS LS 3:2, 8D H17 DAS 3:2,
2D H17 DAS 3:2, 6D H17 DAS 6:5. Penetration 75% and 83% for each.
Target at least 200M rounds per config, or whatever makes SE on edge
below 0.05% for TC buckets -1..+5.

VALIDATION (required, put results in sim/VALIDATION.md):
- Off-the-top house edge for each config vs published values (cite source,
  e.g. Wizard of Odds house edge calculator). Within 0.05%.
- Edge per +1 TC slope roughly 0.5% for 6D H17.
- freq_t sums to 1. Unit tests for the rules engine on hand-constructed decks
  (splits, doubles, dealer soft 17, blackjack vs blackjack, surrender).

CLI: node sim/run.js --config 6d-h17-das-ls-32 --pen 0.75 --rounds 200e6
Print progress and an ETA. Seeded RNG for reproducibility (xoshiro256**).

At the end, print the acceptance checklist with each item ticked or explained.
