# Git State Store for Radius

## Feature Summary

This specification proposes **Git as an alternative state store backend** for Radius. Instead of storing resource state in etcd (via Kubernetes API Server) or PostgreSQL, Radius could persist its application graph and resource data using Git's internal object model.

The Git state store would leverage Git's content-addressable storage primitives:

| Git Primitive | Purpose in Radius |
| --- | --- |
| `git hash-object -w` | Creates **blobs** - stores individual resource JSON documents |
| `git mktree` | Creates **trees** - organizes resources into hierarchical scopes (planes/resourceGroups/providers) |
| `git commit-tree` | Creates **commits** - snapshots of application state with metadata |
| `git update-ref` | Creates **refs** - named anchors to prevent garbage collection and enable branches |

These compose bottom-up: **blobs → trees → commits → refs**, forming a complete state management system with built-in versioning, history, and atomic updates.

### Why This Matters

**User Value**:

- **Version History**: Built-in audit trail of all resource changes with `git log`
- **Rollback Capability**: Restore previous application states by checking out earlier commits
- **GitOps Integration**: State store natively integrates with Git-based workflows
- **Collaboration**: Teams can fork, branch, and merge application configurations
- **Offline/Disconnected**: Work with state locally without cluster connectivity
- **Diff & Comparison**: Compare resource states across time or environments
- **Source Traceability**: Link any Radius state to the exact source code commit that produced it
- **Atomic Operations**: Each user action produces one commit, ensuring transactional consistency

---

## Scope Analysis

### Affected Repositories

| Repository | Impact | Files/Directories |
| --- | --- | --- |
| **radius** | Primary | `pkg/components/database/`, `pkg/components/database/databaseprovider/` |
| **design-notes** | Specification | `specs/<NNN-git-state-store>/` |
| **docs** | User documentation | Configuration guides for Git state store |

### Key Components to Modify

1. **New Git Store Client** (`pkg/components/database/gitstore/`)
   - Implements `database.Client` interface
   - Maps Radius resource operations to Git primitives

2. **Database Provider Factory** (`pkg/components/database/databaseprovider/factory.go`)
   - Add `TypeGit` provider type
   - Add `initGitClient` factory function

3. **Configuration** (`pkg/ucp/config/`)
   - Add Git-specific configuration options (repo path, remote URL, authentication)

---

## Existing Patterns

### Current Storage Interface

The `database.Client` interface ([pkg/components/database/client.go](../radius/pkg/components/database/client.go)) defines the contract:

```go
type Client interface {
    // Query executes a query against the data store and returns the results.
    Query(ctx context.Context, query Query, options ...QueryOptions) (*ObjectQueryResult, error)
    
    // Get retrieves a single resource from the data store by its resource id.
    Get(ctx context.Context, id string, options ...GetOptions) (*Object, error)
    
    // Delete removes a single resource from the data store by its resource id.
    Delete(ctx context.Context, id string, options ...DeleteOptions) error
    
    // Save persists a single resource to the data store.
    Save(ctx context.Context, obj *Object, options ...SaveOptions) error
}
```

### Current Storage Backends

1. **APIServer** (`apiserverstore/`) - Uses Kubernetes CRDs backed by etcd (production)
2. **PostgreSQL** (`postgres/`) - Direct PostgreSQL connection (in development)
3. **InMemory** (`inmemory/`) - Testing and development

### Database Provider Selection

The factory in [databaseprovider/factory.go](../radius/pkg/components/database/databaseprovider/factory.go) currently supports:

- `TypeAPIServer`
- `TypeInMemory`
- `TypePostgreSQL`

A `TypeGit` would be added following the same pattern.

### Reference Data Model

Resources are stored with these key fields:

- `rootScope` - UCP scope (e.g., `/planes/radius/local/resourcegroups/my-rg/`)
- `resourceType` - Fully qualified type (e.g., `Applications.Core/applications`)
- `routingScope` - Resource path within scope
- `ETag` - Optimistic concurrency control

---

## Constitution Alignment

### Principle I: API-First Design

The Git state store implements the existing `database.Client` interface, maintaining API compatibility while adding a new backend.

### Principle III: Multi-Cloud Neutrality

Git storage is cloud-agnostic—works with any Git provider (GitHub, GitLab, Azure DevOps, local repos) or local filesystem.

### Principle VII: Simplicity Over Cleverness

The implementation should use Git's well-understood primitives directly rather than building complex abstractions. Each Git operation maps cleanly to a Radius operation.

