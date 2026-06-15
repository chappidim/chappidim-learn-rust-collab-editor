# Milestone 4: Persistence

## Goal

Documents survive server restarts. This milestone introduces background tasks, trait-based abstractions, and the snapshot scheduling pattern that the production system uses to balance durability against performance.

## What You're Building

- Documents save to disk automatically (every N seconds or every M updates)
- On server start, existing documents load from disk
- A `SnapshotStore` trait that can later be swapped for object store
- Idle documents are evicted from memory after a timeout

## Rust Concepts Introduced

| Concept | Where You'll Encounter It |
|---------|--------------------------|
| Traits + `dyn Trait` | `SnapshotStore` abstraction |
| `tokio::time::interval` | Periodic flush task |
| `tokio::spawn` for background work | Scheduler runs independently of request handlers |
| `#[cfg(test)]` | Mock store for unit tests |
| Builder pattern | `DocManager::new(store).with_flush_interval(30)` |
| `Drop` / shutdown hooks | Flush on Ctrl+C |

## Step-by-Step

### 1. Define the storage trait

```rust
// src/storage.rs
use async_trait::async_trait;

#[async_trait]
pub trait SnapshotStore: Send + Sync {
    /// Save document state. Returns the storage key.
    async fn save(&self, doc_id: &str, data: &[u8]) -> Result<String, String>;

    /// Load the latest state for a document.
    async fn load(&self, doc_id: &str) -> Result<Option<Vec<u8>>, String>;

    /// List all document IDs that have saved state.
    async fn list_docs(&self) -> Result<Vec<String>, String>;
}
```

Add `async-trait = "0.1"` to your dependencies.

### 2. Implement a file-based store

```rust
// src/storage.rs (continued)
pub struct FileStore {
    dir: std::path::PathBuf,
}

impl FileStore {
    pub fn new(dir: impl Into<std::path::PathBuf>) -> Self {
        let dir = dir.into();
        std::fs::create_dir_all(&dir).expect("failed to create storage dir");
        Self { dir }
    }
}

#[async_trait]
impl SnapshotStore for FileStore {
    async fn save(&self, doc_id: &str, data: &[u8]) -> Result<String, String> {
        let path = self.dir.join(format!("{doc_id}.bin"));
        tokio::fs::write(&path, data).await
            .map_err(|e| format!("write failed: {e}"))?;
        Ok(path.to_string_lossy().to_string())
    }

    async fn load(&self, doc_id: &str) -> Result<Option<Vec<u8>>, String> {
        let path = self.dir.join(format!("{doc_id}.bin"));
        match tokio::fs::read(&path).await {
            Ok(data) => Ok(Some(data)),
            Err(e) if e.kind() == std::io::ErrorKind::NotFound => Ok(None),
            Err(e) => Err(format!("read failed: {e}")),
        }
    }

    async fn list_docs(&self) -> Result<Vec<String>, String> {
        let mut docs = Vec::new();
        let mut entries = tokio::fs::read_dir(&self.dir).await
            .map_err(|e| format!("read_dir failed: {e}"))?;
        while let Some(entry) = entries.next_entry().await
            .map_err(|e| format!("entry failed: {e}"))? {
            if let Some(name) = entry.file_name().to_str() {
                if let Some(id) = name.strip_suffix(".bin") {
                    docs.push(id.to_string());
                }
            }
        }
        Ok(docs)
    }
}
```

### 3. Build the DocManager with hydration

```rust
// src/doc_manager.rs
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::RwLock;
use crate::document::CollabDoc;
use crate::storage::SnapshotStore;

pub struct DocManager {
    docs: RwLock<HashMap<String, Arc<CollabDoc>>>,
    store: Arc<dyn SnapshotStore>,
}

impl DocManager {
    pub fn new(store: Arc<dyn SnapshotStore>) -> Self {
        Self {
            docs: RwLock::new(HashMap::new()),
            store,
        }
    }

    pub async fn get_or_create(&self, doc_id: &str) -> Result<Arc<CollabDoc>, String> {
        // Fast path: already loaded
        {
            let docs = self.docs.read().await;
            if let Some(doc) = docs.get(doc_id) {
                return Ok(Arc::clone(doc));
            }
        }

        // Slow path: load from storage or create new
        let mut docs = self.docs.write().await;
        // Double-check (another task may have loaded it)
        if let Some(doc) = docs.get(doc_id) {
            return Ok(Arc::clone(doc));
        }

        let doc = CollabDoc::new();
        if let Some(data) = self.store.load(doc_id).await? {
            doc.apply_update(&data)?;
        }
        let doc = Arc::new(doc);
        docs.insert(doc_id.to_string(), Arc::clone(&doc));
        Ok(doc)
    }

    /// Flush a single doc to storage.
    pub async fn flush(&self, doc_id: &str) -> Result<(), String> {
        let doc = {
            let docs = self.docs.read().await;
            docs.get(doc_id).cloned()
        };
        if let Some(doc) = doc {
            let data = doc.encode_state();
            self.store.save(doc_id, &data).await?;
        }
        Ok(())
    }

    /// Flush all loaded docs.
    pub async fn flush_all(&self) {
        let doc_ids: Vec<String> = {
            let docs = self.docs.read().await;
            docs.keys().cloned().collect()
        };
        for id in doc_ids {
            if let Err(e) = self.flush(&id).await {
                eprintln!("flush error for {id}: {e}");
            }
        }
    }
}
```

