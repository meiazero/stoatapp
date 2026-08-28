# Rewrite Map & Data-Model Design — The "Infinite Time" Plan

> Premise: engineering time is free; the goal is a near-perfect application — excellent DX, UX, and
> UI, and good behavior in resource-constrained environments (server and client). This doc maps
> (a) everything worth rewriting in a language/technology that brings a real advantage, with honest
> "no gain" verdicts, and (b) the data-model design for planned features — because schemas and
> persisted formats are the one layer that is expensive to get wrong and expensive to reverse.
> Companions: [../NITRO-PARITY-MAP.md](../NITRO-PARITY-MAP.md), [04-discord-ux-gaps.md](./04-discord-ux-gaps.md),
> [05-platform-and-stack-decisions.md](./05-platform-and-stack-decisions.md).

# Part 1 — Data-model design (do this before any rewrite)

**Principle:** implementation can be simplified and redone freely; persisted data formats, database
schemas, and wire contracts cannot. Every feature below goes model-first through the pipeline
(`core/models` → `core/database` → `delta` → `bonfire` events → OpenAPI → SDK), and the model gets
designed for its final shape even when the first implementation ships a subset.

## 1.1 Cross-cutting structural decisions

1. **`UserCapabilities` — kill the magic numbers.** The backend already shows the disease: webhooks
   ignore user tier (`webhook_execute.rs:66`), embed-count on edit hardcoded 10 vs config 5
   (`v0/messages.rs:351`), group size hardcoded 49 (`v0/channels.rs:224`), `limits.roles` parsed but
   never consulted (`core/config/src/lib.rs:414`), `max_mega_pixels` dead. End-state: one resolved
   `UserCapabilities` struct (upload size, message length, stream res/fps, feature booleans) computed
   in ONE place from config (+ future per-role overrides via the already-parsed `limits.roles`),
   consulted everywhere. This is additive and upstreamable.
2. **Flag-space discipline.** `MessageFlags` uses bit 1 only (`SuppressNotifications`); plan the
   space: `SuppressEmbeds = 2`, `Silent`-adjacent bits, `Forwarded`, `VoiceMessage` marker. Same for
   `UserBadges`/member flags. Never repurpose a shipped bit.
3. **Notification preferences become a server-side model** (today: opaque client sync blob — doc 04
   §2). Schema: `notification_settings` collection keyed `(user, target)` where target ∈
   {server, channel}, fields `{ level: all|mention|none, muted_until: Option<Timestamp>,
   suppress_mass_mentions: bool }`, inheritance resolved channel→server→default at read time.
   `pushd` consults it in the fan-out loop. Designed once, it also serves mobile forever.
4. **IDs stay ULID** — creation time is free (`decodeTime`), lexicographic order is exploited by the
   client; never introduce a second ID scheme.
5. **User-owned assets as a first-class concept** (the model beneath personal emoji/stickers/sounds
   — beyond-Discord doc): `Asset { id, owner: User|Server, kind: emoji|sticker|sound|avatar|banner,
   file, animated, pack: Option<PackId> }` + `Pack { id, owner, name, kind }`. Server emoji becomes a
   view over the same table (owner = Server). Permission `USE_EXTERNAL_ASSETS` gates cross-server
   use. Designing emoji/stickers/soundboard as three tables would be the mistake; one asset model
   serves all three plus future avatars/banner libraries.

## 1.2 Per-feature models (final shapes)

