# Nitro-Parity Implementation Map — Stoat Stack

> Goal: cancel Discord Nitro. Every feature mapped to the exact repo, file, and change required.
> Mapped 2026-08-27 from full-source sweeps of all five repos in this workspace.
> Companion: [for-web/PROJECT-REPORT.md](./for-web/PROJECT-REPORT.md), [for-web/DISCORD-PARITY-ROADMAP.md](./for-web/DISCORD-PARITY-ROADMAP.md).

## Workspace

```
stoat/
├── stoatchat/              # backend Rust workspace: delta (REST), bonfire (WS), autumn (files),
│                           #   january (embeds), gifbox, pushd, crond, voice-ingress, core/*
├── for-web/                # Solid.js web client (+ submodules: stoat.js SDK, solid-livekit-components)
├── for-desktop/            # Electron shell that loads the web client
├── javascript-client-api/  # `stoat-api` npm pkg: OpenAPI.json → generated TS types
└── self-hosted/            # docker compose deployment (the running instance)
```

**How a new feature flows through the stack** (needed for every Part 4 item):

1. `stoatchat/crates/core/models` — data model + validators → `crates/core/database` — DB model/queries → `crates/delta` — REST route (rocket/okapi, auto-feeds OpenAPI) → `crates/bonfire` — WS event fan-out.
2. Regenerate `javascript-client-api/OpenAPI.json` from delta, then `pnpm build` there (`openapi-typescript`).
3. `for-web/packages/stoat.js` — SDK class/method consuming the new types.
4. `for-web/packages/client` — UI.
5. `self-hosted` — bump images (or build your own: `ghcr.io/stoatchat/api:v0.15.1` etc. in `compose.yml`).
6. `for-desktop` — nothing, unless the feature needs native OS access.

---

# Part 1 — Config-only unlocks (no code; this alone kills most of Nitro)

**There is no premium gating anywhere in the backend.** Exhaustive grep for premium/subscription/billing/mollie/stripe found nothing enforceable — only two cosmetic badge bits (`Supporter`, `ActiveSupporter` in `stoatchat/crates/core/models/src/v0/users.rs:178,186`) never checked against limits. The only tier split is account age: ULID timestamp ≤ `new_user_hours` → `new_user` limits, else `default` (`stoatchat/crates/core/database/src/models/users/model.rs:237-250`). A `limits.roles` map exists in the config struct (`core/config/src/lib.rs:414-415`) but is **dead code** — parsed, never consulted.

### Where config lives
- Authoritative defaults are **compiled into the binaries** from `stoatchat/crates/core/config/Revolt.toml` (`include_str!`, `core/config/src/lib.rs:72-75`).
- Load order (later wins): baked defaults → `./Revolt.toml` → `./Revolt.overrides.toml` → `/Revolt.toml` → env vars `REVOLT__SECTION__FIELD` (`lib.rs:57-64,113`). Cached 30 s (`lib.rs:526`).
- In `self-hosted/`: one `./Revolt.toml` is bind-mounted into **every** backend service (`compose.yml:90-191`). **`generate_config.sh --overwrite` rewrites it from scratch** — put your overrides in `secrets.env` as `REVOLT__…` env vars so upgrades can't erase them (`secrets.env.example` shows the naming pattern).
- Caddy has **no body-size limit** (unlimited by default); MinIO has no quota. The only upload ceilings are in Revolt.toml.

### The knobs (defaults from `stoatchat/crates/core/config/Revolt.toml`)

