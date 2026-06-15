# Milestone 7: AI Integration

## Goal

Add an AI assistant that can read the document, propose edits, and have those edits reviewed by the user before applying. This teaches HTTP client usage, streaming responses, and the "shadow document" pattern for safe AI editing.

## What You're Building

- AI can read the current document content
- AI can propose text edits (str_replace operations)
- User sees proposed changes and accepts/rejects them
- Accepted edits flow through the normal CRDT path
- A "tool use" loop where the model can call read/write tools multiple times

## Rust Concepts Introduced

| Concept | Where You'll Encounter It |
|---------|--------------------------|
| HTTP client (`reqwest`) | Calling the Claude API |
| Streaming responses | Reading Server-Sent Events chunk by chunk |
| The builder pattern | Constructing API request bodies |
| `Clone` on `yrs::Doc` | Creating a shadow document |
| Serde with enums | Parsing tool_use / tool_result blocks |
| Error handling chains | Multiple fallible steps in the AI pipeline |

## Background: The Shadow Document Pattern

The critical insight from The reference system's design:

```
WRONG approach:
  AI calls tool → directly edits the real CRDT doc → broadcasts to all users
  Problem: user can't reject, can't undo, partial edits if AI errors mid-turn

RIGHT approach (shadow doc):
  1. Clone the document state at turn start
  2. AI tool calls mutate the CLONE (the "shadow")
  3. At turn end, diff the shadow vs original
  4. Send the diff as a PROPOSAL to the user
  5. User accepts → apply the diff to the real doc via normal CRDT path
  6. User rejects → discard the shadow, nothing happened
```

This is The reference system's "consent model" — AI never mutates the shared doc directly.

## Step-by-Step

### 1. Add dependencies

```toml
[dependencies]
reqwest = { version = "0.12", features = ["json", "stream"] }
```

### 2. Define the AI service trait

```rust
// src/ai/mod.rs
use async_trait::async_trait;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AiTool {
    pub name: String,
    pub description: String,
    pub input_schema: serde_json::Value,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolCall {
    pub id: String,
    pub name: String,
    pub input: serde_json::Value,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolResult {
    pub tool_use_id: String,
    pub content: String,
}

#[derive(Debug)]
pub enum AiEvent {
    TextChunk(String),      // Streaming text from the model
    ToolUse(ToolCall),      // Model wants to call a tool
    TurnEnd,                // Model is done
}

#[async_trait]
pub trait AiService: Send + Sync {
    async fn converse(
        &self,
        messages: Vec<serde_json::Value>,
        tools: Vec<AiTool>,
    ) -> Result<Vec<AiEvent>, String>;
}
```

### 3. Implement the Claude API client

```rust
// src/ai/claude.rs
use super::*;
use reqwest::Client;

pub struct ClaudeService {
    client: Client,
    api_key: String,
    model: String,
}

impl ClaudeService {
    pub fn new(api_key: String) -> Self {
        Self {
            client: Client::new(),
            api_key,
            model: "claude-sonnet-4-20250514".to_string(),
        }
    }
}

#[async_trait]
impl AiService for ClaudeService {
    async fn converse(
        &self,
        messages: Vec<serde_json::Value>,
        tools: Vec<AiTool>,
    ) -> Result<Vec<AiEvent>, String> {
        let body = serde_json::json!({
            "model": self.model,
            "max_tokens": 4096,
            "messages": messages,
            "tools": tools.iter().map(|t| serde_json::json!({
                "name": t.name,
                "description": t.description,
                "input_schema": t.input_schema,
            })).collect::<Vec<_>>(),
        });

        let response = self.client
            .post("https://api.anthropic.com/v1/messages")
            .header("x-api-key", &self.api_key)
            .header("anthropic-version", "2023-06-01")
            .header("content-type", "application/json")
            .json(&body)
            .send()
            .await
            .map_err(|e| format!("API call failed: {e}"))?;

        let result: serde_json::Value = response.json().await
            .map_err(|e| format!("Failed to parse response: {e}"))?;

        // Parse content blocks into AiEvents
        let mut events = Vec::new();
        if let Some(content) = result["content"].as_array() {
            for block in content {
                match block["type"].as_str() {
                    Some("text") => {
                        if let Some(text) = block["text"].as_str() {
                            events.push(AiEvent::TextChunk(text.to_string()));
                        }
                    }
                    Some("tool_use") => {
                        events.push(AiEvent::ToolUse(ToolCall {
                            id: block["id"].as_str().unwrap_or("").to_string(),
                            name: block["name"].as_str().unwrap_or("").to_string(),
                            input: block["input"].clone(),
                        }));
                    }
                    _ => {}
                }
            }
        }
        events.push(AiEvent::TurnEnd);
        Ok(events)
    }
}
```