| Feature | Model sketch | Notes / traps |
|---|---|---|
| **Polls** | `Poll { id, channel, message, question, answers: Vec<{id, text, emoji?}>, votes: collection (poll, answer, user), expires_at, allow_multiselect, closed }` | Votes as separate collection (not embedded array) — unbounded growth; live results via bonfire delta events, not full re-fetch |
| **Voice messages** | Message flag bit + `Metadata::Audio { duration_ms, waveform: [u8; 100] }` on the attachment | Waveform precomputed server-side (media worker), fixed-size, never raw samples |
| **Threads** | New channel kind `Thread { parent_channel, root_message, auto_archive_after, archived }` reusing channel unreads/permissions with inheritance from parent | The hardest model; design permission inheritance NOW even if forums come later — forums are a channel kind whose children are threads |
| **Scheduled events** | `Event { id, server, channel?, name, description, start, end?, rsvp: collection, status }` + crond reminders | RSVP separate collection, same reason as votes |
| **Server profiles** | Already half-exists: `Member.nickname` + `Member.avatar`. Extend `Member` with `banner`, `bio` — do NOT create a parallel `ServerProfile` object | The model was right all along; ChatGPT's separate-object design would duplicate identity |
| **Status v2** | `UserStatus { text, emoji: Option<AssetRef>, presence, clear_after: Option<Timestamp> }` + daemon sweep | Emoji as asset ref, not unicode string — works for custom emoji day one |
| **Relationships v2** | `Relationship { user_id, status, nickname: Option<String> }`; separate `user_notes (owner, subject, note)` | Notes are not relationship-dependent (you can note a non-friend) |
| **Activities** | `activities: Vec<Activity { kind, application, title, subtitle?, started_at, image?, buttons?, metadata }>` on presence fan-out, open API (no whitelist) | Volatile — presence-layer only, never persisted; cap vec length in validator |
| **Bookmarks** | `bookmark (user, message, collection?, note?, tags?)` + `bookmark_collection (user, name)` | Purely additive, zero risk |
| **Scheduled messages** | `scheduled_message { user, channel, content…, send_at, created }` drained by crond | Reuse the full message validation at schedule time AND at send time (permissions may have changed) |
| **Uploads v2** | `upload_session { id, user, size, sha256, chunks_done, tag }` → tus-style resumable; file table gains `sha256` for dedup + `quota` accounting per user | Content-hash column enables dedup + "media vault" search later; design the column now even if dedup ships later |

## 1.3 Process rule

Every new model lands with: validators at the trust boundary, a bonfire event shape, an OpenAPI
regeneration, and an explicit pagination story (nothing unbounded returns whole collections). A
feature's UI can ship in waves; its schema ships once.

---

# Part 2 — Backend & infra rewrite map

Topology inspected: 18 crates / ~52k LOC Rust / 1,026 locked deps; 15 compose containers. Largest:
`core/database` 22.7k LOC, `delta` 14.3k (Rocket), `pushd` 2.3k, `bonfire` 1.8k (raw TCP+tungstenite),
`autumn`/`january`/`gifbox` ~1k each (axum).

## 2.1 The two structural findings

1. **The DB seam is genuinely good.** `core/database/src/drivers/mod.rs` = `enum Database
   { Reference, MongoDb }`; all 26 models declare `Abstract*` traits (155 methods) with a working
   in-memory `ReferenceDb` impl that passes the test suite (`TEST_DB=REFERENCE`). A third backend =
   one enum variant + 26 trait impls, with Reference as an executable conformance spec. **No route
   code changes.** (Trap: `authifier` keeps its own account/session storage — it also has an abstract
   DB trait; a swap must cover it.)
2. **The microservice split is broker-mediated, not HTTP-mediated.** Services never call each other
   over HTTP; coupling = Valkey pub/sub (→bonfire) + RabbitMQ (→pushd only) + shared DB. All 8
   binaries already link the same `revolt-database`/`revolt-config`. A single `stoat-monolith`
   binary (workspace `monolith` feature, one tokio runtime, axum routers merged under the existing
   Caddy path prefixes, pushd/crond as background tasks) is straightforwardly buildable.

## 2.2 Verdicts

