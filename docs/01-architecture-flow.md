# How Stoat Works — Architecture & Flows

> Source: full-source mapping of the five repos in this workspace (2026-08-27). File references are exact.

## 1. Service topology

Everything sits behind Caddy on one domain. Every backend service is Rust, shares one MongoDB, and reads the **same** `Revolt.toml` (bind-mounted by `self-hosted/compose.yml`).

```mermaid
flowchart LR
    subgraph Clients
        WEB["for-web<br/>(Solid.js PWA)"]
        DESK["for-desktop<br/>(Electron shell)"]
        MOB["for-android / for-ios"]
    end
    DESK -- "loadURL(BUILD_URL)" --> WEB

    CADDY["Caddy<br/>(reverse proxy, TLS)"]
    WEB -->|HTTPS| CADDY
    MOB -->|HTTPS| CADDY

    subgraph Backend["stoatchat (Rust workspace)"]
        DELTA["delta<br/>REST API :14702<br/>route /api*"]
        BONFIRE["bonfire<br/>WebSocket :14703<br/>route /ws"]
        AUTUMN["autumn<br/>files :14704<br/>route /autumn*"]
        JANUARY["january<br/>link embeds :14705<br/>route /january*"]
        GIFBOX["gifbox :14706<br/>route /gifbox*"]
        PUSHD["pushd<br/>push notifications"]
        CROND["crond<br/>scheduled jobs"]
        INGRESS["voice-ingress :8500<br/>route /ingress*"]
    end

    CADDY --> DELTA & BONFIRE & AUTUMN & JANUARY & GIFBOX & INGRESS
    CADDY -->|"fallback /"| WEBIMG["web (for-web image :5000)"]

    subgraph Voice
        LK["LiveKit SFU :7880<br/>UDP 50000-50100<br/>route /livekit*"]
    end
    CADDY --> LK
    LK -- "webhooks" --> INGRESS

    subgraph Data
        MONGO[("MongoDB")]
        REDIS[("Valkey/Redis")]
        RABBIT[("RabbitMQ")]
        MINIO[("MinIO (S3)<br/>bucket revolt-uploads")]
    end

    DELTA & BONFIRE & AUTUMN & PUSHD & CROND --> MONGO
    DELTA & BONFIRE & INGRESS --> REDIS
    DELTA -- "events" --> RABBIT --> PUSHD
    AUTUMN --> MINIO
```

Key facts:
- **delta** serves `GET /` with the instance config — hosts, LiveKit nodes, VAPID key, and **all limits** (`stoatchat/crates/delta/src/routes/root.rs:136-152`). Clients drive every quota from this payload.
- **bonfire** is event fan-out only; it serves no config.
- Instance discovery for third-party clients: `https://<host>/.well-known/stoat` → `{"api": "..."}` (served as static `stoat.json` by Caddy).

## 2. Client boot & session lifecycle

```mermaid
sequenceDiagram
    participant U as User
    participant C as for-web
    participant D as delta (REST)
    participant B as bonfire (WS)

    U->>C: open app
    C->>C: hydrate 15 stores from IndexedDB<br/>(components/state, localforage)
    C->>D: GET / (instance config: limits, livekit nodes, vapid)
    alt no session
        U->>C: credentials
        C->>D: POST /auth/session/login
        D-->>C: session token (or MFA challenge loop)
    end
    C->>D: GET /onboard/hello
    C->>B: WS connect + Authenticate
    B-->>C: Ready (users, servers, channels)
    Note over C,B: Lifecycle machine (client/Controller.ts):<br/>Connected → Disconnected → Reconnecting<br/>with exponential backoff + jitter
    B-->>C: UserSettingsUpdate → SyncWorker merges<br/>(ordering, notifications, release-notes)
```

## 3. Sending a message (with attachment)

```mermaid
sequenceDiagram
    participant C as for-web
    participant A as autumn
    participant D as delta
    participant B as bonfire
    participant P as pushd

    C->>C: Draft store: outbox entry,<br/>idempotency key (ulid)
    C->>A: POST /attachments (XHR, progress)
    A->>A: size vs limits.file_upload_size_limit<br/>+ body_limit_size (api.rs:52-57, 205-214)
    A-->>C: file id (cached; retry won't re-upload)
    C->>D: POST /channels/:id/messages<br/>(stoat.js Channel.sendMessage)
    D->>D: validate: message_length,<br/>attachments ≤ N, replies, embeds<br/>(messages/model.rs)
    D-->>B: event publish
    B-->>C: Message event (all clients)
    D-->>P: via RabbitMQ → web push (VAPID)
    Note over D: january fetches link embeds async<br/>(process_embeds.rs, max message_embeds)
```

## 4. Joining a voice call

```mermaid
sequenceDiagram
    participant C as for-web
    participant D as delta
    participant LK as LiveKit
    participant VI as voice-ingress
    participant B as bonfire

    C->>C: latency-race config.features.livekit.nodes<br/>(rtc/state.tsx:312-318)
    C->>D: POST /channels/:id/join_call {node}
    D->>D: perms: Speak / Video + limits.video<br/>(voice/mod.rs:156-171)
    D-->>C: { url, token } (JWT, 10s TTL,<br/>grants per permission)
    C->>LK: room.connect(url, token, autoSubscribe:false)
    C->>C: publish mic through VoiceProcessor<br/>(RNNoise → highpass → compressor → gain)
    LK-->>VI: webhook (participant/track events)
    VI->>B: mirror to voice state (Redis) + events
    B-->>C: voice state updates (all clients)
```

Quality chain (who caps what):
```mermaid
flowchart TD
    CFG["Revolt.toml<br/>voice_quality / video_resolution<br/>(advisory, served via GET /)"] --> TIERS["for-web tier gate<br/>rtc/state.tsx:443 (1080p needs limit ≥1920x1080)"]
    TIERS --> CLAMP["getDisplayMedia clamp<br/>rtc/index.ts:9-21<br/>HARD 640x480@5fps ← the bottleneck"]
    CLAMP --> LKP["LiveKit presets<br/>h720 camera / h720fps30 share<br/>(no simulcast/codec config)"]
    style CLAMP fill:#7f1d1d,color:#fff
```

## 5. How a new feature crosses the stack

```mermaid
flowchart LR
    M["1. core/models<br/>types + validators"] --> DB["2. core/database<br/>DB model, queries"]
    DB --> RT["3. delta route<br/>(rocket/okapi)"]
    RT --> EV["4. bonfire event"]
    RT --> OAPI["5. OpenAPI.json<br/>(javascript-client-api)"]
    OAPI --> API["6. stoat-api types<br/>(pnpm build)"]
    API --> SDK["7. stoat.js<br/>class/method"]
    SDK --> UI["8. for-web UI"]
    UI --> DEP["9. self-hosted<br/>image bump / compose"]
    UI -.->|"only if native OS access"| DT["for-desktop"]
```
