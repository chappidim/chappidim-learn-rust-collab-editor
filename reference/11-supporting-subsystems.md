# Supporting Subsystems

This file covers the smaller but essential subsystems that don't warrant their own dedicated document.

## Comments

Source: `src/comments.rs`

### What It Is

Threaded comments on documents, with replies, reactions (emoji), upvotes, and resolution state. Comments are stored in the database (not in the CRDT) and broadcast to connected clients via Redis pub/sub.

### How It Works

```
POST /api/docs/{id}/comments → create top-level or reply
GET  /api/docs/{id}/comments → list all (threaded)
PATCH /api/docs/{id}/comments/{num} → edit text, resolve/unresolve
DELETE /api/docs/{id}/comments/{num} → soft-delete
POST /api/docs/{id}/comments/{num}/upvote → toggle upvote
POST /api/docs/{id}/comments/{num}/react → toggle emoji reaction
```

Comments reference a position in the document via an **anchor** (block ID + text offset). When the CRDT content shifts, anchors may become stale — the frontend re-resolves them on load.

After any comment mutation, the server broadcasts a `MSG_COMMENT` (0x06) message via Redis pub/sub so all connected clients update their sidebar in real time.

### Why Separate from CRDT?

Comments are metadata _about_ the document, not content _of_ it. Putting them in the CRDT would:
- Bloat the document state (comments can be long, have threads)
- Make comment deletion complex (CRDT tombstones vs. hard delete)
- Complicate the sync protocol (comment JSON mixed with binary Yrs updates)

## Folders

Source: `src/folders.rs`

### What It Is

A hierarchical folder structure for organizing documents. Each folder has a parent (or `ROOT_PARENT_SENTINEL` for top-level), an owner, an optional ACL, and metadata.

### How It Works

```
POST   /api/folders → create
GET    /api/folders → list user's folders
GET    /api/folders/{id} → get one
PATCH  /api/folders/{id} → rename, move
DELETE /api/folders/{id} → delete (must be empty)
GET    /api/folders/{id}/contents → list docs + subfolders
GET    /api/folders/{id}/ancestors → breadcrumb path to root
```

Key behaviors:
- **Depth limit**: 20 levels max (prevents infinite inheritance walks)
- **ROOT_PARENT_SENTINEL**: The string `"ROOT"` represents "no parent" in the database (which requires non-null partition keys for the GSI)
- **Move semantics**: `PATCH` with `parent_id` moves a folder. The server validates no circular references.
- **Cascade delete**: Not implemented — folders must be empty to delete.

### Why the database (Not Nested in CRDT)?

Folder structure is shared across all documents. It doesn't belong inside any single document's CRDT state. the database's `parent-id-index` GSI provides efficient "list children of folder X" queries.

## Notifications

Source: `src/notifications.rs`, `src/notifier.rs`

### What It Is

In-app notifications when users are mentioned in documents or granted access to resources. Notifications are produced by a a serverless function consumer (reading from message queue) and stored in the database. The backend exposes read/dismiss endpoints.

### How It Works

**Producing notifications:**
```
User @mentions "bob" in doc → MSG_MENTION (0x19) over WS
  │
  ▼
Server: validate mention (user has permission, dedup/debounce)
  │
  ▼
message queue → Notification a serverless function → the database write
```

**Consuming notifications:**
```
GET    /api/notifications → paginated list for current user
DELETE /api/notifications → dismiss (by ID, by doc, or all)
```

The `Notifier` trait abstracts the producer:
- `QueueNotifier` — Production: sends to message queue
- `NoopNotifier` — Local dev / when queue URL isn't configured

### Why message queue + a serverless function (Not Direct Write)?

