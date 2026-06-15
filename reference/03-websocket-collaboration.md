# WebSocket Collaboration

## What It Is

The WebSocket layer is how human users (and the in-process AI teammate) collaborate in real time. Each browser tab opens a persistent WebSocket connection to the backend, through which all document edits, cursor positions, comments, and AI interactions flow.

Source: `src/ws/mod.rs`, `src/ws/protocol.rs`

## How It Works

### Connection Lifecycle

```
Browser                         Backend
  │                                │
  │── GET /ws/{doc_id} ───────────▶│  HTTP upgrade request
  │                                │  1. Extract user alias (the mTLS gateway auth)
  │                                │  2. Check ACL (via the group membership service)
  │                                │  3. Register doc with router (affinity)
  │                                │  4. Load doc (DocManager::get_or_create)
  │◀── 101 Switching Protocols ────│
  │                                │
  │◀── SyncStep1 (state vector) ──│  Server sends its state fingerprint
  │── SyncStep2 (missing diff) ──▶│  Client sends what server is missing
  │◀── SyncStep2 (missing diff) ──│  Server sends what client is missing
  │◀── SyncDone ──────────────────│  Initial sync complete
  │                                │
  │◀─▶ Update (incremental) ──────│  Bidirectional CRDT updates
  │◀─▶ Awareness ────────────────▶│  Cursor positions, selection
  │── AiRequest ─────────────────▶│  User triggers AI
  │◀── AiResponse / AiEdits ──────│  AI streams back
  │                                │
  │── close ─────────────────────▶│  Cleanup: remove presence, dec editor count
```

### Binary Protocol

All messages are binary frames with a 1-byte tag prefix:

| Tag | Name | Direction | Payload |
|-----|------|-----------|---------|
| `0x01` | SyncStep1 | S→C | Yrs state vector |
| `0x02` | SyncStep2 | Bidi | Yrs update (diff) |
| `0x03` | Update | Bidi | Yrs incremental update |
| `0x04` | Awareness | Bidi | JSON (cursor pos, selection, user info) |
| `0x05` | Typing | C→S | 1 byte: 0=stopped, 1=typing |
| `0x06` | Comment | S→C | JSON comment event (create/update/delete) |
| `0x10` | AiRequest | C→S | JSON (prompt, model selection) |
| `0x11` | AiResponse | S→C | JSON (streaming AI text chunks) |
| `0x12` | FormatOp | Bidi | JSON (cell formatting operation) |
| `0x13` | FormatResolve | S→C | JSON (conflict resolution result) |
| `0x14` | AiEdits | S→C | JSON (proposed edits for user consent) |
| `0x15` | UpdateAck | S→C | Acknowledgment of applied update |
| `0x16` | Reveal | S→C | JSON (who's currently in the doc) |
| `0x17` | SyncDone | S→C | Empty; signals initial sync is complete |
| `0x18` | Title | S→C | UTF-8 new title bytes |
| `0x19` | Mention | C→S | JSON `{"alias":"<user>"}` |
| `0x1A` | WriteRejected | S→C | JSON `{"kind":"...","message":"..."}` |
| `0x1B` | DocReset | S→C | Empty; client must drop cache and reload |

### Message Flow for an Edit

1. User types in the browser → TipTap/ProseMirror generates a transaction
2. Yjs client encodes the transaction as a binary update
3. Client sends `[0x03][update bytes]` over WebSocket
4. Server receives, calls `CollabDoc::apply_update_from()`
5. If `ApplyOutcome::Applied`:
   - Publish to Redis PUBLISH (`doc:{id}:crdt` channel)
   - Append to Redis Stream (durable catch-up)
   - Notify snapshot scheduler (dirty doc)
6. Other nodes subscribed to the Redis channel receive the update
7. Those nodes forward it to their locally-connected clients

### Presence & Awareness

Awareness messages carry JSON with cursor position, selection range, user color, and display name. These flow through a separate Redis pub/sub channel (`doc:{id}:awareness`) so cursor jitter doesn't contend with CRDT updates.

The `PresenceTracker` (Redis HASH + TTL) maintains a per-doc set of active users. When a WebSocket disconnects, the user is removed from presence.

### Rate Limiting

Per-user, per-type sliding window counters in Redis:

| Type | Limit | Window |
|------|-------|--------|
| WebSocket messages | 1000 | 60s |
| AI requests | 20 | 60s |
| HTTP API calls | 100 | 60s |
| Media uploads | 100 | 60s |

Exceeding a limit returns a `429 Too Many Requests` or drops the WS message silently (for CRDT updates — the client will resync).

### Connection Teardown

On disconnect (graceful close or timeout):

1. Remove user from Redis presence set
2. Decrement editor count in snapshot scheduler
3. If editor count reaches 0, doc becomes eligible for idle eviction
4. Unsubscribe from Redis pub/sub channels

## Why WebSockets?

- **Low latency**: Sub-50ms round-trip for edits (no HTTP overhead per operation)
- **Server push**: The server can stream AI responses, comment events, title changes, and presence updates without polling
- **Binary protocol**: Yrs updates are binary blobs; WebSocket binary frames avoid base64 encoding overhead
- **Connection state**: The server tracks which docs each connection is editing, enabling efficient fan-out

The alternative (HTTP long-polling or Server-Sent Events) would add latency and complexity for bidirectional flows like CRDT sync.
