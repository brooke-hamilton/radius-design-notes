# Go API Contracts: Git Graph State Provider

**Feature**: 002-graph-state-provider
**Date**: 2026-03-18

## Package: `pkg/components/database/graphstore`

### Client Struct

```go
package graphstore

import (
    "context"
    "github.com/radius-project/radius/pkg/components/database"
)

// Compile-time interface check.
var _ database.Client = (*Client)(nil)

// Client implements database.Client using the git-infra-graph (grif) library
// to persist resource state as a versioned graph inside a Git repository.
type Client struct {
    repoPath  string // Absolute path to the Git repository
    graphName string // Graph name (default: "radius"), maps to refs/infra/<name>
}

// NewClient creates a new graph store client for the given Git repository
// and graph name. The graph is initialized if it does not already exist.
func NewClient(repoPath string, graphName string) (*Client, error)
```

### database.Client Implementation

```go
// Get retrieves a resource by ID from the Git graph.
//
// Returns ErrNotFound if the resource does not exist.
// Returns ErrInvalid if the ID is empty or unparseable.
func (c *Client) Get(ctx context.Context, id string, options ...database.GetOptions) (*database.Object, error)

// Save persists a resource to the Git graph. Creates a new entry or updates
// an existing one. Each save produces a Git commit.
//
// When an ETag is provided via options, returns ErrConcurrency if the stored
// ETag does not match (resource modified or deleted since ETag was obtained).
// Returns ErrInvalid if obj is nil or obj.ID is empty/unparseable.
func (c *Client) Save(ctx context.Context, obj *database.Object, options ...database.SaveOptions) error

// Delete removes a resource from the Git graph by ID. Each delete produces
// a Git commit.
//
// Returns ErrNotFound if the resource does not exist (when no ETag provided).
// When an ETag is provided via options, returns ErrConcurrency if the stored
// ETag does not match (resource modified or deleted since ETag was obtained).
// Returns ErrInvalid if the ID is empty or unparseable.
func (c *Client) Delete(ctx context.Context, id string, options ...database.DeleteOptions) error

// Query executes a scope-based query against the Git graph, returning all
// matching resources with optional filtering and pagination.
//
// Returns ErrInvalid if the query fails validation (missing RootScope or ResourceType).
func (c *Client) Query(ctx context.Context, query database.Query, options ...database.QueryOptions) (*database.ObjectQueryResult, error)
```

### Internal Types

```go
// storedObject is the JSON envelope persisted as a Git blob for each resource.
type storedObject struct {
    ID           string `json:"id"`
    ETag         string `json:"etag"`
    RootScope    string `json:"rootScope"`
    ResourceType string `json:"resourceType"`
    RoutingScope string `json:"routingScope"`
    Data         any    `json:"data"`
}
```

### Path Mapping Functions

```go
// idToTreePath converts a Radius resource ID to a grif tree path.
// The resource ID segments are lowercased and the leading slash is stripped.
// The graph name is NOT included in the returned path.
//
// Example: "/planes/radius/local/resourceGroups/rg1" → "planes/radius/local/resourcegroups/rg1"
func idToTreePath(id string) string

// idToGrifPath converts a Radius resource ID to a full grif path including
// the graph name prefix.
//
// Example: (graphName="radius", id="/planes/radius/local/resourceGroups/rg1")
//          → "radius/planes/radius/local/resourcegroups/rg1"
func (c *Client) idToGrifPath(id string) string

// scopeToGrifPath converts a query root scope to a grif tree path prefix
// for tree walking.
func (c *Client) scopeToGrifPath(rootScope string) string
```

## Package: `pkg/components/database/databaseprovider`

### Type Registration (types.go)

```go
const (
    // TypeGraphStore represents the Git graph store provider.
    TypeGraphStore DatabaseProviderType = "graphstore"
)
```

### Options (options.go)

```go
// GraphStoreOptions represents options for the Git graph store.
type GraphStoreOptions struct {
    // RepoPath is the absolute path to the Git repository used for state storage.
    RepoPath string `yaml:"repoPath"`

    // GraphName is the name of the graph within the repository.
    // Defaults to "radius" if empty. Maps to refs/infra/<graphName>.
    GraphName string `yaml:"graphName"`

    // RemoteURL is the Git remote URL to clone from if the repository
    // does not exist at RepoPath. Optional — if empty, RepoPath must
    // already contain a valid Git repository.
    RemoteURL string `yaml:"remoteUrl,omitempty"`
}
```

```go
// Options (updated)
type Options struct {
    Provider   DatabaseProviderType `yaml:"provider"`
    APIServer  APIServerOptions     `yaml:"apiserver,omitempty"`
    InMemory   InMemoryOptions      `yaml:"inmemory,omitempty"`
    PostgreSQL PostgreSQLOptions    `yaml:"postgresql,omitempty"`
    GraphStore GraphStoreOptions    `yaml:"graphstore,omitempty"` // NEW
}
```

### Factory Registration (factory.go)

```go
var databaseClientFactory = map[DatabaseProviderType]databaseClientFactoryFunc{
    TypeAPIServer:   initAPIServerClient,
    TypeInMemory:    initInMemoryClient,
    TypePostgreSQL:  initPostgreSQLClient,
    TypeGraphStore:  initGraphStoreClient,  // NEW
}

// initGraphStoreClient creates a new graph store client.
// Clones from RemoteURL if configured and RepoPath does not exist.
// Initializes the graph if it does not already exist.
func initGraphStoreClient(ctx context.Context, opt Options) (store.Client, error)
```

## Error Semantics

The graph store client returns the same error types as other database.Client implementations:

| Condition | Error Type | When |
|-----------|-----------|------|
| Resource not found | `database.ErrNotFound{ID: id}` | Get, Delete (without ETag) |
| Invalid arguments | `database.ErrInvalid{Message: ...}` | Nil context, unparseable ID, invalid query |
| ETag mismatch | `database.ErrConcurrency{}` | Save/Delete with stale ETag |
| Resource deleted + ETag provided | `database.ErrConcurrency{}` | Save/Delete when resource gone but ETag was given |

## Configuration Examples

### Local development (in-process)

```yaml
databaseProvider:
  provider: "graphstore"
  graphstore:
    repoPath: "/home/user/my-project"
    graphName: "radius"
```

### Control plane with remote clone

```yaml
databaseProvider:
  provider: "graphstore"
  graphstore:
    repoPath: "/var/lib/radius/state"
    graphName: "radius"
    remoteUrl: "https://github.com/org/infra-state.git"
```

### Environment variables for Git credentials

```bash
# HTTPS token authentication
GIT_TOKEN=ghp_xxxxxxxxxxxx

# SSH key authentication (base64-encoded private key)
GIT_SSH_KEY=LS0tLS1CRUdJTi4uLg==
```