### 4. Add the background scheduler

```rust
// In main.rs
let mgr = Arc::new(DocManager::new(Arc::new(FileStore::new("./data"))));

// Background flush every 30 seconds
let flush_mgr = Arc::clone(&mgr);
tokio::spawn(async move {
    let mut interval = tokio::time::interval(Duration::from_secs(30));
    loop {
        interval.tick().await;
        flush_mgr.flush_all().await;
    }
});
```

### 5. Graceful shutdown (flush on exit)

```rust
// In main.rs
let shutdown_mgr = Arc::clone(&mgr);
axum::serve(listener, app)
    .with_graceful_shutdown(async move {
        tokio::signal::ctrl_c().await.ok();
        println!("Shutting down, flushing all docs...");
        shutdown_mgr.flush_all().await;
        println!("Done.");
    })
    .await
    .unwrap();
```

### 6. Test it

```bash
cargo run
# Open browser, edit a doc, wait 30 seconds (or Ctrl+C)
# Check ./data/ — you should see .bin files

cargo run  # restart
# Open browser again — your edits are still there!
```

## Key Design Decisions

### Why Not Save on Every Edit?

At 5 keystrokes/second, that's 5 file writes/second per doc. With 10 docs open, that's 50 IOPS — fine for local files, but unsustainable for object store ($0.005 per 1000 PUTs × 50/sec = $21/hour).

Batching to every 30 seconds: 10 docs × 1 write/30s = 0.33 writes/second. 450x cheaper.

The tradeoff: up to 30 seconds of data can be lost on a hard crash. For a collaborative editor (not a bank), this is acceptable.

### Why a Trait (Not Just `FileStore`)?

When you move to the object store (production) you'll want:
- `FileStore` for local dev
- `object storeStore` for production
- `InMemoryStore` for unit tests

The trait lets you swap implementations without touching business logic.

### Why Double-Check in `get_or_create`?

Between releasing the read lock and acquiring the write lock, another task might have loaded the same doc. The double-check prevents creating two `CollabDoc` instances for the same document.

## Exercises

1. **Update counting**: Only flush when `updates_since_flush >= 50` (not just on timer). Combine both triggers.
2. **Idle eviction**: If no one has accessed a doc in 5 minutes, remove it from the HashMap. Load from disk on next access.
3. **InMemoryStore**: Write a `HashMap`-based store for tests. Write a unit test that creates a doc, flushes, and reloads.
4. **Version history**: Save with incrementing sequence numbers (`{doc_id}/{seq}.bin`). Keep the last 10 versions.

## How a Production System Does This

The reference system's `SnapshotScheduler` (`src/persistence/snapshot_scheduler.rs`) is this pattern at scale:
- **object store** instead of local files (cross-node durability)
- **the database pointer** to the latest object store key (fast lookup without listing object store)
- **Adaptive version interval**: 30s with 4+ editors, 2 min with 1 editor
- **Title extraction**: Reads the first H1 from the CRDT and updates the database metadata
- **Search indexer trigger**: Re-indexes content in the search engine after each flush
- **Editor count tracking**: Docs with 0 editors become eligible for eviction

The DocManager hydration path (`get_or_create`) is nearly identical to what you're building here.

## Rust Book Chapters to Read

- [Chapter 10: Generic Types, Traits, and Lifetimes](https://doc.rust-lang.org/book/ch10-00-generics.html)
- [Chapter 17: Object-Oriented Features (Trait Objects)](https://doc.rust-lang.org/book/ch17-02-trait-objects.html)
- [Tokio Tutorial: I/O](https://tokio.rs/tokio/tutorial/io)

## Done When

- [ ] Documents persist across server restarts
- [ ] Background scheduler flushes dirty docs every 30 seconds
- [ ] Ctrl+C flushes all docs before exiting
- [ ] A `SnapshotStore` trait exists with at least `FileStore` implementing it
- [ ] `cargo clippy` passes
