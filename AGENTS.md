# AGENTS.md

Guidance for coding agents working on **opencode-mem** — a persistent memory plugin for OpenCode (embedded Turso/libSQL vector search, auto-capture, Svelte web UI).

## Setup

Two independent Bun projects. A root-only install is **not** enough:

```bash
bun install                # root plugin: src/, tests/, scripts/
cd web && bun install      # web UI: own package.json + bun.lock
```

`bun run build` runs `tsc` **and** `bun run web:build`, so missing `web/node_modules` breaks the build.

## Commands

| Command                                      | Notes                                                                                                       |
| -------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `bun test`                                   | Full suite (~103 files). Several tests import from `dist/` — run `bun run build` first or they fail.        |
| `bun test tests/turso-vector-search.test.ts` | Single file; `bun test -t "<name>"` filters by test name.                                                   |
| `bun run typecheck`                          | `tsc --noEmit` over **`src/**` only** (`tsconfig.json` `include`). Tests and web are not covered.           |
| `cd web && bun run check`                    | `svelte-check` for the web UI — the only type check it gets.                                                |
| `bun run lint`                               | ESLint with `--max-warnings=0`; `no-explicit-any` is **off**, unused vars only flagged unless prefixed `_`. |
| `bun run format` / `format:check`            | Prettier: printWidth 100, double quotes, semicolons, LF, `trailingComma: es5`.                              |
| `bun run check`                              | `format:check && lint && typecheck` — no tests.                                                             |
| `bun run build`                              | `rm -rf dist && bunx tsc && bun run web:build`; Vite emits the UI into **`dist/web`**.                      |
| `bun run dev`                                | `tsc --watch` only — does not rebuild the web UI.                                                           |
| `bun run web:dev`                            | Vite dev server proxying `/api` to `127.0.0.1:4747`; needs a running plugin.                                |
| `cd web && bun run dev:sim`                  | Same UI with `/api/*` served from `web/sim` fixtures — no backend needed.                                   |

`.husky/pre-commit` runs `bun run typecheck && bunx lint-staged`, so typecheck runs on every commit even for docs-only changes.

## What CI actually runs

