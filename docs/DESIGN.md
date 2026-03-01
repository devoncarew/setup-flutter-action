# setup-flutter-action Design

## Overview

`devoncarew/setup-flutter-action` is a GitHub Action that installs the Flutter SDK
into a workflow environment. It is optimized for fast startup — particularly on
repeated runs — by caching the SDK transparently via `actions/cache`.

The action is authored in TypeScript and compiled to a single `dist/index.js`
for execution, following the standard GitHub Actions pattern.

## Goals

- **Simple**: minimal configuration surface, easy to understand and maintain.
- **Fast**: cache the SDK automatically so short workflows pay the download cost
  only once per SDK version.
- **Correct**: resolve "latest stable/beta" to a specific version before
  caching, so cache keys are stable and deterministic.

## Inputs

| Input     | Required | Default    | Description |
|-----------|----------|------------|-------------|
| `channel` | No       | `"stable"` | Flutter release channel: `stable`, `beta`, or `main`. |
| `version` | No       | —          | Flutter version or prefix, e.g. `3.19.6` or `3.19` (resolves to the latest `3.19.x` stable release). Overrides `channel`. |

If both are omitted, the action defaults to the latest `stable` release.

Partial version prefixes (e.g. `3.19`) are intended for use with stable
releases. Behavior with beta or main channel releases is unspecified.

## Outputs

| Output           | Description |
|------------------|-------------|
| `flutter-version` | The fully resolved Flutter version string, e.g. `3.19.6`. |
| `flutter-root`    | Absolute path to the Flutter SDK root directory. |

`flutter` and `dart` are both added to `PATH` via Flutter's `bin/` directory.

## Release Resolution

Flutter publishes a releases manifest per platform at:

```
https://storage.googleapis.com/flutter_infra_release/releases/releases_linux.json
```

The manifest structure is:

```json
{
  "current_release": {
    "stable": "<hash>",
    "beta": "<hash>",
    "main": "<hash>"
  },
  "releases": [
    {
      "hash": "...",
      "channel": "stable",
      "version": "3.19.6",
      "archive": "stable/linux/flutter_linux_3.19.6-stable.tar.xz",
      ...
    }
  ]
}
```

**Resolution algorithm:**

1. Fetch the releases manifest for the current OS.
2. If `version` is specified, find the release entry whose `version` field
   matches exactly, or find all releases whose `version` starts with the given
   prefix and pick the highest one.
3. If `channel` is specified (or defaulted), use `current_release[channel]` to
   get the hash of the latest release on that channel, then look up that hash in
   the `releases` array to get the version string and archive URL.
4. The resolved version string becomes the cache key.

The archive URL from the manifest is used directly for downloads, avoiding any
URL construction guesswork.

## Caching

Caching is handled transparently using `@actions/cache`. Users do not need to
add a separate cache step.

**Cache key:** `setup-flutter-<os>[-<arch>]-<resolved-version>`

| Platform | Cache key |
|----------|-----------|
| Linux | `setup-flutter-linux-<version>` |
| macOS (Apple Silicon) | `setup-flutter-macos-arm64-<version>` |
| macOS (Intel) | `setup-flutter-macos-x64-<version>` |
| Windows | `setup-flutter-windows-<version>` |

**Cache path:** the directory containing the extracted Flutter SDK
(e.g. `$RUNNER_TOOL_CACHE/flutter/<version>/`).

On a cache hit, the action skips the download and extract steps entirely and
proceeds directly to adding the SDK to PATH and setting outputs. On a cache
miss, the action downloads, extracts, caches, and then proceeds.

No warning or annotation is emitted on a cache miss — this is intentional to
keep logs clean.

## Download & Extraction

Archives are downloaded from `storage.googleapis.com` using the URL provided
directly in the releases manifest.

- **Linux:** `.tar.xz` archive, extracted with `tar xJ`
- **macOS / Windows:** `.zip` archive, extracted with the platform unzip utility

On macOS, the manifest contains separate entries for Apple Silicon (`arm64`) and
Intel (`x64`). The action detects the runner architecture and selects the
correct archive automatically.

Extraction is handled by `@actions/tool-cache` utilities (`extractZip` /
`extractTar`).

## Pub Cache

The action sets `PUB_CACHE` to `~/.pub-cache` and adds `~/.pub-cache/bin` to
`PATH`. This ensures a consistent pub cache location across all platforms and
makes globally-activated Dart tools (e.g. `dart pub global activate`) available
without extra configuration.

Pub package caching (caching the contents of `PUB_CACHE` between runs) is not
currently implemented. See issue #14.

## Platform Support

Linux, macOS (Apple Silicon and Intel), and Windows are all supported.

## Implementation Language & Build

- **Language:** TypeScript
- **Runtime:** Node.js 20 (per `action.yml` `using: node20`)
- **Entry point:** `dist/index.js` (compiled from `src/index.ts`)
- **Build:** `tsc` + `ncc` to bundle into a single file with dependencies
- **Key dependencies:**
  - `@actions/core` — inputs, outputs, PATH manipulation, logging
  - `@actions/cache` — transparent SDK caching
  - `@actions/tool-cache` — download and extract utilities
  - `@actions/http-client` — fetching the releases manifest

## Repository Structure

```
setup-flutter-action/
├── action.yml          # Action metadata (inputs, outputs, entry point)
├── src/
│   └── index.ts        # Main action logic
├── dist/
│   └── index.js        # Compiled + bundled output (committed to repo)
├── docs/
│   ├── DESIGN.md       # Design rationale and architecture
│   └── PLAN.md         # Implementation history
├── .github/
│   └── workflows/
│       ├── ci.yml      # Build, lint, test on PRs
│       └── smoke.yml   # End-to-end test: run the action itself
├── package.json
├── tsconfig.json
└── README.md
```
