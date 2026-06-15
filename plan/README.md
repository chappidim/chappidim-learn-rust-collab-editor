# Learn Rust: Build a Collaborative Editor

A structured learning plan to build a real-time collaborative document editor in Rust — inspired by the production architecture at the organization. Each milestone produces a working artifact and introduces new Rust concepts incrementally.

## Milestones

| # | Name | Est. Time | Core Concept |
|---|------|-----------|--------------|
| 1 | [Hello Axum](./milestone-01-hello-axum.md) | 1-2 days | Async Rust, HTTP, serde |
| 2 | [WebSocket Echo & Broadcast](./milestone-02-websocket-broadcast.md) | 2-3 days | Channels, Arc, spawning tasks |
| 3 | [CRDT Document](./milestone-03-crdt-document.md) | 3-5 days | Yrs, binary protocols, conflict-free editing |
| 4 | [Persistence](./milestone-04-persistence.md) | 2-3 days | File I/O, background tasks, traits |
| 5 | [Rich Text Frontend](./milestone-05-rich-text-frontend.md) | 3-5 days | Full-stack integration, awareness protocol |
| 6 | [Multi-User Features](./milestone-06-multi-user.md) | 3-5 days | Middleware, dyn traits, permission models |
| 7 | [AI Integration](./milestone-07-ai-integration.md) | 3-5 days | HTTP clients, streaming, shadow-doc pattern |

## Prerequisites

- Comfortable with at least one other programming language
- Basic understanding of HTTP, JSON, WebSockets
- A text editor / IDE with rust-analyzer installed
- Rust toolchain installed (`rustup`)

## How to Use This Plan

1. Work through milestones in order — each builds on the previous
2. At each milestone, get it WORKING before moving on (resist the urge to gold-plate)
3. Use `cargo clippy` after every change — it teaches idiomatic Rust better than any book
4. Commit after each working state (git log becomes your learning journal)
5. When stuck on a Rust concept, refer to the linked Rust Book chapters

## Reference Architecture

The `../reference/` folder contains detailed architecture docs from a production collaborative editor. Use these as reference for HOW a real system solves each problem — but don't try to replicate the complexity. Start simple, add complexity when you feel the pain.

Key reference docs per milestone:
- M1-M2: `01-overview-and-startup.md`
- M3: `02-crdt-engine.md`, `03-websocket-collaboration.md`
- M4: `05-persistence-layer.md`
- M5: `03-websocket-collaboration.md` (protocol details)
- M6: `06-auth-and-acl.md`, `11-supporting-subsystems.md`
- M7: `07-ai-teammate.md`
