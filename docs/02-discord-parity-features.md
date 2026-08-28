# Discord-Like Features — Status Map

> What Discord (free + Nitro) offers vs. what this stack has, organized by what it takes to close each gap.
> Exact file/line targets live in [../NITRO-PARITY-MAP.md](../NITRO-PARITY-MAP.md); this doc is the presentational view.

## The whole picture

```mermaid
mindmap
  root((Discord parity))
    ✅ Already have
      Text: servers, channels, categories, roles, permissions
      DMs, groups, friends, blocking
      Replies, reactions, pins no-cap, search, edit/delete
      Custom + animated emoji everywhere
      Animated avatars & banners
      Masquerade impersonation
      Markdown BEYOND Discord: tables, headings, KaTeX
      Webhooks, bots, invites, bans, reports
      Full theming Material You
      Voice channels LiveKit, screenshare, camera
      Settings sync, PWA
    🟢 Config only
      500 MB uploads
      4000-char messages
      More attachments & emoji slots
      48 kHz voice quality
      1080p-4K resolution allowance
      Kill new-account restrictions
    🟡 Client code
      Remove 640x480@5fps screenshare clamp
      Camera above 720p
      Quality tiers 1080p60 / 4K
      Simulcast + VP9/AV1
      Per-user volume
      Desktop notifications & web push polish
      Image lightbox
      Keybinds UI
    🟠 Desktop shell
      Default to own instance
      Own update channel
      Global push-to-talk
      Tray & autostart exist
    🔴 Backend + full stack
      Polls
      Voice messages
      Voice moderation disconnect/mute
      Scheduled events
      Threads
      Stickers
      Soundboard
      Forum channels
    ⛔ Not worth cloning
      Super reactions
      Activities embedded games
      Stages
      Profile shop & effects
      Server boosts meaningless self-hosted
```

## Nitro perks specifically

| Nitro sells | Here | Path |
|---|---|---|
| 500 MB uploads | 🟢 | `REVOLT__FEATURES__LIMITS__*` env overrides + restart |
| 4000-char messages | 🟢 | same |
| HD (1080p60/4K) streaming | 🟡 | the one perk locked in *code*: client clamps (see map §2.1) |
| Animated avatar/banner | ✅ | free, ungated |
| Emoji anywhere / animated | ✅ | free, ungated |
| Custom profiles | ✅ mostly | banner+bio exist; accent colors optional cosmetic work |
| Custom themes | ✅ better | full theming vs Discord's paid palettes |
| Server boosts | N/A | everything already unlocked |
| Super reactions, shop | ⛔ | cosmetic monetization, skip |

**Conclusion the numbers support:** with one config change-set and one client patch (streaming quality), every *utility* Nitro perk is covered. The 🔴 tier is about matching free Discord's breadth, not Nitro.

## Feature-by-feature ownership

```mermaid
flowchart TD
    subgraph G1["🟢 self-hosted/Revolt.toml"]
        L1["limits: uploads, msg length,<br/>voice_quality, video_resolution,<br/>emoji, servers, new_user_hours"]
    end
    subgraph G2["🟡 for-web (+stoat.js)"]
        C1["rtc clamps & tiers"]
        C2["per-user volume"]
        C3["push/notifications polish"]
        C4["age-bug fix in stoat.js"]
    end
    subgraph G3["🟠 for-desktop"]
        D1["BUILD_URL → own instance"]
        D2["global PTT (Zig N-API)"]
        D3["own updater"]
    end
    subgraph G4["🔴 stoatchat + full pipeline"]
        B1["polls → voice msgs → voice mod →<br/>events → threads → stickers →<br/>soundboard → forums"]
    end
    G1 --> READY1["Phase 1: cancel Nitro"]
    G2 --> READY1
    G3 --> READY2["Phase 2: daily driver"]
    G4 --> READY3["Phase 3: beyond free Discord"]
```

## Current client feature matrix (upstream-tracked)

From `for-web/doc/src/feature-matrix.md`: **160 ✅ / 8 🚧 / 12 ❌** for this client. The ❌/🚧 that overlap this plan: desktop notifications (P0), update indicator (P0), member list, GIF picker, voice UX (volume/hide/ignore), voice moderation, desktop app items. Everything else in the matrix is done.