- `quality.yml` — format:check → lint → typecheck. **No tests.**
- `platform-smoke.yml` — builds, runs `bun test`, then `npm pack` + installs the tarball into a temp project and runs `scripts/native-deps-smoke.mjs`, `scripts/verify-libsql-vector.mjs`, `scripts/smoke-test.mjs` under Node.
- `embedding-backend.yml` — `bun install --ignore-scripts` (mimics OpenCode's script-skipping install) plus real ONNX inference under both Bun and Node.
- `release.yml` — triggers on tag `v*` and **fails unless the tag equals `package.json` version**, then `npm publish`. Merged release commits read `chore(release): release v2.29.2`.

## Architecture

- `src/plugin.ts` — package entrypoint. Default export merges the **v2** plugin (`src/v2/plugin.ts`) with `server:` bound to the **v1** plugin (`src/index.ts`).
- Root `index.js` re-exports `dist/plugin.js` on purpose: OpenCode resolves directory plugins via `<dir>/index` and ignores `exports`/`main`.
- `src/index.ts` — v1 plugin: the single `memory` tool (one tool, `mode` enum), commands, auto-capture, profile learning, web server startup.
- `src/v2/` — v2 adapter + legacy client. **v2 no longer publishes `session.idle`**, so `legacy-client.ts` maps `session.execution.succeeded/failed/interrupted` → `session.idle`. Auto-capture triggers on `session.idle` (`src/index.ts`); breaking that mapping silently disables capture.
- `src/services/` — flat modules by concern (`auto-capture.ts`, `embedding.ts`, `migration-service.ts`, `web-server.ts`, …) plus subpackages: `turso/` (shards, vector search, ordered schema migrations), `ai/` (`providers/`, `session/`, `tools/`, `validators/`), `user-profile/`, `user-prompt/`.
- `src/shared/api/` — zod schemas shared by the plugin **and** the web UI through the Vite `$shared` alias; changing one moves both sides.
- `src/config.ts` — config resolution and secret handling.
- `web/` — private Svelte 5 + Tailwind 4 app. Aliases: `$lib` → `web/src/lib`, `$shared` → `src/shared`.

Structured-output calls (auto-capture, profile learning) run through an internal least-privilege agent `opencode-mem-structured` with `thinking: { type: "disabled" }`, re-applied in `chat.params` after OpenCode merges options. Forced `tool_choice` breaks thinking models, so don't re-enable thinking there.

## Configuration

Two layers, merged shallowly in `initConfig()` (`src/config.ts`):

1. **Global** — `~/.config/opencode/opencode-mem.jsonc`, falling back to `.json`. On Windows that is `%USERPROFILE%\.config\opencode\`, **not** AppData.
2. **Project** — `<project>/.opencode/opencode-mem.jsonc` or `.json`.

Gotchas that are easy to get wrong:

- Within one layer the files are **not merged**: `loadConfigFromPaths()` returns the first existing file and silently ignores parse failures. A broken `.jsonc` falls through to `.json`, then to `{}`.
- A project config that sets `embeddingApiUrl`, `embeddingApiKey`, `memoryProvider`, `memoryApiUrl`, or `memoryApiKey` **throws at startup** (`assertProjectRemoteProviderConfigIsSafe`). Remote providers are global-only by design.
- `autoCleanupEnabled` / `autoCleanupRetentionDays` are deleted from the project layer before merging — global only.
- Secrets accept `literal`, `env://VAR`, or `file://path` (`src/services/secret-resolver.ts`).
- `opencode-mem.example.jsonc` is the documented template; the plugin writes a commented template on first start. Update both when adding config keys.
- Storage defaults to `~/.opencode-mem/data`; project shards are keyed by a project-identity hash.

There is **no `embeddingProvider` key**. Local vs remote is inferred: `embeddingApiUrl && embeddingApiKey` both present → remote OpenAI-compatible `/embeddings`, otherwise local transformers. Setting only the URL silently stays local. Because `embeddingApiUrl` is the gate, a missing `embeddingApiKey` falls back to `process.env.OPENAI_API_KEY` — an env var alone can flip the backend to remote.

## Local embedding load path

`EmbeddingService.ensureTransformersLoaded()` (`src/services/embedding.ts`) has an ordering contract — reordering it breaks OpenCode's compiled Bun host (#210):

1. Resolve the transformers entry via `requireFromHere.resolve()` **before** installing the onnx shim. After the shim is in place, a plain package resolve fails with `Cannot find module '@huggingface/transformers' from ''`.
2. `prepareOnnxruntimeForTransformers()` installs a `Module._resolveFilename` patch that redirects `onnxruntime-node` / `onnxruntime-common` to this package's pinned copy, then asserts the platform binding exists.
3. Load through the **CJS** entry — the ESM entry's static `import "onnxruntime-node"` bypasses the shim.
4. Then set `env.allowLocalModels`, `env.allowRemoteModels`, `env.cacheDir = {storagePath}/.cache`, and `env.backends.onnx.wasm.numThreads = 1`.

Model files are fetched from Hugging Face on first use; there is no vendored model path. `pipeline()` gets `progress_callback` plus a `dtype` resolved in two tiers, and only when both decline is nothing passed (letting transformers.js apply its own defaults). Getting this wrong fails as `fetch failed`, so the resolution is load-bearing, not a tuning knob:

1. `readDeclaredDtype()` reads `transformers.js_config.dtype` from `{cacheDir}/{repo}/config.json`. Repos published for the transformers.js ecosystem carry that block; plain PyTorch/HF repos do not, and a **top-level** `dtype` (e.g. `"bfloat16"`) belongs to PyTorch, not to transformers.js. A missing or malformed `config.json` returns `undefined`.
2. `detectCachedDtype()` probes `{cacheDir}/{repo}/onnx/model{suffix}.onnx` in descending precision (fp32 → fp16 → q8 → int8 → q4) and returns `undefined` when nothing is cached. A declaration always wins over what is on disk.
3. Nothing passed → transformers.js resolves the device default (cpu → fp32) and requests `onnx/model.onnx`, which quantized repos don't ship. The local lookup 404s, falls through to a remote fetch, and fails with `fetch failed` on offline hosts.

Tier 2 exists because hardcoding one dtype breaks other repos: `model_int8.onnx` needs `int8`, fp16-only repos need `fp16`. Probing covers every variant and prefers the highest precision present. Tier 1 is read separately because `get_available_dtypes()` is not exported from the package entry, so a deep import would break the lazy specifier in `getTransformersPackageSpecifier()`.

Cache layout must match transformers.js's own resolution: **ONNX weights under `{cacheDir}/{repo}/onnx/`** (`model_quantized.onnx` + external `model_quantized.onnx.data`), while `config.json` / tokenizer files sit directly in `{cacheDir}/{repo}/`. The `onnx/` subfolder applies to weight files only — moving the config/tokenizer files in there breaks their lookup.

`embeddingDimensions` is **not** derived from the model. `getEmbeddingDimensions()` (`src/config.ts`) is a hardcoded lookup table with a `|| 768` fallback, and nothing cross-checks it against what the model actually emits — an unknown model silently writes vectors at the wrong dimensionality. Set it explicitly.

Failure handling: `formatOnnxruntimeInitError()` distinguishes "binding absent" (Intel Mac hint, points at remote config) from "binding present but load failed" (preserves the original dlopen/codesign error). `initError` latches so a failed warmup doesn't retry forever (#184).

Only handlers that compute similarity call `warmup()` — see the comment in `api-handlers.ts` before adding one.

## Native dependency constraints

- `onnxruntime-node` is pinned to the exact version `1.30.0` in **both** `dependencies` and `overrides`: nested OpenCode installs ignore a dependency's overrides, so the direct dependency is what keeps the native binding. `tests/package-dependencies.test.ts` asserts the pin.
- Intel Mac (`darwin/x64`) is unsupported — `@tursodatabase/database` and fixed `onnxruntime-node` releases ship no x64 binding.
- On Windows the Turso engine rejects `multiprocess_wal`, so only one OpenCode session may own the databases at a time.
- Local embeddings run on CPU through `@huggingface/transformers` + ONNX (no MLX); the first use downloads the model into `{storagePath}/.cache`.

## Testing quirks

- Tests are outside `tsconfig.json`, so `bun run typecheck` never catches type errors in `tests/`.
- These load the built output and need `bun run build` first: `plugin-loader-contract`, `plugin-root-entrypoint`, `plugin-v2-loader-contract`, `plugin-bundle-boundary`.
- `tests/turso-test-utils.ts` exposes `cleanupTursoTestDirectory()`; Windows file locks are retried and leftover temp dirs are tolerated on purpose. Suites are timing- and lock-sensitive — use `mkdtemp` dirs, never real storage.
- `tests/windows-path.test.ts` and other platform-shaped fixtures encode path assumptions; check them before "simplifying" path code.

## Commits — Conventional Commits only

All commits **MUST** follow [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/): `<type>[optional scope]: <description>`.

Allowed types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`. Do not invent new ones.

- Description: imperative, concise, no trailing period (`fix: harden provider discovery`)
- Scope is the touched area: `feat(turso):`, `fix(ai):`, `chore(deps):`
- Breaking changes: `!` after type/scope **or** a `BREAKING CHANGE:` footer
- One logical change per commit when practical

Real examples from history:

```
feat(config): add opencodeVariant for internal LLM calls
fix(v2): map failed and interrupted execution events to session.idle
chore(deps): bump the minor-and-patch group across 2 directories
```

## Pull requests

Squash-merge is preferred, so **only the PR title lands on `main`** — it must be a valid Conventional Commit subject. Do not put Conventional Commit syntax in the PR body; use `.github/PULL_REQUEST_TEMPLATE.md` (Summary, Test plan, Checklist).

- Base branch: `main`
- Before opening: `bun test`, `bun run typecheck`, and `bun run check` when code changes
- Update README / `opencode-mem.example.jsonc` when behavior or config changes

## Code & review habits

- Match existing patterns in the touched area; avoid drive-by refactors
- Keep diffs focused on the requested change
- Prefer libraries and utilities already in the tree
- Do not commit secrets or local data under `data/`
- Ask before destructive git operations (`push --force`, hard reset, history rewrite); only commit when explicitly asked
