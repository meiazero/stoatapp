# Frontier Tech — What This Application Can Become

> Premise: infinite engineering time; the question is not "what to rewrite" (that's
> [06-rewrite-map.md](./06-rewrite-map.md)) but **which technologies make the app MORE than
> Discord**. Researched 2026-08-27 against current primary sources; every item carries an honest
> maturity label. Companions: docs 04 (UX gaps), 05 (platform), 06 (rewrites + data model).

## Consolidated top 10 (how much MORE the app becomes × feasibility)

| # | Technology | New capability | Maturity |
|---|---|---|---|
| 1 | **WASM plugin system** (wasmtime + WIT components, Zed's extension system as blueprint) | "BetterDiscord done right, officially": server bots, slash commands, UI extensions as sandboxed capability-based components — the category differentiator | Host stack production; WASI 0.3 async fresh (Jun 2026) — start on 0.2 |
| 2 | **Collaborative CRDT layer (Loro)** | Shared drafts, per-channel collaborative docs/notes, whiteboard — a feature category Discord doesn't have | Production (Loro 1.13, Rust-native, benchmark leader) |
| 3 | **Watch-party / broadcast via Media over QUIC** | Sub-second fan-out to thousands of viewers on one cheap relay; Twitch-class streaming inside the platform | Beta (moq-rs + GStreamer plugin; WebTransport is Baseline since Mar 2026; Cloudflare runs MoQ relays in production) |
| 4 | **Local semantic search** (Qwen3-Embedding-0.6B / EmbeddingGemma + `ort` + sqlite-vec) | "Find that message about the broken deploy" in natural language, multilingual, 100% on-instance | Models + inference production; sqlite-vec alpha (usearch as mature fallback) |
| 5 | **HQ stereo audio + AV1 screenshare** | Music-mode 128 kbps stereo (Discord charges Nitro for less) + ~25% sharper text screenshare | Production — LiveKit config + shipped browser codecs; ~days of work |
| 6 | **Live captions + translation** (LiveKit Agents + self-hosted whisper) | Real-time subtitles/translation in calls — Discord doesn't have it | Production server-side (WebGPU in-browser still experimental) |
| 7 | **Passkeys as primary auth** (webauthn-rs 0.5.5) | Passwordless, phishing-proof login — above Discord's password+2FA floor | Production; RP ID = instance domain (document domain-change caveat); keep recovery codes |
| 8 | **MLS E2EE for DMs** (OpenMLS 0.9, RFC 9420) | Text E2EE Discord does NOT have (their DAVE covers only voice/video, full rollout May 2026 — text stays plaintext) | Production-usable pre-1.0. Honest cost: server search/embeds/push previews break; scope to DMs/GDMs only; web key storage is the weak link |
| 9 | **Reliability + operability as features** | Fuzzing (cargo-fuzz) on protocol parsers, turmoil DST on the event pipeline ("your server doesn't lose messages"), `tracing`→OTLP with optional OpenObserve (single binary) + Bugsink (Sentry-compatible, 1 container, SQLite) — plus a Nix flake/NixOS module as a self-hoster distribution channel | All production; Antithesis skipped (no free tier; watch their open-source dhyve). TLA+ skipped unless we invent a novel sync protocol |
| 10 | **Native weak-connection mode** (libopus 1.6: DRED deep redundancy + NoLACE decoder enhancement) | Packet-loss resilience no competitor ships; browsers don't expose it — native client exclusive | Shipped in libopus 1.6.1 (Jan 2026), off by default, native only |

## Cross-impacts on earlier docs (important)

1. **A11y is a toolkit constraint, not a feature** — AccessKit is active (Jul 2026 releases, all
   platforms) but **GPUI and Iced upstream have no AccessKit integration; Slint, egui, and Qt do**.
   If the native client must be screen-reader accessible (doc 04 already flagged the web app's a11y
   floor), this weighs against the doc 05/06 GPUI/Iced preference or budgets a large integration
   effort. Note: AccessKit still lacks rich-text surfaces — chat message rendering will be the hard
   part on every Rust toolkit. **Revisit the framework choice with a11y as an explicit criterion.**
2. **LiveKit Rust SDK has no E2EE today** (web SDK does, via insertable streams) — if E2EE calls
   matter, the native client needs custom FrameCryptor work; decide per-room since E2EE conflicts
   with server-side agents (captions, recording).
3. **Local-first message store: no engine fits Rust+SQLite** — PowerSync/Electric/Zero all require
   Postgres, and per-user permission filtering + access revocation is unsolved in all of them. The
   plan stays doc 06's own sync protocol (channel buckets, append-only log + per-device cursor);
   steal the bucket pattern, adopt no engine. Track Turso's stable CDC as future plumbing.
4. **Federation: tarpit — explicitly rejected for now.** MIMI is pre-RFC (all drafts still WG
   documents, Jul 2026), ATProto isn't messaging, Matrix federation costs years. Cheap hedge
   instead: portable identity, full data export, bridge-friendly bot API. Revisit when MIMI is an
   RFC.
5. **Codec strategy**: web = Opus inband FEC + RED + VP9/AV1 SVC (AV1 spatial SVC still missing in
   Chrome; AV1 patent litigation pending — cautious default); native = libopus 1.6 DRED/NoLACE.
   Zoom-style raw WebCodecs+WebTransport pipelines: don't hand-roll — that path arrives packaged
   via MoQ (#3).
6. **Design tokens**: DTCG spec still draft ("do not implement" preview Jul 2026) — use the format
   internally as the token source of truth (Style Dictionary v4 supports it), don't expose it as a
   public contract. Rive: web runtime mature, **no official Rust runtime** — motion in the native
   client belongs to the toolkit, not Rive.

## Suggested adoption arc

**Now (config/days):** #5 stereo+AV1, #7 passkeys, observability slice of #9 (tracing→OTLP flag,
Bugsink).
**Next (weeks):** #6 captions agent, #4 semantic search, fuzzing slice of #9, Nix flake.
**Flagship projects (months, in order):** #1 WASM plugins → #2 Loro collaboration → #3 MoQ
watch-party → #8 MLS DMs (after the Rust core exists) → #10 with the native client.