| Nitro perk | Key | Default | Suggested |
|---|---|---|---|
| 500 MB uploads | `features.limits.{new_user,default}.file_upload_size_limit.attachments` (:286/:325) **and** `features.limits.global.body_limit_size` (:248) — hard HTTP cap enforced at `autumn/src/api.rs:52-57`; must exceed every per-tag limit | 20 MB / 20 MB | 500 MB / 550 MB |
| 4000-char messages | `…{tier}.message_length` (:264/:303) — byte length, enforced at `core/database/src/models/messages/model.rs:284-288,975-996` | 2000 | 4000+ |
| Bigger avatars/banners/emoji | `…file_upload_size_limit.avatars/backgrounds/icons/banners/emojis` (:287-291/:326-330). ⚠️ Removing a tag key panics autumn (`api.rs:210` `.expect`) — only change values | 4M/6M/2.5M/6M/500K | at will |
| More attachments per msg | `…{tier}.message_attachments` (:267/:306) | 5 | 10 |
| 48 kHz voice ("Nitro quality") | `…{tier}.voice_quality` (:273/:312) — advisory, client-honored | 16000 | 48000 |
| 1080p+ video/screenshare | `…{tier}.video_resolution` (:279/:318) — `[0,0]` = unlimited; gates the client's 1080p tier | [1280,720] | [1920,1080] or [3996,2160] |
| Video/screenshare on at all | `…{tier}.video = true` — the **only** A/V value enforced server-side (`core/database/src/voice/mod.rs:156-171`, requires `Video` permission too) | true | keep |
| Kill new-account restrictions | `features.limits.global.new_user_hours` (:244) → `0`, or set `new_user` == `default` | 72 | 0 |
| More servers/bots/friend-reqs | `…{tier}.servers` (:270/:309), `.bots`, `.outgoing_friend_requests` | 100/5/10 | at will |
| More emoji per server | `features.limits.global.server_emoji` (:239) — instance-global | 100 | 500 |
| More roles/channels | `global.server_roles` / `server_channels` (:240-241) | 200/200 | keep |

**Apply:** append `[features.limits.default]` + `[features.limits.new_user]` blocks (or `REVOLT__FEATURES__LIMITS__DEFAULT__MESSAGE_LENGTH=4000`-style env in `secrets.env`), then restart `api` + `autumn` (+ `events`). Client picks limits up automatically from `GET /` (`delta/src/routes/root.rs:136-152`) — **all client enforcement is already server-driven** (verified: attachment size `Composition.tsx:228`, count `Draft.ts:411`, msg length `Composition.tsx:72`, emoji size `EmojiList.tsx:84`, avatar/banner `UserProfileEditor.tsx:162-177`).

### Already free (Discord charges for these)
Animated avatars/banners/emoji — fully supported, zero gating (`stoat.js/src/classes/File.ts:124-134`; backend `emoji_create.rs:64`). Custom emoji usable everywhere. Full theming. Real markdown (tables, KaTeX, headings). Message pins with **no cap** (Discord: 50).

---

# Part 2 — Client-only code changes (`for-web`)

### 2.1 HD streaming — the one real "Nitro feature" locked in code
1. **Delete the screenshare clamp** — `packages/client/components/rtc/index.ts:9-21`: global `getDisplayMedia` monkey-patch forcing **640×480 @ 5 fps** at capture time (comment says "never over 720p"; code says 480p). This defeats every quality picker downstream. Remove or make it honor the chosen preset.
2. **Camera is hardcoded 720p** — `components/rtc/state.tsx:223-231` (`VideoPresets.h720` capture + publish). TODO at `:224` already says "Support higher resolutions based on limits" — drive from `limits().video_resolution` like screenshare does.
3. **Add quality tiers** — `state.tsx:424-477` `getEnabledScreenShareQualities()`: `low`=720p30 hardcoded, `high`=1080p30 gated on server `video_resolution ≥ [1920,1080]`, `text`=original res but **forced 5 fps** (`:455`). TODO at `:442` anticipates more tiers. Add 1080p60/1440p/4K presets; un-force the 5 fps; extend the `"low"|"high"|"text"` enum in `state/stores/Voice.ts:17-28` and the picker modal (`modal/modals/ScreenShareSettings.tsx`).
4. **No bitrate/simulcast/codec config exists anywhere** — `new Room({...})` at `state.tsx:212-232` is the only spot. Add `videoCodec` (vp9/av1), simulcast layers, higher `maxBitrate` encodings.
5. Backend side is free: resolution/quality values are advisory; nothing server-side caps bitrate. LiveKit config (`self-hosted/livekit.yml`, generated by `generate_config.sh:212-230`) has no room/video block — defaults are fine, but the UDP range is only 100 ports (`50000-50100`).

