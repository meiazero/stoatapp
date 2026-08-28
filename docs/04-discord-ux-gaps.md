# Discord UX Gaps — Small-Feature Deep Map

> The "little things" Discord has that shape daily UX: call sounds, ringing, notification behavior,
> status niceties, chat QoL. Mapped 2026-08-27 from four full-source audits of `for-web` (client +
> stoat.js + solid-livekit-components), `stoatchat`, and `for-desktop`.
> Companion: [../NITRO-PARITY-MAP.md](../NITRO-PARITY-MAP.md) (Nitro-locked features), [02-discord-parity-features.md](./02-discord-parity-features.md).
>
> **Parity rule:** desktop loads the remote web client, so every `web` item lands on both platforms
> automatically. Items tagged `web+desktop` need a native counterpart in `for-desktop`.
>
> Paths: `client/…` = `for-web/packages/client`, `stoat.js/…` = `for-web/packages/stoat.js`,
> `livekit/…` = `for-web/packages/solid-livekit-components`, `backend/…` = `stoatchat/crates`.

## TL;DR — ranked quick wins

| # | Gap | Why it's cheap | Size | Tag |
|---|---|---|---|---|
| 1 | DM/group call **ringing** | SDK already takes `recipients` (`stoat.js/src/classes/Channel.ts:824-838`); client drops it at `client/components/rtc/state.tsx:321`. Both ringtone `.ogg` assets shipped with zero callers. Needs: pass arg + incoming-call modal + ring event case | S-M | web |
| 2 | **App badge count** (unread on icon) | Electron side fully built, **zero callers** (`for-desktop/src/native/badges.ts:11`, preload `src/world/window.ts:17`); count already computed at `client/src/interface/navigation/servers/ServerList.tsx:141`. Web: `navigator.setAppBadge` + `document.title` | S | web+desktop |
| 3 | **Silent message** UI | `@silent ` prefix already works end-to-end (SDK sets `MessageFlags.SuppressNotifications`); just no composer toggle. Bell-off button in `MessageBox.tsx` → flags in `Composition.tsx:183` | S | web |
| 4 | **Spoiler toggle on upload** | Rendering exists; flag is filename convention `spoiler_` (`stoat.js/src/classes/File.ts:115-117`). Add per-file toggle in `FileCarousel.tsx` that prefixes the name in `Draft.ts` | S | web |
| 5 | **Recent/frequent emoji** in picker | Usage-count store + leading section in `client/…/picker/EmojiPicker.tsx:126`. Zero occurrences of recent/frequent today | S-M | web |
| 6 | **Auto-idle** | Client never auto-sets presence (`setPresence` has one call site: the manual menu). Idle timer store → `user.edit({status})`; desktop can add `powerMonitor.getSystemIdleTime()` | S | web (+desktop optional) |
| 7 | **Quick switcher (Ctrl+K)** | Nothing exists. New modal + `KeybindAction` entry; data already in `client()`/`ordering` stores | M | web |
| 8 | **Message forwarding** | No backend route needed: compose-and-resend from a channel-picker modal, entry at `MessageContextMenu.tsx:316` | M | web |
| 9 | Voice **toasts** (moved/disconnected/stream started) | Snackbar system exists, zero voice usage; hook next to `playSound("streamStart")` at `rtc/state.tsx:295` and `disconnected` listener `:269` | S | web |
| 10 | **Suppress @everyone/@online** per server | Client-side filter in `NotificationsWorker.tsx:57` + field in `NotificationOptions.ts:39` + menu entry | S | web |

**Structural (not quick, highest impact):** notification preferences are a **client-only opaque sync
blob** — mutes/levels/DND silence the browser tab but **not push/mobile/second clients**. See §2.

---

# 1 — Voice & calls

### Exists (verified working)
- **Sound effects**: full `SoundController`, 14 `.ogg` assets, per-sound toggles + preview in settings
  (`client/components/client/Sounds.tsx:22-141`, store `state/stores/Sounds.ts`, settings UI
  `settings/user/notifications/Sounds.tsx`). Wired: join/leave call, others join/leave, mute/unmute,
  deafen/undeafen, stream start/end (`rtc/state.tsx:266,348,282,286,392-394,368-370,295,306`).
- **Deafen** distinct from mute, transported as `is_receiving`, `DeafenMembers` permission exists.
- **Per-user volume** 0–300% + separate screen-share volume, in user context menu
  (`UserContextMenu.tsx:285-385`), persisted, applied in `RoomAudioManager.tsx:51-63`.
- **Device selection** (in/out/camera) with live switching and `setSinkId` output routing
  (`settings/user/voice/VoiceInputOptions.tsx:26-108`), global in/out volume sliders.
