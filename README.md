# Learn Rust: Collaborative Editor

Building a real-time collaborative document editor from scratch to learn Rust — inspired by production collaborative editing architectures.

## Repository Structure

```
.
├── plan/                  # Learning plan with 7 milestones
│   ├── README.md          # Overview and reading order
│   ├── milestone-01-hello-axum.md
│   ├── milestone-02-websocket-broadcast.md
│   ├── milestone-03-crdt-document.md
│   ├── milestone-04-persistence.md
│   ├── milestone-05-rich-text-frontend.md
│   ├── milestone-06-multi-user.md
│   └── milestone-07-ai-integration.md
├── reference/             # Architecture docs from a production system (read-only context)
│   ├── README.md
│   ├── 01-overview-and-startup.md
│   ├── ...
│   └── 11-supporting-subsystems.md
└── src/                   # Code (built incrementally per milestone)
```

## Getting Started

1. [Install Rust](https://rustup.rs/)
2. Read `plan/README.md` for the full learning roadmap
3. Start with `plan/milestone-01-hello-axum.md`
4. Code goes in `src/` — one commit per working state

## Milestones

| # | Name | What You Learn |
|---|------|----------------|
| 1 | Hello Axum | Async Rust, HTTP servers, serde |
| 2 | WebSocket Broadcast | Channels, concurrency, Arc |
| 3 | CRDT Document | Yrs, binary protocols, conflict-free merging |
| 4 | Persistence | File I/O, traits, background tasks |
| 5 | Rich Text Frontend | TipTap + Yjs, full-stack integration |
| 6 | Multi-User | Auth, middleware, permissions |
| 7 | AI Integration | HTTP clients, streaming, shadow-doc pattern |

## Reference Material

The `reference/` folder contains detailed architecture documentation from a production collaborative editor. Use it to understand HOW real systems solve each problem — but don't try to replicate the complexity upfront.
