# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- `npm test` — runs the full gate, same as CI (Node 22 and 26): `xo` lint, then `c8 node --test 'test/[!_]*.js'`, then `tsc --noEmit` over `source/index.d.ts`. All three must pass.
- Single test file: `node --test test/chalk.js`
- Single test by name: `node --test --test-name-pattern='style string' test/chalk.js`
- Lint only / autofix: `npx xo` / `npx xo --fix`
- Benchmarks: `npm run bench` (uses `node:bench`; `benchmark.js` forces `chalk.level = 3` because the bench runner pipes stdout)

Files in `test/` prefixed with `_` are fixtures run as child processes (via `execa`), not test files, which is why the glob excludes them.

## Architecture

Pure ESM, zero runtime dependencies, Node >= 22. The published package is just `source/`.

- `source/index.js` — the entire runtime. `source/index.d.ts` is the hand-written public types, and is type-checked as part of `npm test`.
- `source/vendor/` — vendored copies of `ansi-styles` and `supports-color`, resolved through the `package.json` `imports` map (`#ansi-styles`, `#supports-color`; the latter maps to `browser.js` outside Node). Excluded from xo and coverage. Do not import `ansi-styles` from npm in `source/`; the devDependency exists for tests only.
- `source/utilities.js` — string helpers on the hot path (`stringReplaceAll`, `stringEncaseCRLFWithFirstIndex`).

### How chaining works (`source/index.js`)

- `chalk` / `new Chalk()` / `chalkStderr` are *generators*: functions whose prototype is `createChalk.prototype`. The `Chalk` constructor returns a function from its constructor.
- Every style name (from `ansiStyles`, plus `visible`, plus the `rgb`/`hex`/`ansi256` model functions and their `bg*`/`underline*` variants) is a lazy getter. On first access it builds a *builder* and caches it as an own property with `Object.defineProperty`, so repeated `chalk.red.bold` access is cheap.
- A builder is a function whose prototype is `proto`. It holds three symbol-keyed fields: `GENERATOR` (always the root generator, not its parent, so reading level is one load), `STYLER` (a linked list of `{open, close, openAll, closeAll, parent}` with precomputed concatenated open/close codes), and `IS_EMPTY` (set by `visible`).
- `level` is stored under the `LEVEL` symbol on the generator; the public `level` accessor validates (integer 0–3, throws otherwise). Builders proxy `level` to their generator. `applyOptions` writes `LEVEL` directly because the prototype carrying the accessor is not set yet at that point.
- Color model converters are resolved once per model into an array indexed by level, so `converters[this.level]` does downsampling (truecolor → 256 → 16) at call time without branching.
- `applyStyle` is the hot path: returns input unchanged at level 0 (or `''` for `visible`), re-opens nested styles by replacing inner close codes with open codes when the string already contains `\x1B`, and closes/reopens styles around newlines (handles `\r\n`) to prevent color bleed.

### Conventions

Performance is a core concern: the code deliberately avoids rest arrays, getter dispatch, and allocations on hot paths, and comments explain why. Preserve those micro-optimizations when editing, and check `npm run bench` for changes to `applyStyle`, `createBuilder`, or the style getters. The project follows xo style (tabs, see `.editorconfig`); eslint-disable comments in source carry a justification after `--`.

When behavior changes, update `readme.md` (it is the API documentation) and `source/index.d.ts` alongside the code.
