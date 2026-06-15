# MCP Server

## What It Is

The MCP (Model Context Protocol) server exposes the production system documents to external AI agents over a JSON-RPC 2.0 HTTP endpoint. This allows tools like Q Developer, Claude Code, or any MCP-compatible client to read, edit, and search documents without a browser — using the same CRDT engine and ACL enforcement as the web UI.

Source: `src/mcp/mod.rs`, `src/mcp/session.rs`, `src/mcp/tool_ctx.rs`, `src/mcp/tools/`, `src/mcp/edit.rs`

## How It Works

### Protocol

```
MCP Client                              the production system Backend
    │                                        │
    │── POST /mcp ──────────────────────────▶│
    │   Headers:                             │
    │     x-forwarded-user: alice            │  (auth via the mTLS gateway)
    │     x-agent-platform: ai-assistant      │  (agent identity)
    │     x-agent-instance: session-123      │
    │   Body: JSON-RPC request               │
    │                                        │
    │◀── 200 + mcp-session-id header ────────│
    │   Body: JSON-RPC response              │
    │                                        │
    │── POST /mcp (with session id) ────────▶│  (subsequent calls)
```

Transport: Streamable HTTP (single `POST /mcp` endpoint). Each request is a JSON-RPC 2.0 message. The server returns a session ID header that clients pass on subsequent requests.

### Session Management

Each MCP client gets a session that tracks which documents it has "opened":

```
SessionStore (in-memory, Arc<DashMap>)
  └── session_id → Session {
        user: "alice",
        platform: "ai-assistant",
        opened_docs: HashSet<DocId>,
        last_activity: Instant,
      }
```

When a session first accesses a document:
1. The doc is loaded into memory (via DocManager)
2. The session is registered as an "editor" with the snapshot scheduler
3. The doc stays warm (not evicted) for the session's lifetime

**Idle sweep**: A background task runs every `SESSION_SWEEP_TICK` and evicts sessions idle longer than `SESSION_IDLE_TTL`. On eviction, the session's editor registrations are cleaned up.

### Available Tools

Tools are registered in a `ToolRegistry` and exposed via `tools/list`:

| Tool | Description | Returns |
|------|-------------|---------|
| `list_docs` | List accessible documents | Doc metadata array |
| `create_doc` | Create a new document | New doc metadata |
| `read_markdown` | Read document content as Markdown | Markdown string |
| `write_markdown` | Replace entire document content | Success/failure |
| `edit_markdown` | Apply targeted str_replace edits | Success/failure |
| `search` | Full-text search with ACL filtering | Search results |
| `feedback` | Submit bug/feature reports | Feedback ID |
| `reveal` | List active users on a document | User list |

### Edit Pipeline

MCP edits go through the same CRDT pipeline as WebSocket edits:

```
MCP tool call (edit_markdown)
  │
  ├── Parse edit operations (str_replace pairs)
  ├── Load document (DocManager::get_or_create)
  ├── Apply edits to the real CollabDoc (MCP edits are direct, no consent gate)
  ├── Broadcast update via Redis pub/sub
  ├── Notify snapshot scheduler (dirty doc)
  └── Return success + updated content
```

Note: Unlike the AI teammate (which uses a shadow doc), MCP edits are applied directly. The consent model doesn't apply because the human user explicitly invoked the MCP tool from their agent — the "consent" is the tool invocation itself.

### ToolOutcome

Each tool returns a `ToolOutcome` enum:
- `Unbound(json)` — Result doesn't reference a specific doc (search, feedback)
- `OnDoc(json, doc_id)` — Result is bound to a document; pins the session as an editor

### ACL Enforcement

MCP requests go through the same `check_resource_permission` as HTTP API calls:
- The authenticated user (from `x-forwarded-user`) must have sufficient permission on the target doc/folder
- Group memberships are checked via the group membership service (with Redis caching)
- The markdown tool additionally validates a the CDN origin secret to prevent unauthorized direct access

### Client Version Enforcement

A middleware layer (`enforce_mcp_version`) rejects MCP clients below a minimum protocol version. This allows breaking changes to the MCP schema without maintaining backwards compatibility indefinitely.

## Why This Design?

### Why MCP (Not a Custom API)?

MCP is an emerging standard for AI tool access. By implementing it:
- Any MCP-compatible AI agent can access the production system documents out of the box
- The protocol handles JSON-RPC framing, tool discovery, and error conventions
- Future agents don't need the production system-specific integration code

### Why Sessions (Not Stateless)?

Without sessions:
- Every request would re-load the doc from the object store (50-200ms overhead)
- No way to track which docs an agent is actively using (for eviction)
- No way to detect abandoned agents (stale editor counts)

Sessions amortize the load cost and provide a clean lifecycle for cleanup.

### Why Direct Edits (No Consent Gate)?

The MCP path serves AI agents acting on behalf of an authenticated user who explicitly invoked the tool. The human's consent is implicit in the tool invocation — they chose to run the agent and give it access. Adding a consent UI gate would require the MCP client to implement an approval flow, which most don't support.

If consent is needed, the MCP client can use `read_markdown` first, present the proposed changes to the user, and only call `edit_markdown` after approval — but that's the client's responsibility.

### Why a Sweep Task (Not TTL Eviction)?

Redis keys naturally expire with TTL, but MCP sessions hold in-memory state (editor count registrations in the snapshot scheduler). A simple TTL expiry wouldn't trigger the cleanup logic. The sweep task actively checks for idle sessions and runs the proper teardown sequence.