- **Decoupling**: The WS handler doesn't block on notification delivery
- **Batching**: The a serverless function can debounce (e.g., don't send 5 notifications for 5 mentions within 1 second)
- **Slack integration**: The a serverless function also sends Slack DMs, which has its own rate limits and retry logic

## Presence

Source: `src/presence/mod.rs`, `src/presence/redis.rs`

### What It Is

Real-time tracking of who is currently viewing/editing each document, their cursor position, and typing state.

### How It Works

```rust
trait PresenceTracker: Send + Sync {
    async fn set_cursor(&self, doc_id, user, position) -> Result<()>;
    async fn get_cursors(&self, doc_id) -> Result<Vec<(String, u32)>>;
    async fn remove_user(&self, doc_id, user) -> Result<()>;
    async fn set_typing(&self, doc_id, user, typing: bool) -> Result<()>;
    async fn get_typing(&self, doc_id) -> Result<Vec<String>>;
}
```

Backed by Redis HASHes with TTL:
- `presence:{doc_id}:cursors` — user → position
- `presence:{doc_id}:typing` — user → timestamp

Awareness updates (cursor moves, selection changes) flow via `MSG_AWARENESS` (0x04) over WebSocket and `doc:{id}:awareness` Redis pub/sub channel.

### Why Redis (Not In-Memory)?

Multiple server nodes may have clients on the same doc. Presence must be visible across nodes (user A on Node 1 should see user B's cursor on Node 2).

## Rate Limiting

Source: `src/rate_limit.rs`

### What It Is

Per-user sliding-window rate limits backed by Redis `INCR` + `EXPIRE`. Prevents abuse of expensive operations.

### Limits

| Type | Limit | Window |
|------|-------|--------|
| HTTP API | 100 requests | 60 seconds |
| WebSocket messages | 1000 messages | 60 seconds |
| AI requests | 20 requests | 60 seconds |
| Media uploads | 100 uploads | 60 seconds |
| Media bytes | 100 MB | 60 seconds |

Disabled in `test-mock` builds to avoid test flakiness.

### Why Redis (Not In-Memory)?

Same user hitting different server nodes should see a unified rate limit. Redis provides the shared counter.

## Feedback

Source: `src/feedback.rs`

### What It Is

Users can submit bug reports and feature requests from within the editor. Each submission is packaged as a bundle with optional attachments (document state, AI trace, client info) and stored in the object store.

### How It Works

```
POST /api/feedback
  Body: {
    doc_id: "optional",
    type: "bug" | "feature",
    description: "...",
    include_state: true,      // attach current doc content
    include_ai_trace: false,  // attach recent AI activity
    client_state: {...},      // editor state snapshot
    client_info: {...}        // browser/version info
  }
  │
  ▼
Server:
  1. Generate feedback_id (UUID)
  2. Write envelope.json to the object store (metadata + description)
  3. If include_state: encode doc CRDT state → state.bin in the object store
  4. If include_ai_trace: drain AiTraceRing → trace.jsonl in the object store
  5. Write client_state.json, client_info.json if provided
  └── Return { feedback_id }
```

### Why object store (Not the database)?

Feedback bundles can be large (10+ MB with doc state and AI traces). object store is the natural fit for variable-size blobs. A separate object store prefix (`feedback/{id}/`) keeps them organized and easy to lifecycle (expire old feedback after 90 days).

## Shortcuts (Document Bookmarks)

Source: `src/shortcuts.rs`

### What It Is

Users can create named shortcuts (bookmarks) to specific documents for quick access.

### How It Works

```
POST   /api/docs/{doc_id}/shortcuts → create shortcut
GET    /api/docs/{doc_id}/shortcuts → list shortcuts for a doc
DELETE /api/shortcuts/{shortcut_id} → delete a shortcut
```

Stored in the database (`app-resources` table) with the doc as the partition key.

## External Doc Import

Source: `src/doc_import.rs`

### What It Is

Allows importing content from external document editors into the production system. Accepts HTML content (exported from another editor) and converts it into CRDT operations that populate a the production system document.

### How It Works

```
POST /api/docs/{id}/import
  Body: { html: "<externally exported content>", media: [...] }
  │
  ├── Parse HTML into block structure
  ├── Convert to CRDT operations
  ├── Migrate embedded media to the editor (upload to the object store)
  ├── Apply operations to the document's CollabDoc
  └── Return success
```

Size limit: `MAX_IMPORT_BYTES` (set generously for large imported docs with embedded media).

## Metrics

Source: `src/metrics.rs`

### What It Is

Application-level metrics emitted to CloudWatch via the EMF (Embedded Metric Format) pattern in structured logs.

### Key Metrics

| Metric | Type | Dimensions |
|--------|------|------------|
| `api.requests` | Counter | method, path, status |
| `api.latency_ms` | Timer | path |
| `api.success/error/fault` | Counter | path |
| `doc_manager.docs.loaded` | Gauge | — |

The `normalize_api_path` function collapses dynamic segments (doc IDs, hashes) to keep CloudWatch dimension cardinality bounded.

## Client Version Enforcement

Source: `src/client_version.rs`

### What It Is

A middleware that checks the MCP client version header and rejects outdated clients. This allows the backend to evolve the MCP tool schema without maintaining backwards compatibility with very old clients.

## Recovery

Source: `src/recovery.rs`

### What It Is

A recovery endpoint (`GET /api/recovery/mine`) that lets users find documents they own that may have been "lost" (e.g., parent folder deleted, metadata corrupted). Lists recoverable documents that the user can then re-home into a new folder.