- **Noise suppression** 3-way (off/browser/RNNoise) + echo cancel + AGC checkboxes, applied live
  (`settings/user/voice/VoiceProcessingOptions.tsx:18-46`, `VoiceProcessor.ts:93-131`).
- **Speaking indicator** (primary-color ring) on tiles, PiP, and channel preview.

### Gaps
| Gap | State | Fix sketch | Size |
|---|---|---|---|
| **Call ringing** | Backend+SDK ready, client never uses it. No incoming-call UI, no outgoing "Calling…", ringtone assets dead | Pass `recipients` at `rtc/state.tsx:321`; add ring event case in `stoat.js/src/events/v1.ts:990` switch; incoming-call modal via `modals.openModal` (injected at `rtc/state.tsx:150`); play `ringtoneIncoming/Outgoing` | S-M |
| **5 dead sounds** | `ringtoneIncoming/Outgoing`, `userMoved`, `streamViewerJoin/Leave`: assets + cases exist, zero callers; also missing from settings UI | Callers: ring (above); moved → stubbed SDK events `v1.ts:1008,1023` (both `// todo`); viewer join/leave → `trackPublished/Unpublished` on own screen-share pub | S |
| **"You were moved/disconnected" toasts** | Snackbar system exists (`useSnackbar`), zero voice usage | Hook `disconnected` listener `rtc/state.tsx:269` + the two stubbed move events | S |
| **Reconnecting state** | `RECONNECTING` variant declared (`rtc/state.tsx:49`) but never set — no `reconnecting`/`reconnected` listeners | Register both listeners next to `:250,269` | S |
| **Connection quality** | Only state pill; per-participant quality is an explicit TODO (`livekit/src/signals/index.ts:5`) | Implement `useConnectionQualityIndicator` from LiveKit's `ConnectionQualityChanged` | S-M |
| **Push-to-talk** | Nothing. Keybinds engine works but store is a phantom stub, settings page commented out, zero voice actions, no keyup modeling | See §5 keybinds. In-app PTT: new actions + keyup support in `keybindHandler.tsx` + `toggleMute()`. Global PTT: Zig N-API addon (NITRO-PARITY-MAP Part 5.3) via existing `window.native` bridge | M (in-app) / L (global) | 
| **Input sensitivity / VAD** | Nothing (no gate, no threshold setting) | `inputSensitivity` in `stores/Voice.ts:43`; AnalyserNode+gate in `VoiceProcessor.ts:133-160` chain; slider in `VoiceProcessingOptions.tsx:47` | M |
| **Camera/mic preview** | None — join is immediate, no `PreJoin` component in livekit pkg | Settings: `getUserMedia` preview under device selects (`VoiceInputOptions.tsx:32`). Pre-join: modal wrapping connect at `VoiceCallCardPreview.tsx:38` | M |
| **Per-call quick toggle for noise suppression** | Settings-only today | Button in `VoiceCallCardActions.tsx` | S |

PTT global is the only `web+desktop` item here; everything else is `web`.

---

# 2 — Notifications

### Exists (verified working)
- **Per-server/per-channel levels** (all/mention/none, inheritance channel→server→default) + **mute
  with durations** (15m/1h/3h/8h/24h/forever, expiry enforced): store
  `client/components/state/stores/NotificationOptions.ts`, menus `NotificationContextMenu.tsx`,
  `ServerContextMenu.tsx:204-270`, enforced in `NotificationsWorker.tsx:50-58`.
- **Desktop notifications + web push**: full loop — permission flow, `pushManager.subscribe` with
  VAPID, service worker `push`/`notificationclick`, backend `pushd` daemon (vapid/fcm/apn).
- **Unread system**: channel highlight, per-channel mention badges, per-server mention counts, unread
  DM pills, mark read (channel/category/server/keybind/auto-ack), mark unread, "NEW" divider,
  jump-to-last-read bar.
- **DND/Focus are real client-side**: `Busy` suppresses all, `Focus` non-mentions, sounds included
  (`NotificationsWorker.tsx:61-65`).

