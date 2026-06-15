# Milestone 2: WebSocket Echo & Broadcast

## Goal

Add WebSocket support so two browser tabs can communicate in real time through the server. This teaches Rust's concurrency primitives — channels, Arc, and task spawning — which are the backbone of any real-time system.

## What You're Building

- `GET /ws` → upgrades to WebSocket
- Any message sent by one client is broadcast to ALL other connected clients
- A simple HTML page with a text input that sends messages and displays received ones

## Rust Concepts Introduced

| Concept | Where You'll Encounter It |
|---------|--------------------------|
| `tokio::sync::broadcast` | Fan-out messages to all clients |
| `tokio::spawn` | Separate read/write loops per connection |
| `Arc` without `Mutex` | Sharing the broadcast sender (it's already thread-safe) |
| Splitting a WebSocket | `socket.split()` → separate `SinkExt` + `StreamExt` |
| Lifetimes with async | Why `'static` bounds appear on spawned tasks |
| `select!` macro | Waiting on multiple futures simultaneously |

## Step-by-Step

### 1. Add WebSocket dependency

```toml
[dependencies]
# ... existing deps from milestone 1 ...
futures = "0.3"   # for SinkExt, StreamExt
```

Axum 0.8 includes WebSocket support in `axum::extract::ws`.

### 2. Create the broadcast channel

In `main()`:

```rust
use tokio::sync::broadcast;

// 100 = channel capacity (backpressure if a client is slow)
let (tx, _rx) = broadcast::channel::<String>(100);
```

Pass `tx` as state to the WebSocket handler.

### 3. Write the WebSocket upgrade handler

```rust
use axum::extract::ws::{Message, WebSocket, WebSocketUpgrade};
use axum::extract::State;
use axum::response::IntoResponse;
use futures::{SinkExt, StreamExt};

type BroadcastTx = broadcast::Sender<String>;

async fn ws_handler(
    ws: WebSocketUpgrade,
    State(tx): State<BroadcastTx>,
) -> impl IntoResponse {
    ws.on_upgrade(move |socket| handle_socket(socket, tx))
}

async fn handle_socket(socket: WebSocket, tx: BroadcastTx) {
    let (mut sender, mut receiver) = socket.split();
    let mut rx = tx.subscribe();

    // Task 1: forward broadcast messages → this client
    let mut send_task = tokio::spawn(async move {
        while let Ok(msg) = rx.recv().await {
            if sender.send(Message::Text(msg)).await.is_err() {
                break; // client disconnected
            }
        }
    });

    // Task 2: read from this client → broadcast to all
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(Message::Text(text))) = receiver.next().await {
            // Ignore errors (means no active receivers)
            let _ = tx.send(text.to_string());
        }
    });

    // If either task finishes, abort the other
    tokio::select! {
        _ = &mut send_task => recv_task.abort(),
        _ = &mut recv_task => send_task.abort(),
    }
}
```

### 4. Wire it into the router

```rust
let tx_clone = tx.clone();
let app = Router::new()
    .route("/health", get(health))
    .route("/ws", get(ws_handler))
    .with_state(tx_clone)
    .fallback_service(ServeDir::new("static"));
```

### 5. Create a test client (`static/index.html`)

```html
<!DOCTYPE html>
<html>
<body>
  <h1>Collab Editor - WebSocket Test</h1>
  <input id="input" placeholder="Type a message..." />
  <button onclick="send()">Send</button>
  <div id="messages"></div>

  <script>
    const ws = new WebSocket(`ws://${location.host}/ws`);
    const messages = document.getElementById('messages');

    ws.onmessage = (event) => {
      const div = document.createElement('div');
      div.textContent = event.data;
      messages.appendChild(div);
    };

    function send() {
      const input = document.getElementById('input');
      ws.send(input.value);
      input.value = '';
    }

    document.getElementById('input').addEventListener('keypress', (e) => {
      if (e.key === 'Enter') send();
    });
  </script>
</body>
</html>
```

### 6. Test it

```bash
cargo run
```

Open two browser tabs at `http://localhost:3000`. Type in one tab — the message should appear in both.

## Key Patterns to Understand

### Why `socket.split()`?

A WebSocket is bidirectional. You need to read AND write concurrently. Rust's ownership model won't let two tasks hold `&mut socket` simultaneously. `split()` gives you two independent halves — one for reading, one for writing — each owned by a separate task.

### Why `broadcast` (not `mpsc`)?

- `mpsc` (multi-producer, single-consumer): one receiver. Only one task can read messages.
- `broadcast` (multi-producer, multi-consumer): every subscriber gets every message. Perfect for "send to all connected clients."

### Why `tokio::select!`?

When a client disconnects, one of the two tasks (send or recv) will finish first. `select!` races them and lets you clean up the other. Without this, a dead client's send task would hang forever waiting for broadcast messages.

## Exercises

1. **Username**: Accept a `?name=alice` query param on the WebSocket upgrade. Prefix messages with the username.
2. **Connection count**: Track connected clients. Broadcast "alice joined" / "alice left" messages.
3. **Binary messages**: Change from `Message::Text` to `Message::Binary`. Send raw bytes and parse them on the client. (This prepares you for the CRDT protocol in Milestone 3.)
4. **Per-room broadcast**: Add a `doc_id` to the WebSocket path (`/ws/{doc_id}`). Only broadcast to clients in the same room.

## How a Production System Does This

The reference system's `src/ws/mod.rs` follows the same pattern:
- `ws_upgrade` handler accepts the upgrade with state
- `handle_socket` splits the socket and spawns read/write tasks
- Instead of `broadcast::channel`, it uses Redis PUBLISH/SUBSCRIBE (cross-node fan-out)
- Instead of plain text, it dispatches on the first byte (message tag) to handle CRDT updates, awareness, AI, comments, etc.

The broadcast channel in this milestone is conceptually identical to The reference system's `CrdtBroadcast` trait — just local instead of Redis-backed.

## Rust Book Chapters to Read

- [Chapter 16: Fearless Concurrency](https://doc.rust-lang.org/book/ch16-00-concurrency.html)
- [Tokio Tutorial: Channels](https://tokio.rs/tokio/tutorial/channels)
- [Tokio Tutorial: Select](https://tokio.rs/tokio/tutorial/select)

## Done When

- [ ] Two browser tabs can send messages to each other through the server
- [ ] Closing one tab doesn't crash the server
- [ ] Messages only go to OTHER clients (sender doesn't echo to itself — optional, but good UX)
- [ ] `cargo clippy` passes