### 2.2 Small unlocks
- MIME allowlist for image previews hardcoded: `state/stores/Draft.ts:85-90` (`jpeg/png/gif/webp`) — add avif/jxl (backend autumn accepts 9 formats: `autumn/src/metadata.rs:12-22`).
- Cosmetics counter thresholds hardcoded in `Composition.tsx:73-74`.

### 2.3 SDK bug (fix in `packages/stoat.js`, upstream it)
`src/classes/User.ts:366-370`: account age compared against `new_user_hours * 3600_0000` — **10× too large** (ms math). Every account honors restricted `new_user` limits for ~30 days instead of 3. One-character fix: `3600_000`.

---

# Part 3 — Desktop (`for-desktop`)

Thin Electron shell; loads a **remote** URL, so web features arrive automatically. Mapped surface:
- `src/native/window.ts:23-27`: `BUILD_URL` hardcoded `https://stoat.chat/app`, override only via CLI `--force-server=https://your.domain`. **Change:** make it a persisted setting in `src/native/config.ts` (config schema at `src/config.d.ts`) so your instance is the default.
- `src/main.ts:41`: `updateElectronApp()` auto-updates from **upstream** GitHub releases — will clobber a fork. Disable or point at your own releases.
- Exposed native API (`src/world/window.ts`): window controls, badge count, screen picker (with audio loopback), `isWayland`; virtual mic for Wayland screenshare audio (`src/native/virtualMic.ts`); tray, autostart, Discord RPC (!), spellcheck.
- **No global shortcuts at all** (grep: zero `globalShortcut`) → **push-to-talk is the missing Discord-desktop feature.** Change: `globalShortcut.register` in main + IPC into the web app's mute toggle (`for-web` already has a keybinds subsystem, `components/keybinds/`, currently without UI).

---

# Part 4 — Structural features (backend + full pipeline)

Confirmed absent from the data model (`stoatchat/crates/core/models` — zero occurrences): **polls, threads, forum channels, stickers, voice messages, soundboard**. Channel enum is only `SavedMessages | DirectMessage | Group | TextChannel | VoiceChannel` (`v0/channels.rs`). Building any of these = full pipeline from the top of this doc. In recommended order:

| # | Feature | Backend sketch | Client sketch | Size |
|---|---|---|---|---|
| 1 | **Polls** | New `Poll` model + votes collection; routes `POST /channels/:id/polls`, vote/close; bonfire events | Composer entry + render component + live results | S-M |
| 2 | **Voice messages** | `flags` bit or `Metadata::Audio` marker on message + duration/waveform fields | MediaRecorder → opus upload to autumn `attachments` + waveform player | S-M (backend), M (client) |
| 3 | **Voice moderation** (disconnect/server-mute) | delta route calling LiveKit server API (`RoomServiceClient.removeParticipant/mutePublishedTrack`) — token infra exists at `core/database/src/voice/voice_client.rs` | context-menu entries; per-user volume is pure client | S |
| 4 | **Threads** | Hardest: new channel-like model, parent message link, unread state, permission inheritance | Sidebar + thread panel UI | XL |
| 5 | **Scheduled events** | Event model + RSVP + crond reminders (crond daemon already exists) | Server events tab + banners | M |
| 6 | **Stickers** | Generalize emoji model (new autumn tag, bigger size) | Picker tab | M |
| 7 | **Soundboard** | Sound assets + LiveKit ingress playback (`can_publish_data` currently hardcoded `false` at `voice_client.rs:93` — flipping it enables data-channel triggers) | Sound picker in call UI | M-L |
| 8 | **Forums** | On top of threads | | L |

Skip (cosmetic monetization, YAGNI): super reactions, activities, stages, profile effects, shop.

**Contribute upstream where possible** — every fork item here means carrying patches across `stoatchat` releases (currently v0.15.x, release-please, active).

---

# Part 5 — Where Zig fits

Ranked by fit. The stack is Rust + TS; Zig enters cleanly at three seams — WASM in the browser, N-API in Electron, standalone microservice behind Caddy.

