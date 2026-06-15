# Media & Attachments

## What It Is

the production system supports embedded media in documents: images, videos, audio files, and PDFs. Media files are stored in a dedicated object store bucket, content-addressed by SHA-256 hash, with reference tracking in the database to support cross-document sharing and garbage collection.

Source: `src/media.rs`, `src/media_observer.rs`, `src/persistence/object_store.rs` (ObjectStoreMediaStore), `src/persistence/database.rs` (DatabaseMediaRefStore)

## How It Works

### Upload Flow

```
Browser: user pastes/drags image into editor
  │
  ▼
POST /api/docs/{doc_id}/media (multipart form)
  │
  ├── Auth: extract user, check Editor permission on doc
  ├── Rate limit: per-user media upload counter (100/min)
  ├── Rate limit: per-user bytes counter (100 MB/min)
  ├── Validate:
  │   ├── Content-Type → determine MediaKind (image/video/audio/pdf)
  │   ├── Size limits per kind:
  │   │   ├── Image: max 10 MB
  │   │   ├── Video: max 50 MB
  │   │   ├── Audio: max 30 MB
  │   │   └── PDF: max 20 MB
  │   └── Minimum size: 512 bytes (reject empty/corrupt)
  │
  ├── SHA-256 hash the content
  ├── Check object store: does this hash already exist? (dedup)
  │   ├── Yes → skip upload, reuse existing blob
  │   └── No → PUT to the object store media bucket at key = hash
  │
  ├── Create MediaRef in the database:
  │   { doc_id, hash, state: "active", kind, size, uploaded_by, ... }
  │
  └── Return: { hash, url: "/api/docs/{doc_id}/media/{hash}" }
```

### Read Flow

```
GET /api/docs/{doc_id}/media/{hash}
  │
  ├── Auth: check Viewer permission on doc
  ├── Load blob from the object store by hash
  └── Return with Content-Type and cache headers
```

### Cross-Document References

When a user copies media from Doc A to Doc B (e.g., copy-paste with images):

```
POST /api/docs/{doc_b}/media/reference
  Body: { "source_doc_id": "doc_a", "hash": "abc123" }
  │
  ├── Auth: check Viewer on source doc (user can see the original)
  ├── Auth: check Editor on target doc (user can write to destination)
  ├── Create new MediaRef for doc_b pointing to same hash
  └── Return success (no bytes copied — same object store blob)
```

This is why media is content-addressed: the same image in 10 docs is stored once in the object store with 10 reference records in the database.

### Media Observer

The `MediaObserver` watches for CRDT changes that add or remove media nodes from documents. When the CRDT document is modified:

1. Observer detects media node insertions/deletions
2. Updates `MediaRef` state in the database (`active` → `orphaned` when removed)
3. Orphaned media can be garbage collected (future: lifecycle policy)

This ensures that if a user deletes an image from a doc, the reference is marked as no longer needed — but the object store blob persists (other docs may reference the same hash).

### Storage Architecture

```
object store Media Bucket
├── {sha256_hash_1}  ← raw bytes, no doc_id in path
├── {sha256_hash_2}
└── ...

the database (app-media-refs table)
├── DOC#doc_a | MEDIA#hash_1 → { state: active, kind: image, ... }
├── DOC#doc_a | MEDIA#hash_2 → { state: active, kind: video, ... }
├── DOC#doc_b | MEDIA#hash_1 → { state: active, kind: image, ... }  ← same hash, different doc
└── ...
```

## Why This Design?

### Why Content-Addressing (SHA-256)?

- **Deduplication**: The same screenshot pasted into 50 docs costs one object store blob, not 50.
- **Integrity**: The hash proves the content hasn't been corrupted or tampered with.
- **Simplicity**: No need for unique IDs or name collision handling.

### Why a Separate Media Bucket?

- **Security boundary**: Media may contain sensitive content (screenshots with PII). Separating it from document snapshots allows different access policies.
- **Lifecycle management**: Media can have different retention/cold storage policies than CRDT snapshots.
- **Cost monitoring**: Easy to track media storage costs separately.

### Why Reference Tracking (Not Just Inline)?

Without references:
- Deleting a doc would leave orphaned blobs in the object store (cost leak)
- Cross-doc copy would require duplicating bytes (waste)
- No way to know if a blob is still needed by any doc

With references:
- Garbage collection: delete blobs where all references are `orphaned`
- Cross-doc sharing: create a reference record, no bytes copied
- Audit: know exactly which docs use which media

### Why Per-Kind Size Limits?

- **Memory safety**: The server buffers the upload in memory for hashing. A 2 GB video would OOM the container.
- **Cost control**: object store storage + data transfer costs scale with blob size.
- **UX**: Large files (>50 MB) are better served from dedicated video/file hosting, not an editor.

The limits are generous enough for normal document work (screenshots, diagrams, short screen recordings) while preventing abuse.
