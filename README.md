# @cmrenus/oxlint-plugin-boundaries

Config-driven **cross-package / element-type boundaries** for [oxlint](https://oxc.rs) — enforce an architectural dependency matrix (which parts of your codebase may import which) **without a module resolver**.

You declare your own element table and allow-matrix in `.oxlintrc.json`; the plugin classifies every import by file path and flags edges your matrix disallows. Works in any monorepo (Bun / npm / pnpm / Yarn workspaces) and in single-package repos.

> **Status: alpha.** oxlint's JS plugin system (`jsPlugins`) is itself alpha and not yet semver-stable. This plugin pins a tested `oxlint` range in `peerDependencies` and is exercised against an oxlint-version matrix in CI. Expect a new minor when oxlint changes the plugin API. See [Versioning & the alpha pin](#versioning--the-alpha-pin).

## Why this exists

oxlint has no native cross-package boundaries rule, and the popular [`eslint-plugin-boundaries`](https://github.com/javierbrea/eslint-plugin-boundaries) **cannot run under oxlint's JS-plugin layer**: that layer intentionally exposes _no module resolver_ (no `context.resolve`, empty `parserServices`), and `eslint-plugin-boundaries` depends on one (`eslint-import-resolver-typescript`).

This plugin sidesteps the missing resolver by **classifying purely from the file path**. It walks up to your workspace root (including `pnpm-workspace.yaml` roots), reads each package's `package.json` `name` once to build a `name → directory` index, and resolves bare specifiers like `@scope/pkg` to an exported/source entry path when available — then to an element type. No resolver required.

## Install

```sh
bun add -D @cmrenus/oxlint-plugin-boundaries
# or: npm i -D @cmrenus/oxlint-plugin-boundaries  /  pnpm add -D @cmrenus/oxlint-plugin-boundaries
```

This fork publishes to GitHub Packages. Configure the registry and authenticate with a GitHub personal access token (classic) that has `read:packages`:

```ini
# .npmrc
@cmrenus:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

`oxlint` is a peer dependency — install it yourself and keep it within the supported range.

## Usage

Reference the plugin in `jsPlugins`, describe your architecture under `settings.boundaries`, and turn the rules on:

```jsonc
{
  "jsPlugins": ["@cmrenus/oxlint-plugin-boundaries"],
  "settings": {
    "boundaries": {
      // Ordered: more-specific patterns BEFORE their parents. First match wins.
      "elements": [
        { "type": "core", "pattern": "packages/core/**" },
        { "type": "db", "pattern": "packages/db/**" },
        { "type": "schemas", "pattern": "packages/schemas/**" },
        { "type": "api-client", "pattern": "packages/api-client/**" },
        { "type": "app-web", "pattern": "apps/web/**" },
      ],
      "rules": [
        { "from": "core", "allow": ["db", "schemas"] },
        { "from": "db", "allow": ["schemas"] },
        { "from": "app-web", "allow": ["api-client", "schemas"] },
        {
          "from": "api-client",
          "allow": ["app-api"],
          "importKind": "type",
          "message": "api-client may import apps/api only as `import type`.",
        },
      ],
      "default": "disallow",
      "ignore": ["**/*.test.ts", "**/*.spec.ts"],
    },
  },
  "rules": {
    "boundaries/element-types": "error",
    "boundaries/no-unknown": "error",
  },
}
```

> oxlint resolves `jsPlugins` paths **relative to the config file**. Using the package name (as above) is the normal case; a relative path would be resolved against the `.oxlintrc.json` location, not your shell's cwd.

## Configuration (`settings.boundaries`)

This object is the plugin's public API.

| Field            | Type                      | Required                  | Meaning                                                                                                                                                                                                                                                                                                                              |
| ---------------- | ------------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `elements`       | `Element[]`               | yes                       | Ordered list mapping path patterns to element types. **First match wins**, so list specific patterns before their parents.                                                                                                                                                                                                           |
| `rules`          | `Rule[]`                  | yes                       | Directional allow-matrix. Each entry says what a `from` element may import.                                                                                                                                                                                                                                                          |
| `default`        | `"disallow"` \| `"allow"` | no (default `"disallow"`) | Verdict for an edge not covered by any rule.                                                                                                                                                                                                                                                                                         |
| `ignore`         | `string[]`                | no                        | Glob-ish patterns for files to skip entirely.                                                                                                                                                                                                                                                                                        |
| `workspaceScope` | `string`                  | no (derived)              | The package-name prefix marking workspace-internal imports (e.g. `"@acme/"`). A bare specifier starting with it is a candidate boundary edge; anything else is an external dependency and ignored. When omitted, it is derived from the common scope of your workspace packages; set it explicitly if your packages don't share one. |

**`Element`**

| Field     | Type                   | Meaning                                                                                                                                                                      |
| --------- | ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `type`    | `string`               | Element type name, referenced by `rules`.                                                                                                                                    |
| `pattern` | `string` \| `string[]` | Root-relative path prefix/glob(s) classifying files into this type. Supports `*` within one path segment and `**` across segments. Arrays let one type cover multiple roots. |

**`Rule`**

| Field        | Type                  | Meaning                                                                                          |
| ------------ | --------------------- | ------------------------------------------------------------------------------------------------ |
| `from`       | `string`              | The importing element type this rule governs.                                                    |
| `allow`      | `string[]`            | Element types `from` may import.                                                                 |
| `importKind` | `"value"` \| `"type"` | Optional. `"type"` narrows the `allow` list to `import type` edges only (a type-only carve-out). |
| `message`    | `string`              | Optional. Custom message shown when this edge is violated.                                       |

Self-imports (an element importing its own type) are always allowed. External dependencies (npm packages outside your workspace) are never boundary edges and are ignored.

### Rules

- **`boundaries/element-types`** — the core rule. For every import, classifies both ends and reports edges your matrix disallows (honoring the type-only carve-out).
- **`boundaries/no-unknown`** — flags a workspace-style specifier that resolves to _no_ package (typically a typo or a deleted package). Closes the gap that `element-types` leaves when a target classifies to nothing.

## How classification works

1. **Find the workspace root** — walk up from the file to the nearest `package.json` declaring `workspaces` or a `pnpm-workspace.yaml` (falls back to oxlint's cwd). Keying off the file path keeps results identical no matter which directory you run oxlint from.
2. **Index packages** — expand workspace globs from package.json or pnpm's `packages:` list; read each package's `name` once and memoize a `name → dir` map.
3. **Classify both ends of each import** — the importing file by its path; the target by resolving a relative specifier against the file's directory, or a bare workspace specifier (`@scope/pkg[/sub]`) to its matching export/source entry path (falling back to the package directory).
4. **Evaluate** — `self` → allowed; in the value allow-list → allowed; `import type` and in the type-only allow-list → allowed; otherwise the `default` decides.

## Versioning & the alpha pin

oxlint's `jsPlugins` API is **alpha** and not semver-stable. The policy here:

- `peerDependencies.oxlint` is pinned to a **tested range**; CI verifies against the pinned baseline.
- When oxlint ships a breaking plugin-API change, this package cuts a new release with an updated range. The test suite is the tripwire — there are no defensive version guards in the runtime.
- Keep your `oxlint` within the supported range for predictable behavior.

## Authoring / build (for contributors)

Authored in TypeScript, shipped as compiled **ESM `.js` + `.d.ts`** so the published artifact is a plain, dependency-free ESM module with full types — no reliance on the consumer's Node having TypeScript type-stripping enabled. Built with [tsdown](https://tsdown.dev) (oxc / Rolldown).

```sh
bun install
bun run build        # tsdown -> dist/ (.js + .d.ts), runs publint + attw
bun run type-check
bun run lint         # dogfoods this very plugin
bun test
```

> This package requires **Node ≥ 24** (see `engines` in `package.json`) — for both consumers and contributors. CI runs on Node 24.

## Releasing

Releases are **fully automated from commit messages** — there is no manual version bump. The [CI workflow](.github/workflows/release.yml) runs on every push to `main`: a `verify` job (type-check, lint, build, test) followed by a `release` job that only runs once `verify` is green. The `release` job runs [semantic-release](https://semantic-release.gitbook.io/semantic-release), which reads the commits since the last release, computes the next version, updates `package.json` + `CHANGELOG.md`, tags the release, publishes to GitHub Packages, and opens a GitHub Release.

To make this work, commits must follow [Conventional Commits](https://www.conventionalcommits.org). The commit **type** decides the bump:

| Commit                                                        | Release   | Example (from `0.1.0`) |
| ------------------------------------------------------------- | --------- | ---------------------- |
| `fix:` / `perf:` / `revert:`                                  | **patch** | `0.1.1`                |
| `feat:`                                                       | **minor** | `0.2.0`                |
| `feat!:` / any commit with `BREAKING CHANGE:` in body         | **minor** | `0.2.0` _(see below)_  |
| `docs:` / `chore:` / `refactor:` / `test:` / `ci:` / `style:` | _none_    | —                      |

**Pre-1.0 policy.** While this package is in `0.x`, a breaking change bumps the **minor** version (not `1.0.0`) — matching the alpha status described in [Versioning & the alpha pin](#versioning--the-alpha-pin). This is enforced by a `releaseRules` override in [`.releaserc.json`](.releaserc.json). When the API is ready for 1.0, remove the `{ "breaking": true, "release": "minor" }` rule and the next breaking change will cut `1.0.0`.

A release only fires when at least one commit since the last release warrants one (a lone `docs:`/`chore:` push publishes nothing). The release commit is pushed back to `main` with `[skip ci]`, so it does not re-trigger the pipeline.

> **Maintainer setup.** GitHub Actions publishes with its built-in `GITHUB_TOKEN` and the release job's `packages: write` permission. The first published package is private by default; change its visibility to public in the package settings if it should be installable by everyone. Consumers need a GitHub personal access token (classic) with `read:packages` to authenticate to GitHub Packages.

**Recovering a failed release.** `@semantic-release/git` pushes the version-bump commit during the _prepare_ step, before the npm _publish_ step. If publish fails (e.g. a bad token), the bump commit may already be on `main` while npm never received the package. Fix the cause, then re-run the **Release** workflow from the Actions tab (`workflow_dispatch`) — semantic-release is idempotent and will complete the publish for the pending version without double-releasing.

**First GitHub Packages release.** The fork retains the existing `v0.1.0` baseline tag, so semantic-release will use the commits after that tag to choose the first scoped-package version. Merging the workspace-support `feat:` change to `main` will publish `0.2.0`.

## License

[MIT](./LICENSE) © Paul Cedrick Artigo