### Strong fit
1. **Upload preprocessor (WASM in for-web)** — resize/compress images and strip EXIF *before* they hit autumn. Hooks into the single funnel `Composition.tsx onFiles` (`:228-268`) → `Draft.ts`. Zig → `wasm32-freestanding`, no runtime, tiny binary. Pairs with the 500 MB limit raise (saves bandwidth on your own infra).
2. **Blurhash/thumbhash encoder+decoder (WASM)** — instant image placeholders in the message list (`features/messaging/elements/Attachment`). Small, pure-compute, ideal first Zig-WASM project.
3. **Global push-to-talk native addon (for-desktop, N-API via `zig cc`)** — OS-level key hook (evdev/X11/Wayland portals) compiled as a Node addon; Electron's `globalShortcut` works while app is focused-less but swallows the key globally — a native hook can listen without stealing. This is the Part 3 PTT item.
4. **Voice-message/media transcoder (standalone microservice)** — opus re-encode, waveform JSON, sticker/soundboard asset normalization. The backend is already microservices behind Caddy (`self-hosted/Caddyfile` routes per service) speaking HTTP + RabbitMQ; a Zig HTTP service (e.g. `zap` or std.http) slots in without touching Rust. This is the natural backend companion to Part 4 items 2/6/7.

### Moderate fit
5. **Audio DSP in the voice worklet** — gate/limiter/EQ stages in the existing chain (`components/rtc/VoiceProcessor.ts`: source → RNNoise(WASM) → highpass → compressor → gain). Zig-WASM AudioWorklet module alongside RNNoise.
6. **Markdown/emoji hot-path scanner (WASM)** — the unified/remark pipeline is the heaviest render cost (`components/markdown/`); a Zig pre-scanner for emoji/mention tokenization could cut it, but profile first — likely not the bottleneck.

### Poor fit (don't)
- Rewriting delta/bonfire/autumn (mature Rust, months of work, no gain).
- Anything crypto/auth (rule: mature libs only).
- The LiveKit server (Go upstream, config-driven already).

### Bug found during mapping (deploy, worth fixing regardless)
`for-web/docker/inject.js:16-24` reads `VITE_DEV_WS_URL`/`VITE_DEV_MEDIA_URL`/… but `self-hosted/generate_config.sh:184-187` writes `VITE_WS_URL`/`VITE_MEDIA_URL`/… — mismatched names silently become `void 0`; also `VITE_HOST` is required by `env.ts:48` but never written to `.env.web`. Verify against the pinned `for-web:0c31cf0` image before relying on the config generator.

### Other landmines found
- Backend: webhook send/edit ignore user tier (`webhook_execute.rs:66`), embed-count on edit hardcoded 10 vs config 5 (`v0/messages.rs:351`), group create hardcoded 49 users regardless of `group_size` (`v0/channels.rs:224`), `files.limit.max_mega_pixels/max_pixel_side` are dead config (never enforced), animated flag only set for GIF (animated WebP/AVIF pass but flagged static, `emoji_create.rs:64`).
- LiveKit tokens: 10 s TTL, `empty_timeout` 5 min, both hardcoded (`voice_client.rs:86,114`).

---

# Part 6 — Execution order

**Phase 0 (blocker, ~1h):** verify the self-hosted instance versions match these clones (compose pins `v0.15.1`, backend repo is `0.15.3`); decide upstream-PR vs fork per repo.

**Phase 1 — "cancel Nitro" (a weekend):**
1. Limits: add `REVOLT__FEATURES__LIMITS__*` overrides to `secrets.env` (Part 1 table), restart api+autumn.
2. for-web: delete screenshare clamp, camera from limits, add 1080p60 tier + simulcast/codec (Part 2.1).
3. stoat.js: one-char age-bug fix (Part 2.3).
4. Deploy your own `for-web` image; point desktop at it with `--force-server` (proper config default later).
→ **Nitro is now redundant. Cancel it.**

**Phase 2 — desktop daily driver (~1-2 weeks):** BUILD_URL config + updater redirect; Zig PTT addon (Part 5.3).

**Phase 3 — beyond Discord (ongoing):** polls → voice messages (+ Zig transcoder) → voice moderation → events → threads (Part 4), upstreaming each.
