# Repository Guidelines

## Archived Repository Ownership

This repository is an archived historical snapshot of AdScrubber's Ghostery `adblocker` fork.

- Do not modify this repository; authoritative core ownership moved to `project-product-adblocker-ghostery`.
- Product commit `520e0897c2fd78bb5aa01e3b7474aa3a2772404a` established the active core owner at `/root/projects/project-product-adblocker-ghostery/packages/adblocker`.
- This archive preserves the complete 13-package source snapshot for provenance and research.
- Only `packages/adblocker` moved into active product ownership; no other package migration is implied.
- The imported source snapshot is `e7e8a18d65cad2333bd8041fc48592ca4ff289b4`.
- Ghostery upstream provenance remains pinned at `ff8ed92b3648bdf49ba884b7d90c46d3dda4c366`.
- Preserve all remotes, branches, tags, source, tests, build configuration, and generated distribution output.
- Preserve the MPL-2.0 license and upstream attribution.
- Do not prepare, propose, publish, or track upstream contributions, pull requests, issues, or patches.
- Use Ghostery upstream only as a read-only provenance, comparison, and intentional-sync source.
- CI and GitHub Actions are prohibited; builds, tests, provenance checks, and releases are run manually.
- Do not add or enable `.github/workflows` or any other hosted CI configuration.
- Record product-level findings in `project-product-adblocker-ghostery` instead of adding research ledgers here.

## Project Overview

The Ghostery adblocker is a JavaScript/TypeScript library for blocking ads, trackers, and annoyances in the browser.
It is compatible with uBlock Origin and EasyList filter syntax.
It powers Ghostery and Cliqz, and Brave uses an adapted form of its algorithm.
The historical snapshot is a Yarn 4 workspace monorepo with 13 packages and a `bench/` harness.
Lerna v10 orchestrates cross-package builds and linting.
The inherited `auto` tooling is not our product release owner.
License is MPL-2.0; the core package is published as `@ghostery/adblocker`.

## Codebase Overview & Orientation

Start by reading `packages/adblocker/src/index.ts` (public API) and `engine/engine.ts` (core engine).
Then move to `filters/` and `engine/reverse-index.ts` to see how filters are stored and matched.
Three vertical layers: the core engine, the browser/selector helpers, and the host integrations.
Core engine is `packages/adblocker`; helpers are `adblocker-content` and `adblocker-extended-selectors`.
Host integrations are `*-webextension`, `*-puppeteer`, `*-playwright`, and `*-electron`.
Network blocking path: `Request` → `FilterEngine.match` → bucket → `ReverseIndex` → `NetworkFilter.match`.
Cosmetic path: `getCosmeticsFilters` → `matchCosmeticFilters` → `CosmeticFilterBucket` → stylesheets.
Serialization: every type implements `serialize`/`deserialize` over `StaticDataView`.
The engine flattens all buckets into one compact buffer.
To adapt a filter feature, edit the parser (`lists.ts`, `filters/*.ts`) and the bucket.
Also update `engine/engine.ts` and extend the matching tests.
Regenerate compression codebooks via `yarn generate-codebooks`.
Most extension-level work lands in `adblocker-webextension`; the other hosts are thin analogues.

## Architecture & Data Flow

`packages/adblocker` is the self-contained core engine.
Every other `packages/adblocker-*` package is a thin host integration.
Host packages re-export the core via `export * from '@ghostery/adblocker'`.
They subclass `FiltersEngine` into a `*Blocker` with a `BlockingContext`.
The two filter primitives are `NetworkFilter` and `CosmeticFilter`.
`NetworkFilter` handles URL blocking, redirects, and CSP; `CosmeticFilter` handles CSS/scriptlet hiding.
Build-from-text pipeline: `FiltersEngine.parse` delegates to `parseFilters` in `lists.ts`.
It splits lines, detects type, and parses each into a filter.
Preprocessor directives (`!#if`/`!#else`/`!#endif`) become `Preprocessor` objects.
They toggle whole filter blocks against an `Env`.
Each bucket calls `ReverseIndex.update` to tokenize filters.
It picks the least-frequent (most selective) token, serializes once, and builds lookup indexes.
At runtime `Request.getTokens()` hashes the URL and hostname.
Then `ReverseIndex.iterMatchingFilters` deserializes candidates to run `filter.match`.
`FilterEngine.match` evaluates in priority order: `$important`, `$redirect`, normal filters, exceptions, `$removeparam`.
The serialized engine is one compact `StaticDataView` buffer backed by a `Uint8Array`.
It is optionally compressed and CRC32 checksummed.
`FilterEngine` emits typed events through an async `EventEmitter`.
Examples include `filter-matched`, `request-blocked`, and `request-redirected`.

## Key Directories

- `packages/adblocker` — core engine, filter parsing, reverse index, serialization, compression.
- `packages/adblocker-content` — DOM feature extraction and scriptlet injection shared by content scripts.
- `packages/adblocker-extended-selectors` — extended-CSS parser and evaluator.
- `packages/adblocker-webextension` — MV2/MV3 background integration (`WebExtensionBlocker`).
- `packages/adblocker-webextension-cosmetics` — content-script cosmetics injector.
- `packages/adblocker-puppeteer` — Puppeteer integration (`PuppeteerBlocker`).
- `packages/adblocker-playwright` — Playwright integration (`PlaywrightBlocker`).
- `packages/adblocker-electron` and `packages/adblocker-electron-preload` — Electron main/renderer integration.
- `packages/*-example` — private demos for webextension, Puppeteer, Playwright, and Electron.
- `bench/` — micro-benchmarks and a cross-blocker comparison harness.
- `packages/adblocker/assets` — fetched real uBlock Origin, EasyList, Fanboy lists used as fixtures.
- `packages/adblocker/tools` — compression-codebook and engine-version generation scripts.

