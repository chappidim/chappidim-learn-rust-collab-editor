# Milestone 1: Hello Axum

## Goal

Build a minimal HTTP server in Rust that serves JSON endpoints and static files. This milestone establishes your project scaffold and teaches the foundational Rust concepts you'll use in every subsequent milestone.

## What You're Building

A Rust HTTP server with:
- `GET /health` → returns `{"status": "ok"}`
- `POST /docs` → accepts a JSON body `{"title": "..."}`, generates an ID, returns `{"id": "...", "title": "..."}`
- `GET /docs` → returns a list of all created docs (in-memory, lost on restart)
- A static `index.html` served at `/`

## Rust Concepts Introduced

| Concept | Where You'll Encounter It |
|---------|--------------------------|
| Ownership & borrowing | Passing request bodies to handlers |
| `Result<T, E>` | Error handling in handlers |
| `async`/`await` | Every Axum handler is async |
| Serde derive macros | `#[derive(Serialize, Deserialize)]` on request/response types |
| Module system | Splitting handlers into `mod routes` |
| `Arc<Mutex<T>>` | Sharing state (doc list) across handler calls |

## Step-by-Step

### 1. Create the project

```bash
# From the repo root (already has plan/ and reference/)
cargo init
```

This creates `Cargo.toml` and `src/main.rs` alongside your existing folders.

### 2. Add dependencies to `Cargo.toml`

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tower-http = { version = "0.6", features = ["fs"] }
uuid = { version = "1", features = ["v4"] }
```

### 3. Write the server (`src/main.rs`)

```rust
use axum::{
    extract::State,
    routing::{get, post},
    Json, Router,
};
use serde::{Deserialize, Serialize};
use std::sync::{Arc, Mutex};
use uuid::Uuid;

// --- Types ---

#[derive(Clone, Serialize)]
struct DocMeta {
    id: String,
    title: String,
}

#[derive(Deserialize)]
struct CreateDocRequest {
    title: String,
}

// --- Shared state ---

type AppState = Arc<Mutex<Vec<DocMeta>>>;

// --- Handlers ---

async fn health() -> Json<serde_json::Value> {
    Json(serde_json::json!({"status": "ok"}))
}

async fn create_doc(
    State(state): State<AppState>,
    Json(req): Json<CreateDocRequest>,
) -> Json<DocMeta> {
    let doc = DocMeta {
        id: Uuid::new_v4().to_string(),
        title: req.title,
    };
    state.lock().unwrap().push(doc.clone());
    Json(doc)
}

async fn list_docs(State(state): State<AppState>) -> Json<Vec<DocMeta>> {
    let docs = state.lock().unwrap().clone();
    Json(docs)
}

// --- Main ---

#[tokio::main]
async fn main() {
    let state: AppState = Arc::new(Mutex::new(Vec::new()));

    let app = Router::new()
        .route("/health", get(health))
        .route("/docs", post(create_doc).get(list_docs))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    println!("Listening on http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

### 4. Test it

```bash
cargo run &

# Health check
curl http://localhost:3000/health

# Create a doc
curl -X POST http://localhost:3000/docs \
  -H "Content-Type: application/json" \
  -d '{"title": "My First Doc"}'

# List docs
curl http://localhost:3000/docs
```

### 5. Add static file serving

Create `static/index.html`:
```html
<!DOCTYPE html>
<html>
<body>
  <h1>Collab Editor</h1>
  <p>Server is running!</p>
</body>
</html>
```

Add to your router:
```rust
use tower_http::services::ServeDir;

let app = Router::new()
    .route("/health", get(health))
    .route("/docs", post(create_doc).get(list_docs))
    .with_state(state)
    .fallback_service(ServeDir::new("static"));
```

## Exercises (stretch goals)

1. **Error handling**: Return a 400 if `title` is empty. Use `Result<Json<DocMeta>, StatusCode>` as the return type.
2. **Structured logging**: Add `tracing` and `tracing-subscriber`. Log each request.
3. **Module split**: Move handlers to `src/routes.rs` and types to `src/types.rs`.
4. **CORS**: Add `tower-http`'s `CorsLayer` so a browser can call your API.

## How a Production System Does This

In a production collaborative editor, the equivalent is:
- An `AppState` struct with ~15 store references instead of one `Vec`
- A large router assembly with many route groups (docs, folders, comments, media, etc.)
- Static file serving gated by an env var (production uses a CDN for the frontend)

The pattern is identical; production just has more state and more routes. If you understand this milestone, you understand the skeleton.

## Rust Book Chapters to Read

- [Chapter 4: Understanding Ownership](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)
- [Chapter 5: Structs](https://doc.rust-lang.org/book/ch05-00-structs.html)
- [Chapter 7: Modules](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html)

## Done When

- [ ] `cargo clippy` passes with no warnings
- [ ] `curl /health` returns JSON
- [ ] `curl -X POST /docs` creates a doc and `GET /docs` returns it
- [ ] `http://localhost:3000/` shows your HTML page
