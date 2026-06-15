# Persistence Layer

## What It Is

The persistence layer is responsible for durably storing document content, metadata, and all associated resources (comments, versions, folders, etc.). It is split across two cloud services — object storage for large binary blobs and the database for structured metadata — connected by a background snapshot scheduler that periodically checkpoints in-memory state.

Source: `src/persistence/mod.rs`, `src/persistence/object_store.rs`, `src/persistence/database.rs`, `src/persistence/snapshot_scheduler.rs`

## How It Works

### Storage Split

```
┌─────────────────────────────────────────────────┐
│                 In-Memory                        │
│                                                 │
│  DocManager → HashMap<DocId, Arc<CollabDoc>>     │
│  (ground truth while doc is loaded)             │
└─────────────────────┬───────────────────────────┘
                      │ Snapshot Scheduler (every 30s)
                      ▼
┌──────────────────────────────┐    ┌─────────────────────────┐
│            object store                │    │        the database          │
│                              │    │                         │
│  {doc_id}/{seq}.bin          │    │  app-resources      │
│  (binary Yrs state)         │    │  ├── DOC#{id} (metadata)│
│                              │    │  ├── FLD#{id} (folders) │
│  feedback/{id}/envelope.json │    │  ├── CMT#{id} (comments)│
│  feedback/{id}/state.bin     │    │  └── ...               │
│                              │    │                         │
│  media/{hash} (blobs)        │    │  app-recents        │
│                              │    │  app-versions       │
└──────────────────────────────┘    │  app-notifications  │
                                    └─────────────────────────┘
```

### Snapshot Scheduler

The scheduler is the bridge between in-memory CRDT state and durable storage. It runs on a 30-second tick:

```
Every 30 seconds:
  for each registered doc:
    if updates_since_flush >= 100 OR timer expired:
      1. Encode the CollabDoc to binary (encode_state)
      2. Write to the object store: {doc_id}/{seq}.bin
      3. Update Database metadata: snapshot_key, updated_at, title
      4. Optionally record a VersionEntry (based on editor count):
         - 1 editor:  version every 2 min
         - 2-3 editors: version every 1 min
         - 4+ editors: version every 30s
      5. Trigger search re-index (if search is enabled)
      6. Reset updates_since_flush counter
```

Key design decisions:
- **UPDATE_THRESHOLD = 100**: Don't flush on every keystroke — batch for efficiency
- **Adaptive version interval**: More editors = more frequent versions (higher conflict risk)
- **Title extraction**: The scheduler reads the first H1 heading and updates Database metadata so the doc list stays current without loading every doc

### DocManager Hydration

When a document is first accessed:

```
DocManager::get_or_create("doc_abc")
  │
  ├── Fast path: already in HashMap? Return Arc<CollabDoc>
  │
  └── Slow path (write lock):
        1. Query the database for metadata → get snapshot_key
        2. Load binary blob from the object store at that key
        3. Create new yrs::Doc
        4. Apply the object store blob as a Yrs update (hydrate)
        5. If CRDT streams enabled: XREAD any updates after the snapshot
        6. Insert into HashMap, return Arc<CollabDoc>
```

### Idle Eviction

Every 60 seconds, the eviction loop:

1. Asks the scheduler: which docs have 0 editors AND haven't been flushed recently?
2. For those docs: one final snapshot flush
3. Removes them from DocManager's HashMap
4. Frees memory

This prevents unbounded memory growth. A doc that was popular an hour ago but has no one editing it gets unloaded.

### Version History

`VersionStore` (the database) records a timeline of snapshots:

```json
{
  "doc_id": "doc_abc",
  "seq": 42,
  "storage_key": "doc_abc/42.bin",
  "timestamp": 1718451200,
  "contributors": ["alice", "bob"],
  "title": "Design Review Notes"
}
```

The frontend can list versions and request a diff between any two. The rollback endpoint loads a historical snapshot and writes it as the new HEAD.

### Store Traits

Every store is a trait with `Send + Sync` bounds:

```rust
#[async_trait]
trait SnapshotStore: Send + Sync {
    async fn save(&self, doc_id: &str, seq: u64, data: &[u8]) -> Result<String, PersistenceError>;
    async fn load_latest(&self, doc_id: &str) -> Result<Option<(u64, Vec<u8>)>, PersistenceError>;
    async fn load_by_key(&self, key: &str) -> Result<Option<Vec<u8>>, PersistenceError>;
    async fn delete_all(&self, doc_id: &str) -> Result<(), PersistenceError>;
    async fn set_storage_class(&self, doc_id: &str, cold_storage: bool) -> Result<(), PersistenceError>;
}
```

Benefits:
- **Testability**: Unit tests use `NoopSnapshotStore` (in-memory HashMap)
- **Local dev**: Works without cloud credentials
- **Flexibility**: Could swap object storage for another blob store without touching business logic

## Why This Design?

### Why object storage for Snapshots (Not the database)?

- **Size**: A Yrs document state can be 1-10 MB (or more for large docs with history). the database items cap at 400 KB.
- **Cost**: object store is ~$0.023/GB/month vs the database's ~$0.25/GB/month for on-demand.
- **Lifecycle**: object store supports automatic cold-storage tiering for deleted/archived docs.

### Why the database for Metadata?

- **Low-latency queries**: Doc listings, folder contents, recent items all need fast indexed queries.
- **Conditional writes**: the database's `ConditionExpression` prevents concurrent metadata corruption.
- **GSIs**: The `parent-id-index` enables efficient folder-contents queries.

### Why Not Write Every Edit to the object store?

At 5 edits/second from 3 users, that's 15 object store writes/second per doc. With 100 active docs, that's 1500 PUTs/second = $5.40/hour. The 30-second batch window reduces this to ~3 PUTs/second total, saving >99% on object store costs while keeping data loss window under 30 seconds.

### Why the In-Memory Model?

The alternative is a stateless design where every CRDT update reads from the object store, applies, and writes back. This would:
- Add 50-200ms latency per edit (object store round-trip)
- Require locking to prevent concurrent writes from diverging
- Make real-time collaboration impossible

The in-memory model gives sub-millisecond apply times and relies on the snapshot scheduler for durability. The worst-case data loss window is ~30 seconds (last snapshot to crash).
