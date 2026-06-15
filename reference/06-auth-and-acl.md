# Authentication & Authorization

## What It Is

A production system's auth system has two layers:

1. **Authentication** — Who is this user? Handled by mTLS auth via the CDN an edge auth function.
2. **Authorization (ACL)** — What can this user do on this resource? Handled by the group membership service group membership checks with folder-tree inheritance.

Source: `src/auth.rs`, `src/acl.rs`, `src/acl_routes.rs`, `src/groups.rs`

## How Authentication Works

### Production Flow

```
Browser                the CDN              an edge auth function           the load balancer              the production system
  │                       │                       │                   │                  │
  │── HTTPS request ────▶│                       │                   │                  │
  │                       │── Invoke ────────────▶│                   │                  │
  │                       │                       │── mTLS auth ──▶│                  │
  │                       │                       │  (verify client   │                  │
  │                       │                       │   certificate)    │                  │
  │                       │◀── x-forwarded-user ──│                   │                  │
  │                       │                       │                   │                  │
  │                       │── Forward + headers ─────────────────────▶│                  │
  │                       │   x-forwarded-user: alice                 │── extract ──────▶│
  │                       │   x-origin-verify: <secret>              │                  │
```

Key headers:
- `x-forwarded-user` — The authenticated user alias (stamped by an edge auth function after the mTLS gateway validation)
- `x-origin-verify` — A shared secret proving the request came through OUR the CDN's distribution (prevents origin confusion attacks where an attacker's CDN distribution reaches the load balancer)

### Origin Verification

Problem: The load balancer security group allows all CDN IPs (shared managed prefix list). An attacker could create their own CDN distribution, point it at the load balancer, and forge `x-forwarded-user`.

Solution: Infrastructure-as-code injects a secret as a custom origin header on the CDN distribution. The backend validates this header on every request. An attacker's distribution can't know this secret.

Rollout modes:
- `monitor` — Log mismatches but allow (safe rollout during edge propagation)
- `enforce` — Reject requests with missing/wrong secret (production)

### Local Development

In debug/test builds, if `x-forwarded-user` is absent, the server falls back to `APP_DEV_ALIAS` env var. This allows local development without the mTLS gateway infrastructure.

### Agent Identity (MCP)

External AI agents authenticate as a user but also carry:
- `x-agent-platform` — Which tool/platform (e.g., "ai-assistant")
- `x-agent-instance` — Unique instance ID

Both must be present (all-or-nothing). This creates a `Principal` with distinct attribution for audit logs.

## How Authorization (ACL) Works

### Permission Levels

```
Owner > Editor > Viewer
```

| Level | Can do |
|-------|--------|
| Owner | Everything + manage ACL + delete |
| Editor | Read + write content + comment |
| Viewer | Read content + comment (no edits) |

Additionally, a doc/folder may have a `link_access` setting:
- `None` — Only explicitly listed users/groups can access
- `View` — Anyone with the link can view
- `Edit` — Anyone with the link can edit

### ACL Structure

Each document or folder can have an explicit ACL:

```json
{
  "entries": [
    { "type": "user", "id": "alice", "level": "Owner" },
    { "type": "posix_group", "id": "example-team", "level": "Editor" },
    { "type": "ldap_group", "id": "example-viewers", "level": "Viewer" },
    { "type": "team", "id": "T12345", "level": "Editor" }
  ],
  "link_access": "View"
}
```

### ACL Inheritance

Documents and folders can inherit permissions from their parent folder:

```
Folder A (ACL: team=Editor)
├── Folder B (inherits=true, no explicit ACL)
│   └── Doc X (inherits=true) ← Effective ACL: team=Editor (from Folder A)
└── Doc Y (inherits=false, own ACL) ← Uses its own ACL only
```

Resolution algorithm (`resolve_effective_acl`):
1. Check current resource: has explicit ACL AND `inherits_permissions == false`? → Use it.
2. If `inherits_permissions == true`: walk up `parent_id` chain.
3. First ancestor with `inherits_permissions == false` AND an explicit ACL wins.
4. If we reach the root (parent = "ROOT") with no ACL found → owner-only access.
5. Safety cap: max 20 levels of inheritance to prevent infinite loops.

### the group membership service Client

the group membership service is an internal group membership service. The `GroupClient` trait:

```rust
trait GroupClient: Send + Sync {
    async fn is_member_of(&self, user: &str, group: &AclEntry) -> Result<bool>;
    async fn batch_is_member_of(&self, user: &str, groups: &[AclEntry]) -> Result<Vec<bool>>;
    async fn list_groups_for_user(&self, user: &str) -> Result<Vec<AclEntry>>;
}
```

Implementations:
- `RemoteGroupClient` — Production. Calls the group membership service with Redis caching (5-minute TTL).
- `EnvMockGroupClient` — Test builds. Reads memberships from `APP_GROUPS_MOCK_MEMBERSHIPS` env var.
- `NoopGroupClient` — When the group membership service isn't configured. All group checks return `false`.

### Permission Check Flow

```
Request arrives (e.g., PATCH /api/docs/{id})
  │
  ├── Extract user alias from headers
  ├── Load doc metadata from the database
  ├── Resolve effective ACL (walk inheritance chain)
  ├── Check: is user the owner? → Owner access
  ├── Check: is user in any ACL entry? (the group membership service batch check)
  ├── Check: does link_access grant sufficient level?
  └── Return highest matching permission level (or 403)
```

## Why This Design?

### Why the mTLS gateway + the CDN (Not Direct the load balancer Auth)?

- **Zero-trust perimeter**: mTLS auth validates the user's X.509 client certificate at the edge. No passwords, no tokens to steal.
- **Edge caching**: The CDN caches static assets (frontend JS/CSS) at the edge while routing API/WS to the origin.
- **WAF**: The CDN integrates with a web application firewall for additional protection.

### Why the group membership service Groups (Not Role-Based)?

- **Organizational alignment**: Teams in the organization already manage access via the group membership service groups (posix, LDAP, org directory teams). Users expect to share docs with "my team" using the same groups they use elsewhere.
- **Dynamic membership**: When someone joins a team, they immediately get access to all that team's docs without manual grants.

### Why Folder-Level Inheritance?

Without inheritance, sharing a folder of 50 docs with a new team member would require 50 individual ACL updates. With inheritance, one ACL on the parent folder covers everything inside it.

The `inherits_permissions` flag allows opting out: a confidential doc inside a shared folder can break inheritance and have its own restricted ACL.

### Why Redis Caching for the group membership service?

the group membership service calls take 50-200ms and the same user's group membership rarely changes within minutes. A 5-minute Redis TTL means:
- First ACL check for a user: ~100ms (the group membership service call)
- Subsequent checks: <1ms (Redis lookup)
- Group membership changes propagate within 5 minutes
