# Milestone 6: Multi-User Features

## Goal

Add user identity, permissions, document listing, and presence tracking. Transform the prototype into something that feels like a real multi-user product with access control.

## What You're Building

- Authentication middleware (simple header-based or token-based)
- Document metadata: owner, title, created_at, updated_at
- Permission model: owner (full access), editor (read+write), viewer (read-only)
- REST API: list my docs, create doc, get doc metadata, update doc metadata
- Presence: who's currently in a document

## Rust Concepts Introduced

| Concept | Where You'll Encounter It |
|---------|--------------------------|
| Axum middleware | Auth layer that runs before every handler |
| Axum extractors | Custom `UserAlias` extractor from headers |
| `dyn Trait` objects | `Arc<dyn MetadataStore>` for different storage backends |
| Enums with data | `PermissionLevel::Owner / Editor / Viewer` |
| Conditional compilation | `#[cfg(test)]` for mock stores |
| `impl FromRequestParts` | Teaching Axum how to extract your custom types |

## Step-by-Step

### 1. Define the user extractor

```rust
// src/auth.rs
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode, HeaderMap},
};

pub struct UserAlias(pub String);

impl<S: Send + Sync> FromRequestParts<S> for UserAlias {
    type Rejection = StatusCode;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        // In production: validate a token or read from a proxy header
        // For learning: read from a simple header
        parts.headers
            .get("x-user")
            .and_then(|v| v.to_str().ok())
            .map(|s| UserAlias(s.to_string()))
            .ok_or(StatusCode::UNAUTHORIZED)
    }
}
```

### 2. Add auth middleware

```rust
// src/auth.rs (continued)
use axum::{middleware::Next, response::Response, body::Body, extract::Request};

pub async fn require_auth(
    headers: HeaderMap,
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    // Skip auth for health check
    if request.uri().path() == "/health" {
        return Ok(next.run(request).await);
    }

    let has_user = headers.get("x-user")
        .and_then(|v| v.to_str().ok())
        .filter(|s| !s.is_empty())
        .is_some();

    if has_user {
        Ok(next.run(request).await)
    } else {
        Err(StatusCode::UNAUTHORIZED)
    }
}
```

Wire it in:
```rust
let app = Router::new()
    // ... routes ...
    .layer(middleware::from_fn(require_auth));
```

### 3. Define document metadata

```rust
// src/types.rs
use serde::{Deserialize, Serialize};

#[derive(Clone, Serialize, Deserialize)]
pub struct DocMetadata {
    pub id: String,
    pub title: String,
    pub owner: String,
    pub created_at: u64,  // unix timestamp
    pub updated_at: u64,
    pub parent_id: Option<String>,
}

#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
pub enum PermissionLevel {
    Viewer,
    Editor,
    Owner,
}
```

### 4. Build the metadata store

```rust
// src/metadata_store.rs
use async_trait::async_trait;
use crate::types::DocMetadata;

#[async_trait]
pub trait MetadataStore: Send + Sync {
    async fn put_doc(&self, meta: &DocMetadata) -> Result<(), String>;
    async fn get_doc(&self, doc_id: &str) -> Result<Option<DocMetadata>, String>;
    async fn list_docs_by_owner(&self, owner: &str) -> Result<Vec<DocMetadata>, String>;
    async fn delete_doc(&self, doc_id: &str) -> Result<(), String>;
}

/// In-memory implementation for development and tests.
pub struct InMemoryMetadataStore {
    docs: tokio::sync::RwLock<std::collections::HashMap<String, DocMetadata>>,
}

impl InMemoryMetadataStore {
    pub fn new() -> Self {
        Self { docs: tokio::sync::RwLock::new(std::collections::HashMap::new()) }
    }
}

#[async_trait]
impl MetadataStore for InMemoryMetadataStore {
    async fn put_doc(&self, meta: &DocMetadata) -> Result<(), String> {
        self.docs.write().await.insert(meta.id.clone(), meta.clone());
        Ok(())
    }

    async fn get_doc(&self, doc_id: &str) -> Result<Option<DocMetadata>, String> {
        Ok(self.docs.read().await.get(doc_id).cloned())
    }

    async fn list_docs_by_owner(&self, owner: &str) -> Result<Vec<DocMetadata>, String> {
        let docs = self.docs.read().await;
        Ok(docs.values().filter(|d| d.owner == owner).cloned().collect())
    }

    async fn delete_doc(&self, doc_id: &str) -> Result<(), String> {
        self.docs.write().await.remove(doc_id);
        Ok(())
    }
}
```

### 5. Add permission checks

```rust
// src/permissions.rs
use crate::types::{DocMetadata, PermissionLevel};

pub fn check_permission(doc: &DocMetadata, user: &str) -> PermissionLevel {
    if doc.owner == user {
        PermissionLevel::Owner
    } else {
        // For now: everyone else is a viewer
        // Later: check ACL entries, group memberships
        PermissionLevel::Viewer
    }
}

pub fn can_edit(doc: &DocMetadata, user: &str) -> bool {
    check_permission(doc, user) >= PermissionLevel::Editor
}
```

### 6. Add REST endpoints