### 4. Define AI tools

```rust
// src/ai/tools.rs
use super::AiTool;
use serde_json::json;

pub fn document_tools() -> Vec<AiTool> {
    vec![
        AiTool {
            name: "read_document".to_string(),
            description: "Read the current document content as plain text.".to_string(),
            input_schema: json!({
                "type": "object",
                "properties": {}
            }),
        },
        AiTool {
            name: "str_replace".to_string(),
            description: "Replace a specific string in the document with new text. \
                          The old_str must match exactly (including whitespace).".to_string(),
            input_schema: json!({
                "type": "object",
                "properties": {
                    "old_str": {
                        "type": "string",
                        "description": "The exact text to find and replace"
                    },
                    "new_str": {
                        "type": "string",
                        "description": "The replacement text"
                    }
                },
                "required": ["old_str", "new_str"]
            }),
        },
    ]
}
```

### 5. Implement the shadow document + tool execution

```rust
// src/ai/turn.rs
use yrs::{Doc, Text, Transact, ReadTxn, GetString, updates::decoder::Decode, updates::encoder::Encode};

pub struct AiTurnContext {
    /// Shadow: clone of the real doc at turn start
    shadow: Doc,
}

impl AiTurnContext {
    pub fn new(original_state: &[u8]) -> Result<Self, String> {
        let shadow = Doc::new();
        let update = yrs::Update::decode_v1(original_state)
            .map_err(|e| format!("decode: {e}"))?;
        shadow.transact_mut().apply_update(update)
            .map_err(|e| format!("apply: {e}"))?;
        Ok(Self { shadow })
    }

    /// Read current shadow state as text.
    pub fn read_text(&self) -> String {
        let txn = self.shadow.transact();
        let text = txn.get_or_insert_text("content");
        text.get_string(&txn)
    }

    /// Apply a str_replace to the shadow.
    pub fn str_replace(&self, old_str: &str, new_str: &str) -> Result<(), String> {
        let txn = self.shadow.transact();
        let text = txn.get_or_insert_text("content");
        let current = text.get_string(&txn);
        drop(txn);

        let pos = current.find(old_str)
            .ok_or_else(|| format!("text not found: {:?}", old_str))?;

        let mut txn = self.shadow.transact_mut();
        let text = txn.get_or_insert_text("content");
        text.remove_range(&mut txn, pos as u32, old_str.len() as u32);
        text.insert(&mut txn, pos as u32, new_str);
        Ok(())
    }

    /// Compute the diff between shadow and original, as a Yrs update.
    pub fn compute_diff(&self, original_sv: &[u8]) -> Result<Vec<u8>, String> {
        let sv = yrs::StateVector::decode_v1(original_sv)
            .map_err(|e| format!("decode sv: {e}"))?;
        let txn = self.shadow.transact();
        Ok(txn.encode_diff_v1(&sv))
    }

    /// Execute a tool call against the shadow.
    pub fn execute_tool(&self, name: &str, input: &serde_json::Value) -> Result<String, String> {
        match name {
            "read_document" => Ok(self.read_text()),
            "str_replace" => {
                let old_str = input["old_str"].as_str()
                    .ok_or("missing old_str")?;
                let new_str = input["new_str"].as_str()
                    .ok_or("missing new_str")?;
                self.str_replace(old_str, new_str)?;
                Ok("Replacement applied.".to_string())
            }
            _ => Err(format!("unknown tool: {name}")),
        }
    }
}
```

### 6. Wire it into the WebSocket handler

When the client sends an AI request:

