# AI Teammate

## What It Is

The AI teammate is a the LLM provider-powered assistant that can read, edit, and comment on documents. It participates in the editing experience as a distinct author — its edits are tagged with AI attribution and go through a user-consent gate before being applied to the real document.

Source: `src/ai/mod.rs`, `src/ai/turn.rs`, `src/ai/tools/`, `src/ai_config.rs`

## How It Works

### Request Flow

```
User clicks "Ask AI" in frontend
  │
  ▼
Browser sends MSG_AI_REQUEST (0x10) over WebSocket
  │
  ▼
WS handler: check rate limit (20 AI requests/min/user)
  │
  ▼
Check AiConfigStore: is AI enabled? (Andon Cord)
  │
  ▼
Create AiTurnContext (shadow doc + format ops accumulator)
  │
  ▼
Call the LLM streaming API (Claude model)
  │ ◀── streaming response chunks
  │      MSG_AI_RESPONSE (0x11) → browser (UI shows streaming text)
  │
  ├── Model emits tool_use → execute tool against shadow doc
  │   ├── read_document: read shadow's current state
  │   ├── markdown_edit: apply text edits to shadow
  │   ├── table / sheet: structural table operations on shadow
  │   ├── list_comments / create_comment: read/write comments
  │   └── react_to_comment: add emoji reactions
  │
  ├── Tool result → feed back to model as tool_result
  │   (loop until model emits end_turn)
  │
  ▼
AiTurnContext::flush()
  │
  ├── Diff shadow vs original → compute minimal block-level edits
  ├── Package as MSG_AI_EDITS (0x14) → browser
  └── Browser shows proposed edits with Accept/Reject UI
```

### The Shadow Document (Consent Model)

This is the critical architectural invariant: **AI never mutates the real `CollabDoc`.**

```
Real CollabDoc (shared by all users)
       │
       │ clone at turn start
       ▼
Shadow yrs::Doc (private to this AI turn)
       │
       │ AI tool calls mutate the shadow
       ▼
Diff (shadow vs original) → proposed edits
       │
       │ User clicks "Accept"
       ▼
Apply to real CollabDoc (normal CRDT update path)
```

Why:
- **User consent**: The user sees exactly what the AI wants to change before it takes effect
- **Undo-friendly**: If the user rejects, nothing happened
- **Multi-tool coherence**: An AI turn may call 5 tools in sequence. Each reads from the shadow (seeing prior tool results), producing one coherent proposal at the end — not 5 separate diffs

### AI Tools

Tools exposed to the model (via the LLM provider's tool_use API):

| Tool | Purpose | Mutates Shadow? |
|------|---------|-----------------|
| `read_document` | Read document structure as block list | No |
| `markdown_edit` | `str_replace` or full `write` to the doc | Yes |
| `table` | Create/read/update/delete table rows/cols/cells | Yes |
| `sheet` | Spreadsheet-specific operations (A1 addressing) | Yes |
| `list_comments` | Read existing comments | No |
| `create_comment` | Add a new comment | No (direct the database write) |
| `update_comment` | Edit/resolve a comment | No (direct the database write) |
| `react_to_comment` | Add emoji reaction | No (direct the database write) |

Note: Comment tools bypass the shadow entirely because comments aren't part of the CRDT document — they're stored in the database.

Tool exposure is gated by doc type:
- Prose documents get the `table` tool (reference-addressed: "row 3, column B")
- Spreadsheet documents get the `sheet` tool (A1-addressed: "cell C4")

### Mark-Based Addressing

AI edits use **text marks** rather than character offsets:

```json
{
  "text": "the quick brown fox",
  "occurrence": 1
}
```

This is resilient to concurrent edits. If another user inserts text above the AI's target, character offsets would be wrong, but a text match still finds the right location. The `resolve_mark_text_in_edits()` function converts marks to positions at apply time.

### AiTraceRing (Diagnostics)

An in-memory ring buffer records AI activity per doc:
- `turn_start`: user prompt, model selection
- `round`: each model→tool→model round-trip
- `tool_call`: tool name, input, output, latency
- `turn_end`: final proposal summary

This trace is never persisted unless a user explicitly opts in via the feedback form (checkbox: "Include AI trace"). Privacy-first: unchecked by default.

### Andon Cord (AI Kill Switch)

The `AiConfigStore` polls cloud a remote config service every 30 seconds:

```json
{ "ai_enabled": true }
```

If an operator sets `ai_enabled: false`:
- All new AI requests are rejected (403)
- In-flight turns are NOT interrupted (they'll complete, but the proposal may be stale)
- No deploy required — takes effect within 30 seconds globally

Safety properties:
- **Fail-safe default**: AI is disabled at startup. Only a successful a remote config service fetch enables it.
- **Malformed config**: Keeps previous state (doesn't auto-enable)
- **Missing a remote config service**: AI stays disabled forever (explicit warning in logs)

### Model Selection

Users can choose which Claude model to use. `ModelProvider::all()` returns the available models, exposed via `GET /api/models`. The frontend presents these as a dropdown.

## Why This Design?

### Why a Shadow Doc (Not Direct Edits)?

Alternative: AI could apply edits directly to the CRDT (like a human typing). Problems:
- **No undo**: CRDT updates are permanent once broadcast. The user can't "reject" something already merged.
- **Partial application**: If the AI's 3rd tool call fails, the first 2 are already applied. The user gets a half-baked edit.
- **Consent**: the organization's responsible AI guidelines require user review before AI content appears in shared documents.

### Why Streaming Responses?

The model may take 5-30 seconds to generate a full response. Streaming the text chunks over `MSG_AI_RESPONSE` lets the user see the AI "thinking" in real time, matching the ChatGPT-style UX they expect.

### Why Text Marks Over Offsets?

In a collaborative editor, character positions shift constantly as other users type. An AI turn takes 5-30 seconds — in that time, other users may have added 100 characters. Text marks are position-independent and resolve at apply time, making them naturally compatible with concurrent editing.

### Why Redis Rate Limiting for AI?

the LLM provider (Claude) calls cost $3-15 per 1M tokens. A single user spam-clicking "Ask AI" could generate significant cost. The 20 requests/minute/user limit prevents abuse while allowing normal usage patterns.
