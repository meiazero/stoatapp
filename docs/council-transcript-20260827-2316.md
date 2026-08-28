# LLM Council Transcript: Stoat Complete Refactor

Generated: 2026-08-27 23:16 America/Sao_Paulo.

## Original question

Analyze the decisions for a complete refactor of Stoat using several independent agents. Assume unlimited implementation time. Optimize for technologies that improve what the application is and what it can become, without bias.

## Framed question

Design the end-state architecture and complete refactor strategy for Stoat. The product is desktop-first on Linux and Windows, with a web client at feature parity, exceptional UX/UI/accessibility, constrained self-hosting, realtime voice/video/screen sharing, reliability, security, and room to become more capable than Discord. Unlimited implementation time removes schedule pressure but does not remove ecosystem maturity, intrinsic complexity, maintenance, security, operability, or platform constraints. Decide what to retain, rewrite, delete, and introduce across UI, shared core, backend, data/sync, media, extensions, E2EE, infrastructure, and testing.

## Independent advisor responses

### The Contrarian

The safest recommendation is incremental refactoring with frozen contracts, not the settled end state proposed by doc 06. Keep Solid, `stoat.js`, LiveKit, and Electron while fixing measurable defects. Replacing Electron with GPUI or Iced currently conflicts with the accessibility requirement.

Reject “SQLite everywhere” as an architectural decision today. The existing 155 database methods provide a seam, not proof of semantic migration. Preserve MongoDB until a reversible adapter has load, migration, RPO, and RTO evidence. Keep S3-compatible storage; filesystem storage can be an optional adapter. Consolidate the API and event plane, but isolate media and workers. Remove RabbitMQ only after a durable outbox proves replay, poison-message, and idempotency semantics.

Treat the Rust shared SDK, WASM plugins, CRDTs, MoQ, and MLS as product programs rather than refactoring. Start with versioned schemas, generated types, conformance tests, and a pure permission/sync core. Plugins begin without arbitrary UI access, in an isolated process with quotas. MLS requires a threat model, recovery, and moderation/search policy. Require p95 interaction, 4 GB memory, lossless reconnection, WCAG 2.2 AA, and restorable migration gates. Unlimited time does not remove maintenance complexity.

### The First Principles Thinker

Stoat should become a local-first, self-hostable communication platform, not merely a Discord clone. Preserve working product surface while moving complexity behind deep seams.

Keep Solid and correct its locality defects. Absorb `solid-livekit-components` only after fixing cleanup errors. Electron remains the compatibility shell while a native client is developed. Prefer accessible candidates such as Qt/QML or Slint in the probe; do not default to GPUI or Iced while accessibility remains unresolved.

Use Rust for a shared protocol, permission, validation, and sync core, extracted incrementally. Do not make it own Solid's reactive state. Build a Rust modular monolith with explicit module interfaces and keep LiveKit, Caddy, and specialized workers. Make SQLite the default single-node adapter only after evidence; retain a scale adapter. Use filesystem and S3-compatible blob adapters.

Beyond-Discord leverage should come from capability-based WASM plugins, local search, focused Loro documents, captions, and selective MLS rooms. Every feature requires a schema validator, versioned event, bounded pagination, migration, accessibility test, and failure-injection test.

### The Expansionist

Adopt a Rust core with specialized clients, preserving Solid during migration. The core should eventually own protocol, permissions, sync, eviction, durable cache contracts, full-text search, and normalized message parsing. Expose flat change events through generated web and native bindings.

Keep Electron until a native client proves parity. Use the probe to evaluate Qt/QML and other candidates. Move the server toward a Rust/Axum modular monolith with a simple single-node data profile while retaining LiveKit and specialized media processes. Use GStreamer for native capture, transcode, and hardware-accelerated media workflows.

Make WASM Components the strategic differentiator: official, capability-scoped extensions rather than unofficial client modification. Add CRDTs only for clear collaborative artifacts, not ordinary messages. Scope MLS to direct messages and private groups. Fix production devtools, RxJS leaks, and O(k²) hydration before any rewrite so the baseline remains usable.

### The Outsider

The product is coherent as user-controlled, extensible communication. It becomes incoherent if it simultaneously attempts to be Discord, Matrix, Slack, Twitch, and Notion. Establish reliable local-first communication as the north star.

Retain the mature Solid UI, SDK, and media bridge. Extract validators, permissions, event decoding, sync invariants, and message normalization into a shared Rust core only after conformance tests exist. A native client is not automatically better. Prototype multiple toolkits against the same screen-reader, IME, Wayland, Windows, GPU-disabled, and media tests.

Ship a modular monolith but preserve durable queue and scale adapters. Local persistence has high UX value only after revocation, conflict, deletion, and shared-device privacy semantics are specified. Sequence accessibility and reliability first, offline UX second, then captions, search, and capability plugins. Delay federation, MoQ, generalized CRDTs, and E2EE until product ownership and threat models are explicit.