```rust
// src/routes.rs
use axum::{extract::{Path, State}, http::StatusCode, Json};
use crate::auth::UserAlias;
use crate::types::DocMetadata;

// POST /api/docs
async fn create_doc(
    UserAlias(user): UserAlias,
    State(state): State<AppState>,
    Json(req): Json<CreateDocRequest>,
) -> Result<Json<DocMetadata>, StatusCode> {
    let now = std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH).unwrap().as_secs();
    let meta = DocMetadata {
        id: uuid::Uuid::new_v4().to_string(),
        title: req.title,
        owner: user,
        created_at: now,
        updated_at: now,
        parent_id: None,
    };
    state.metadata.put_doc(&meta).await.map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    Ok(Json(meta))
}

// GET /api/docs
async fn list_docs(
    UserAlias(user): UserAlias,
    State(state): State<AppState>,
) -> Result<Json<Vec<DocMetadata>>, StatusCode> {
    let docs = state.metadata.list_docs_by_owner(&user).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    Ok(Json(docs))
}

// GET /api/docs/:id
async fn get_doc(
    UserAlias(user): UserAlias,
    State(state): State<AppState>,
    Path(id): Path<String>,
) -> Result<Json<DocMetadata>, StatusCode> {
    let doc = state.metadata.get_doc(&id).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?
        .ok_or(StatusCode::NOT_FOUND)?;
    // Anyone can read metadata for now; expand with ACL later
    Ok(Json(doc))
}
```

### 7. Add presence tracking

```rust
// src/presence.rs
use std::collections::{HashMap, HashSet};
use tokio::sync::RwLock;

pub struct PresenceTracker {
    /// doc_id → set of user aliases currently connected
    docs: RwLock<HashMap<String, HashSet<String>>>,
}

impl PresenceTracker {
    pub fn new() -> Self {
        Self { docs: RwLock::new(HashMap::new()) }
    }

    pub async fn join(&self, doc_id: &str, user: &str) {
        self.docs.write().await
            .entry(doc_id.to_string())
            .or_default()
            .insert(user.to_string());
    }

    pub async fn leave(&self, doc_id: &str, user: &str) {
        let mut docs = self.docs.write().await;
        if let Some(users) = docs.get_mut(doc_id) {
            users.remove(user);
            if users.is_empty() {
                docs.remove(doc_id);
            }
        }
    }

    pub async fn who_is_in(&self, doc_id: &str) -> Vec<String> {
        self.docs.read().await
            .get(doc_id)
            .map(|s| s.iter().cloned().collect())
            .unwrap_or_default()
    }
}
```

Wire into the WebSocket handler: `presence.join(doc_id, user)` on connect, `presence.leave(doc_id, user)` on disconnect.

### 8. Permission check on WebSocket upgrade

```rust
async fn ws_handler(
    ws: WebSocketUpgrade,
    Path(doc_id): Path<String>,
    UserAlias(user): UserAlias,
    State(state): State<AppState>,
) -> Result<impl IntoResponse, StatusCode> {
    let meta = state.metadata.get_doc(&doc_id).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?
        .ok_or(StatusCode::NOT_FOUND)?;

    let perm = check_permission(&meta, &user);
    // For now: viewers can connect but can't send updates (enforce in recv loop)

    Ok(ws.on_upgrade(move |socket| {
        handle_socket(socket, doc_id, user, perm, state)
    }))
}
```

## Exercises

1. **ACL entries**: Add a `Vec<AclEntry>` to `DocMetadata` where `AclEntry = { user: String, level: PermissionLevel }`. Let the owner grant Editor/Viewer access to others.
2. **Share endpoint**: `PUT /api/docs/{id}/acl` to add/remove users from the ACL.
3. **Read-only enforcement**: If the user is a Viewer, reject `MSG_UPDATE` messages in the WebSocket recv loop (they can see edits but not make them).
4. **Recent docs**: Track the last 10 docs each user accessed. Return them from `GET /api/recent`.

## How a Production System Does This

The reference system's auth system (`src/auth.rs`, `src/acl.rs`) extends this pattern significantly:
- **mTLS auth** instead of a simple header (production-grade auth)
- **Group membership service** for team-based access (not just individual users)
- **Folder-level ACL inheritance** (a folder's ACL applies to all docs inside)
- **Origin verification** to prevent CDN impersonation attacks
- **Redis-cached group membership calls** for performance

The presence system (`src/presence/redis.rs`) uses Redis instead of in-memory HashMap to work across multiple server nodes.

## Rust Book Chapters to Read

- [Chapter 10.2: Traits](https://doc.rust-lang.org/book/ch10-02-traits.html)
- [Chapter 17.2: Trait Objects](https://doc.rust-lang.org/book/ch17-02-trait-objects.html)
- [Chapter 19.1: Unsafe Rust](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html) (you probably won't need this, but good to know what it is)

## Done When

- [ ] Requests without `x-user` header get 401
- [ ] `POST /api/docs` creates a doc owned by the authenticated user
- [ ] `GET /api/docs` returns only docs the user owns
- [ ] WebSocket connections require authentication
- [ ] A `GET /api/docs/{id}/presence` endpoint shows who's currently editing
- [ ] Non-owners can't delete docs (403)
- [ ] `cargo clippy` passes
