# Implementation Plan — setup-flutter-action

## Milestone 1 — Scaffold ✓

Initialized the repo with `package.json`, `tsconfig.json`, `action.yml`, and a
minimal `src/index.ts`. Set up dependencies (`@actions/core`, `@actions/cache`,
`@actions/tool-cache`, `@actions/http-client`, `@vercel/ncc`, TypeScript, Jest,
ESLint) and a CI workflow that builds and tests on PRs.

## Milestone 2 — Release Resolution ✓

Implemented manifest-fetching and version-resolution logic with unit tests
covering channel resolution, exact version lookup, and version-not-found errors.

## Milestone 3 — Download & Extract ✓

Added download and extraction of the SDK archive on cache miss, PATH setup, and
outputs. Added a smoke test workflow verifying `flutter --version` on an
`ubuntu-latest` runner.

## Milestone 4 — Caching ✓

Wrapped the download/extract step with `@actions/cache` restore and save.
Verified cache hits in the smoke test.

## Milestone 5 — Polish & Release ✓

Added input validation with clear error messages, wrote `README.md` with usage
examples, and published to the GitHub Marketplace as v1.

## Milestone 6 — Pub Cache ✓

Set `PUB_CACHE` to a fixed location (`~/.pub-cache`) and exported
`~/.pub-cache/bin` to `PATH`, making globally-activated Dart tools available
without extra configuration.

## Milestone 7 — macOS & Windows ✓

Added platform detection and archive format selection (`.zip` on macOS/Windows,
`.tar.xz` on Linux). Added per-OS manifest URLs and updated the smoke test
matrix to cover all three platforms. On macOS, automatically selects the correct
archive for Apple Silicon or Intel based on runner architecture.

## Milestone 8 — Partial Version Matching ✓

Added support for version prefixes like `3.19`, resolving to the latest `3.19.x`
release in the manifest. Intended for use with stable channel releases.
