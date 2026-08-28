# Project Report — Stoat for Web

> Snapshot: 2026-08-27, branch `refactor` (clean, in sync with `main`), version 0.15.2.
> Companion document: [DISCORD-PARITY-ROADMAP.md](./DISCORD-PARITY-ROADMAP.md) — the feature plan to replace paid Discord.

## 1. What this is

The official **web client for Stoat** (formerly Revolt — rebranded October 2025 after a cease-and-desist), the open-source, self-hostable Discord alternative. This repo is the frontend only; the platform is a set of separate services:

| Piece | Repo/tech | Role |
|---|---|---|
| **This repo** | Solid.js + TypeScript | Web client (`stoat.chat/app`) |
| Delta | Rust (stoatchat/backend) | REST API |
| Bonfire | Rust | WebSocket events |
| Autumn | Rust | File/media server (uploads, avatars, emoji) |
| January | Rust | Link-embed proxy |
| LiveKit | Go (upstream project) | Voice/video SFU |
| stoat.js | TS submodule here | Client SDK wrapping Delta/Bonfire |

**Key implication for the Discord-parity goal:** anything involving new data types (threads, events, polls…) needs backend work in the Rust repos, not here. Anything that is UI, limits, or presentation can be done in this repo plus instance config.

## 2. How it works

### Monorepo & tooling
- pnpm workspace with 3 packages: `packages/client` (the app), `packages/stoat.js` (SDK submodule), `packages/solid-livekit-components` (submodule). **All submodules are currently uninitialized in this checkout — the app cannot build until `git submodule init && git submodule update` + `mise build:deps` run.**
- Task runner is **mise** (`mise dev`, `mise build`, `mise check`). Vite + VitePWA (service worker, installable PWA). Docker image uses `__VITE_*__` placeholder injection at container start, so one image serves any instance.
- Import aliases `@revolt/<dir>` are auto-generated from `packages/client/components/*` (naming is Revolt-era debt; product is Stoat).

### App bootstrap
`packages/client/src/index.tsx` → `DeviceContext` → i18n → Router. Routes `/i/:host` (multi-instance, **currently disabled** — redirects to default) and `/` both mount `InstanceContext`, which resolves the backend: `.well-known/stoat` discovery for foreign hosts, else `VITE_API_URL`. Then providers stack in order: State → Keybinds → Modals → Client → Theme → Sound → Voice → TanStack Query.

### State
`components/state/` — a `State` object with 15 stores (Auth, Draft, Layout, Settings, Theme, Ordering, NotificationOptions, Keybinds, Voice…), each a Solid store persisted per-key to **localforage/IndexedDB** (1.2 s debounced writes; auth writes immediately). Three stores (`ordering`, `notifications`, `release-notes`) sync to the server with revision tracking via `SyncWorker`.

### Session lifecycle
`components/client/Controller.ts` — an explicit state machine (`Ready → LoggingIn → Connecting → Connected → Disconnected → Reconnecting…`) with exponential backoff + jitter, MFA flow, onboarding, and session invalidation handling. `useClient()` gives reactive access to the stoat.js `Client`.

### Voice/video
`components/rtc/state.tsx` — LiveKit. Health-checks `config.features.livekit.nodes`, joins via `channel.joinCall()`, `autoSubscribe: false`. RNNoise noise-suppression processor chain (source → RNNoise → highpass → compressor → gain). Camera capped at 720p; **screenshare hard-clamped to 640×480@5fps by a `getDisplayMedia` monkey-patch** (`components/rtc/index.ts`) — this is the single biggest "worse than Discord" limitation and it lives entirely in this repo.

### Messaging
CodeMirror 6 editor (`TextEditor2`) — a legacy ProseMirror editor still ships in parallel (dead weight). Drafts + outbox with idempotency keys and retry; uploads go straight to Autumn with per-file progress. Rendering is a unified/remark/rehype pipeline with GFM, KaTeX math, syntax highlighting, custom emoji, mentions, spoilers, timestamps — markdown support is already **richer than Discord's**.

### What a self-hosted instance must provide
API (Delta), WebSocket (Bonfire), Autumn, January, LiveKit nodes (from API config), VAPID keys for push, optionally hCaptcha. GIF picker is hardcoded to `api.gifbox.me` (TODO to source from backend). `VITE_HOST` must be set when pointing at a custom API.

## 3. Current momentum

23 releases, release-please driven. 0.15.0 (Aug 12) was the last feature release (role management UI, self-hosting instance context, LiveKit work, Material Symbols migration). Since then: pure stabilization — 14 fixes, 0 features across 0.15.1/0.15.2. Active themes upstream: LiveKit voice v2, self-hosting/multi-instance, Panda CSS design-system migration, TextEditor2.

## 4. What is pending (condensed inventory)

Full sweep done 2026-08-27; 71 TODOs, 2 FIXMEs, 35 hand-written `@deprecated`.

### Feature gaps (from `doc/src/feature-matrix.md`: 160 done, 12 missing, 8 in progress)
- **In progress:** voice v2 (LiveKit), screen sharing, webcam, desktop notifications (P0), member list, GIF picker, language settings, text-formatting extensions.
- **Missing:** update indicator (P0), desktop app (P1), changelogs, voice UX (per-user volume, hide non-video, ignore blocked), voice moderation (disconnect/server-mute), inline pronouns, cancel in-flight message.
- **Web Push:** marked won't-do for frontend in the matrix, but the push controller and service-worker rendering already exist (`NotificationsController.ts`) — matrix is stale.

### Debt clusters (largest first)
1. **Design-system migration** — `components/ui/components/features/legacy/` (9 files, ~2 real consumers left, nearly deletable), 12 deprecated Button variants, 3 icon packages, 2 Material libs, 3 theming layers coexisting.
2. **Two full text-editor stacks** shipped simultaneously (ProseMirror 11 deps + CodeMirror 6 deps).
3. **ServerSidebar.tsx** — densest TODO file: category visibility broken (category state not persisted anywhere), channel reordering disabled on mobile, a11y gaps.
4. **Drafts/uploads** — unsent files not persisted, no upload-progress %, known backend bug at `Draft.ts:366`.
5. **Multi-instance** — gated off at `components/instance/index.tsx:55`; Gifbox URL and status endpoint hardcoded to official infra. This directly affects self-hosting polish.
6. **Dead switches:** experiments store has only two placeholder flags that gate nothing; 6 settings pages commented out of navigation (Sync and Keybinds are *implemented* but unreachable).

### Health risks
- **Effectively zero test coverage**: one 10-line Playwright smoke test for ~58 k LOC. Playwright infra fully wired in CI; tests never written. No unit test framework at all.
- `solid-js` hard-pinned to 1.9.14 (newer versions break reactivity); `livekit-client` needs a local patch.
- `/dev` playground routed and linked in production without a DEV guard.
- Docs mdBook is mostly empty shells; `NOTES` at repo root is a stale personal backlog.

## 5. Suggested first moves (independent of the Discord-parity plan)

1. Initialize submodules and get a local build running (`git submodule update --init`, `mise install:frozen`, `mise build:deps`, `mise dev`).
2. Delete `features/legacy/` after migrating its 2 remaining consumers — biggest debt payoff per hour.
3. Remove the ProseMirror stack once nothing imports `design/TextEditor.tsx`.
4. Add vitest + a handful of tests around `Draft.ts` and `Controller.ts` before touching them — they are the highest-churn, highest-risk files.
5. Un-hardcode Gifbox/status URLs (instance config) — required for a clean self-hosted experience anyway.