## Development Commands

Run everything from the repo root with `yarn` (Yarn 4; `corepack enable` first).
- `yarn install --immutable` — install with locked `yarn.lock`.
- `yarn build` — `lerna run build` (tshy dual build + rollup, topologically ordered).
- `yarn test` — `yarn workspaces foreach -A run test` across all packages.
- `yarn lint` — `lerna run --parallel lint` (per-package `eslint src [test|tools]`).
- `yarn format-check` / `yarn format-fix` — Prettier over `./packages/**/*.ts`.
- `yarn workspace @ghostery/adblocker test` — test a single package.
- `yarn workspace @ghostery/adblocker run test:fuzz` — property tests controlled by `FUZZ_RUNS`.
- `yarn workspace @ghostery/adblocker-extended-selectors run test:browser` — Playwright browser tests.
- Requires `yarn build` first.
- `make` in `bench/` — run micro-benchmarks with the repository runner.
- `yarn generate-codebooks` (in `packages/adblocker`) — regenerate SmaZ compression codebooks.
- This is maintenance and not part of the normal `build`.

## Code Conventions & Common Patterns

Use strict TypeScript and ES modules with `.js` extensions on relative imports.
Prettier uses print width 99, single quotes, trailing commas, semicolons, and arrow-function parentheses.
Every serializable type implements `serialize`, `getSerializedSize`, and static `deserialize` over `StaticDataView`.
Use lazy/memoized getters for expensive derived state.
Examples include `Request.tokens`, `NetworkFilter.id`, and bucket caches.
Dependency injection is by delegate.
An injected `fetch` powers `fromLists`/`fromPrebuilt*`.
A `Caching {path, read, write}` powers on-disk caching.
Fail fast with thrown errors on `ENGINE_VERSION` or CRC32 `checksum` mismatches.
Also enforce `ReverseIndex.merge` precondition violations.
Parse failures are non-throwing; `parse()` returns `null` for bad lines.
Unsupported lines are reported in `notSupportedFilters`, not raised.
Avoid allocation in hot paths by reusing typed arrays (`StaticDataView`, `TokensBuffer`, `compact-set.ts`).
Async event dispatch goes through `queue-microtask` to avoid re-entrancy.
Do not add typechecking to the build beyond what tshy already runs; there is no separate typecheck script.

## Important Files

- `packages/adblocker/src/index.ts` — public API barrel for the core package.
- `packages/adblocker/src/engine/engine.ts` — core parse, update, match, serialization, and deserialization.
- Also defines `ENGINE_VERSION`.
- `packages/adblocker/src/request.ts` — `Request`, request-type unions, hostname hashing.
- `packages/adblocker/src/filters/network.ts` and `filters/cosmetic.ts` — the two filter primitives.
- Each implements parse, match, and getTokens.
- `packages/adblocker/src/engine/reverse-index.ts` — token lookup and candidate iteration.
- `packages/adblocker/src/lists.ts` — raw-list parsing pipeline and diff types.
- `packages/adblocker/src/config.ts` — `Config` load/enable flags.
- `packages/adblocker/src/data-view.ts` — zero-copy serialization cursor.
- `packages/adblocker/src/compression.ts` and `src/codebooks/*` — SmaZ compression and generated codebooks.
- Root tooling lives in `package.json`, `lerna.json`, TypeScript configs, and `eslint.config.js`.

## Runtime/Tooling Preferences

- Runtime: Node.js 24.15.0 (pinned by `asdf` in `.tool-versions`).
- Package manager: Yarn 4.18.0 with `node-modules` linking and the global cache disabled.
- Orchestration: Yarn workspaces + Lerna v10; no nx config is present in the repo.
- Resolutions: `@remusao/counter` is overridden by a patch under `.yarn/patches/`.
- Toolchain: TypeScript 6, `tsx`, `tshy`, and Rollup with terser.
- ESLint 10 flat config in `eslint.config.js`.
- It uses `typescript-eslint` `recommendedTypeChecked` with many strict rules relaxed.
- Code style is enforced by Prettier and `.editorconfig` (2-space indent, LF, UTF-8).

## Testing & QA

Unit tests use Mocha 11 + Chai (`expect`) through the `tsx` loader.
`nyc` wraps mocha for coverage; there is no Jest or Vitest.
The root `.mocharc.json` sets timeout 10000, `extension: ["ts"]`, and `recursive`.
Mocha finds it by walk-up, so per-package configs are unnecessary.
Tests live in `packages/<pkg>/test/*.test.ts` under `test/engine/` and `test/filters/`.
Structure them with `describe/context/it` and Chai `expect`.
Property-based invariants use `fast-check` (`test/fuzz.test.ts`), gated by `FUZZ_RUNS`.
Golden-style tests assert exact parse output against large inline fixtures like `parsing.test.ts`.
They also use real request data from `test/data/requests.ts`.
Browser E2E is limited to `adblocker-extended-selectors` via Playwright.
It splits `test/unit` (mocha) and `test/e2e` (browsers) and requires `yarn build` first.
Real filter-list fixtures live in `packages/adblocker/assets/` and drive parsing tests and codebook generation.
`nyc` runs with defaults and no coverage thresholds.
`clean` wipes `coverage`, `.nyc_output`, `.tshy*`, and `.rollup.cache`.
Inherited CI runs build, lint, browser installation, and tests on supported Node versions.
The inherited repository has no commit hooks or commit-message enforcement.
Our fork releases follow the owning product's explicit release process.
