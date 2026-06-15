# Search

## What It Is

the production system provides full-text search across all documents a user can access. Built on an the search engine cluster, it indexes document content at snapshot time and enforces ACL-aware filtering at query time so users only see results they're authorized to view.

Source: `src/search/mod.rs`, `src/search/service.rs`, `src/search/indexer.rs`, `src/search/routes.rs`, `src/search/text_extractor.rs`, `src/search/worker_message.rs`

## How It Works

### Architecture

```
Document Edit Flow:
  User edits doc → Snapshot scheduler flushes → SearchIndexer re-indexes

Query Flow:
  User searches → SearchService → the search engine query → ACL postfilter → results

                    ┌─────────────────────┐
                    │  the organization the search engine   │
                    │  (app-docs idx)  │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
     SearchIndexer      SearchService      SearchEngine
     (write path)       (query path)       (low-level client)
```

### Index Schema

Each document is indexed as:

```json
{
  "doc_id": "doc_abc",
  "title": "Design Review Notes",
  "content": "plain text extraction of document body",
  "owner": "alice",
  "parent_id": "fld_xyz",
  "updated_at": 1718451200,
  "acl_principals": ["user:alice", "posix_group:example-team", "link:view"]
}
```

The `acl_principals` field stores a pre-computed list of all users/groups that have access. This enables efficient index-side filtering (see below).

### Indexing Pipeline

```
Snapshot Scheduler flush
  │
  ├── Save binary to the object store
  ├── Update Database metadata
  └── Trigger SearchIndexer::index_doc(doc_id)
        │
        ├── Extract plain text from CRDT (TextExtractor)
        │   ├── Walk XmlFragment tree
        │   ├── Concatenate text nodes
        │   └── Strip formatting attributes
        │
        ├── Compute ACL principals
        │   ├── Resolve effective ACL (inheritance walk)
        │   ├── Expand to principal set: ["user:owner", "group:X", ...]
        │   └── If link_access is set: add "link:view" or "link:edit"
        │
        └── PUT to the search engine index
```

### ACL-Aware Search Queue

When ACL changes happen (doc shared/unshared, folder ACL updated), the affected documents need re-indexing. An message queue (`APP_SEARCH_ACL_QUEUE_URL`) handles this:

```
ACL change on folder "fld_xyz"
  │
  └── Enqueue all docs under fld_xyz to message queue
        │
        └── Search Worker (separate binary: search-worker)
              │
              └── Re-index each doc with updated acl_principals
```

### Query Flow

```
GET /api/search?q=design+review&limit=20
  │
  ▼
SearchService::search(query, user)
  │
  ├── Build the search engine query:
  │   {
  │     "query": {
  │       "bool": {
  │         "must": { "multi_match": { "query": "design review", "fields": ["title^3", "content"] } },
  │         "filter": {
  │           "terms": { "acl_principals": ["user:alice", "posix_group:team-a", "link:view"] }
  │         }
  │       }
  │     }
  │   }
  │
  ├── Execute against the search engine
  │
  ├── Postfilter: for each result, verify live ACL (in case index is stale)
  │   └── check_resource_permission(user, doc_id, Viewer)
  │
  └── Return filtered results with snippets
```

The two-layer filtering strategy:
1. **Index-side filter** (`acl_principals` terms query): Efficient, handles 99% of cases. May be slightly stale (up to the re-index lag).
2. **Postfilter** (live the group membership service check): Catches the edge case where an ACL changed after the last index but before the message queue worker processed it.

### Public Dictionary

A `PublicDictionary` provides autocomplete/suggestion support. It maintains a set of common terms that can be suggested without ACL concerns (they appear in enough documents to not leak information).

### Search Backfill

A separate binary (`src/bin/search-backfill.rs`) handles bulk re-indexing:
- Initial index population when search is first enabled
- Re-index after schema changes
- Index listing and deletion via `--delete-index` flag

## Why This Design?

### Why the search engine (Not the database Queries)?

the database is a key-value store — it can't do full-text search, fuzzy matching, or relevance ranking. the search engine provides:
- BM25 relevance scoring
- Tokenization and stemming
- Phrase matching
- Highlighting/snippets

### Why ACL at Index Time AND Query Time?

**Index-time only**: Fast but potentially stale. A user who just lost access might still see results until the index catches up.

**Query-time only**: Always correct but potentially slow (checking permissions for every search result is N the group membership service calls).

**Both**: Index-time filtering removes 99% of unauthorized results cheaply (no the group membership service call), and the postfilter catches the remaining <1% staleness window. This gives both performance and correctness.

### Why a Separate Search Worker?

When a folder ACL changes, potentially hundreds of child documents need re-indexing. Doing this synchronously in the ACL update handler would make the API response unacceptably slow. The message queue decouples the ACL change (fast) from the re-index work (slow, bursty).

### Why Gated by Feature Flag?

Search requires an the search engine cluster ($$$). Not all environments need it:
- **Production**: Full search enabled
- **Gamma/staging**: Search enabled for testing
- **Local dev**: Typically disabled (no local the search engine)
- **Integration tests**: May enable with a local the search engine container

The `APP_SEARCH_ENDPOINT` env var gates the entire subsystem. Absent = search disabled, server starts fine.
