# Discord-Parity Roadmap — replacing paid Discord with self-hosted Stoat

> Goal: cancel the Discord Nitro subscription. The self-hosted instance + this client must cover everything Nitro provides, everything free Discord provides that matters day-to-day, and ideally a bit more.
> Companion document: [PROJECT-REPORT.md](./PROJECT-REPORT.md) — architecture and current state.

## 0. The core insight

**Most Nitro perks are artificial limits, not features.** Upload caps, streaming quality, animated avatars, emoji across servers, message length — Discord charges to *lift restrictions*. On a self-hosted instance you control the limits, so a large chunk of "Nitro parity" is instance configuration plus small client changes, not engineering. The real engineering is in Discord's *structural* features (threads, forums, events, soundboard), and those need backend (Rust) work, not just this repo.

Effort legend: 🟢 config/hours · 🟡 client-only, days · 🔴 backend + client, weeks each.

---

## 1. Nitro perks — status on self-hosted Stoat

| Nitro perk | Status | What it takes |
|---|---|---|
| Bigger uploads (500 MB) | 🟢 already yours | Autumn config: raise size limits per tag (`attachments`, etc.) + reverse-proxy body limit. Client reads limits from API config (`instance.limits()`), no client change. |
| Longer messages (4000 chars) | 🟢 | Backend message-length config. Client honors server limit. |
| Animated avatars/banners | ✅ have it | GIF avatars/banners already supported, free. |
| Custom emoji everywhere | ✅ have it | Stoat emoji from any server you're in are usable anywhere — Nitro behavior, free. |
| Animated emoji | ✅ have it | Supported. |
| Profile customization (banner, bio, accent) | ✅ mostly | Profile background + bio exist. Missing vs Discord: per-profile accent color/theme, profile effects (🟡 cosmetic, low value). |
| Custom themes | ✅ better than Discord | Full Material-You theming + legacy custom themes. Discord charges for less. |
| HD streaming (1080p60 / 4K) | 🟡 **top priority** | Client currently clamps screenshare to **640×480@5fps** (`components/rtc/index.ts` monkey-patch) and camera to 720p (`state.tsx`). Remove/raise clamps, add a quality picker (existing TODO at `rtc/state.tsx:224` says tiers were meant to be server-driven), configure LiveKit for higher bitrate. This is the single most tangible "Nitro" win. |
| Super reactions | 🔴 | Cosmetic; skip (YAGNI) unless you really want it. |
| Custom sounds / soundboard | 🔴 | See §3. |
| Server boosts | N/A | Meaningless self-hosted — everything is already unlocked. |
| HD video backgrounds, clips, profile shop | skip | Cosmetic monetization surface, not utility. |

**Bottom line: cancel-Nitro parity ≈ Autumn/backend config + the streaming-quality work in this repo.** Everything else in this doc is about matching *free* Discord's feature breadth.

## 2. Free-Discord features Stoat already matches or beats

Text channels, categories, roles + granular permissions, DMs, group DMs, friends/blocking, mentions, replies, reactions, pins, message search, invites, bans, webhooks, bots, masquerade (better than Discord webhook-impersonation), settings sync, PWA/mobile web, reporting. Markdown is **richer than Discord**: real tables, headings, KaTeX math, spoilers, timestamps, syntax highlighting. Theming is far beyond Discord. That's the "e talvez até mais um pouco" part — already true for text chat.

## 3. Gap analysis vs Discord — the real work

### Tier A — client-only (this repo), backend already supports it
| Feature | Notes |
|---|---|
| Voice UX: per-user volume, hide non-video participants, ignore blocked users in voice | All LiveKit-client-side. Feature matrix lists them ❌. |
| Voice moderation: disconnect member, server-mute | Needs the LiveKit room-admin token path; mostly client + LiveKit API. |
| Screenshare/camera quality picker | The HD-streaming item from §1. |
| Desktop notifications (P0, in progress) + Web Push | Push controller + service worker already exist; finish and re-enable. |
| Update indicator (P0), in-app changelogs | Small. |
| Image preview/lightbox modal | Listed in `NOTES`; users notice its absence immediately. |
| Keybinds settings UI | Subsystem implemented, page commented out of nav. |
| GIF picker on your own infra | Un-hardcode `api.gifbox.me`; self-host gifbox or proxy Tenor. |
| Full member list, drafts persistence, upload progress % | Existing TODOs. |

