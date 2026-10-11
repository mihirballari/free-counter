# free-counter

Free, open-source Hi-Lo card counting trainer. Static site on GitHub Pages. MIT license.

## Hard constraints
- Vanilla HTML/CSS/JS, ES modules, zero runtime dependencies, no build step.
- Total shipped app (excluding data/*.json and sim/) under 60 KB uncompressed.
- Pure logic in core.js (no DOM). UI in app.js. Tests in tests/*.test.js via `node --test`.
- Randomness via crypto.getRandomValues, Fisher-Yates shuffle.
- Persistence: localStorage only, single versioned key `fc:v1`, with JSON export/import.
- No accounts, no analytics, no network calls at runtime.
- Cards drawn with CSS + Unicode suits. No image assets, no third-party branding.
- Keyboard-first: every action reachable without a mouse.
- Mobile works at 375px width.
- Specs live in docs/prompts/. Follow the one for your branch exactly.

## Done means
- `node --test` passes.
- You opened index.html via a local static server and exercised the feature.
- README updated if behavior changed.
- Work committed on your branch with clear messages.