| Component | Verdict | Concrete advantage |
|---|---|---|
| **MongoDB** | **Replace with SQLite** (in-process, WAL, FTS5 for message search, JSON functions for flexible docs) | mongod idles 300–500 MB RSS + 750 MB image on a small box → **0 MB, 0 containers**, single-file backup. Postgres only if staying multi-process (LISTEN/NOTIFY as bus). FoundationDB: no |
| **RabbitMQ** | **Replace** | Carries ONLY push fan-out to pushd (`amqp/amqp.rs`: 8 channels, one consumer). → Valkey Streams now, in-process mpsc after consolidation. ~130–150 MB RSS + `lapin` gone from 4 crates. Loss (broker-restart persistence) covered by a tiny outbox table |
| **MinIO** | **Replace with filesystem backend** | Blobs are already app-side encrypted (AES-GCM) + content-hashed — S3 buys nothing. `S3Storage` seam in `revolt-files` = ~100 LOC `FsStorage`. ~150 MB RSS, −2 containers. Garage if S3 API must survive |
| **Rocket (delta, voice-ingress)** | **Improve: migrate to axum** | Workspace is split-brained (axum+utoipa vs Rocket+okapi with git-pinned forks). Unifies OpenAPI on utoipa, ~150 fewer deps, enables type-gen overhaul |
| **Caddy** | **Keep** (+fold the `web` container into `file_server`) | Auto-TLS is the thing you don't reimplement; −1 container |
| **LiveKit (Go)** | **Keep + tune** | SFU rewrite = negative value. Single-node needs no Redis (delete the block `generate_config.sh` writes), `GOMEMLIMIT=128MiB`, `use_external_ip: false` for LAN |
| **Rust → Zig (any service)** | **No — honest verdict** | Ecosystem loss (push crypto, S3, LiveKit protocol, image/ffmpeg) dwarfs the 1–3 MB/service RSS win. Only bounded candidate: bonfire gateway (1.8k LOC, no framework, ~4-8 KB/conn hand-rolled vs 20-60 KB tokio) — legitimate learning project, irrelevant at personal scale |
| **Type-gen chain** | **Improve** | Today: hand-rolled `cli.js` + openapi-typescript v5 + an `anyOf→oneOf` regex hack over a spec produced by TWO generators. Path: utoipa-only → openapi-typescript v7/openapi-fetch. Endgame: **ts-rs straight from `revolt-models`** — Rust structs as single source of truth, no lossy JSON-schema hop |
| Build hygiene | Improve | `codegen-units=1`, `panic=abort`, mimalloc / `MALLOC_ARENA_MAX=2`: 20–40% RSS off every daemon for a day of work. Pin the unpinned `mongo:latest` until the swap |
| bonfire deps | Improve | async-tungstenite 0.17 (2022) + patched redis 0.23 fork → tokio-tungstenite ≥0.24 + modern client, drop the `[patch.crates-io]` |

## 2.3 End state

If SQLite + monolith + no-Rabbit + fs-storage land: idle stack RAM **~0.9–1.3 GB → ~200–350 MB**,
containers **15 → 4** (monolith, Caddy, LiveKit, and web folded into Caddy), backup = one SQLite
file + one blob directory. Compose profiles keep the split-process mode for future scale-out.

# Part 3 — Client rewrite map

## 3.1 Local persistence — the largest pure-UX win available anywhere in this doc

Today localforage/IndexedDB persists **settings only**; messages die on reload (`MessageCache.tsx`
is RAM-only), every cold start renders nothing until WS+REST round-trip, PWA offline = blank shell.

**End-state: SQLite everywhere.** `sqlite3.wasm` + OPFS SyncAccessHandle-pool VFS in a worker on
web (~300-400 KB lazy-loaded, no COOP/COEP needed); plain native SQLite in the desktop client —
same schema/queries, shipped inside the shared core (§3.3). Tables: messages (ULID-ordered),
users/members snapshot, unreads, drafts; **FTS5** over content. Wins: first-frame paint of
last-known state (Discord-class cold start, impossible today), instant channel switch everywhere,
offline reading + queued sends (nonce field already exists), search-as-you-type over local history,
and **bounded memory by construction** (hot window in RAM, rest on disk — 400 MB → ~150 MB tabs on
long sessions).

## 3.2 WASM candidates — honest verdicts

