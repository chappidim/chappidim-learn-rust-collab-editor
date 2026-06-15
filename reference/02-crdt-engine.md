# CRDT Engine

## What It Is

The CRDT (Conflict-free Replicated Data Type) engine is the heart of A production system's collaboration model. It uses **Yrs** (the Rust port of Yjs) to maintain a conflict-free document state that multiple users can edit simultaneously without coordination.

Source: `src/crdt/mod.rs`

## How It Works

### The CollabDoc

`CollabDoc` wraps a `yrs::Doc` and adds the production system-specific concerns:

```
CollabDoc
├── yrs::Doc          — the CRDT document (XmlFragment tree)
├── FormatOpStore     — cell formatting not representable in Yrs (bg color, validation)
├── broadcast channel — notify local subscribers of updates
├── CancellationToken — for cleanup on eviction
└── StructuralRateState — per-connection throttling
```

The Yrs document contains an `XmlFragment` tree representing the document's block structure (paragraphs, headings, tables, lists). Each block is a Yrs XML element; text within blocks is stored as Yrs `Text` with rich-text attributes (bold, italic, links, etc.).

### Applying Updates

When a CRDT update arrives (from a WebSocket client or Redis broadcast):

```
binary update bytes
       │
       ▼
CollabDoc::apply_update_from(conn_id, update)
       │
       ├── Pre-snapshot: capture table dimensions
       ├── yrs::Doc::apply_update() — merge into local state
       ├── Post-snapshot: check new table dimensions
       ├── Validate structural constraints
       │     ├── Table size caps (max rows/cols)
       │     └── Structural rate limit (updates/sec per connection)
       └── Return ApplyOutcome
```

### ApplyOutcome

| Variant | Meaning | Caller Action |
|---------|---------|---------------|
| `Applied` | Update merged successfully | Broadcast to other clients |
| `InternalReject` | Yrs rejected (duplicate, out-of-order) | Silently ignore |
| `RejectedTableCap` | Table exceeded max dimensions | Send `WriteRejected` toast to client |
| `RejectedStructuralRate` | Too many structural edits/sec | Send `WriteRejected` toast to client |

### Structural Rate Limiting

A sliding window counter tracks how many CRDT updates from a single connection grew a table (added rows/columns). If a connection exceeds 100 structural changes in 10 seconds, further structural edits are rejected. This prevents runaway "column cascade" bugs where a client programmatically inserts thousands of cells.

### State Encoding

`CollabDoc::encode_state()` serializes the entire document to a binary Yrs state vector. This blob is what gets stored in the object store as a snapshot.

`CollabDoc::encode_state_vector()` produces a compact fingerprint of which updates the doc has seen — used in the sync protocol to compute the minimal diff a peer needs.

## Why CRDTs?

Traditional collaborative editors use **Operational Transform (OT)**, which requires a central sequencer to order operations. This creates:

- A single point of failure
- A bottleneck for high-throughput editing
- Complexity in the transformation logic

CRDTs eliminate these problems:

- **No central sequencer**: Any node can accept writes independently
- **Convergence guaranteed**: All replicas that see the same set of updates converge to the same state, regardless of order
- **Multi-node friendly**: the production system can run N container tasks, each accepting writes in parallel

The tradeoff is that CRDTs use more memory (they carry tombstones for deleted content) and the merge semantics can sometimes surprise users (e.g., concurrent edits to the same word). Yrs mitigates this with garbage collection of old tombstones and well-tested merge algorithms.

## Key Invariants

1. **The `CollabDoc` in memory is always the ground truth** for a loaded document. object store snapshots are periodic checkpoints.
2. **All mutations go through `apply_update`** — there is no direct manipulation of the Yrs doc outside this path (except during initial hydration).
3. **AI edits do NOT mutate the real CollabDoc** — they operate on a shadow copy (see [AI Teammate](./07-ai-teammate.md)).
4. **FormatOps are separate from Yrs state** — cell formatting (background colors, number formats, data validation) lives in a sidecar `FormatOpStore` because Yrs doesn't natively support cell-level attributes on table cells.
