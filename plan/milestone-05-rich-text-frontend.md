# Milestone 5: Rich Text Frontend

## Goal

Replace the plain textarea with a real rich-text editor (TipTap/ProseMirror + Yjs) that supports headings, bold, italic, lists, and live cursors. The Rust backend doesn't change much — the heavy lifting is on the frontend — but you'll refine the sync protocol and add awareness support.

## What You're Building

- A React/TypeScript frontend with TipTap editor
- `y-websocket` provider connecting to your Rust backend
- Awareness protocol: see other users' cursors in real time
- Basic toolbar: headings, bold, italic, bullet list

## Rust Concepts Introduced

| Concept | Where You'll Encounter It |
|---------|--------------------------|
| Awareness protocol (new message tag) | Handling `0x04` messages alongside CRDT updates |
| Binary frame parsing with multiple tags | Switch-dispatching on first byte |
| Per-connection state | Tracking which user is on which doc |
| Static file serving in development | Vite dev server proxy OR serving the build |

## Step-by-Step

### 1. Create the frontend project

```bash
# From your project root
mkdir frontend && cd frontend
npm create vite@latest . -- --template react-ts
npm install
npm install yjs y-websocket @tiptap/react @tiptap/starter-kit @tiptap/extension-collaboration @tiptap/extension-collaboration-cursor
```

### 2. Build the editor component

```tsx
// frontend/src/Editor.tsx
import { useEditor, EditorContent } from '@tiptap/react'
import StarterKit from '@tiptap/starter-kit'
import Collaboration from '@tiptap/extension-collaboration'
import CollaborationCursor from '@tiptap/extension-collaboration-cursor'
import * as Y from 'yjs'
import { WebsocketProvider } from 'y-websocket'
import { useEffect, useMemo } from 'react'

export function Editor({ docId, username }: { docId: string; username: string }) {
  const ydoc = useMemo(() => new Y.Doc(), [])

  const provider = useMemo(
    () => new WebsocketProvider(
      `ws://localhost:3000`,
      docId,
      ydoc,
      { connect: true }
    ),
    [ydoc, docId]
  )

  useEffect(() => {
    provider.awareness.setLocalStateField('user', {
      name: username,
      color: '#' + Math.floor(Math.random() * 16777215).toString(16),
    })
    return () => { provider.destroy() }
  }, [provider, username])

  const editor = useEditor({
    extensions: [
      StarterKit.configure({ history: false }),
      Collaboration.configure({ document: ydoc }),
      CollaborationCursor.configure({ provider }),
    ],
  })

  return (
    <div>
      <div className="toolbar">
        <button onClick={() => editor?.chain().focus().toggleBold().run()}>B</button>
        <button onClick={() => editor?.chain().focus().toggleItalic().run()}>I</button>
        <button onClick={() => editor?.chain().focus().toggleHeading({ level: 1 }).run()}>H1</button>
        <button onClick={() => editor?.chain().focus().toggleBulletList().run()}>• List</button>
      </div>
      <EditorContent editor={editor} />
    </div>
  )
}
```

### 3. Update your Rust server to speak y-websocket's protocol

The `y-websocket` library uses a slightly different wire format than raw Yjs:
- Messages are prefixed with a **message type** byte
- Sync messages have an additional sub-type byte

You have two options:

**Option A**: Match `y-websocket`'s protocol exactly (recommended for learning):
```
[0x00][sync sub-type][payload]  — sync messages
[0x01][payload]                 — awareness messages
```

**Option B**: Write a custom Yjs provider on the frontend that speaks YOUR protocol (from Milestone 3). This is what a production system does — it has a custom `a custom WebSocket provider` class.

For learning, Option A is easier: just adjust your message tags to match `y-websocket`'s expectations.

### 4. Add awareness support to the server

Awareness messages carry cursor position and user metadata. They don't go through the CRDT — they're broadcast directly:

```rust
const MSG_AWARENESS: u8 = 0x01; // in y-websocket's encoding

// In the recv task:
MSG_AWARENESS => {
    // Don't apply to doc — just broadcast to other clients
    let _ = awareness_tx.send(data.to_vec());
}
```

You'll need a separate broadcast channel for awareness (or use the same one with a tag prefix).

### 5. Proxy the frontend in development

Add to your Vite config (`frontend/vite.config.ts`):
```ts
export default defineConfig({
  server: {
    proxy: {
      '/ws': { target: 'ws://localhost:3000', ws: true },
      '/api': { target: 'http://localhost:3000' },
    }
  }
})
```

Run both:
```bash
# Terminal 1
cargo run

# Terminal 2
cd frontend && npm run dev
```

### 6. Test it

Open two browser tabs at `http://localhost:5173`. You should see:
- A rich-text editor with a toolbar
- Colored cursors showing the other user's position
- Real-time sync of all formatting (bold, headings, lists)

## Understanding Awareness

Awareness is SEPARATE from the document CRDT:

| | Document (CRDT) | Awareness |
|---|---|---|
| Data | Text content, formatting | Cursor position, user name, color |
| Persistence | Yes (saved to disk) | No (ephemeral) |
| Conflict resolution | CRDT merge | Last-write-wins (per user) |
| Delivery | Must be reliable | Best-effort (stale cursors are fine) |

Awareness updates are typically sent:
- On every cursor move (debounced to ~50ms)
- On selection change
- On user state change (idle, active, typing)

## Exercises

1. **User list sidebar**: Show a list of connected users (name + color dot) by reading awareness states.
2. **Typing indicator**: When a user is actively typing, show "alice is typing..." (debounced).
3. **Document title**: Store the title in the Yjs doc (as a separate `Y.Text`). Show it in the browser tab.
4. **Multiple documents**: Add routing (`/docs/{id}`) so users can create and switch between docs.

## How a Production System Does This

The reference system's frontend (`CollabEditorUI`) uses:
- TipTap with `@tiptap/extension-collaboration` (same as here)
- A **custom** `a custom WebSocket provider` (not y-websocket) that speaks the binary protocol from `src/ws/protocol.rs`
- Awareness flows over a separate Redis pub/sub channel (`doc:{id}:awareness`)
- Additional message types: AI responses, comments, format ops, write-rejected toasts

The backend awareness handling is in `src/ws/mod.rs` — it's a simple broadcast without touching the CRDT doc.

## Rust Book Chapters to Read

- [Chapter 18: Patterns and Matching](https://doc.rust-lang.org/book/ch18-00-patterns.html) (for dispatching on message types)

## External Resources

- [TipTap Collaboration Guide](https://tiptap.dev/docs/editor/collaboration)
- [y-websocket source](https://github.com/yjs/y-websocket) (to understand the wire protocol)
- [Yjs Awareness Protocol](https://github.com/yjs/y-protocols/blob/master/awareness.js)

## Done When

- [ ] Rich-text editor with bold, italic, headings, lists
- [ ] Two tabs show each other's cursors (colored, with username labels)
- [ ] Formatting syncs in real time (bold in one tab appears in the other)
- [ ] Documents persist across server restarts (from Milestone 4)
- [ ] `cargo clippy` passes
