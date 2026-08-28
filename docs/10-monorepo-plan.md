# Monorepo Consolidation Plan

Status date: 2026-08-28. Branch: `chore/monorepo`.
Scope: make every application run in dev and build successfully, then turn the vendored
directories into a real monorepo. This document is the reference for that work; it does
not cover the product rewrite described in `09-complete-refactor-decision.md`.

---

## 0. Verified current state

Everything below was executed on this machine, not inferred.

| Project | Install | Build | Typecheck | Dev |
|---|---|---|---|---|
| `web/` | `pnpm install --frozen-lockfile` — 1370 pkgs, 5.2s | `mise build` — 25s, `packages/client/dist` | `mise build:check` — clean | `mise dev` — Vite 6.4.3 on `:5173`, HTTP 200 |
| `for-desktop/` | `pnpm install --frozen-lockfile` — 11.5s, native modules compiled | `pnpm package` — `out/` produced for linux/x64 | not run | `pnpm start` loads `https://stoat.chat/app` |
| `javascript-client-api/` | `pnpm install --frozen-lockfile` — 0.7s | `pnpm build` — regenerates `src/*`, emits `lib/` | via `tsc` in build | n/a (library) |
| `stoatchat/` | `mise install` — rust 1.92.0, cargo-nextest | **BLOCKED** — `dav1d-sys v0.8.3` build script | blocked | blocked |

Notes:
- `web` and `for-desktop` have no `.mise` drift problems in practice because mise resolves
  per-directory. The versions do diverge (see §4.5).
- `javascript-client-api` regenerated its committed `src/*` byte-identically, so the tree
  stayed clean. The generator is deterministic against the checked-in `OpenAPI.json`.
- The `lingui:compile` step prints compilation errors for the `sq` and `sv` catalogs
  (plural syntax written in the target language instead of ICU keywords). Non-fatal today
  because `--strict` is not passed. Two strings, worth fixing.

### 0.1 The one hard blocker

`stoatchat` depends on `image` with the `avif-native` feature
(`crates/core/files/Cargo.toml:37`, `crates/services/autumn/Cargo.toml:19`), which pulls
`dav1d-sys`. `stoatchat/.mise/config.toml:24` sets
`SYSTEM_DEPS_DAV1D_BUILD_INTERNAL = "auto"`, so when no system dav1d is present the crate
tries to build dav1d from source with meson — which is not installed, and neither are
ninja or nasm. `mise` cannot supply nasm (not in its registry), so the internal-build path
is a dead end on this machine.

The project's own `stoatchat/Dockerfile` solves this by installing `libdav1d-dev`. Do the
same locally:

```bash
sudo apt install -y libdav1d-dev
# undo: sudo apt remove -y libdav1d-dev
```

Pop!_OS 24.04 ships `libdav1d-dev 1.4.1-1build1`, which satisfies `dav1d-sys 0.8.3`.
`libssl-dev`, `pkg-config`, `make`, `cc`, `g++` are already present.

---

## 1. Tooling decision: stay on mise, do not adopt Nx

**Verdict: mise-en-place, with a root monorepo config. No Nx, no Turborepo, no moon, no Bazel.**

mise is already the toolchain manager and task runner for three of the four subprojects
(28 task scripts in `web/`, 18 in `stoatchat/`, 8 in `for-desktop/`). As of 2026 it also
ships the features that would have been the reason to add a second tool: a project graph,
`--affected`, BLAKE3 content-addressed task caching, and a self-hostable remote cache.

Verified locally on mise 2026.8.2 in a scratch repo:

- `monorepo_root = true` plus `[monorepo] config_roots` makes `mise run "//...:build"`
  fan out across subprojects.
- Cross-config task dependencies work: a task in `backend/mise.toml` can `depends` on
  `//web:build` and mise orders them correctly.
- `[tools]` layering works: the root declares shared pins, each subproject overrides only
  what it must. This resolves node 26.2.0 vs 25.4.0 and the three pnpm versions **without
  merging any workspace**.

Why not the alternatives:

| Option | Reason rejected |
|---|---|
| **Nx 23** | Rust is still a community plugin (`@monodon/rust` 3.0.0, last publish 2026-06-01, official status unanswered in nrwl/nx#34963). Self-hosted cache packages were deprecated 2026-05-21 after CVE-2025-36852 (CREEP), pushing users to paid Nx Cloud. Nx's own polyglot-Rust blog post uses mise to install Rust — it stacks on top of mise rather than replacing it. |
| **Turborepo 2.10** | Requires a single root `package.json` + one workspace, and a fake `package.json` per non-JS project — 18 of them for the crates. Its prerequisite is the unified pnpm workspace, which is blocked (see §4.4). Remote cache self-hosting is genuinely the best of the bunch, but that alone does not pay for the rest. |
| **moon 2.5** | Philosophically closest, but its own Rust handbook says it "is not a build system and does not replace Cargo", requires rustup in the environment, and warns against caching the `target` dir. Swapping a working mise + 28 task scripts for a second toolchain manager delivering the same features is churn. |
| **Bazel / Pants / Buck2** | Requires generating build rules for every crates.io dependency and rewriting Vite, PandaCSS, Lingui, Playwright, and electron-forge as rules. Nothing here needs hermetic builds or remote execution. The one legitimate draw — caching `rustc` — is nearly free with `sccache`. |

Known mise limitations to design around:

1. `mise tasks graph` does not discover a **nested** Cargo workspace or a nested pnpm
   workspace, which is exactly this layout. Consequence: `mise run --affected` reports
   "none" out of the box. Workaround: declare `[monorepo.projects]` explicitly, generated
   from `cargo metadata` and `pnpm -r list --json` by a small task (~21 entries). Do this
   in Phase 5, not before — everything else works without it.
2. `--affected` compares git revisions only; uncommitted edits are invisible, and the
   `HEAD~1...HEAD` default fails on a single-commit repo.
3. Flag order matters: `mise run --task-cache-stats //x:build`, not `mise //x:build --task-cache-stats`.
4. The monorepo lockfile default changes (warning in 2026.12.0, root default in 2027.6.0).
   Set `[monorepo] lockfile` explicitly now.
5. `jdx/mise-cache` (the remote cache server) states in its own README that it is
   experimental and not intended for outside use. Do not put CI on it yet.

---

## 2. What is actually broken, ranked

Severity is about consequence, not effort.

| # | Severity | Problem | Evidence |
|---|---|---|---|
| 1 | Critical | **The repository has zero CI.** GitHub only reads `.github/workflows` at the repo root; there is no root `.github/`. All 40 workflow files across 7 nested `.github/` directories are inert. | `find . -type d -name .github` |
| 2 | Critical | `stoatchat` cannot build without a system dav1d. | §0.1 |
| 3 | High | **The OpenAPI contract loop is severed.** `stoatchat/.github/workflows/rust.yaml:77-85` checks out `stoatchat/javascript-client-api` as a separate repo and curls `/openapi.json` into it; `javascript-client-api/.github/workflows/build_and_publish.yml:67-77` then checks out `stoatchat/javascript-client-sdk` to bump the pin. All three are now local directories. No local replacement script exists. | — |
| 4 | High | **`javascript-client-api/` is orphaned.** Nothing in the repo resolves to it. `web/packages/stoat.js/package.json:34` pins `stoat-api: "0.14.3"` from npm, while the vendored source is `0.15.3`. Editing it has zero effect on any build. | `pnpm-lock.yaml:563,6191` |
| 5 | High | `mise assets` was deleted in `b0a25aa9` but is still invoked by `for-desktop/.github/workflows/build.yml:32`, `build_appimage.yml:25`, `web/.github/workflows/canary-release.yml:26`, `production-release.yml:26`, and documented as required in `for-desktop/README.md:83`. | — |
| 6 | Medium | Docker build contexts assume the old repo roots. Every workflow uses `context: .` with an implicit `./Dockerfile`; `web/Dockerfile:14` needs `package.json`/`pnpm-workspace.yaml`/`pnpm-lock.yaml` at the context root. | — |
| 7 | Medium | **Three release-please configs collide in one tag namespace**, all with `include-component-in-tag: false`: `web` (0.15.2), `for-desktop` (1.5.3), `stoatchat` (0.15.3). | — |
| 8 | Medium | No root orchestration of any kind: no root `mise.toml`, `pnpm-workspace.yaml`, `compose.yml`, `.github/`, `git-town.toml`, `default.nix`, or meaningful `.gitignore`. `mise <task>` from the root finds nothing. | — |
| 9 | Medium | `stoatchat`'s config crate walks **upward** from cwd looking for `Revolt.toml` (`crates/core/config/src/lib.rs:41-51`). It now traverses the monorepo root — a new config-bleed surface that did not exist before vendoring. | — |
| 10 | Low | Vestigial ex-submodule scaffolding committed as plain files: orphan `pnpm-lock.yaml` in `web/packages/stoat.js` and `web/packages/solid-livekit-components`, a nested `pnpm-workspace.yaml` in the latter, two nested `.github/workflows` sets, leftover `.git/modules/packages/`, and six `packageManager` fields with six different pnpm versions. | — |
| 11 | Low | Both VS Code `nixEnvSelector.nixFile` settings resolve to `${workspaceFolder}/default.nix`, which no longer exists at the monorepo root. | — |
| 12 | Low | Stale docs: `web/README.md:30,33-34` still says `git clone --recursive`; `NITRO-PARITY-MAP.md:15-19` and `docs/01-architecture-flow.md` still say `for-web/`; `stoatchat/README.md:20-46` documents a `justfile` that does not exist and claims MSRV 1.86.0 against a pinned 1.92.0. | — |

Pre-existing bugs worth fixing while nearby: `pnpm ci` is not a real command
(`for-desktop/.github/workflows/multiplatform_build.yml:45`, `release-please.yml:80`);
`for-desktop/package.json:13` has transposed `--ext --fix` args; there is no
`service:voice-ingress` mise task even though the binary is built and containerized;
`stoatchat/Revolt.toml:32` points LiveKit at port 14706, which collides with gifbox's
listen port.

---

## 3. Target layout

Directory names stay as they are. `stoatchat/` in particular **must not** be renamed:
`stoatchat/.mise/config.toml:25` derives `DOCKER_NETWORK_NAME = "stoatchat_default"` from
the directory name, and `compose.yml` has no explicit `name:`.

```
/                          root mise.toml (monorepo_root), .github/, .gitignore,
                           release-please-config.json, git-town.toml, default.nix
├── web/                   pnpm workspace  — client, stoat.js, solid-livekit-components
│                          (absorbs javascript-client-api as a 4th member)
├── for-desktop/           separate pnpm workspace (nodeLinker: hoisted — cannot merge)
├── stoatchat/             cargo workspace, 18 crates, 8 binaries
├── self-hosted/           deployment compose (published images)
└── docs/                  architecture and decision records
```

Two package managers, three project roots, one task runner.

---

## 4. Phased plan

Each phase ends in a state where everything still builds. Do not start a phase before the
previous one is green.

### Phase 1 — Unblock the build (blocking, ~15 min)

1. `sudo apt install -y libdav1d-dev` (§0.1). **Requires the user.**
2. `cd stoatchat && mise exec -- cargo check --workspace --all-targets` — confirm clean.
3. `mise docker:start && mise start` — confirm the 7 services come up. Requires
   `cp livekit.example.yml livekit.yml` first; that file is gitignored.
4. Point the web client at the local backend: copy `web/packages/client/.env.example` to
   `.env` (API `:14702`, WS `:14703`, media `:14704`, proxy `:14705`, gifbox `:14706`).
5. Attach the desktop shell: `pnpm start -- --force-server http://localhost:5173`.

Exit criterion: all four projects build, and the full stack runs locally end to end.

### Phase 2 — Root orchestration — DONE (branch `chore/root-orchestration`)

Delivered:

1. **Root `mise.toml`** with `monorepo_root = true`, `[settings] experimental = true`, and
   `config_roots = ["web", "for-desktop", "stoatchat", "javascript-client-api"]`.
2. **Shared tool pins at the root**: `node = "26.2.0"`, `gh = "2.93.0"`,
   `git-town = "23.0.1"`. Removed from all three subproject configs, which now declare
   only what actually differs. `stoatchat`'s `node = "25.4.0"` was dropped: its only node
   consumer is the docs site, whose `engines` field asks for `>=20`.
3. **pnpm stays per-project** — 11.3.0 (web), 11.17.0 (for-desktop), 10.28.1
   (stoatchat/docs), 11.1.2 (javascript-client-api). Each is tied to a `packageManager`
   field and a lockfile; unifying them means regenerating three lockfiles and belongs in
   phase 5, not here.
4. **`javascript-client-api/.mise/config.toml`** created so the directory participates in
   the aggregates at all.
5. **Aggregate tasks at the root**: `install`, `build`, `lint`, `format`, `format-fix`,
   `test`, `dev`, `dev:backend`, `dev:web`, `down`.
6. **Two tasks added to `stoatchat`** so the aggregates reach it: `lint` (delegates to the
   existing clippy `check`) and `install:frozen` (`cargo fetch --locked`).
7. **Root `.gitignore`** now guards against a stray root `Revolt.toml` /
   `Revolt.overrides.toml` / `Revolt.test-overrides.toml`, closing the config-bleed hole
   in §2.9, plus root-level `node_modules/`, `target/`, `.env*`.
8. **Root `.vscode/settings.json`** with `rust-analyzer.linkedProjects =
   ["stoatchat/Cargo.toml"]` — without it rust-analyzer finds no cargo workspace when the
   monorepo root is opened.

Verified on mise 2026.8.2, in this repository:

```
mise run install     # 4 projects,  6.3s
mise run build       # 4 projects, 42.0s -> web/packages/client/dist,
                     #   for-desktop/out, javascript-client-api/lib, stoatchat/target
mise tasks --all     # 69 tasks across the 4 projects, fully namespaced
cd web && mise dev   # still resolves to //web:dev, unchanged workflow
```

Behaviours confirmed by experiment rather than documentation:

- `.mise/config.toml` (not just `mise.toml`) is discovered as a monorepo project config,
  and file-based tasks under `.mise/tasks/` are addressable as `//project:task`.
- `[tools]` layers correctly: the root supplies a floor, each project overrides.
- A root task can `depends = ["//...:task"]`, and a `run = [{ tasks = ["//a:x", "//b:y"] }]`
  block does resolve monorepo references, contrary to what the mise discussion thread
  suggests.
- A `//...:task` wildcard silently **skips** projects that do not define the task and
  still exits 0. Aggregate coverage is therefore spelled out in each task description.

Deviations from the original phase-2 list:

- **No `[monorepo] lockfile` key.** It is not present in `mise settings --all` on 2026.8.2,
  so setting it would be writing an unverified option. Revisit before 2026.12.0, when mise
  starts warning about the default.
- **No root `default.nix`.** Nix is not installed on this machine, so a merged shell could
  not be tested, and each subproject's own `default.nix` still works when entered directly.
  The two `nixEnvSelector.nixFile` settings are not in fact broken — VS Code only reads
  `.vscode/` at the workspace root, so the nested ones are inert when the monorepo root is
  opened and correct when the subdirectory is opened alone.
- **Root `git-town.toml` added** with `main = "main"` and `perennials = ["develop"]`; the
  two inert per-project copies were deleted (§6).

Exit criterion met: `mise install`, `mise build`, and `mise lint` from the repo root do
the right thing for every project.


### Phase 3 — Revive CI (~1 day)

This is the highest-consequence phase. The repo currently has no CI at all.

1. Create `/.github/workflows/` with one workflow per concern, path-filtered by
   subdirectory so a web change does not rebuild the Rust workspace:
   - `web.yml` — i18n check, typecheck, lint/format, Playwright e2e
   - `desktop.yml` — lint, format, package (linux/win/mac matrix)
   - `backend.yml` — clippy, nextest against REFERENCE and MONGODB
   - `docker.yml` — the base image plus the 8 service images plus the web image
2. Fix every path assumption: `context: ./web` + `file: ./web/Dockerfile`, artifact paths,
   `paths-ignore` globs, `config-file` locations.
3. Delete the `mise assets` steps (§2.5) and every `submodules: recursive` / `git submodule
   update --init assets` line.
4. Fix `pnpm ci` → `pnpm install --frozen-lockfile`.
5. Delete the 7 nested `.github/` directories after their content is either migrated or
   consciously dropped. Keep `stoatchat/.github/ISSUE_TEMPLATE/` and
   `pull_request_template.md` by moving them to the root `.github/`.
6. Move `web/.github/CODEOWNERS` to the root and re-path its rules (`/web/packages/client`).

Exit criterion: a PR against this repo runs checks for exactly the projects it touches.

### Phase 4 — Make the contract loop local (~1 day)

The OpenAPI chain is the only real coupling between the Rust and TypeScript halves, and
it is currently three cross-repo CI hops that no longer exist.

1. Absorb `javascript-client-api` into the `web/` pnpm workspace as a fourth member, and
   change `web/packages/stoat.js/package.json:34` from `"stoat-api": "0.14.3"` to
   `"stoat-api": "workspace:*"`.
   **This is a behavior change**, not a refactor: it moves the client from the published
   0.14.3 to the local 0.15.3. Diff the two before flipping, and run the e2e suite after.
   Remove `minimumReleaseAgeExclude: [stoat-api]` from `web/pnpm-workspace.yaml` once done.
2. Write a root task `contract:sync` replacing `stoatchat/.github/workflows/rust.yaml:62-95`:
   ```
   cargo run --bin revolt-delta &   # or mise //stoatchat:service:api
   wait for http://localhost:14702/
   curl http://localhost:14702/openapi.json -o javascript-client-api/OpenAPI.json
   pnpm --filter stoat-api build
   ```
3. Add a CI check that fails when `OpenAPI.json` is out of date with the Rust routes —
   the same shape as the existing `lingui:check` dirty-tree check.
4. Decide whether `stoat-api` still needs to be published to npm. If the only consumer is
   in this repo, stop publishing and delete `build_and_publish.yml`. If external consumers
   exist, keep publishing but drive it from the root release config.

Exit criterion: changing a Rust route and running one command updates the TypeScript
client, with a CI check that catches drift.

### Phase 5 — Hygiene, release, and caching (~1 day)

1. One root `release-please-config.json` with `packages` keyed by subdirectory and
   `include-component-in-tag: true`, plus one manifest holding all three versions. Delete
   the three per-project configs.
2. One root `renovate.json`; delete the two per-project ones and the dead
   `cloneSubmodules` config.
3. Delete vestigial files: `web/packages/*/pnpm-lock.yaml`,
   `web/packages/solid-livekit-components/pnpm-workspace.yaml`, the two nested
   `.github/` directories under `web/packages/`, `web/packages/stoat.js/default.nix`, and
   `.git/modules/packages/`. Remove the `packageManager` field from every non-root
   manifest.
4. Fix the stale docs listed in §2.12.
5. Only now, add the project graph: generate `[monorepo.projects]` from `cargo metadata`
   and `pnpm -r list --json` so `mise run --affected` works (§1, limitation 1). Roughly 21
   entries; generate, never hand-maintain.
6. Add `sccache` for the Rust half. Given `lto = true` on the release profile, this buys
   more CI time than any task-level cache would.

Exit criterion: one release process, one dependency bot, `--affected` working, no dead
files.

### 4.4 Explicitly not doing: one pnpm workspace at the root

`nodeLinker` is a workspace-global setting with no per-package override. `web/` requires
`isolated`; `for-desktop/` requires `hoisted`, and electron-forge additionally expects
`node_modules` in the app directory rather than at the monorepo root
(electron/forge#4188, open since 2026-03-24, no maintainer response).
`blockExoticSubdeps: false` — needed for `discord-rpc` — is likewise workspace-global.

`for-desktop/` stays a separate pnpm workspace. This is a real constraint, not a
preference. Absorbing `javascript-client-api` into `web/` (Phase 4) is the only merge
that is actually available.

### 4.5 Version drift to resolve in Phase 2

| Tool | web | for-desktop | stoatchat | Target |
|---|---|---|---|---|
| node | 26.2.0 | 26.2.0 | 25.4.0 | 26.2.0 unless proven otherwise |
| pnpm | 11.3.0 | 11.17.0 | 10.28.1 | 11.17.0 |
| gh | 2.93.0 | 2.93.0 | 2.25.0 | 2.93.0 |
| git-town | 23.0.1 | 23.0.1 | 22.7.1 | 23.0.1 |

Also: `web/Dockerfile` builds on `node:24-alpine` while mise pins 26.2.0, and
`for-desktop/.github/workflows/release-please.yml:76` uses node 24 with a
`# TODO: Figure out why 26 isn't working` comment. Resolve both rather than papering over.

---

## 5. Sequencing summary

| Phase | Blocks what | Effort | Needs the user |
|---|---|---|---|
| 1 — Unblock build | everything backend-side | ~15 min | yes, one `apt install` |
| 2 — Root orchestration | phases 3-5 | ~half a day | no |
| 3 — Revive CI | safe iteration on everything | ~1 day | secrets, if release workflows are wanted |
| 4 — Contract loop | the rewrite in doc 09 | ~1 day | decision on npm publishing |
| 5 — Hygiene and release | nothing; pure cleanup | ~1 day | no |

Phase 3 is the one to prioritize if time is limited. A repository with 40 dead workflows
and no checks will not survive a rewrite of the scale described in
`09-complete-refactor-decision.md`.

---

## 6. Branch model

`main` is the main branch. `develop` is the long-lived integration branch for the monorepo
rewrite and is declared perennial, so branches stacked on it get the correct parent and
nothing attempts to ship it into `main`.

Root `git-town.toml`:

```toml
[branches]
main = "main"
perennials = ["develop"]

[hosting]
dev-remote = "origin"
forge-type = "github"
github-connector = "gh"
```

Verified with `git-town config`: main branch `main`, perennial branches `develop`.
`web/git-town.toml` and `stoatchat/git-town.toml` were deleted — git-town only reads the
repository root, so both were inert and only invited confusion.

Phase 3 workflow triggers should therefore fire on `develop` (and on pull requests
targeting it) for the duration of the rewrite, with `main` reserved for releases.
