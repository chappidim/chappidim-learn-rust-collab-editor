# Collaborative Editor Backend Architecture

This folder contains detailed documentation for each subsystem of the reference collaborative editor backend. Each file explains **what** the subsystem is, **how** it works, and **why** it exists.

## Reading Order

For someone new to the codebase, the recommended path is:

1. [Overview & Startup](./01-overview-and-startup.md) — What the system is and how the server boots
2. [CRDT Engine](./02-crdt-engine.md) — The Yrs document model at the core
3. [WebSocket Collaboration](./03-websocket-collaboration.md) — Real-time sync protocol
4. [Multi-Node Sync & Routing](./04-multi-node-sync-and-routing.md) — How multiple server nodes coordinate
5. [Persistence Layer](./05-persistence-layer.md) — Database, object store, and the snapshot scheduler
6. [Authentication & Authorization](./06-auth-and-acl.md) — mTLS auth, group membership, ACL inheritance
7. [AI Teammate](./07-ai-teammate.md) — LLM integration, tools, consent model
8. [MCP Server](./08-mcp-server.md) — External AI agent access via Model Context Protocol
9. [Search](./09-search.md) — Full-text search with ACL filtering
10. [Media & Attachments](./10-media.md) — Upload, storage, and cross-doc references
11. [Supporting Subsystems](./11-supporting-subsystems.md) — Comments, folders, notifications, rate limiting, feedback, presence

## Package Map

| Package | Role |
|---------|------|
| `backend` | The Rust backend server |
| `frontend` | React/TypeScript frontend |
| `crdt-ops` | Shared CRDT operations library |
| `crdt-ops-wasm` | WASM bridge for the frontend |
| `mcp-server` | Standalone MCP server binary |
| `infra` | Infrastructure as code |
| `integration-tests` | Playwright E2E test suite |
| `dev-tools` | Dev-server tooling |

## Key Architectural Decisions

- **CRDT over OT**: Yrs (Rust Yjs) provides conflict-free merging without a central sequencer. Any node can accept writes independently.
- **Redis for ephemeral state**: Presence, routing, pub/sub, and rate limits live in Redis. If Redis dies, docs are still durable (object store + database) — only real-time sync degrades.
- **Object storage for durability**: Binary CRDT snapshots in the object store are the source of truth for document content. Database metadata points to the latest object store key.
- **AI as a participant, not a controller**: AI edits flow through the same CRDT protocol as human edits, gated by user consent before application.
- **Trait-based dependency injection**: Every external service (group membership, object store, database, Redis) is behind a trait with noop/mock implementations for testing.