### Gaps
| Gap | State | Fix sketch | Size |
|---|---|---|---|
| **Prefs are client-only** ⚠️ | The single biggest structural gap. `/sync/settings/set` stores an opaque blob; backend has **no notification-preference model**; push fan-out filters only on online-ness. **Muting a server silences the tab but not the phone.** DND user who closes the tab still gets push (`backend/core/database/src/amqp/amqp.rs:191-194`) | Real model in `core/database/src/models/user_settings/` + check in `pushd/src/consumers/inbound/message.rs:53` (per-session loop). Vestige confirms original intent: commented `user_settings[notifications]` in `stoat.js/src/events/EventClient.ts:174` | L, backend |
| **Suppress @everyone/@online** | `MentionsEveryone/Online` flags exist and fan out; no suppression pref anywhere | Field in `NotificationOptions.ts:39` + filter `NotificationsWorker.tsx:57` + entry in `ServerContextMenu.tsx:232` | S |
| **App icon / favicon badge** | Nothing on web; Electron `setBadgeCount` fully built with zero callers, not even declared in web's `window.native` typing | Worker beside `<NotificationsWorker/>` (`src/Interface.tsx:136`): `navigator.setAppBadge` + `document.title` + call the native bridge | S |
| **Inbox / recent mentions** | No route, no component. Data already exists both sides (`Channel.mentions`, backend `fetch_unread_mentions` — used only for APNs badge today) | `<Route path="/inbox">` at `Content.tsx:44`; button near `ServerList.tsx:141`; optional delta route later | M |
| **Notification overrides page** | All config is context-menu-only; settings page has just 2 toggles + sounds. No per-server override list (Discord has one) | New page under `settings/user/notifications/`, data already keyed by id in `NotificationOptions.ts:43-58` | S-M |
| **Sound customization** | Single `message` sound, hardcoded assets, no volume slider, no mention-vs-message distinction. Sound gated behind desktop-notification permission — denying notifications kills sounds too (`NotificationsWorker.tsx:196-202`) | Split the gate; per-event asset override in `Sounds.tsx:79-136`; volume field in `Sounds.ts` store | S-M |
| **In-app silent-flag check** | `Message.isSuppressed` exists; `NotificationsWorker.onMessage` never checks it (backend push does honor it) | One condition at `NotificationsWorker.tsx:37-66` | S |
| **Muted DMs still counted in rail** | Explicit `// TODO: muting channels` at `client/src/interface/Sidebar.tsx:55-59` | Honor `isChannelMuted` there | S |

---

# 3 — Presence, status, profile

### Exists (verified working)
- **Custom status text** (128 chars, full model→API→UI loop) — but see gaps for reachability.
- **5 presence states** (Online/Idle/Focus/Busy/Invisible), invisible enforced server-side, Busy/Focus
  honored by the notification worker.
- **Per-server nickname AND per-server avatar** — self-set, with `ChangeNickname`/`ChangeAvatar`
  permissions, UI in `ServerIdentity.tsx`. *Discord charges Nitro for server profiles; Stoat has the
  core of it free.*
- **Profile editor**: avatar, display name, pronouns, bio, banner (`UserProfileEditor.tsx:162-215`).
- **Status text renders** in member sidebar + DM list with emoji shortcodes + presence dot.

### Gaps
| Gap | State | Fix sketch | Size |
|---|---|---|---|
| **Auto-idle** | Client never auto-sets presence; no idle/visibility tracking anywhere | Idle-timer store driving `user.edit({status:{presence}})`; desktop: `powerMonitor.getSystemIdleTime()` via preload | S |
| **Status emoji + expiry** | No emoji field/picker (shortcodes render in some surfaces, not `ProfileStatus.tsx:21-32` / `UserMenu.tsx:242-244` — inconsistent); no "clear after 1h" | Backend: `emoji` + `clear_after` on `UserStatus` (`backend/core/models/src/v0/users.rs:147`) + daemon sweep; picker + duration dropdown in `CustomStatus.tsx:61-69`; fix the two non-emoji render surfaces | M, backend |
| **Status editor reachability** | `CustomStatusModal` reachable only from server-list user menu; not in Settings profile editor | Add status field to `UserProfileEditor.tsx` | S |
| **Streamer mode** | Zero hits anywhere; no privacy settings group | `"privacy:streamer_mode"` key in `Settings.ts:34-82` + consume at invite/email/ID render sites | M |
| **Friend nicknames** | `Relationship` is `{user_id, status}` only | Extend model + fallback in `stoat.js` `displayName` (`User.ts:82-87`) | S-M, backend |
| **User notes** | Nothing (no field, no collection, no UI) | New `user_notes` model + route; `Profile.Note` card in profile popup | S-M, backend |
| **Profile accent color** | `UserProfile` is `{content, background}` only | `accent` field reusing `RE_COLOUR` validator; render in `ProfileBanner.tsx` | S, backend |
| **Activity / "Playing X"** | No model at all. Desktop's Discord RPC only *advertises Stoat to Discord* (hardcoded static activity, `for-desktop/src/native/discordRpc.ts:17-30`) | `activities: Vec<Activity>` on `User` + bonfire fan-out; render slot exists next to status in `MemberSidebar.tsx:389-404`. Pairs with the open Presence API idea (suggestion-of-chatgpt.md §9) | L, backend |

---

# 4 — Chat & navigation QoL