### Principle VIII: Separation of Concerns

The Git store is a pure storage backend, isolated behind the `database.Client` interface, with no impact on resource providers or controllers.

### Principle IX: Incremental Adoption & Backward Compatibility

Git store would be opt-in via configuration. Existing deployments continue using APIServer/etcd by default.

### Principle XVII: Polyglot Project Coherence

Git is a universal tool understood by all developers. Using it as a state store leverages existing skills and tools.

---

## Technical Considerations

### Git Object Mapping

| Radius Concept | Git Object | Implementation |
| --- | --- | --- |
| Resource (JSON) | Blob | `git hash-object -w --stdin` |
| Scope hierarchy | Tree | `git mktree` with entries for each resource |
| State snapshot | Commit | `git commit-tree` pointing to root tree |
| Named state | Ref | `refs/radius/<scope>` branches |

### Example Storage Structure

```text
refs/radius/planes/radius/local/
├── HEAD -> commit (latest state)
└── history/ -> tagged commits

Tree structure:
/
├── resourcegroups/
│   └── my-rg/
│       └── providers/
│           └── Applications.Core/
│               └── applications/
│                   └── my-app.json (blob)
```

### Key Operations Mapping

| `database.Client` Method | Git Operations |
| --- | --- |
| `Get(id)` | `git cat-file blob <hash>` |
| `Save(obj)` | `git hash-object -w`, `git mktree`, `git commit-tree`, `git update-ref` |
| `Delete(id)` | Rebuild tree without entry, commit, update-ref |
| `Query(scope, type)` | `git ls-tree -r` with filtering |

### Concurrency Control (ETags)

ETags can be implemented using:

- Git blob hashes (content-based)
- Or explicit version counters stored in blob metadata

Atomic updates use `git update-ref` with expected old value for compare-and-swap semantics.

### Index/Query Support

