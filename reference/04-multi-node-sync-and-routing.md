# Multi-Node Sync & Routing

## What It Is

the production system runs multiple container orchestration tasks behind an the load balancer. Since any task can receive a WebSocket connection for any document, the system needs a mechanism to:

1. Route all connections for a given doc to the same node (affinity)
2. Propagate CRDT updates between nodes when affinity fails
3. Allow late-joining nodes to catch up on missed updates

Source: `src/routing/mod.rs`, `src/sync/mod.rs`, `src/sync/redis.rs`, `src/sync/streams.rs`

## How It Works

### Doc-Affinity Routing

```
                     the load balancer (round-robin)
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
          Node A       Node B       Node C
              │
              │  First connection for doc "abc":
              │  1. Check Redis: who owns "abc"?
              │  2. Nobody → claim it (SET doc:abc → node-A, TTL 60s)
              │  3. Load doc from the object store into memory
              │
              │  Second connection for doc "abc" hits Node B:
              │  1. Check Redis: who owns "abc"? → Node A
              │  2. (Future: proxy to Node A. Today: co-load)
```

The `DocRouter` trait:

```rust
trait DocRouter: Send + Sync {
    async fn register_doc(&self, doc_id, node_id) -> Result<()>;
    async fn lookup_node(&self, doc_id) -> Result<Option<NodeId>>;
    async fn heartbeat(&self, node_id) -> Result<()>;
    async fn unregister_node(&self, node_id) -> Result<()>;
}
```

**Heartbeat**: Every 30 seconds, each node refreshes its Redis TTL. If a node dies without graceful shutdown, its keys expire within 60s and other nodes can reclaim its docs.

**Unregister**: On graceful shutdown, the node explicitly removes all its routing entries so failover is instant (no TTL wait).

### CRDT Broadcast (Redis Pub/Sub)

Every accepted CRDT update is published to `doc:{id}:crdt`:

```
Node A (owner)              Redis              Node B (co-loaded)
     │                        │                       │
     │── PUBLISH doc:abc ───▶│                       │
     │                        │── message ──────────▶│
     │                        │                       │── apply to local doc
```

This is **fire-and-forget**: if Node B isn't subscribed (doc not loaded there), the message is simply lost. That's fine — Node B will catch up from the stream or object store when it next loads the doc.

The `CrdtBroadcast` trait:

```rust
trait CrdtBroadcast: Send + Sync {
    async fn publish_update(&self, doc_id: &str, update: &[u8]) -> Result<()>;
    async fn subscribe(&self, doc_id: &str) -> Result<broadcast::Receiver<Vec<u8>>>;
}
```

### CRDT Streams (Durable Catch-Up)

Redis Pub/Sub is ephemeral — if a node wasn't subscribed at publish time, the message is gone. Redis Streams solve this:

```
Node A writes update #42
  │
  ├── PUBLISH doc:abc:crdt (real-time)
  └── XADD doc:abc:stream (durable)

Node B loads doc "abc" 5 minutes later:
  1. Load object store snapshot (state as of update #38)
  2. XREAD doc:abc:stream after snapshot-entry-id
  3. Apply updates #39, #40, #41, #42
  4. Subscribe to pub/sub for future updates
```

The `CrdtStream` trait:

```rust
trait CrdtStream: Send + Sync {
    async fn append(&self, doc_id: &str, payload: &[u8]) -> Result<String>;
    async fn read_after(&self, doc_id: &str, from_id: &str) -> Result<Vec<(String, Vec<u8>)>>;
    async fn trim(&self, doc_id: &str, max_len: usize) -> Result<()>;
}
```

Streams are **optional** (gated by `APP_STREAMS_CATCHUP=true`). Without them, catch-up relies solely on object store snapshots (which may be up to 30 seconds stale).

### Multiple Pub/Sub Channels Per Doc

Each doc uses several channels for different message types:

| Channel | Content |
|---------|---------|
| `doc:{id}:crdt` | Binary CRDT updates |
| `doc:{id}:awareness` | Cursor/presence JSON |
| `doc:{id}:comments` | Comment create/update/delete events |
| `doc:{id}:title` | Title change notifications |
| `doc:{id}:reset` | Doc reset signal (rollback) |
| `doc:{id}:format` | Cell formatting operations |

Separating channels prevents cursor noise from delaying CRDT delivery and allows selective subscription.

## Why This Design?

### Why Not Sticky Sessions?

the load balancer sticky sessions (cookie-based) don't help because:
- Different users editing the same doc may be on different machines
- WebSocket connections can outlive the load balancer's session cookie TTL
- Failover on node death would still require state transfer

### Why Redis Over Kafka/message queue?

- **Latency**: Redis pub/sub delivers in <1ms. Kafka/message queue add 10-100ms.
- **Simplicity**: Redis is already needed for presence and rate limiting.
- **Ephemeral is fine**: CRDT updates are idempotent and object store is the durable store. Missing a pub/sub message is recoverable.

### Why Both Pub/Sub AND Streams?

- **Pub/Sub alone**: Fast but lossy. A node that loads a doc misses everything published before it subscribed.
- **Streams alone**: Durable but higher latency (polling). Not suitable for sub-50ms sync.
- **Both**: Real-time delivery via pub/sub, gap recovery via streams. Best of both worlds.

### Why Doc-Affinity At All?

Without affinity, every CRDT update for a popular doc would need to be applied on N nodes simultaneously. With affinity, most operations stay local to one node, reducing Redis traffic by ~Nx. The few cases where a doc is co-loaded (the load balancer split, failover) are handled gracefully by the broadcast mechanism.
