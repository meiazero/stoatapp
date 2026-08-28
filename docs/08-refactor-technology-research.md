# Refactor Technology Research

Research date: 2026-08-27. Scope: complete refactor for a desktop-first Linux/Windows client, web parity, accessible UX, self-hosting, resource efficiency, realtime media, and extensibility. Sources are first-party documentation, specifications, or the owning project's source repository. “Verified” means the source explicitly supports the statement; “projection” is an engineering inference; “unverified” must not be used as an architectural premise.

## Executive decision

Use a staged shared-domain architecture, not a total rewrite:

1. Keep the existing Solid web UI and Electron bridge while fixing measured correctness/performance defects. Electron has a documented Chromium multi-process model and sandbox, but its security is explicitly dependent on application configuration and staying current ([Electron process model](https://www.electronjs.org/docs/latest/tutorial/process-model), [Electron security checklist](https://www.electronjs.org/docs/latest/tutorial/security)).
2. Extract protocol/domain algorithms into a Rust crate with generated TypeScript/WASM and native bindings only where tests show a real cross-client benefit. Keep UI state adapters platform-specific; do not force a reactive store ABI across Solid and a future native toolkit.
3. Retain LiveKit for SFU/media. Its official docs cover H.264, VP8, VP9/SVC, and AV1/SVC, with simulcast enabled by default and dynacast as an explicit bandwidth optimization ([LiveKit codecs](https://docs.livekit.io/transport/media/advanced/)).
4. Make SQLite a measured candidate, not an automatic “everywhere” decision. SQLite’s official WASM build supports OPFS, but OPFS is worker-only and the VFS choices trade performance against multi-tab concurrency ([SQLite WASM overview](https://sqlite.org/wasm/doc/tip/about.md), [SQLite persistence](https://www.sqlite.org/wasm/doc/trunk/persistence.md)). Benchmark migration, concurrent writers, backup/restore, and multi-instance deployment before replacing MongoDB.
5. Adopt capability-based WASM extensions only behind a narrow host API, quotas, signature/revocation policy, and a compatibility test matrix. WIT defines contracts; a component can access only imported host interfaces, but that boundary is a security mechanism—not proof that arbitrary third-party code is safe ([WIT overview](https://component-model.bytecodealliance.org/design/wit.html), [component worlds and isolation](https://component-model.bytecodealliance.org/design/worlds.html), [Wasmtime](https://docs.wasmtime.dev/)).

## Technology findings

### Desktop UI and shell

**Verified:** LiveKit maintains an official Flutter client repository with Linux and Windows plugin targets and documentation for camera, microphone, desktop screenshare, and E2EE ([LiveKit Flutter SDK](https://github.com/livekit/client-sdk-flutter)). LiveKit also maintains an official C++ SDK with raw-frame access and integration targets such as GStreamer ([LiveKit C++ SDK](https://github.com/livekit/client-sdk-cpp)). These are stronger facts than the claim that Flutter is the only end-to-end native option.

**Correction:** docs 05 and 06 make categorical claims that Tauri/WebKitGTK cannot provide WebRTC and that Electron is the only proven path. The cited evidence in those docs is issue reports and third-party field reports, not an owning-platform specification. Tauri’s official materials identify WebKitGTK as the Linux webview ([Tauri prerequisites](https://v2.tauri.app/start/prerequisites/)), while actual WebRTC capability depends on the deployed WebKitGTK build. Record a per-distro capability probe and media smoke test before a “no” verdict. Electron remains the lowest-risk bridge because it bundles Chromium and has explicit sandbox guidance, not because every alternative is categorically impossible.

**Projection:** Flutter is the fastest native media prototype; Qt/GStreamer or a native C++/Rust client may offer more rendering and OS control. GPUI/Iced memory numbers, “only stack” claims, and exact RSS figures in docs 05/06 are not verified here and should be removed or labeled as measurements with reproducible hardware/build parameters.

### Shared core, storage, and sync

**Verified:** the SQLite project provides an official WASM/JS subproject and OPFS VFS. OPFS requires worker contexts; `opfs-sahpool` favors performance when concurrency is less important, while other VFS choices address multi-tab concurrency with different constraints ([SQLite persistence](https://www.sqlite.org/wasm/doc/trunk/persistence.md)). Therefore “SQLite-WASM in a worker” is feasible, but “no COOP/COEP needed,” “same schema everywhere,” and exact bundle/RAM savings are deployment hypotheses.

**Decision:** first define an append-only sync protocol with per-device cursors, idempotency keys, tombstones, permission revocation, and bounded retention. Then implement a storage interface with SQLite and the current database as conformance backends. Do not let local-first storage imply offline authorization: the server remains authoritative for send, membership, and revocation.

**Correction:** docs 06/07 call a custom sync protocol the end state while also presenting SQLite as settled. It is neither settled nor validated. The architecture should make SQLite replaceable until load, recovery, migration, and multi-process tests pass.

### Backend topology and self-hosting

**Projection:** a monolith can reduce operational burden for a single-node self-hosted deployment, but it changes failure domains, upgrade isolation, and scaling. Preserve process boundaries behind one deployment profile until equivalent integration, crash-recovery, and migration tests exist. Do not claim exact container/RAM reductions from docs 06 without published benchmark scripts and workload definitions.

**Decision:** remove a broker or object store only after proving delivery semantics (at-least-once behavior, retries, poison messages, durable outbox, and replay), blob durability, range requests, quotas, and backup restore. Resource efficiency is a constraint, not sufficient evidence for replacing MongoDB, RabbitMQ, or S3-compatible storage.

### Media, codecs, and E2EE

**Verified:** LiveKit documents E2EE for media tracks and data channels, says keys are generated/distributed by the application, and states signaling/API traffic remains readable by the server ([LiveKit encryption overview](https://docs.livekit.io/transport/encryption/)). LiveKit also documents agents participating in E2EE rooms when supplied the key ([E2EE with agents](https://docs.livekit.io/transport/encryption/agents/)).

**Correction:** docs 07’s statement that LiveKit E2EE conflicts with server-side agents is too broad: agents are supported when trusted with the key. The real product decision is per-room trust policy: a zero-knowledge room cannot provide server-side transcription, moderation, indexing, or recording. Document this explicitly and test key rotation, device loss, recovery, and metadata leakage.

**Correction:** docs 07 says native LiveKit Rust has no E2EE, but the research does not establish that claim from the Rust SDK source. The official C++ SDK exposes E2EE options ([C++ E2EE reference](https://docs.livekit.io/reference/client-sdk-cpp/structlivekit_1_1E2EEOptions.html)). Treat native-language support as an SDK-version capability to verify, not as a reason to invent FrameCryptor.

**Decision:** default browser media to interoperable Opus plus VP8/H.264 fallback; selectively benchmark VP9/AV1 SVC on target hardware. LiveKit’s own docs say SVC changes dynacast behavior and that codecs trade device performance against quality ([codec details](https://docs.livekit.io/transport/media/advanced/)). Do not promise a fixed “25% sharper” or “5–10% CPU” gain without a captured test matrix.

### Plugins and frontier features

**Verified:** WIT is an IDL for language-neutral component contracts, and worlds restrict interaction to declared imports/exports ([WIT](https://component-model.bytecodealliance.org/design/wit.html), [worlds](https://component-model.bytecodealliance.org/design/worlds.html)). The Component Model guide currently identifies WASI 0.2 as the stable release and describes newer async primitives separately ([Component Model FAQ](https://component-model.bytecodealliance.org/reference/faq.html)).

**Correction:** docs 07’s “start on WASI 0.2” is prudent, but its statement that WASI 0.3 is simply “fresh” should be version-pinned to the runtime and toolchain actually shipped. Plugin ABI stability, package signing, permissions, resource limits, host-call timeouts, and update rollback matter more than the runtime brand.

**Unverified:** Loro benchmark leadership, MoQ relay production claims, local embedding model suitability, exact model sizes, and “Discord does not ship” comparisons require direct benchmark/product evidence. Keep them as experiments, not architecture commitments. WebTransport is now broadly available in current browsers but requires HTTPS and has older-device limits ([WebTransport API](https://developer.mozilla.org/en-US/docs/Web/API/WebTransport_API)); it does not by itself provide a broadcast protocol or relay.

### Accessibility, observability, and quality

**Verified:** AccessKit provides cross-platform accessibility infrastructure and integrations for several Rust toolkits, but its own documentation says rich text/hypertext are not yet supported ([AccessKit](https://accesskit.dev/), [how it works](https://accesskit.dev/how-it-works/)). This makes native toolkit selection an accessibility decision, not only a memory/performance decision. Keep HTML text rendering for chat until an equivalent accessible rich-text tree is demonstrated.

**Verified:** OpenTelemetry Rust currently labels traces, metrics, and logs as beta, while OTLP itself is stable for traces/metrics/logs ([OpenTelemetry Rust](https://opentelemetry.io/docs/languages/rust/), [OTLP specification](https://opentelemetry.io/docs/specs/otlp/)). Use feature-gated OTLP export with local buffering and redact message contents, tokens, and media identifiers by default.

**Decision:** require contract tests for protocol/schema generation, property tests for permissions and sync, media-device matrix tests, accessibility audits with real screen readers, crash/restart tests, migration/restore tests, and deterministic end-to-end fixtures before deleting current paths. “Infinite implementation time” permits these tests; it does not justify speculative rewrites.

## Strongest corrections to docs 05–07

1. Replace “Tauri no / Electron only” with “Electron lowest-risk bridge; verify WebKitGTK capabilities per target distro.”
2. Replace settled “SQLite everywhere” with a backend-conformance experiment and explicit sync/authorization invariants.
3. Remove unreferenced exact RSS, CPU, bundle, and sharpness percentages; publish reproducible benchmarks before using them in decisions.
4. Correct E2EE: LiveKit supports media/data E2EE and trusted agents; signaling remains server-visible. Verify each native SDK version instead of asserting Rust lacks support.
5. Treat Flutter as an official media path, not proof that it is the best final UI. Compare Flutter, Qt, and Rust toolkits using accessibility, IME, rich text, GPU fallback, packaging, and media tests.
6. Keep WASM plugins as a bounded prototype; pin stable component/WASI versions and design revocation/permissions before calling it production-ready.
7. Keep frontier items (CRDT, MoQ, embeddings, MLS) behind measurable experiments and threat models. Standards maturity and a library’s release tag are not production readiness.

