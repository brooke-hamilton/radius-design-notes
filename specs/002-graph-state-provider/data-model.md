# Data Model: Git Graph State Provider

**Feature**: 002-graph-state-provider
**Date**: 2026-03-18

## Entity Overview

```text
┌─────────────────────────────────────────────────────────────────┐
│                        Git Repository                           │
│                                                                 │
│  refs/infra/<graphName>  ──► Commit ──► Root Tree               │
│                                           │                     │
│                              ┌────────────┼────────────┐        │
│                              ▼            ▼            ▼        │
│                           planes/      (other)      (other)     │
│                              │                                  │
│                              ▼                                  │
│                           radius/                               │
│                              │                                  │
│                              ▼                                  │
│                           local/                                │
│                              │                                  │
│                    ┌─────────┼─────────┐                        │
│                    ▼                   ▼                         │
│              resourcegroups/     providers/                      │
│                    │                   │                         │
│                    ▼                   ▼                         │
│                  rg1/          applications.core/                │
│                    │                   │                         │
│                    ▼                   ▼                         │
│              providers/         applications/                    │
│                    │                   │                         │
│                    ▼                   ▼                         │
│           system.resources/      my-app  ◄── Blob (storedObject)│
│                    │                                            │
│                    ▼                                            │
│            resourcegroups/                                      │
│                    │                                            │
│                    ▼                                            │
│                  rg1  ◄── Blob (storedObject)                   │
│                                                                 │
│  refs/infra-stage/<graphName>  ──► Tree (uncommitted changes)   │
└─────────────────────────────────────────────────────────────────┘
```

## Entities

### 1. storedObject (Blob Content)

The JSON payload stored in each Git blob representing a Radius resource.

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | Original resource ID (preserved case) |
| `etag` | `string` | ETag computed from JSON-marshaled `data` via `sha1` |
| `rootScope` | `string` | Normalized root scope (lowercase, leading/trailing `/`) |
| `resourceType` | `string` | Fully-qualified resource type (lowercase) |
| `routingScope` | `string` | Normalized routing scope (lowercase, leading/trailing `/`) |
| `data` | `any` | The resource data payload (arbitrary JSON) |

**Validation rules**:

- `id` must be a valid Radius resource ID parseable by `resources.Parse()`
- `etag` is recomputed on every Save from `json.Marshal(data)` using `etag.New()`
- `rootScope`, `resourceType`, `routingScope` are extracted via `databaseutil.ExtractStorageParts()`

**Example blob content**:

```json
{
  "id": "/planes/radius/local/resourceGroups/rg1/providers/Applications.Core/applications/my-app",
  "etag": "20-a3f2b1c4d5e6f7890123456789abcdef01234567",
  "rootScope": "/planes/radius/local/resourcegroups/rg1/",
  "resourceType": "applications.core/applications",
  "routingScope": "/applications.core/applications/my-app/",
  "data": {
    "name": "my-app",
    "type": "Applications.Core/applications",
    "properties": {
      "application": "my-app",
      "environment": "/planes/radius/local/resourceGroups/rg1/providers/Applications.Core/environments/default"
    }
  }
}
```

### 2. Tree Path (Derived from Resource ID)

The deterministic mapping from Radius resource ID to Git tree path.

| Component | Derivation |
|-----------|-----------|
| Graph name | Configured (default: `radius`), prefixed to path for grif API calls |
| Path segments | Each `/`-delimited segment of the resource ID, lowercased |

**Mapping function**: `idToTreePath(id string) string`

```text
Input:  "/planes/radius/local/resourceGroups/rg1/providers/Applications.Core/applications/my-app"
Output: "planes/radius/local/resourcegroups/rg1/providers/applications.core/applications/my-app"

Full grif path: "radius/planes/radius/local/resourcegroups/rg1/providers/applications.core/applications/my-app"
               ^^^^^^^
               graph name
```

**Reversibility**: `treePathToID(path string) string` — prepend `/` and join segments with `/`. Case is lost (all lowercase) but Radius IDs are case-insensitive.

**Special Character Handling**: Radius resource IDs are URL-path-safe by convention (alphanumeric, hyphens, dots, forward slashes). The path mapper validates inputs and rejects IDs containing empty segments (`//`) or NUL bytes. No percent-encoding or escaping is required because the character set maps directly to valid Git tree entry names. The lowercasing step (`NormalizePart`) is the only transformation applied.

### 3. Graph Ref

A Git reference tracking the current state of all Radius resources.

| Property | Value |
|----------|-------|
| Ref name | `refs/infra/<graphName>` (e.g., `refs/infra/radius`) |
| Points to | Latest commit object |
| Commit tree | Root tree containing all resource blobs |
| Commit parent | Previous graph commit (linear history) |
| Commit message | Includes `Source-Commit: <HEAD>` trailer |

### 4. Staging Ref

A Git reference tracking uncommitted changes during a Save or Delete operation.

| Property | Value |
|----------|-------|
| Ref name | `refs/infra-stage/<graphName>` (e.g., `refs/infra-stage/radius`) |
| Points to | Tree hash of staged (uncommitted) changes |
| Lifecycle | Created by `Put`/`DeleteNode`, consumed by `Commit`, deleted after commit |

## State Transitions

### Resource Lifecycle

```text
[Not Exists] ──Save──► [Staged] ──Commit──► [Committed]
                                                │
                                         Save───┤───Delete
                                                │       │
                                         [Staged]  [Staged]
                                                │       │
                                         Commit─┘  Commit─┘
                                                │       │
                                         [Committed] [Not Exists]
```

### Operation Flow

```text
Save Operation:
  1. Parse resource ID → extract storage parts
  2. Compute tree path from normalized ID
  3. Get existing blob (if any) for ETag check
  4. Compute new ETag from marshaled data
  5. Build storedObject envelope
  6. Marshal envelope to JSON
  7. Put blob at tree path (stages change)
  8. Commit staged change
  9. Return updated Object with new ETag

Delete Operation:
  1. Parse resource ID → compute tree path
  2. Get existing blob for ETag check
  3. DeleteNode at tree path (stages change)
  4. Commit staged change

Get Operation:
  1. Parse resource ID → compute tree path
  2. Get blob from grif
  3. Unmarshal storedObject from blob
  4. Reconstruct database.Object

Query Operation:
  1. Convert query root scope to tree path prefix
  2. Recursive tree walk from scope path
  3. Collect all blobs (leaf nodes)
  4. Unmarshal and filter by resource type
  5. Apply property filters (MatchesFilters)
  6. Apply pagination (slice windowing)
  7. Return ObjectQueryResult with pagination token
```

## Relationships

```text
database.Client (interface)
    │
    ▼
graphstore.Client (implementation)
    │
    ├── Uses: graph.Init(), graph.Put(), graph.Get(),
    │         graph.DeleteNode(), graph.Commit()
    │
    ├── Uses: databaseutil.ExtractStorageParts() for ID normalization
    │
    ├── Uses: etag.New() for ETag computation
    │
    └── Uses: database.Object.MatchesFilters() for query filtering

databaseprovider.Options
    │
    ├── TypeGraphStore → initGraphStoreClient()
    │
    └── GraphStoreOptions { RepoPath, GraphName, RemoteURL }
```