| Candidate | Verdict | Reasoning |
|---|---|---|
| **Markdown core** (Zig/Rust → WASM + native static lib) | **Rewrite — but as the shared-core play** | Custom syntax is small (`:emoji:`, mentions, timestamps, `!!spoiler!!`); a single parser shared byte-for-byte between web and native kills dialect drift forever + removes ~200 KB of unified/remark JS. Boundary design decides success: return ONE flat postorder node buffer per message (not a wasm-bindgen object graph), decode into the shape `childrenToSolid` already consumes; render stays Solid (mentions/spoilers are interactive). Honest note: a TS `Map<messageId, renderResult>` cache delivers most of the *felt* win in an afternoon — do that first |
| KaTeX / highlighting | Keep JS, lazy | No credible KaTeX WASM (consider Temml→MathML); hljs `common` + on-demand grammars. `shiki` and `rehype-prism` are **dead deps — delete** |
| Emoji/mention tokenizer | Fold into markdown core | Standalone WASM win is negligible (V8 regex is fast); free inside the tokenizer |
| Upload preprocessing | **Platform first, WASM later** | `createImageBitmap` + `OffscreenCanvas.convertToBlob` in a worker resizes + re-encodes + strips ALL metadata natively, zero bundle bytes. WASM mozjpeg only if quality-targeted encoding is demanded later. Today raw files upload with GPS EXIF intact (`Draft.ts:316-341`) |
| Blurhash/thumbhash | JS first | Feature doesn't exist client-side yet (server already computes thumbhash!); decode is <1 ms in JS |
| Compression | **No WASM** | Browser does permessage-deflate; the declared-but-unimplemented msgpack path (`EventClient.ts:84`) is a small TS task or falls out of the type-chain endgame |
| Client search index | Not standalone | Collapses into SQLite FTS5 (§3.1) |

## 3.3 SDK end-state — Rust core with mirror-store bindings (the LiveKit FFI pattern)

Keep-TS is near-optimal *for web only*; the native client changes the answer. End-state: core owns
WS state machine, reconnection, entity cache **with eviction**, permission calculus, unreads,
pagination, persistence (§3.1), markdown (§3.2) — compiled to WASM (web) + native lib (desktop).
**The reactivity boundary** (the known hard part): reads never cross it. Core emits per-frame
coalesced change events `(entity_kind, id, field_bitmask)` + values in one flat buffer; a ~1-2k LOC
shim per platform applies them — into Solid stores on web (preserving today's fine-grained DOM
updates exactly), into GPUI/Iced/Flutter state natively. Proven shape (Automerge/Yjs across WASM;
LiveKit across FFI). Mirror only subscribed entities — which fixes unboundedness by construction.
**Migration path:** extract pure functions first (protocol types, permissions, markdown — trivial
boundary), stateful cache second. Rust over Zig here: uniffi/flutter_rust_bridge/wasm-bindgen
codegen is decisive for a many-typed protocol core. Fix the TS SDK's O(k²) hydration + eviction NOW
regardless — those are bugs, not architecture.

## 3.4 Rendering — keep Solid, reject canvas

Solid is already top-tier for constrained devices (no VDOM, ~7 KB runtime); no framework migration
buys anything measurable. **Canvas/WebGL message list: rejected** — forfeits a11y tree, IME, text
selection, find-in-page, bidi, spellcheck, OS text scaling; rebuilding those is Google-Docs-team
cost to render text the browser's C++ already renders faster. The real fixes are application-level
TS: virtualization (virtual container already installed, used only by the emoji picker), markdown
render cache, devtools strip. What legitimately leaves JS: parsing and storage — compute, not
rendering.

## 3.5 Shell economics on constrained machines

Electron 43 idle with this SPA + LiveKit ≈ 350-500 MB RSS; on 4 GB machines that's swap territory.
Native client targets: Rust **GPUI/Iced ~40-90 MB** (best synergy: §3.3 core becomes in-process, no
FFI at all), **Qt-QML ~60-120 MB** (best software-render fallback for GPU-less old laptops — GPUI
and Flutter assume working GPU drivers), **Flutter ~80-150 MB** (fastest to polished UI + calls SDK
ready, per doc 05). Zig honesty: no production Zig UI toolkit exists — Zig's client role is compute
core behind a C ABI, not the shell. ⚠️ The audit suggested Tauri as an interim shell (a dead
`@tauri-apps/api` dep suggests someone tried); **doc 05 research overrules this: Tauri cannot run
LiveKit calls on Linux (WebKitGTK ships without WebRTC)** — interim Tauri would ship a chat app
without working calls. Electron stays the bridge until the native client lands.

# Part 4 — Media & voice path rewrite map

## 4.1 Server-side media (autumn / revolt-files)