### Exists (verified working)
- **Typing indicator** (send+debounce+render, filters self/blocked).
- **Composer autocomplete**: emoji `:`, user `@`, role `%`, channel `#` (skipped in code blocks).
- **GIF picker** (Gifbox-backed, configurable endpoint) — note stale experiment flag, see landmines.
- **Drafts per channel** persisted to IndexedDB incl. outbox of unsent messages (not files — TODO in
  `Draft.ts`).
- **Hover-to-react toolbar** (reply/react/edit/delete, permission-gated) + inline "+" chip.
- **Server + category + channel drag reorder** (disabled on mobile, TODOs noted).
- **Text spoilers** `||…||` with live composer decoration and click-to-reveal; attachment spoiler
  overlay renders.

### Gaps
| Gap | State | Fix sketch | Size |
|---|---|---|---|
| **Quick switcher (Ctrl+K)** | Nothing; message search exists but doesn't list channels/servers | `KeybindAction.OPEN_QUICK_SWITCHER` + sequence + modal in `modal/types.ts:30` fed by existing stores | M |
| **Message forwarding** | No action, no backend route | Client-only: channel-picker modal → `channel.sendMessage()` with copied content+attachment URLs; entry at `MessageContextMenu.tsx:316` | M |
| **Server folders** | Flat `servers: string[]` in `Ordering.ts:7-12`; nothing in backend models | Tree in `TypeOrdering` + group nodes in `ServerList.tsx:213` (client-side, syncs via settings blob) | M |
| **Silent-message toggle** | `@silent ` prefix works end-to-end incl. badge render; zero discoverability | Bell-off toggle in `MessageBox.tsx` setting flags in `Composition.tsx:183` | S |
| **Image spoiler toggle** | Flag = filename convention `spoiler_` only | Per-file toggle in `FileCarousel.tsx` prefixing name in `Draft.ts` | S |
| **Recent/frequent emoji** | Zero occurrences client-wide | Usage store + leading section `EmojiPicker.tsx:126` | S-M |
| **Suppress embeds per message** | Backend-blocked: `MessageFlags` has only `SuppressNotifications = 1` | `SuppressEmbeds = 2` in Rust enum + PATCH handling + mirror in `stoat.js/src/hydration/message.ts:88` + context-menu toggle | S-M, backend |
| **Keybinds resurrection** | Worse than roadmap says: settings page commented out (`UserSettings.tsx:292-294`) AND store gutted to `_phantom` stub (`Keybinds.ts:50-52`) — defaults read from a module constant, rebinding impossible. 12 actions exist, no Ctrl bindings, no voice actions, no keyup modeling | Restore store + `KeyComboSequence` parser + settings page; then PTT/mute/deafen actions (§1) | M-L |
| **Message scheduling** | Nothing; needs backend queue | Pairs with crond daemon; see suggestion-of-chatgpt.md §8 | L, backend |
| **TTS + a11y** | Zero `speechSynthesis`/`aria-live` hits; message list has **no aria roles at all**; Accessibility settings page is an empty shell | Minimum bar: `aria-live="polite"` + `role="log"` on `Messages.tsx` list, `role="article"` per message in `Container.tsx:338`. TTS optional later | S (a11y floor) |

---

# 5 — Landmines found during these audits

- `Experiments.ts:31-34` still advertises `gif_picker` as "Not available yet" while it ships
  unconditionally — stale flag, confusing in settings.
- Notification sound requires desktop-notification permission (gate at
  `NotificationsWorker.tsx:196-202`) — denying one kills the other.
- Stubbed SDK events `VoiceChannelMove` / `UserMoveVoiceChannel` (`stoat.js/src/events/v1.ts:1008,1023`,
  both `// todo`) block moved-user UX and the already-shipped `user_moved.ogg`.
- Status emoji rendering inconsistent across surfaces (renders in sidebars, raw text in profile card
  and user menu).
- Muted DMs still count in the unread rail (`Sidebar.tsx:55-59` TODO).

# 6 — Suggested execution order

**Wave 1 — pure client, each ≤ a day:** silent toggle, spoiler toggle, badge count (web+desktop),
suppress-@everyone, in-app silent-flag check, muted-DM rail fix, voice toasts + reconnecting state,
auto-idle, status editor in settings.

**Wave 2 — small features:** call ringing (+ dead sounds), recent emoji, quick switcher, forwarding,
notification overrides page + sound customization, a11y floor.

**Wave 3 — backend-touching:** status emoji/expiry, accent color, friend nicknames, user notes,
suppress-embeds flag, **server-side notification preferences** (the structural one — do it before
mobile/push matters), inbox view with delta route.

**Wave 4 — big:** keybinds resurrection → in-app PTT → global PTT (Zig), camera/mic preview + VAD,
activity/presence API, server folders, streamer mode, scheduling.