### The Executor

Use a migration with preserved contracts, not a big-bang rewrite. Keep Solid, `stoat.js`, Electron, and LiveKit. Fix production devtools, O(k²) hydration, unbounded stores, RxJS leaks, unordered bulk events, missing virtualization, and uncached Markdown first.

Introduce a Rust crate for pure invariants: identifiers, permissions, validation, ordering, sync, cryptography, and serialization. Do not share reactive UI state. Define an append-only protocol with per-device cursors, idempotency, tombstones, revocation, and a durable outbox. Keep MongoDB, RabbitMQ/Valkey, and S3/MinIO until alternatives prove equivalent recovery, migration, backup, and load behavior.

Prototype Flutter and Qt/GStreamer or other accessible native candidates. Replace Electron only after functional parity, real accessibility, and distro media tests. Treat WASM plugins, Loro, local search, captions, MLS, and MoQ as bounded experiments. Remove an old path only after two stable releases and a rehearsed rollback.

## Anonymous mapping

- Response A: Executor
- Response B: Expansionist
- Response C: Contrarian
- Response D: First Principles Thinker
- Response E: Outsider

## Peer reviews

### Reviewer 1

Response A is strongest because it provides a reversible sequence, defines sync semantics, and requires recovery, load, and rollback evidence. Its blind spot is failure to commit to a product-shaped destination. All responses lack quantitative exit thresholds, ownership, compatibility guarantees, rollback triggers, and conclusive decision rules.

### Reviewer 2

Response A is strongest for preserving optionality while demanding evidence. It risks indefinite architecture runway. All responses under-specify SLOs, observability, incident response, patch ownership, upgrade support, compatibility/deprecation policy, and maintenance budgets.

### Reviewer 3

Response D is strongest because it joins product direction to deep seams and adapter-based storage without prematurely rewriting. It lacks migration ownership, compatibility windows, rollback mechanics, and measurable exits. All responses miss identity recovery, moderation, privacy/retention, the canonical domain model, and accessibility as a continuing process.

### Reviewer 4

Response A is strongest. It lacks enough detail on deployment evolution, device identity/key recovery, and parity governance. All responses miss canonical entities, permissions, event ordering, conflict resolution, retention, indexing, media lifecycle, telemetry-backed deletion, and a coherent local-first/federation/E2EE contract.

### Reviewer 5

Response A is strongest but does not define security architecture: identity, authorization, key management, plugin trust, and isolation. All responses miss operational governance, ownership and deletion criteria, lifecycle management, migration SLOs, and a cross-platform capability matrix.

## Chairman synthesis

### Where the Council Agrees

Stoat should center user-controlled, extensible communication, preserve its working Solid/Electron/LiveKit path, and migrate through deep Rust modules with narrow interfaces. The Rust core owns canonical domain rules, permissions, deterministic sync, validation, and cache semantics—not reactive state or UI. The server moves toward a modular monolith with isolated media and hazardous workers. A simple single-node data profile is desirable, but replacements must prove their semantics. Local-first is a protocol property. Security begins with device identity, recovery, revocation, retention, and capability models. WASM plugins are the strongest expansion direction when tightly constrained.

### Where the Council Clashes

The main conflict is commitment versus evidence. The council resolves it by committing to Rust for pure invariants, intending SQLite/filesystem for the default profile while preserving adapters until proven, running a native toolkit probe instead of naming a winner, and treating WASM as the strategic extension direction without prematurely publishing a broad plugin interface. Electron remains baseline. LiveKit remains the media plane.

### Blind Spots the Council Caught

The architecture needs measurable SLOs, migration ownership, compatibility windows, upgrade topology, identity recovery, abuse and moderation policy, observability, privacy and retention behavior, and a canonical domain model. Every subsystem creates a permanent support and patching obligation.

### The Recommendation

Keep Solid and Electron in production. Create a pure Rust domain/sync core. Build a Rust modular-monolith server with versioned HTTP and event interfaces. Intend SQLite/filesystem/local search as the default single-node profile and preserve scale adapters. Keep LiveKit and isolate media workers. Select a native toolkit through one fixed accessibility/media/platform probe. Make WASM plugins, local search, captions, and focused collaborative documents the first expansion bets. Keep federation, generalized E2EE, MoQ, embeddings, and broad CRDT adoption experimental.

Migrate by vertical slices. Freeze the canonical domain and event envelope, extract invariants, add outbox and sync, add a replaceable local cache, then migrate messages, membership, attachments, and calls. Remove paths only after conformance, recovery, two stable releases, production telemetry, and rehearsed rollback.

### The One Thing to Do First

Write and test the canonical domain model and versioned sync/event envelope against the existing stack. The first runnable simulation covers duplicate delivery, event gaps, tombstones, permission revocation, device loss, and eventual convergence.
