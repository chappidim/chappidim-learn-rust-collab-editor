# Overview & Startup

## What is the production system?

A production collaborative editor is a real-time collaborative document workspace for users. It combines:

- A rich-text editor with tables, headings, lists, code blocks, and media
- Real-time multi-user collaboration (Google Docs-style)
- An AI "teammate" that can read, edit, and comment on documents
- Access control via a group membership system
- Full-text search with ACL-aware filtering
- Version history with rollback
- MCP (Model Context Protocol) access for external AI agents

The backend is a single Rust binary built with Axum (HTTP/WS framework) and Tokio (async runtime), deployed on container orchestration behind the CDN + the load balancer.

## How the Server Boots

Entry point: `src/main.rs` → `#[tokio::main] async fn main()`

### Phase 1: Core Infrastructure

```
rustls crypto provider → tracing (JSON) → cloud SDK config
```

The server first installs the TLS provider and structured logging, then loads cloud credentials from the environment (container task role in prod, local credentials in dev).

### Phase 2: Storage Clients

the database and object store clients are created with optional endpoint overrides (`APP_DB_ENDPOINT`, `APP_STORAGE_ENDPOINT`) for local development against a local database emulator and MinIO/LocalStack.

### Phase 3: Store Initialization

~15 trait-based stores are wired up:

| Store | Backend | What it holds |
|-------|---------|---------------|
| `SnapshotStore` | Object Store | Binary Yrs CRDT state blobs |
| `MetadataStore` | Database | Doc/folder metadata (title, owner, parent, timestamps) |
| `CommentStore` | Database | Threaded comments on docs |
| `FolderStore` | Database | Folder hierarchy |
| `ShortcutStore` | Database | Document shortcuts/bookmarks |
| `RecentsStore` | Database | Per-user recently accessed docs |
| `FavoritesStore` | Database | Per-user favorited docs |
| `VersionStore` | Database | Version history entries |
| `AccessCountStore` | Database | View counts |
| `NotificationsStore` | Database | User notification inbox |
| `MediaRefStore` | Database | Media reference tracking |
| `MediaStore` | Object Store | Media file blobs (images, video, audio, PDF) |
| `FeedbackStore` | Object Store | Bug/feature report bundles |

Each has a Noop variant for local dev when the backing service isn't configured.

### Phase 4: Redis Subsystems

A single Redis URL (`APP_REDIS_URL`) backs multiple logical subsystems:

- **CrdtBroadcast** — Redis PUBLISH/SUBSCRIBE for real-time CRDT fan-out
- **CrdtStream** — Redis Streams for durable catch-up (optional, gated by `APP_STREAMS_CATCHUP=true`)
- **PresenceTracker** — Cursor positions and typing indicators
- **DocRouter** — Doc-to-node affinity mapping
- **RateLimiter** — Per-user sliding window counters
- **the group membership service cache** — Group membership lookups with Redis TTL
- **CARDS cache** — Team name resolution with Redis TTL

### Phase 5: AI & Config

- **AiService**: `the LLM providerAiService` (prod) or `MockAiService` (test-mock feature flag)
- **AiConfigStore**: Polls cloud a remote config service every 30s for the "Andon Cord" kill switch. Fail-safe: AI is disabled by default until a remote config service explicitly enables it.
- **AiTraceRing**: In-memory ring buffer recording AI tool calls for diagnostics/feedback.

### Phase 6: Background Tasks

Before accepting connections, the server spawns:

| Task | Interval | Purpose |
|------|----------|---------|
| Snapshot scheduler | 30s | Flush dirty docs to the object store, record versions |
| Idle doc eviction | 60s | Unload docs with no editors from memory |
| Router heartbeat | 30s | Renew Redis TTL on node's doc-affinity keys |
| MCP session sweeper | Periodic | Evict stale MCP sessions |
| a remote config service poller | 30s | Refresh AI enabled/disabled flag |

### Phase 7: Router Assembly

The Axum router is built bottom-up:

1. Route groups (docs, folders, comments, ACL, media, search, MCP, etc.)
2. Middleware stack (applied in reverse order of declaration):
   - MCP client version enforcement
   - Request ID generation
   - API metrics (latency, status codes, CloudWatch counters)
   - Authentication (`require_auth`)
   - CORS

### Phase 8: Listen & Serve

```rust
axum::serve(listener, app).with_graceful_shutdown(shutdown_signal(...))
```

Binds `0.0.0.0:3000` (or `APP_ADDR`), serving both HTTP REST and WebSocket upgrade requests.

## Why This Design?

- **Single binary**: Simplifies deployment (one container task definition), reduces operational overhead.
- **Trait-based stores**: Every external dependency can be swapped for a mock/noop, enabling fast unit tests and local dev without cloud credentials.
- **Fail-safe defaults**: Missing env vars disable features (AI, search, media) rather than crashing. The server always starts.
- **Background tasks are non-blocking**: The server is ready to serve immediately; background processes (snapshot flush, eviction) run in parallel.

## Graceful Shutdown

On SIGTERM (containers rolling deploy) or Ctrl-C:

1. Abort background tasks (scheduler, eviction, heartbeat)
2. Encode every in-memory document's CRDT state
3. Write final snapshots to the object store
4. Update Database metadata with the new object store keys
5. Unregister this node from the Redis doc router

This guarantees zero data loss during deployments.
