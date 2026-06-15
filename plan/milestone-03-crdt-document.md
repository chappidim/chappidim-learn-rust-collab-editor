# Milestone 3: CRDT Document

## Goal

Replace the plain-text broadcast with a **conflict-free replicated data type (CRDT)**. Two clients edit the same document simultaneously, and their changes merge without conflicts. This is the conceptual core of the entire project.

## What You're Building

- A shared `yrs::Doc` on the server that multiple clients sync with
- The Yjs sync protocol over WebSocket (SyncStep1 → SyncStep2 → incremental Updates)
- Two browser textareas that stay in sync regardless of edit order

## Rust Concepts Introduced

| Concept | Where You'll Encounter It |
|---------|--------------------------|
| Binary data (`&[u8]`, `Vec<u8>`) | Yrs updates are binary, not JSON |
| `yrs` crate API | `Doc`, `Text`, `Transact`, `Update::decode` |
| Pattern matching on bytes | Dispatching on message tag (first byte) |
| Interior mutability in async | `Arc<RwLock<Doc>>` for the shared document |
| Error propagation | Decoding failures, apply failures |

## Background: What is a CRDT?

A CRDT guarantees that if two replicas see the same set of operations (in any order), they converge to the same state. No central sequencer needed.

**Yrs** (the Rust port of Yjs) implements a text CRDT where:
- Each character has a unique ID (client_id + sequence_number)
- Inserts reference "to the left of" another character
- Deletes are tombstones (the character stays in memory but is hidden)
- Any two concurrent edits merge deterministically

## Step-by-Step

### 1. Add the `yrs` crate

```toml
[dependencies]
yrs = "0.21"
```

### 2. Define the sync protocol

Create `src/protocol.rs`:

```rust
/// Message tags — first byte of every WebSocket binary frame.
pub const MSG_SYNC_STEP_1: u8 = 0x01;  // Server → Client: "here's my state vector"
pub const MSG_SYNC_STEP_2: u8 = 0x02;  // Bidi: "here's what you're missing"
pub const MSG_UPDATE: u8 = 0x03;       // Bidi: incremental update

/// Parse the first byte and return (tag, payload).
pub fn parse_message(data: &[u8]) -> Option<(u8, &[u8])> {
    if data.is_empty() {
        return None;
    }
    Some((data[0], &data[1..]))
}

/// Prepend a tag byte to a payload.
pub fn encode_message(tag: u8, payload: &[u8]) -> Vec<u8> {
    let mut msg = Vec::with_capacity(1 + payload.len());
    msg.push(tag);
    msg.extend_from_slice(payload);
    msg
}
```

### 3. Create the document wrapper

Create `src/document.rs`:

```rust
use std::sync::Arc;
use tokio::sync::RwLock;
use yrs::{Doc, Text, Transact, ReadTxn, updates::decoder::Decode, updates::encoder::Encode};

pub struct CollabDoc {
    doc: Doc,
}

impl CollabDoc {
    pub fn new() -> Self {
        Self { doc: Doc::new() }
    }

    /// Get the document's state vector (compact fingerprint of what we have).
    pub fn state_vector(&self) -> Vec<u8> {
        let txn = self.doc.transact();
        txn.state_vector().encode_v1()
    }

    /// Compute the diff between our state and a remote state vector.
    /// Returns the updates the remote is missing.
    pub fn encode_diff(&self, remote_sv: &[u8]) -> Result<Vec<u8>, String> {
        let sv = yrs::StateVector::decode_v1(remote_sv)
            .map_err(|e| format!("bad state vector: {e}"))?;
        let txn = self.doc.transact();
        Ok(txn.encode_diff_v1(&sv))
    }

    /// Apply a binary update from a remote peer.
    pub fn apply_update(&self, update: &[u8]) -> Result<(), String> {
        let update = yrs::Update::decode_v1(update)
            .map_err(|e| format!("bad update: {e}"))?;
        self.doc.transact_mut().apply_update(update)
            .map_err(|e| format!("apply failed: {e}"))
    }

    /// Get the current text content (for debugging).
    pub fn get_text(&self) -> String {
        let txn = self.doc.transact();
        let text = txn.get_or_insert_text("content");
        text.get_string(&txn)
    }
}

pub type SharedDoc = Arc<RwLock<CollabDoc>>;
```

### 4. Implement the sync protocol in the WebSocket handler