### Tier B — desktop app (its own project, high daily-driver value)
Discord's desktop app is half the product: global push-to-talk, tray, autostart, native notifications, screenshare with audio. The client already probes `window.native` (Wayland virtual-mic path exists). Wrap this client in **Tauri** (Rust, matches the ecosystem; lighter than Electron) exposing: global keybinds (PTT), tray/minimize, autostart, native capture pickers. Feature matrix marks Desktop App P1 with ❌ across the board — nobody upstream is on it.

### Tier C — backend + client (fork/patch the Rust backend, weeks each)
Ordered by day-to-day impact for a small self-hosted community:

1. **Polls** — small data model, high utility.
2. **Voice messages** — record → upload to Autumn → new message flag + waveform player. Mostly reuses existing pieces.
3. **Threads** — biggest structural gap vs Discord. Large: new channel semantics, message routing, unread state.
4. **Forum channels** — builds on threads; do after.
5. **Scheduled events** — event object + RSVP + reminders.
6. **Soundboard** — audio assets on Autumn + LiveKit playback injection.
7. **Stickers** — emoji pipeline generalized to larger assets.
8. **Slash commands / interactions API** — only matters if you rely on Discord's bot ecosystem; Stoat bots are message-based today.
9. **AutoMod** — regex/wordlist filters server-side; a bot can cover this meanwhile (do the bot first, YAGNI on backend).
10. **Message forwarding, role icons, per-server profile banners** — small, low priority.
11. Not worth cloning: Activities (embedded games), Stages, server subscriptions, shop/cosmetics.

**Cost warning:** every Tier C item means maintaining a fork of `stoatchat/backend` (and often `stoat.js`). Before forking, check upstream's tracker — voice v2, e.g., is actively moving. Contributing upstream beats carrying patches.

### Where Stoat can go *beyond* Discord
E2EE DMs (on Stoat's own long-term roadmap; Discord text has nothing), full theming (done), real markdown (done), no message-history paywall, your own retention/limits, plugins (`plugins` experiment flag exists as a placeholder — a plugin API would leapfrog Discord).

## 4. Suggested execution order

**Phase 1 — "Cancel Nitro" (config + small client work, ~a weekend + a week of client work)**
1. Instance config: raise Autumn upload limits, message length, emoji limits.
2. Remove screenshare 640×480 clamp; add quality settings UI (1080p60 source cap); tune LiveKit bitrates.
3. Verify camera 720p → 1080p path.
   → At the end of this phase, Nitro is redundant. Cancel it here.

**Phase 2 — daily-driver polish (client-only, ~2–4 weeks of evenings)**
4. Desktop notifications + web push, update indicator.
5. Image lightbox, per-user volume, keybinds UI, member list, GIF picker on own infra.

**Phase 3 — desktop app (Tauri shell, ~2–3 weeks for v1)**
6. Tray, autostart, global PTT, native notifications.

**Phase 4 — structural features (backend forks, ongoing)**
7. Polls → voice messages → threads → events, in that order; upstream each if possible.

## 5. Immediate prerequisites (blocking everything)

1. `git submodule update --init packages/stoat.js packages/solid-livekit-components` + `mise install:frozen` + `mise build:deps` — the repo cannot build right now.
2. Confirm the self-hosted instance runs the LiveKit-based voice stack (voice v2) — Phase 1's streaming work assumes it.
3. Point `packages/client/.env` at the self-hosted API and verify login end-to-end (`mise dev`).