Current pipeline facts: image-rs full-buffer decode (40 MP cap ≈ **160 MB RAM per concurrent
decode**), WebP-only thumbnails **recomputed on every preview request** (`api.rs:369-445` — the moka
cache holds raw bytes, not derived variants), thumbhash already computed at upload, SHA-256 dedup
already exists, no video transcode/posters.

**Active UX defect found:** EXIF strip **fully re-encodes every JPEG at quality ~75**
(`autumn/src/exif.rs:55-152`) — every JPEG upload suffers generational quality loss today. The PNG
re-encode (acropalypse CVE) can be replaced by lossless chunk truncation.

| Item | Verdict | Advantage |
|---|---|---|
| Pixel pipeline → **libvips** | Partial rewrite | Demand-driven streaming: O(strips) memory not O(pixels), ~3-6× faster at ~1/10 peak RAM, JPEG shrink-on-load makes 40 MP→128 px avatars 10-20× cheaper; AVIF encode built in |
| **Persist derived variants** keyed `(hash, variant)` | Improve | Kills per-request recompute — biggest server-resource win |
| **Lossless metadata strip** (byte-level APP1/chunk surgery; good small Zig or Rust routine, no decode) | Rewrite-small | Fixes the JPEG quality loss + privacy intact |
| **Queue-driven transcode worker** (Zig over ffmpeg/GStreamer C APIs — the best genuine backend Zig fit: crash isolation + simple C orchestration) | New | Video posters, AV1/H264 ladder, voice-message opus + waveform, sticker/soundboard normalization — unblocks 3 planned features |
| autumn HTTP/auth/S3 layer, **january**, **gifbox** | **Keep — no gain** | Network-bound; TLS/HTML/IP-filter ecosystems are Rust strengths |

## 4.2 Client audio chain

Today: source → highpass → RNNoise (worklet loaded **from a CDN URL** — self-hosting smell) →
compressor-as-limiter → gain (`VoiceProcessor.ts:93-131`). Native WebAudio nodes are cheap C++ —
WASM-ifying them gains nothing. What's missing: noise gate, VAD, true lookahead limiter.

**End-state:** one consolidated **Zig-WASM AudioWorklet** (gate → RNNoise → limiter → makeup gain +
VAD side-channel), self-hosted; **DeepFilterNet3 as opt-in "studio" tier** (WASM ports exist; ~12 MB
ONNX + ~10× RNNoise CPU + ~40 ms lookahead — gate it behind a quick CPU benchmark); on the native
desktop client, run the DSP natively (no WASM tax). VAD feeds speaking-indicator/PTT UX.

## 4.3 Screenshare encode — the largest UX delta in the whole map

Browser ceiling (honest): Chromium software-encodes VP8/VP9/openh264 for screenshare on Linux
(VAAPI-for-WebRTC is flag-gated and brittle); 1080p60 saturates weak machines — the current presets
(720p30/1080p30/source@5fps) encode that reality. **The native client unlocks:** PipeWire portal →
DMABUF zero-copy → `vah264enc`/`vaav1enc`/NVENC/QSV (Linux) and MediaFoundation/AMF/NVENC (Windows)
via GStreamer → **1080p60 at ~5-10% CPU on an iGPU**. Integration: libwebrtc external encoder
factory in the LiveKit native SDK (full simulcast semantics) or GStreamer→WHIP ingress (simpler).
Native capture also retires the `virtualMic.ts` Wayland hack and the `getDisplayMedia` monkey-patch.

## 4.4 livekit-client 4.6 MB patch

Misleading size: one real method (`applyScreenShareConstraints`) — the bytes are the patched `dist`
bundle + sourcemap. Public APIs can't replicate it (publishOptions/sender-encoding are private).
**Fix: upstream PR (it mirrors the existing `restartTrack` pattern) + thin source-level fork until
merged.** Never patch `dist` again.

## 4.5 Upload path end-state

Client (**Zig-WASM — the best genuine client Zig fit**): pre-upload resize/recompress, client-side
EXIF strip (GPS never leaves the device), local thumbhash for instant placeholder, SHA-256 for
hash-first upload. Server (extend autumn, Rust): hash-check endpoint (dedup skips the transfer —
table already exists), **tus resumable protocol** (thin PATCH-with-offset, small in axum) +
streaming-to-storage to kill full-file RAM buffering, quota accounting.

