# Research: Git Graph State Provider

**Feature**: 002-graph-state-provider
**Date**: 2026-03-18
**Status**: Complete

## R1: grif Library API Compatibility

**Decision**: The grif library provides sufficient operations for all required functionality.

**Rationale**: The grif public API covers all core operations needed:

| Radius Operation | grif API | Notes |
|-----------------|----------|-------|
| Save (create/update) | `Put(repoPath, path, blob)` then `Commit(repoPath, graphName, message)` | Put stages changes; Commit persists them |
| Get | `Get(repoPath, path)` | Returns `NodeContent` with blob data |
| Delete | `DeleteNode(repoPath, path)` then `Commit(repoPath, graphName, message)` | DeleteNode stages; Commit persists |
| Query (tree walk) | `Get(repoPath, path)` recursively | Get on tree nodes returns immediate `Children` — recursive walking is required |
| Init graph | `Init(repoPath, name)` | Creates orphan commit with empty root tree |

**Gap: No public recursive tree walk.** The library has internal `collectAllLeaves()` for the `Status()` function but does not expose a public API for listing all blobs under a subtree. The graph store client must implement recursive tree walking by calling `Get()` on tree nodes and recursively descending into children of type `TreeNode`.

**Alternatives considered**:

- Propose upstream API addition to grif: Rejected for now — keeps the grif library simple and the recursive walk in the graph store is straightforward
- Use gitops internals directly: Rejected — `gitops` is an internal package and should not be imported

## R2: Resource ID to Tree Path Mapping

**Decision**: Direct segment mapping with lowercasing — each `/`-delimited segment of the normalized resource ID becomes a tree path segment.

**Rationale**: The spec defines this mapping. A resource ID like `/planes/radius/local/resourceGroups/rg1/providers/Applications.Core/applications/my-app` maps to tree path `planes/radius/local/resourcegroups/rg1/providers/applications.core/applications/my-app` (all segments lowercased, leading slash stripped). The mapping is reversible.

The grif `Put` call uses path format `<graphName>/<segments...>`, so the full path becomes `<graphName>/planes/radius/local/resourcegroups/rg1/providers/applications.core/applications/my-app`.

**Alternatives considered**:

- Hash-based paths: Rejected — not human-readable, prevents Git-native browsing of state
- Flattened key encoding: Rejected — loses hierarchical structure needed for scope queries

## R3: ETag Computation Strategy

**Decision**: Use the existing `etag.New()` function from `pkg/ucp/util/etag`, computing ETags on the JSON-marshaled resource data (identical to `inmemory` and `postgres` implementations).

**Rationale**: Consistency with existing backends. The ETag is computed as `sha1(json_bytes)` formatted as `"<length>-<hex>"`. This is stored alongside the resource data in the blob (as part of the serialized object metadata). On Get, the ETag is recomputed from the stored data for validation.

**Implementation detail**: The graph store wraps the `database.Object` (including its `Metadata.ETag`) into JSON before storing as a grif blob. On retrieval, the blob is deserialized back to the object and the ETag is verified by recomputing from the `Data` field.

**Alternatives considered**:

- Use Git blob SHA as ETag: Rejected — Git blob hash includes the blob header (`blob <size>\0`) making it non-portable; also, the ETag must be computed from the *data* not the storage representation
- Use `etag.NewFromRevision()`: Rejected — this is PostgreSQL-specific (uses revision counters)

## R4: Query Implementation via Recursive Tree Walk

**Decision**: Implement scope queries as recursive tree walks starting from the scope prefix subtree, using grif `Get()` on tree nodes to enumerate children.

**Rationale**: The grif library does not provide a bulk query API. The database.Client `Query` method requires returning all matching resources within a scope, optionally filtered by resource type and properties. The implementation:

1. Convert query root scope to a tree path prefix (via the same ID-to-path mapping)
2. Call `Get()` on the scope path to get the tree node and its immediate children
3. Recursively walk subtrees to find all blob (resource) nodes
4. For each blob, deserialize the stored `database.Object`
5. Apply resource type filters using `databaseutil.IDMatchesQuery()`
6. Apply property filters using `database.Object.MatchesFilters()`
7. Apply pagination (in-memory slice windowing with index-based continuation tokens)

**Performance note**: For the initial implementation, all matching resources are loaded into memory before pagination. This is acceptable for the target scale (100 resources across 10 scopes per SC-005). Caching and lazy loading are deferred optimizations.

**Alternatives considered**:

- Secondary index for type/scope lookups: Rejected — violates constitution principle VII (simplicity) and the grif library's storage-only philosophy
- Pre-computed type manifests: Rejected — adds complexity; in-process filtering is sufficient at target scale

## R5: Pagination Strategy

**Decision**: Index-based continuation tokens encoded as base64 integers, similar to the PostgreSQL implementation pattern.

**Rationale**: Since queries load all matching results in-memory before paginating, the continuation token is simply the index of the next item to return. The token is base64-encoded for opacity. When the result set is exhausted, no token is returned.

**Implementation**:

- `MaxQueryItemCount` controls page size (from `QueryOptions`)
- Continuation token is `base64(next_start_index)`
- If `len(results) > maxCount`, return first `maxCount` items and set token
- If token is provided in query, skip items before the decoded index

**Alternatives considered**:

- Cursor-based with resource IDs: Rejected — more complex and unnecessary given in-memory result sets
- No pagination (return all): Rejected — violates FR-010

## R6: Concurrency Control Model

**Decision**: Optimistic concurrency using ETags computed on JSON data, with grif's stage-then-commit workflow providing atomic commits.

**Rationale**: The graph store enforces OCC at the application level:

1. **Save with ETag**: Read current blob → compute ETag on stored data → compare with provided ETag → reject if mismatch → write new blob → commit
2. **Delete with ETag**: Read current blob → compute ETag on stored data → compare → reject if mismatch → delete node → commit
3. **Save without ETag**: Write blob → commit (upsert behavior, no concurrency check)
4. **Delete without ETag**: Delete node → commit (if exists)

The spec notes that the initial implementation targets single-process deployment. Multi-process concurrency relies on Git file-level locking (lockfile on `.git/refs`). This is acceptable per the spec assumptions.

**Rollback on commit failure**: Per the spec edge case, if a commit fails after staging, the staging ref must be rolled back. This is implemented by saving the pre-operation staging ref hash and restoring it on error.

**Alternatives considered**:

- Use Git commit hashes as version markers: Rejected — ETags must be per-resource, not per-commit
- File locking for multi-process safety: Deferred — out of scope per spec assumptions

## R7: Provider Registration Pattern

**Decision**: Register as `TypeGraphStore = "graphstore"` in the existing `databaseprovider` factory pattern, with a `GraphStoreOptions` configuration struct.

**Rationale**: Follows the established pattern exactly:

1. Add `TypeGraphStore DatabaseProviderType = "graphstore"` to `types.go`
2. Add `GraphStoreOptions` struct to `options.go` with fields: `RepoPath string`, `GraphName string`
3. Add `GraphStore GraphStoreOptions` field to the `Options` struct
4. Add `TypeGraphStore: initGraphStoreClient` to the factory map in `factory.go`
5. The factory function creates the graph store client with the configured repo path and graph name

**Configuration example**:

```yaml
databaseProvider:
  provider: "graphstore"
  graphstore:
    repoPath: "/path/to/git/repo"
    graphName: "radius"
```

**Alternatives considered**:

- Separate configuration mechanism: Rejected — breaks consistency with existing providers
- Environment variable-only config: Rejected — other providers use YAML; env vars are used only for secrets (like `GIT_TOKEN`)

## R8: CLI Remote URL Detection and Control Plane Clone

**Decision**: Extend the CLI to detect the graph store provider and read the Git remote URL. Extend control plane startup to clone from the remote URL.

**Rationale**: Per FR-017 through FR-019:

1. **CLI side**: When `databaseProvider.provider` is `"graphstore"`, the CLI reads the Git remote/origin URL from the local repository at `databaseProvider.graphstore.repoPath` using go-git. This URL is passed to the control plane configuration (e.g., via environment variable or Helm values during `rad install`).

2. **Control plane side**: On startup, if the graph store provider is configured and a `remoteURL` is provided but `repoPath` doesn't exist locally, the control plane clones the repository using go-git's `git.PlainClone()`. Git credentials come from environment variables (`GIT_TOKEN` for HTTPS, `GIT_SSH_KEY` for SSH) per FR-019.

**Key implementation note**: The CLI and control plane are independently configured (they don't share config at runtime). The remote URL must be passed through the deployment mechanism (Helm chart values → ConfigMap → control plane YAML config).

**Extended `GraphStoreOptions`**:

```go
type GraphStoreOptions struct {
    RepoPath  string `yaml:"repoPath"`
    GraphName string `yaml:"graphName"`
    RemoteURL string `yaml:"remoteUrl,omitempty"`
}
```

**Alternatives considered**:

- Direct Git library dependency in CLI: Required (go-git) — already a transitive dependency
- Remote URL auto-discovery at control plane startup: Rejected — control plane may not have access to the "local" repo that the CLI uses

## R9: Stored Object Format

**Decision**: Store the full `database.Object` (Metadata + Data) as JSON in each grif blob.

**Rationale**: When `Save` is called, the graph store client:

1. Extracts storage parts from the resource ID using `databaseutil.ExtractStorageParts()`
2. Computes a new ETag from the JSON-marshaled `Data` field using `etag.New()`
3. Creates a storage envelope containing the object metadata (ID, ETag, rootScope, resourceType, routingScope) and the data payload
4. JSON-marshals the envelope and stores it as a blob at the mapped tree path

On `Get`, the blob is read, deserialized, and the `database.Object` is reconstructed. On `Query`, all blobs under the scope subtree are deserialized and filtered.

The envelope structure enables efficient filtering during queries without needing secondary indexes:

```go
type storedObject struct {
    ID           string `json:"id"`
    ETag         string `json:"etag"`
    RootScope    string `json:"rootScope"`
    ResourceType string `json:"resourceType"`
    RoutingScope string `json:"routingScope"`
    Data         any    `json:"data"`
}
```

**Alternatives considered**:

- Store only `Data` and reconstruct metadata from path: Rejected — loses ETag and requires re-parsing paths on every read
- Store as binary (gob encoding): Rejected — JSON is human-readable and consistent with Git/grif philosophy

## R10: Graph Initialization Lifecycle

**Decision**: Initialize the graph ref on first use via `Init()` if it does not already exist, using the configured graph name (default: `radius`).

**Rationale**: Per FR-014, the graph ref must be created on first use. The factory function (`initGraphStoreClient`) calls `graph.Init()` during client initialization. If the graph already exists, this is a no-op (the error from Init indicating "already exists" is handled gracefully). If the repository doesn't exist and a remote URL is configured, the clone happens first.

**Lifecycle**:

1. If `remoteURL` is set and `repoPath` doesn't exist → clone
2. Call `graph.Init(repoPath, graphName)` → creates graph if new, no-op if exists
3. Return initialized client

**Alternatives considered**:

- Lazy initialization on first operation: Rejected — better to fail fast at startup if the repository is inaccessible
- Require manual `graph init` before use: Rejected — poor operator experience; violates SC-002 (configure in under 5 minutes)