Similar to the Dapr state store design ([architecture/2024-05-radius-on-dapr.md](../design-notes/architecture/2024-05-radius-on-dapr.md#building-our-own-index)), the Git store may need supplementary index structures for efficient queries:

- Maintain an `.index` blob per scope listing all resources
- Update index atomically with resource changes

### Source Code Commit Traceability

A critical feature of the Git state store is **linking each Radius state commit to the user's source code commit**. This enables answering: "What source code was deployed when this state existed?"

#### Commit Metadata Structure

Each Radius state commit includes metadata linking to the user's source:

```json
{
  "radius_state_version": "1.0",
  "timestamp": "2026-02-07T15:30:00Z",
  "source_commit": {
    "repository": "github.com/user/my-app",
    "ref": "refs/heads/main",
    "sha": "abc123def456...",
    "author": "user@example.com"
  },
  "operation": {
    "type": "deploy",
    "resource_id": "/planes/radius/local/resourceGroups/my-rg/providers/Applications.Core/applications/my-app",
    "triggered_by": "rad deploy app.bicep"
  }
}
```

#### Traceability Flow

```text
┌─────────────────────────────────────────────────────────────┐
│                    User Source Repository                    │
│  commit abc123 ─── "Add Redis connection"                   │
│  commit def456 ─── "Update container image"                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ rad deploy (from commit def456)
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Radius State Repository                    │
│  commit rs001 ─── state after deploy                        │
│    └── metadata: { source_commit: "def456", ... }           │
│  commit rs002 ─── state after recipe completes              │
│    └── metadata: { source_commit: "def456", ... }           │
└─────────────────────────────────────────────────────────────┘
```

#### Source Commit Detection

Radius captures the user's current source commit via:

1. **Environment variable**: `RADIUS_SOURCE_COMMIT` (explicit override)
2. **Git detection**: Run `git rev-parse HEAD` in the working directory
3. **CI/CD context**: Extract from `GITHUB_SHA`, `CI_COMMIT_SHA`, etc.
4. **Fallback**: Record "unknown" if not determinable

### Atomic Commit Events

Each user action that updates Radius state MUST result in **exactly one atomic git commit**. This ensures:

- State changes are transactional (all-or-nothing)
- Each commit represents a complete, consistent state
- History shows discrete user operations, not partial updates

#### Atomicity Guarantees

| User Action | State Changes | Git Commits |
| --- | --- | --- |
| `rad deploy app.bicep` | Create/update N resources | 1 commit |
| `rad resource delete` | Delete 1 resource | 1 commit |
| Recipe execution | Create backing resources | 1 commit per async operation |

#### Implementation Approach

```go
// Transaction groups multiple Save/Delete operations into one commit
type Transaction interface {
    Save(ctx context.Context, obj *Object) error
    Delete(ctx context.Context, id string) error
    Commit(ctx context.Context, metadata CommitMetadata) error
    Rollback() error
}

// CommitMetadata links state change to source
type CommitMetadata struct {
    SourceRepository string
    SourceRef        string
    SourceCommitSHA  string
    Operation        string
    TriggeredBy      string
    Timestamp        time.Time
}
```

#### Commit Message Format

State commits use a structured message format:

```text
[radius] deploy: Applications.Core/applications/my-app

Source: github.com/user/my-app@def456
Operation: rad deploy app.bicep
Resources: 3 created, 1 updated, 0 deleted

Triggered-By: user@example.com
Radius-State-Version: 1.0
```

---

## Open Questions for Specification

1. **Sync Strategy**: Should commits push to a remote automatically, or should sync be explicit?
2. **Branching Model**: Should different environments (dev/staging/prod) be separate branches?
3. **Merge Conflicts**: How to handle concurrent modifications from multiple Radius instances?
4. **Large Binaries**: Should large output resources (container images, etc.) be stored via Git LFS?
5. **Garbage Collection**: When/how to prune old history while preserving audit trail?
6. **Authentication**: How to handle Git credentials for remote repositories?
7. **Performance**: What are acceptable latency bounds for Git operations vs. etcd/PostgreSQL?
8. **Source Commit Resolution**: How to handle cases where source commit cannot be determined (e.g., API calls without git context)?
9. **Cross-Repository Linking**: Should state commits include links to multiple source repos (e.g., app repo + recipes repo)?
10. **Async Operations**: How to handle long-running operations (recipes) that span multiple source commits?

---

## Spec Kit Prompt for `/speckit.specify`

```text
Build a Git-based state store backend for Radius that implements the database.Client interface using Git's internal object model (blobs, trees, commits, refs).

Context:
- This affects the radius repository, specifically pkg/components/database/
- The existing database.Client interface (pkg/components/database/client.go) defines Query, Get, Save, Delete operations
- Current backends include APIServer (etcd), PostgreSQL, and InMemory
- Related design: architecture/2024-05-radius-on-dapr.md discusses state store design principles

Problem being solved:
Users need state storage that provides built-in version history, rollback capability, and native GitOps integration. Current storage backends (etcd, PostgreSQL) don't provide audit trail without additional tooling. A Git state store would enable: reviewing change history, rolling back to previous states, diffing configurations across time, and working with state offline.

Design constraints:
- Must implement the existing database.Client interface without changes
- Must support optimistic concurrency control via ETags
- Must handle Query operations efficiently (consider index strategy from Dapr design)
- Must be additive—existing storage backends remain default and unchanged
- Should follow Constitution Principle VII (Simplicity Over Cleverness)
- Should use standard Git primitives (hash-object, mktree, commit-tree, update-ref)

Critical requirements:
- Each Radius state commit MUST be linked to the user's source code commit hash
- Each user action (rad deploy, rad delete, etc.) MUST produce exactly one atomic git commit
- Commit metadata MUST include source repository, ref, SHA, timestamp, and operation type
- Must support transaction semantics: group multiple Save/Delete ops into single commit

Git primitives to use:
- git hash-object -w: creates blobs (resource JSON documents)
- git mktree: creates trees (scope/type hierarchies)  
- git commit-tree: creates commits (state snapshots with source commit metadata)
- git update-ref: creates refs (named state anchors, prevents GC)

Expected outcomes:
- New gitstore package implementing database.Client
- Transaction interface for grouping operations into atomic commits
- TypeGit added to database provider factory
- Configuration options for Git repository (path, remote, auth)
- Source commit detection (env var, git rev-parse, CI context)
- Query support via index strategy (similar to Dapr design)
- Atomic Save/Delete operations with ETag enforcement
- Structured commit messages with source commit metadata
- Documentation for enabling and configuring Git state store

User value:
- Version history of all resource changes (git log)
- Rollback to previous application states (git checkout)
- Source code traceability: find exact source commit for any state
- Native GitOps workflow integration
- Collaboration via branches and merges
- Offline state access without cluster connectivity
- Cross-environment comparison (git diff)
```

---

## Next Steps

After running `/speckit.specify`, move this prompt file from `.copilot-tracking/` into your new `specs/<NNN-git-state-store>/` folder. This preserves the original research and reasoning alongside your specification.