```rust
MSG_AI_REQUEST => {
    let prompt: String = serde_json::from_slice(payload).unwrap();
    let doc_state = doc.read().await.encode_state();
    let doc_sv = doc.read().await.state_vector();

    tokio::spawn(async move {
        // Create shadow
        let ctx = AiTurnContext::new(&doc_state).unwrap();

        // Build initial messages
        let doc_text = ctx.read_text();
        let messages = vec![serde_json::json!({
            "role": "user",
            "content": format!("Document content:\n\n{doc_text}\n\n---\n\nUser request: {prompt}")
        })];

        // Call AI (may loop for tool use)
        let events = ai_service.converse(messages, document_tools()).await.unwrap();

        for event in &events {
            if let AiEvent::ToolUse(call) = event {
                let result = ctx.execute_tool(&call.name, &call.input);
                // Feed result back for next round (simplified)
            }
        }

        // Compute proposed diff
        let diff = ctx.compute_diff(&doc_sv).unwrap();

        // Send proposal to client as MSG_AI_EDITS
        let proposal = serde_json::json!({
            "diff": base64::encode(&diff),
            "description": "AI proposed changes"
        });
        // Send via the client's sender channel...
    });
}
```

### 7. Frontend: show accept/reject UI

When the client receives `MSG_AI_EDITS`:
1. Decode the diff (Yrs update bytes)
2. Show a "Proposed Changes" panel with before/after
3. "Accept" → apply the update to the local Yjs doc (which syncs to server)
4. "Reject" → discard the payload

## The Multi-Tool Loop

A real AI turn may involve multiple tool calls:

```
User: "Fix the typos in this doc"
  │
  Model: calls read_document
  │ ← returns current text
  │
  Model: calls str_replace("teh", "the")
  │ ← returns "Replacement applied"
  │
  Model: calls str_replace("recieve", "receive")
  │ ← returns "Replacement applied"
  │
  Model: responds with text "I fixed 2 typos."
  │
  Turn end → diff shadow vs original → propose to user
```

The shadow accumulates all edits across multiple tool calls, and the user sees ONE coherent proposal at the end.

## Exercises

1. **Streaming text**: Instead of waiting for the full response, stream `TextChunk` events to the client as they arrive (show the AI "thinking").
2. **Rate limiting**: Add a counter per user — max 20 AI requests per minute.
3. **Mock AI service**: Create a `MockAiService` that returns canned responses for testing without API keys.
4. **Kill switch**: Add a config flag to disable AI globally (like The reference system's a remote config service Andon Cord).
5. **Tool result loop**: Implement the full converse loop: when the model returns `tool_use`, execute it, then call the API again with the tool result. Repeat until the model returns `end_turn`.

## How a Production System Does This

The reference system's AI system (`src/ai/`) extends this with:
- **the LLM streaming API** instead of direct Anthropic API (runs in cloud)
- **7 tools** including table operations, comments, and formatting
- **Mark-based addressing** (text matches instead of character offsets for resilience to concurrent edits)
- **AiTraceRing** for recording tool calls for diagnostics
- **a remote config service kill switch** (fail-safe: AI disabled until config says otherwise)
- **Per-doc-type tool gating**: prose docs get `table` tool, spreadsheets get `sheet` tool
- **Format ops accumulator**: Cell styling ships alongside the text diff

The shadow doc pattern (`src/ai/turn.rs`) is exactly what you're building here — `AiTurnContext` with a cloned `yrs::Doc`.

## Rust Book Chapters to Read

- [Chapter 12: An I/O Project](https://doc.rust-lang.org/book/ch12-00-an-io-project.html) (building real programs)
- [Reqwest documentation](https://docs.rs/reqwest/latest/reqwest/)
- [Serde guide on enums](https://serde.rs/enum-representations.html)

## Done When

- [ ] User can type a prompt and the AI reads the document
- [ ] AI can propose str_replace edits
- [ ] User sees proposals and can accept or reject
- [ ] Accepted edits sync to other connected clients via CRDT
- [ ] AI requests are rate-limited (20/minute/user)
- [ ] A mock AI service exists for testing without API keys
- [ ] `cargo clippy` passes

## What's Next?

After completing all 7 milestones, you'll have built a working collaborative editor with:
- Real-time multi-user editing (CRDT-based)
- Persistent documents
- Rich-text formatting
- User authentication and permissions
- AI assistance with a consent model

To continue learning, consider:
- **Multi-node**: Add Redis for cross-process broadcast (mirrors The reference system's production setup)
- **Tables**: Use `yrs::XmlFragment` for structured block content
- **Search**: Add a search engine (Meilisearch is simpler than the search engine for learning)
- **Deploy**: Containerize with Docker and deploy to containers/Fly.io/Railway
