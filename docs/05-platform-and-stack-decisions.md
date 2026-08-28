# Platform & Stack Decisions — Measured Verdicts

> Answers four questions with measured evidence (three source audits + current-web research, 2026-08-27):
> desktop shell (Electron vs Electrobun vs Tauri vs native), stoat.js / solid-livekit-components
> (rewrite vs improve), PandaCSS → StyleX, and ESLint/Prettier → oxc.
> Context: desktop app is the priority, web is the clone, UX above all. Origin: `meiazero/stoatapp`
> (detached from the fork network; no upstream remote).
> Companions: [../NITRO-PARITY-MAP.md](../NITRO-PARITY-MAP.md), [04-discord-ux-gaps.md](./04-discord-ux-gaps.md).

## Verdict summary

| Axis | Verdict | One-line reason |
|---|---|---|
| Desktop shell | **Keep Electron** (Vesktop blueprint); native GStreamer+LiveKit is the long-term escape | Every non-Chromium shell on Linux = WebKitGTK, and distro WebKitGTK ships with WebRTC **compiled out** — LiveKit cannot even feature-detect |
| stoat.js | **Keep and fix hot paths** — do not rewrite | It IS revolt.js (+~14% divergence); reactivity is Solid at the data-model layer, so "another lib" = rewriting the data model, for zero UX gain |
| solid-livekit-components | **Fix the adapter, then extend or absorb** | 1,063 LOC, 18% hook coverage, and a systematic port bug that **leaks every RxJS subscription** |
| PandaCSS → StyleX | **Don't migrate** | Panda config has zero tokens/recipes to port; measured runtime cost ~6 calls/message. The perf problems are elsewhere (see §5) |
| oxc (oxlint+oxfmt) | **Adopt oxlint now, oxfmt when it exits beta** | oxlint 1.80 stable + type-aware; `eslint-plugin-solid` runs via alpha jsPlugins; oxfmt 0.65 beta lacks unused-import removal |

The single highest-ROI action in this entire document is **one line**: `src/index.tsx:55` ships the
Solid devtools overlay **in production** (§5.1).

---

# 1 — Desktop shell