```rust
use crate::protocol::*;
use crate::document::SharedDoc;

async fn handle_socket(socket: WebSocket, doc: SharedDoc, tx: BroadcastTx) {
    let (mut sender, mut receiver) = socket.split();
    let mut rx = tx.subscribe();

    // --- Initial sync: send our state vector so client knows what to send us ---
    {
        let d = doc.read().await;
        let sv = d.state_vector();
        let msg = encode_message(MSG_SYNC_STEP_1, &sv);
        let _ = sender.send(Message::Binary(msg.into())).await;
    }

    // --- Send task: forward broadcast updates to this client ---
    let send_doc = doc.clone();
    let mut send_task = tokio::spawn(async move {
        while let Ok(update) = rx.recv().await {
            let msg = encode_message(MSG_UPDATE, &update);
            if sender.send(Message::Binary(msg.into())).await.is_err() {
                break;
            }
        }
    });

    // --- Recv task: process incoming messages from this client ---
    let recv_doc = doc.clone();
    let recv_tx = tx.clone();
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(Message::Binary(data))) = receiver.next().await {
            let data = data.to_vec();
            if let Some((tag, payload)) = parse_message(&data) {
                match tag {
                    MSG_SYNC_STEP_1 => {
                        // Client sent its state vector; respond with our diff
                        let d = recv_doc.read().await;
                        if let Ok(diff) = d.encode_diff(payload) {
                            // Send back as SyncStep2 (we'd need sender here;
                            // in practice use a channel back to the send task)
                            let _ = recv_tx.send(diff);
                        }
                    }
                    MSG_SYNC_STEP_2 | MSG_UPDATE => {
                        // Client sent an update; apply to our doc
                        let d = recv_doc.write().await;
                        if d.apply_update(payload).is_ok() {
                            // Broadcast to other clients
                            let _ = recv_tx.send(payload.to_vec());
                        }
                    }
                    _ => {} // unknown tag, ignore
                }
            }
        }
    });

    tokio::select! {
        _ = &mut send_task => recv_task.abort(),
        _ = &mut recv_task => send_task.abort(),
    }
}
```

### 5. Client-side (use Yjs library)

```html
<!DOCTYPE html>
<html>
<body>
  <h1>CRDT Collaborative Editor</h1>
  <textarea id="editor" rows="20" cols="80"></textarea>
  <script src="https://cdn.jsdelivr.net/npm/yjs/dist/yjs.js"></script>
  <script>
    const Y = window.yjs || window.Y;
    // For a real implementation, use y-websocket provider
    // This is simplified — see exercises below
  </script>
</body>
</html>
```

**Practical approach**: Use the `y-websocket` npm package on the client side. It implements the exact SyncStep1/SyncStep2/Update protocol your server speaks. You just need to match the wire format.

### 6. Test it

Open two tabs. Type in one. The other should update in real time — even if both type simultaneously in different positions.

## The "Aha" Moment

Try this:
1. Tab A: type "Hello" at position 0
2. Tab B: simultaneously type "World" at position 0
3. Result: both tabs show "HelloWorld" (or "WorldHello") — but BOTH show the SAME thing

No conflicts. No last-write-wins. No lost edits. That's the CRDT guarantee.

## How the Sync Protocol Works

```
Server                          Client
  │                                │
  │── SyncStep1(my_state_vec) ───▶│  "Here's what I have"
  │                                │
  │◀── SyncStep2(your_diff) ──────│  "Here's what you're missing"
  │                                │  (computed from my_state_vec)
  │                                │
  │── SyncStep2(my_diff) ────────▶│  "And here's what YOU'RE missing"
  │                                │  (computed from client's state_vec,
  │                                │   which was in the SyncStep1 it sent us)
  │                                │
  │◀─▶ Update(incremental) ──────▶│  All subsequent edits
```

After the initial sync handshake, both sides have identical state. From then on, every edit is sent as a small incremental `Update`.

## Exercises

1. **Multiple documents**: Add a `doc_id` path parameter. Maintain a `HashMap<String, SharedDoc>`. Each WebSocket room syncs a different doc.
2. **State inspection**: Add a `GET /docs/{id}/text` endpoint that returns the current plain text of a doc (for debugging).
3. **Update counting**: Track how many updates each doc has received (this leads into the snapshot scheduler concept in Milestone 4).
4. **Use y-websocket**: Instead of building the client from scratch, use `y-websocket`'s `WebsocketProvider`. Match your server's wire format to what it expects.

## How a Production System Does This

The reference system's `CollabDoc` (`src/crdt/mod.rs`) is this same pattern plus:
- **Structural validation**: Rejects updates that make tables too large
- **Rate limiting**: Per-connection throttle on structural changes
- **FormatOps sidecar**: Cell-level formatting stored outside Yrs
- **XmlFragment** instead of Text: Rich document structure (not just plain text)
- **ApplyOutcome enum**: Tells the caller what happened (applied, rejected, internal error)

But the core — `Doc`, `apply_update`, `encode_state_vector`, `encode_diff` — is exactly what you're building here.

## Rust Book Chapters to Read

- [Chapter 8: Common Collections](https://doc.rust-lang.org/book/ch08-00-common-collections.html) (Vec, HashMap)
- [Chapter 9: Error Handling](https://doc.rust-lang.org/book/ch09-00-error-handling.html) (Result, ?)
- [Chapter 15: Smart Pointers](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html) (Arc, Box)

## External Resources

- [Yrs documentation](https://docs.rs/yrs/latest/yrs/)
- [Yjs protocol specification](https://github.com/yjs/y-protocols)
- [CRDT explainer (interactive)](https://crdt.tech/)

## Done When

- [ ] Two browser tabs edit the same text and stay in sync
- [ ] Concurrent edits in different positions merge correctly (no data loss)
- [ ] Server restart loses state (expected — persistence is Milestone 4)
- [ ] `cargo clippy` passes