## 4.6 Voice server

LiveKit routes RTP, never transcodes — codec choice costs clients, not the server. Keep Go server +
keep `voice-ingress` (420 LOC on official LiveKit crates; rewrite = pure loss). Config wins:
single-port UDP mux, TURN/TLS on 443 fallback ("calls just connect" on hostile networks), opus
DTX+RED, simulcast ON (it's what lets a no-transcode SFU serve weak downlinks; cap weak senders to
2 layers).

# Part 5 — Consolidated ranking (all layers, by end-state advantage)

1. **SQLite everywhere** — server: MongoDB→SQLite via the existing 155-method trait seam
   (300-500 MB RAM reclaimed, 1-file backup); client: SQLite-WASM/OPFS + FTS5 (instant cold start,
   offline, local search, bounded memory). One storage philosophy, both sides.
   *Frontier check (2026-08):* stay on vanilla SQLite — server pins rusqlite with bundled
   **SQLite ≥ 3.53** (WAL-corruption fix + `ALTER TABLE` constraint add/drop for migrations); web
   uses **wa-sqlite `OPFSCoopSyncVFS`** in a dedicated worker + Web Locks for multi-tab (official
   Promiser API deprecated 2026-04). Skip libSQL (vendor-frozen), skip Turso rewrite until 1.0
   (track its stable CDC for future sync), skip DuckDB (bolts on later via sqlite scanner). Design
   our own sync protocol over plain SQLite — that layer of the frontier is still in flux.
2. **Native client** (Linux+Windows) with **hardware encode** — the two biggest UX deltas combined:
   60-120 MB instead of 350-500 MB, and 1080p60 screenshare at iGPU cost instead of saturating the
   CPU. Framework: Rust GPUI/Iced for the best end-state (in-process Rust core, no FFI) with
   Qt-QML as GPU-less fallback; Flutter if shipping speed ever outranks the end-state premise.
3. **Rust SDK core with mirror-store bindings** — protocol/permissions/cache/persistence/markdown
   written once for web+native; enables #1-client and #2 without triple-writing stateful logic.
4. **Backend consolidation** — monolith feature + drop RabbitMQ (→mpsc) + MinIO→filesystem + web
   into Caddy: 15 containers → 4, ~1 GB idle → ~250 MB. Deployment near-perfection for self-hosters.
5. **Pure-TS perf fixes** (not rewrites — do first, days not months): devtools out of prod, floating
   registry, virtualization + markdown cache, SDK hydration/eviction, lossless-EXIF fix server-side.
   Most of the *felt* UX gap closes here, in the current languages.
6. **Server media pipeline** — libvips + persisted variants + transcode worker: ~10× memory, AVIF,
   video posters, voice-message waveforms; fixes the silent JPEG quality loss.
7. **Shared markdown core** (WASM + native lib) — dialect parity forever, −200 KB JS.
8. **Single-source types** — delta→axum→utoipa now; endgame ts-rs/bridge codegen straight from Rust
   models (zero drift by construction), WS events inside the contract, binary frames for free.
9. **Upload path end-state** — client preprocessing (Canvas-native → later WASM) + tus resumable +
   hash-first dedup skip + quota.
10. **Audio chain** — unified worklet (gate+RNNoise+limiter+VAD, self-hosted), DFN3 opt-in tier,
    native DSP on desktop; LiveKit config tuning (UDP mux, TURN/443, DTX/RED).

## Where Zig actually lands (consolidated honest verdict)

Wins: **transcode worker** (C-library orchestration, crash isolation), **upload-preprocess WASM**
(later stage), **audio DSP worklet**, **lossless EXIF surgery**, optionally the markdown tokenizer.
Loses: any backend service (ecosystem), the UI shell (no toolkit), the SDK core (codegen decides
for Rust). The bonfire gateway remains a legitimate learning project only.

## Explicitly not worth rewriting (final list)

Solid.js, canvas rendering of text UI, january, gifbox, voice-ingress, LiveKit server, autumn's
HTTP layer, Caddy, WebAudio native nodes, standalone WASM for emoji/blurhash/compression, any
web-framework migration.