**Decisive fact:** distro builds of WebKitGTK (Ubuntu, Debian, Fedora, Arch) compile with
`ENABLE_WEB_RTC` off — `RTCPeerConnection` does not exist, so LiveKit fails at feature detection
(verified in distro build files + tauri#13143). Separately, screenshare-with-audio is unsupported by
every engine's `getDisplayMedia` on Linux; Chromium apps solve it with a PipeWire virtual-mic native
module ([venmic](https://github.com/Vencord/venmic), used by Vesktop, works with "any electron app").

| Option | Can it ship this app on Linux (X11+Wayland) today? | Detail |
|---|---|---|
| **Electron 44** (Aug 2026) | **Yes — only proven path** | Full Chromium WebRTC, Wayland screenshare via PipeWire portal (default since M110), `setDisplayMediaRequestHandler` + `useSystemPicker`; audio share via venmic. Everyone with working voice ships this (Vesktop, official Discord, Slack, Element). Cost: ~250-400 MB disk, 300-500 MB RAM |
| **Electrobun v2** (Aug 2026) | Risky | Active but bus-factor 1; WebRTC requires its optional CEF bundle (~100 MB) which on Linux is X11-only (XWayland), weeks old, open HiDPI/resize bugs; a Feb 2026 field report abandoned it for Electron. No LiveKit app exists on it |
| **Tauri v2** | **No** | Stock WebKitGTK: no `RTCPeerConnection` at all. Hopp (Tauri+LiveKit) publicly gave up on Linux (Jan 2026); Dorion ships "voice not supported on Linux". Verso/Servo has no WebRTC; official CEF backend in private dogfooding — re-check in 6-12 months |
| Wails v3 / Neutralino / Photino / Pake | No | All WebKitGTK underneath |
| **Truly native** (LiveKit Rust SDK + GStreamer) | Possible — it's a whole project | livekit rust-sdks 0.8.4: solid transport + WebRTC APM, but **no capture/render** — you push raw frames. GStreamer has first-class `livekitwebrtcsink/src` elements, and `pipewiresrc` is the only stack where Wayland screenshare-with-audio is a *solved, packaged* problem. Precedent scale: Telegram Desktop is the only full native voice+video+screenshare client, built by a dedicated team. LiveKit Flutter: works on X11, Wayland picker buggy |

### Decision
1. **Now:** stay on Electron; invest the "shell budget" in the Vesktop blueprint — venmic for
   screenshare audio, portal picker on Wayland, native badge/tray already built (doc 04 §2).
   Known trap: `globalShortcut` broken on GNOME 50 / xdg-desktop-portal ≥ 1.20 (electron#51875) —
   PTT needs the Zig native hook (NITRO-PARITY-MAP Part 5.3) or a compositor keybind → IPC fallback.
2. **Desktop-first UX without changing shells:** the wins are in the web client's perf (§5) and the
   native integrations, not in the shell choice. Electron is not what makes the app feel slow today.
3. **Long-term native track (optional, ongoing):** prototype = LiveKit Rust crate for room state +
   GStreamer for capture/render. This is also the only credible Wayland-native path. Re-evaluate
   Tauri when its CEF backend ships publicly.

---

# 2 — stoat.js: rewrite vs improve

### Measured anatomy
- 56 files / **9,182 LOC**, 7 runtime deps, **zero tests**. Largest: `events/v1.ts` 1,038, `Channel.ts` 900, `Server.ts` 855.
- **It is revolt.js**: full history back to 2020, renamed Oct 2025; divergence since = 58 commits,
  +1,258/−339 LOC (~14%), concentrated in voice states/slowmode/audit-log. Upstream fixes remain mergeable.
- **Solid coupling is structural, not cosmetic**: collections are `createStore` (`storage/ObjectStorage.ts:17`),
  class getters read through the store (property access *is* the signal read), hydrated shapes embed
  `ReactiveMap`/`ReactiveSet`, writes wrapped in `batch()`. 20/56 files import solid-js. There is no
  adapter seam to swap.
- Client contract: 112 import sites, 48 symbols, but really ~15 classes; `.displayName` (487 sites) and
  `.havePermission` (74) are the true API.
- Type safety is fine (3 `any` total); the `as never` idiom (21×) is a workaround for `stoat-api` route typing.

### Verdict: rewriting to "another lib" buys nothing
A non-Solid rewrite means replacing the data-model layer and re-touching 112 client import sites — for a
client that is itself Solid. The SDK's problems are hot-path bugs, not architecture. **Fix these instead**
(ranked; each is small and upstreamable):

| # | Fix | Where | Why |
|---|---|---|---|
| 1 | Age-unit bug `3600_0000` → `3_600_000` | `src/classes/User.ts:366-370` | Everyone stuck on new_user limits ~30 days (already in NITRO-PARITY-MAP Phase 1) |
| 2 | Hydration is **O(k²)**: `reduce((acc,k)=>({...acc,…}))` spreads the whole object per key, unknown keys throw+log | `src/hydration/index.ts:45-66` | Runs for every object on `Ready` and every message. Mutate one `acc` object; drop try/catch-per-key |
| 3 | **Unbounded stores**: zero eviction anywhere — every message/user/member ever seen stays in the ReactiveMap forever | `collections/MessageCollection.ts` | Long sessions bloat; add LRU/eviction keyed to channel focus |
| 4 | Unmemoized full re-derivation getters: `orderedChannels` rebuilds per read; `ServerMember` does **4 full sorts per member render** (`orderedRoles`+`hoistedRole`+`roleColour`+`roleIcon`) | `classes/Server.ts:211-301`, `ServerMember.ts:123-156` | Member list renders pay this per row. Wrap in `createMemo` at construction |
| 5 | `Bulk` events: async `handleEvent` not awaited → ordering not guaranteed, rejections unhandled | `events/v1.ts:286-291` | Correctness under `Ready`-in-`Bulk` |
| 6 | Client-side `toSorted((a,b)=>b.id.localeCompare(a.id))` — full Intl collation resort of an already-sorted ULID list, on every page append (5 sites) | `client/…/Messages.tsx:186` | Plain `<` comparison + merge instead |
| 7 | Stubbed events: `Auth` session revocation silently ignored (security), `VoiceChannelMove`/`UserMoveVoiceChannel` empty, voice joins don't emit client events, server delete leaks members/emoji | `events/v1.ts:988,1011,1023,992,1003`; `Server.ts:487` | Blocks doc-04 voice UX; the Auth one matters regardless |

Also: `stoat.js` is declared twice in client package.json (dev + prod deps); dedupe.

---

# 3 — solid-livekit-components: fix, then extend or absorb

### Measured anatomy
- **1,063 LOC / 20 files**, 18 commits in 2.4 years, zero tests. Mechanical port of
  `@livekit/components-react` **2.0.4** (provenance headers intact) — all real logic lives in
  `@livekit/components-core` (RxJS), adapted to Solid by one 20-line `useObservableState`.
- Coverage: **8 of 44 hooks (18%)**, 3 of ~40 components; prefabs (`PreJoin`, `ControlBar`,
  `VideoConference`) never started — `signals/index.ts:1-38` is literally a 36-hook TODO list.
- The client already routed around it: 7 direct `livekit-client` import sites and **~3,181 LOC of its
  own voice UI** (`components/rtc/` + `features/voice/`) sitting on a 1,063 LOC library. It uses 14
  of the 26 exports.
- Submodule remote is **`revoltchat/solid-livekit-components`** — not owned by stoatchat, third
  repo name in package.json. You don't control this upstream.

### The port carries a systematic correctness bug — fix before anything else
React idioms pasted into Solid:
1. **Every RxJS subscription leaks.** `return () => subscription.unsubscribe()` inside `createEffect`
   is not a cleanup in Solid (return value feeds the next run). 5 sites: `internal/useObservableState.ts:17`,
   `useTracks.ts:80`, `useMediaTrackBySourceOrName.ts:50,66`, `useIsMuted.ts:46`. Since
   `useObservableState` underlies most signals, subscriptions accumulate per participant, per track,
   per channel switch, for the life of the page. Fix = `onCleanup(() => sub.unsubscribe())` (4 correct
   examples already exist in the package).
2. **React dependency arrays passed as `createEffect`'s 2nd arg** (silent no-op; `JSON.stringify(sources)`
   at `useTracks.ts:81` runs every execution for nothing). 4 sites.

This leak plausibly relates to the workspace pin comment "solid-js 1.9.6→1.9.14 breaks reactivity
across dependencies" (`pnpm-workspace.yaml`).

### Verdict
1. **Week one:** fix the 9 adapter sites (leak + dep arrays). Small, mechanical, testable.
2. **Then choose:**
   - **Absorb into the client** (recommended given you now own everything under `meiazero/stoatapp`
     and the upstream is a 18-commit repo you don't control): move the 20 files into
     `packages/client/lib/livekit/`, drop the submodule. −1 repo, −1 build step, same code.
   - Or keep as package and extend mechanically: missing observable-backed hooks are ~20-40 LOC each
     (`useObservableState(coreObservable(...), initial)` + context read); `useConnectionQualityIndicator`
     (doc 04) is exactly this shape. Prefabs are full rewrites either way — but the client already
     wrote its own prefab layer, so that's fine.
3. Landmine either way: `patches/livekit-client.patch` is a **4.6 MB patch against the shipped bundle**
   (adds `applyScreenShareConstraints`) — must be regenerated on every livekit-client bump. Consider
   upstreaming that API to livekit-client or reimplementing via public constraint APIs.

---

# 4 — PandaCSS → StyleX: don't

Measured footprint (client: 403 files / 57,777 LOC):
- 157 files (39%) import styled-system; **469 call sites** (`styled` 368, `cva` 43, `css` 58), 89 variant blocks.
- **panda.config defines essentially nothing**: 5 keyframes. Zero tokens, semanticTokens, recipes,
  patterns, utilities. Panda is a pure atomic-CSS compiler over raw `var(--*)` strings.
- Theming is 100% decoupled from Panda: ~254 CSS custom properties written imperatively to
  `document.body` per theme change (`LoadTheme.tsx:60-68`); components hardcode
  `"var(--md-sys-color-…)"` literals. **A StyleX migration has zero token-porting benefit because
  there is no token layer.**
- Runtime cost measured: ~6 cva/css calls per message + one genuinely dynamic `css()` merge in
  `Symbol.tsx:57-68` (124 usages). Real but small next to §5.

**Cost:** rewriting 469 call sites + 89 variant blocks across 157 files, StyleX has no `cva`-style
variants primitive (manual composition), and `jsxFramework: "solid"` styled-JSX would need rebuilding.
**Benefit:** ~nothing measurable. If styling work has budget, spend it on: deleting the legacy Revolt
theme shim (118 `--colours-*` vars, `legacyThemeGeneratorCode.ts` 13.6K), making `Symbol` static,
and DEV-gating `Inspect()`/`devtools()` in vite.config (§5).

---

# 5 — What actually makes the UX fast (ranked, measured)

The audits found where the frame time really goes. In order of blast radius:

1. **Devtools instrumented in production** — `attachDevtoolsOverlay()` unguarded at
   `client/src/index.tsx:55`; `devtools()` + `Inspect()` unconditional in `vite.config.ts:19-27`;
   `@solid-devtools/*` in prod dependencies. The whole reactive graph is instrumented for every user.
   **Fix: 1 line + 2 config gates + move deps.** Likely the single largest runtime cost in the app.
2. **Floating-elements registry is O(n²) + leaks listeners** — `directives/floating.ts:31,38`
   copies/filters a global array per register/unregister; 9 registrations per message → ~1,350 entries
   at the 150-message cap; `FloatingManager.tsx:59` re-reconciles the full array on every mount/unmount;
   `floating.ts:190-191` re-`addEventListener`s touch handlers inside `onCleanup` instead of removing
   (2 leaked listeners per unmount). Fix: keyed map + fix the two lines.
3. **Markdown: 16-plugin unified pipeline runs synchronously per message, uncached** —
   `markdown/index.tsx:365,393`; edits re-run the whole pipeline; `rehypeHighlight` registers all
   ~190 highlight.js grammars up front (`:270-272`); KaTeX CSS eager. 150 full parses on first paint,
   +1 per reply. Fix: memoize per message id+content, lazy-register grammars on first code block,
   defer KaTeX CSS. (Zig/WASM pre-scanner from NITRO-PARITY-MAP Part 5.6 only *after* this.)
4. **No virtualization in the message list** — plain `<For>` capped at 150 (`Messages.tsx:55,938`);
   `ListView2` is infinite-scroll, not a virtualizer. Each row eagerly renders a full
   `CompositionMediaPicker` (emoji+GIF picker per message! `Message.tsx:335`), `MessageToolbar`
   hidden by CSS, `Reactions` even when empty, SVG avatar with `foreignObject`, up to 6 tooltips.
   `@minht11/solid-virtual-container` is already a dependency (used in Friends/MemberSidebar). Fix
   order: lift the per-message picker to a single shared floating instance (biggest DOM cut), then
   virtualize.
5. **SDK hot paths** — §2 items 2/3/4/6 (O(k²) hydration, unbounded stores, unmemoized getters,
   localeCompare resort).
6. **Theme apply churn** — 254 `setProperty` calls in one effect + O(n²) reduce-spread rebuild
   (`LoadTheme.tsx:36-68`). Only matters on theme change; fix opportunistically.
7. **Images** — 11 of 19 `<img>` sites eager, zero `decoding="async"`; emoji are individual remote
   SVGs (lazy at least). Cheap sweep.

Items 1, 2, and the `Symbol` fix are a day of work combined and will do more for perceived UX than
any shell/framework migration in this document.

---

# 6 — Toolchain: oxc

Current: ESLint 9 flat config + Prettier + `eslint-plugin-solid` + `eslint-config-prettier` +
`prettier-plugin-organize-imports`, duplicated at root and in `stoat.js`.

As of Aug 2026: **oxlint 1.80 stable** (type-aware stable since Jul 2026 via tsgolint);
`eslint-plugin-solid` runs under the **alpha** jsPlugins layer (oxc discussion #19936); **oxfmt 0.65
beta** — 100% Prettier JS/TS conformance + built-in import sorting, but does **not** remove unused
imports or sort named specifiers.

**Verdict: partial today.**
1. **Now:** adopt oxlint as the primary linter (native rules + type-aware). Keep a minimal ESLint
   pass *only* for `eslint-plugin-solid` until jsPlugins leaves alpha — Solid reactivity rules are
   exactly the ones that catch real bugs here (§3's `createEffect` misuse is their bread and butter;
   note the current ESLint setup didn't catch those in the submodule — make sure the lint config
   actually covers `packages/solid-livekit-components`).
2. **Formatter:** stay on Prettier until oxfmt is stable *or* accept beta now — conformance is 100%
   for TS and import sorting is built in; the gap (unused-import removal) is a lint fix
   (`noUnusedImports`) anyway. Low risk either way; don't run both.
3. Unify the duplicated configs at the workspace root while migrating (stoat.js's own
   eslint/prettier copies exist because of "ancient prettiers" in submodules — absorbing the
   livekit package (§3) removes one of those).

---

# 7 — Execution order (all axes)

**Phase A — free UX (days):** devtools guard + vite gating; floating registry fix + listener leak;
`Symbol` static; livekit adapter leak fix (9 sites); SDK age bug; `localeCompare` sort.
**Phase B — perf structure (1-2 weeks):** markdown memoization + lazy grammars; shared reaction
picker; virtualized message list; SDK hydration/getter/eviction fixes.
**Phase C — shell investment (parallel):** venmic integration; Zig PTT (portal-proof); badge/tray
wiring (doc 04 quick win #2).
**Phase D — structural (when justified):** absorb solid-livekit-components; oxfmt when stable;
Tauri-CEF re-check (~2027); native GStreamer+LiveKit prototype as the long-term Wayland-native bet.

---

# 8 — Router (web) and native-client framework (desktop)

## 8.1 TanStack Router for Solid: yes it exists, but **wait**

Researched 2026-08-27. `@tanstack/solid-router` v1 is stable (since Jan 2025, now 1.170.30) with high
parity: file-based routing, typed params/search params, loaders, preloading, auto code splitting,
devtools. Works in a plain Vite SPA with the Solid 1.9.x pin. Hooks return `Accessor`s (`params()`).

Facts for this codebase: client uses `@solidjs/router ^0.15.3` at only **12 import sites**, and
`@tanstack/solid-query ^5.76` is already a dependency. Migration ≈ 1 day (mostly the route tree; no
official migration guide exists).

**Decision: wait.** TanStack Router **v2 (for Solid 2.0) is in RC** (announced Apr 2026, explicitly
not production-ready) — migrating to v1 now likely means a second migration when Solid 2.0 lands.
Adoption is real (~102k dl/wk ≈ 27% of @solidjs/router) but @solidjs/router remains the community
default and causes no pain here. Revisit alongside the Solid 2.0 upgrade.

## 8.2 Native desktop client — targets: **Linux + Windows** (no macOS)

Full research 2026-08-27 with the make-or-break lens: can it do LiveKit voice + camera +
screenshare-with-audio, and render video frames efficiently (GL/zero-copy)?

| Framework | LiveKit path | Linux (X11/Wayland) | Windows | Verdict |
|---|---|---|---|---|
| **Flutter** | **Official full SDK** (`livekit_client` 2.8.1 via flutter-webrtc: voice+camera+screenshare) | GTK embedder = native Wayland+X11; Canonical became lead steward of Flutter desktop (I/O 2026). Known bug: Wayland picker ignores portal (flutter-webrtc#1542, contained fix — delegate to xdg portal) | First-class; flutter-webrtc supports Windows capture | **Yes — only stack where calls exist end-to-end today** |
| **Qt 6/QML** (C++ or cxx-qt) | Build it on GStreamer: `livekitwebrtcsink/src` + `pipewiresrc`+portal capture + `qml6glsink` render | Best Wayland client in class; **true zero-copy** (qml6glsink + DMA-BUF) | First-class | **Yes — most engineering, most control** (Telegram precedent) |
| **GPUI** (Zed's, Rust) | **Proven in prod**: Zed runs LiveKit Rust SDK on Linux — voice + screenshare view, X11 capture via scap, Wayland issue closed; camera never exercised (add via GStreamer → `capture_frame`) | wgpu renderer since Feb 2026 | Zed Windows shipped; GPUI Windows newer, less battle-tested | Risky-yes — the proven Rust+LiveKit path; pre-1.0, thin docs |
| **Iced** 0.14 (Rust) | DIY: livekit Rust crate + GStreamer capture; video = CPU→wgpu texture copy (not zero-copy) | Engine of COSMIC/Pop!_OS — perfect distro alignment; a11y weak | Works | Risky-yes — stable API, more assembly |
| Slint | livekit crate + GStreamer; GL import via EGL (official example) | Good | Good | Risky — widget ecosystem too thin for Discord-class UI |
| Avalonia (.NET) | No client SDK; FFI bindings are server-focused; capture/render all DIY | X11 solid, Wayland recent | First-class | Risky — calls ≈ all DIY |
| Compose Multiplatform | **None** (Kotlin SDK is Android-only; webrtc-kmp JVM PR unmerged since 2024) | XWayland only | OK | **No** |
| Dioxus | Native renderer (Blitz) is 0.7 preview: no `<video>`, no WebRTC; stable path is webview | — | — | **No** (for now) |
| egui, Lynx, Freya, Xilem, Makepad | No / not desktop / experimental | — | — | **No** (Makepad+Robrix: watch only) |

Dropping macOS **helps**: Windows capture/loopback is mature (WASAPI/WGC), and the top options are
exactly the ones with first-class Linux+Windows.

### Shortlist (ranked for "Discord-class UX, Linux+Windows, LiveKit calls")
1. **Flutter + livekit_client** — ship-ready calls stack; the only gap is the Wayland picker patch
   (days). Cost: Dart, a second UI codebase to keep in parity with the web client.
2. **Qt 6/QML** — maximum control and the best video pipeline (zero-copy); you assemble room
   state/simulcast/adaptive-stream yourself on GStreamer. Biggest build, biggest ceiling.
3. **GPUI or Iced (Rust)** — GPUI if following Zed's proven integration; Iced for Pop!_OS/COSMIC
   alignment and API stability. Both: more assembly than Flutter, smaller binaries, no GC.

### How this composes with §1
Electron remains the **now** path (web client is the product today, parity is free). The native
client is a parallel track, not a replacement gate: pick the framework, prototype the call path
first (join room → hear audio → see a screenshare → publish camera), because that is the part that
kills frameworks — the chat UI ports anywhere. Note: Zig (NITRO-PARITY-MAP Part 5) fits none of the
three shortlisted stacks directly; the PTT/native-addon plan stays tied to the Electron track, while
a native client would get PTT for free from its own event loop.
