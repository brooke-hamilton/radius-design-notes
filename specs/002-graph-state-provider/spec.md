# Feature Specification: Git Graph State Provider

**Feature Branch**: `002-graph-state-provider`
**Created**: 2026-03-18
**Status**: Draft
**Input**: User description: "Build a new database.Client implementation for Radius that uses the git-infra-graph (grif) Go library to persist resource state as a versioned graph inside a Git repository."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Store and Retrieve Resource State in Git (Priority: P1)

As a platform engineer, I want Radius to persist resource state directly in a Git repository so that I can manage infrastructure state without provisioning a separate database.

**Why this priority**: This is the core value proposition — without the ability to save, retrieve, and delete resources, no other functionality is possible. It enables the zero-infrastructure state storage promise.

**Independent Test**: Can be fully tested by configuring Radius to use the graph store backend, deploying a resource, and verifying the resource is retrievable via `Get` and appears in `Query` results. Delivers value by proving the end-to-end storage path works.

**Acceptance Scenarios**:

1. **Given** Radius is configured with the graph store backend and a valid Git repository path, **When** a resource is saved via `Save`, **Then** the resource is persisted as a Git blob within the graph's tree structure and a commit is created.
2. **Given** a resource has been saved, **When** `Get` is called with the resource's ID, **Then** the full resource object is returned with correct data and a valid ETag.
3. **Given** a resource has been saved, **When** `Delete` is called with the resource's ID and correct ETag, **Then** the resource is removed from the graph and a commit records the deletion.
4. **Given** a resource has been saved, **When** `Delete` is called with an incorrect ETag, **Then** the operation fails with a concurrency error.

---

### User Story 2 - Query Resources by Scope and Type (Priority: P2)

As a platform engineer, I want to query resources by scope and resource type so that I can list all resources within a resource group, plane, or other scope boundary.

**Why this priority**: Querying is essential for resource listing and discovery operations that Radius performs constantly. Without scope-based queries, the control plane cannot enumerate resources.

**Independent Test**: Can be tested by saving multiple resources across different scopes and resource types, then querying by root scope and resource type to verify correct filtering.

**Acceptance Scenarios**:

1. **Given** multiple resources exist across different resource groups, **When** a query is issued with a specific root scope, **Then** only resources within that scope are returned.
2. **Given** multiple resources of different types exist within a scope, **When** a query is issued with a resource type filter, **Then** only resources matching that type are returned.
3. **Given** a query matches more resources than the page size, **When** pagination is used, **Then** continuation tokens allow retrieval of all matching resources across pages.
4. **Given** resources have varying property values, **When** a query includes property-level filters (QueryFilter), **Then** only resources matching those filter criteria are returned.

---

### User Story 3 - Optimistic Concurrency Control (Priority: P2)

As a Radius control plane operator, I want the graph store to enforce optimistic concurrency control (OCC) via ETags so that concurrent writers do not silently overwrite each other's changes.

**Why this priority**: OCC is a correctness requirement for all `database.Client` implementations. Without it, concurrent operations could corrupt state.

**Independent Test**: Can be tested by saving a resource, obtaining its ETag, modifying the resource through a separate save, and verifying the first ETag is now stale and produces a concurrency error on save.

**Acceptance Scenarios**:

1. **Given** a resource is saved and an ETag is returned, **When** `Save` is called again with the correct ETag and updated data, **Then** the save succeeds and a new ETag is returned.
2. **Given** a resource's ETag has changed due to another write, **When** `Save` is called with the stale ETag, **Then** the operation fails with `ErrConcurrency`.
3. **Given** a resource's ETag has changed, **When** `Delete` is called with the stale ETag, **Then** the operation fails with `ErrConcurrency`.

---

### User Story 4 - Provider Registration and Configuration (Priority: P3)

As a Radius operator, I want to enable the graph store backend by specifying it in the Radius YAML configuration so that I can switch between storage backends without code changes.

**Why this priority**: Configuration-driven provider selection follows Radius's established pattern and is required for operators to adopt the graph store. However, the core storage logic (P1-P2) must work first.

**Independent Test**: Can be tested by creating a Radius configuration file with the graph store provider type and options, starting Radius, and verifying it initializes the graph store client successfully.

**Acceptance Scenarios**:

1. **Given** a Radius configuration file specifies the graph store as the database provider type, **When** Radius starts, **Then** the graph store client is initialized with the configured Git repository path and graph name.
2. **Given** an invalid or inaccessible Git repository path is configured, **When** Radius starts, **Then** an informative error message is returned and startup fails gracefully.
3. **Given** the graph store is configured, **When** the provider initializes, **Then** the graph ref is created in the Git repository if it does not already exist.

---

### User Story 5 - Full Audit Trail via Git History (Priority: P3)

As a platform engineer, I want every state mutation to be recorded as a Git commit so that I have a complete audit trail of all resource changes.

**Why this priority**: The audit trail is a differentiating capability of the Git-backed store, but it is a byproduct of the core storage mechanism (P1) rather than independent functionality.

**Independent Test**: Can be tested by performing a series of save and delete operations, then inspecting the Git log for the graph ref to verify each mutation is captured as a distinct commit with a meaningful message.

**Acceptance Scenarios**:

1. **Given** a resource is saved, **When** the Git log for the graph ref is inspected, **Then** a commit exists recording the save operation.
2. **Given** a resource is deleted, **When** the Git log for the graph ref is inspected, **Then** a commit exists recording the deletion.
3. **Given** multiple mutations have occurred, **When** the commit history is listed, **Then** mutations are ordered chronologically and each commit references the affected resource path.

---

### Edge Cases

- What happens when the Git repository's disk is full and a save is attempted?
- How does the system handle a corrupted Git object database?
- What happens when two concurrent save operations target the same resource simultaneously?
- How does the system behave when the configured graph name contains invalid characters?
- What happens when a `Get` is issued for a resource ID that was never saved?
- How does the system handle extremely deep resource hierarchies (e.g., deeply nested scopes)?
- What happens when the Git repository path does not exist and cannot be created?
- How does the system handle resource IDs with special characters that may conflict with Git path conventions?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST implement the `database.Client` interface (`Query`, `Get`, `Delete`, `Save`) with all method signatures and return types matching the existing contract.
- **FR-002**: System MUST persist each resource as a JSON-serialized blob in the Git object database, with the resource's hierarchical ID mapped deterministically to a tree path in the graph.
- **FR-003**: System MUST normalize resource IDs using established Radius ID normalization utilities before mapping to tree paths, consistent with other `database.Client` implementations.
- **FR-004**: System MUST compute ETags on JSON-marshaled resource data and enforce optimistic concurrency control on `Save` and `Delete` operations.
- **FR-005**: System MUST return a not-found error when `Get` or `Delete` is called with an ID that does not exist in the graph.
- **FR-006**: System MUST return a concurrency error when `Save` or `Delete` is called with an ETag that does not match the current stored ETag.
- **FR-007**: System MUST support scope-based queries, mapping the root scope to a subtree path and returning all matching resources within that scope.
- **FR-008**: System MUST support resource type filtering in queries, returning only resources matching the specified resource type.
- **FR-009**: System MUST support property-level filtering using query filter values, applying filters in-process after retrieving candidate resources.
- **FR-010**: System MUST support pagination for query results using continuation tokens.
- **FR-011**: System MUST create a Git commit for each state-mutating operation (`Save`, `Delete`), capturing the mutation in the graph's commit history.
- **FR-012**: System MUST handle scope queries by converting scope types to resource types using established Radius scope conversion utilities.
- **FR-013**: System MUST register as a new database provider type in the Radius provider/factory pattern, selectable via YAML configuration.
- **FR-014**: System MUST initialize the graph ref on first use if it does not already exist, creating the named graph in the Git repository.
- **FR-015**: System MUST pass all shared conformance tests that validate `database.Client` behaviors including CRUD, optimistic concurrency, scope queries, filters, and error semantics.
- **FR-016**: System MUST include a compile-time interface check to verify interface compliance with `database.Client`.

### Key Entities

- **Graph Store Client**: The `database.Client` implementation that translates Radius storage operations into graph library calls against a Git repository.
- **Resource Object**: A database object containing the resource ID, resource type, root scope, routing scope, data payload, and ETag. Serialized as JSON for blob storage.
- **Graph Ref**: A Git ref that identifies the named graph within the Git repository. Each graph ref points to the latest commit representing the current state snapshot.
- **Tree Path**: The deterministic mapping of a Radius resource ID to a hierarchical path within the graph's tree structure. Derived from the normalized and lowercased segments of the resource ID.
- **Staging Ref**: A Git ref used internally to track uncommitted changes before they are committed to the graph ref.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: All shared conformance tests pass against the graph store client, demonstrating behavioral parity with existing backends.
- **SC-002**: Platform engineers can configure the graph store backend in under 5 minutes by adding a database provider section to the Radius YAML configuration file.
- **SC-003**: Resources saved through the graph store are recoverable from a fresh clone of the Git repository — state survives process restarts when backed by a persistent repository.
- **SC-004**: Every state mutation (save, delete) produces a distinct Git commit, enabling operators to inspect the full change history using standard Git tooling.
- **SC-005**: The graph store handles at least 100 resources across 10 scopes without query response degradation noticeable to the control plane (sub-second for typical scope queries).
- **SC-006**: Concurrent save operations to different resources succeed without data loss; concurrent saves to the same resource are correctly rejected via OCC when ETags conflict.
- **SC-007**: A contributor can run Radius locally with the graph store backend without installing any external database — only a Git repository on the local filesystem is required.

## Assumptions

- The initial implementation targets a single-process deployment model. Multi-process or multi-replica concurrency is out of scope for the first version and will rely on the version control system's built-in file locking.
- Remote synchronization (push/pull to a remote repository) is out of scope for the initial implementation. The graph store operates on a local repository only.
- The graph library provides sufficient operations for all required functionality (initialization, put, get, delete, commit, tree walking for queries). Any library gaps will be addressed as separate contributions.
- The initial implementation prioritizes correctness over performance. Performance optimization (e.g., caching, batch commits) can be addressed in future iterations.
- Resource IDs are case-insensitive, matching the behavior of other Radius database backends. The ID-to-path mapping lowercases all path segments.
- The repository used by the graph store may be the same repository that contains the Radius source code, or it may be a separate dedicated repository. The implementation is agnostic to this choice.
- Deployment configurations (Helm charts, Kubernetes manifests) for the graph store are out of scope for the initial implementation. The initial target is local development and testing.

## Scope Boundaries

### In Scope

- New `database.Client` implementation backed by the graph library
- Provider registration in the database provider/factory pattern
- YAML configuration options for the graph store (repository path, graph name)
- Shared conformance test integration
- ID-to-path mapping logic
- ETag computation and OCC enforcement
- Scope-based and type-based query support with pagination
- Unit and integration tests for the graph store package

### Out of Scope

- Remote synchronization (push/pull to hosted repositories)
- Multi-process or multi-replica concurrency support
- Deployment configurations (Helm chart, Kubernetes manifests)
- Credential management for remote repositories
- Container filesystem / persistent volume provisioning strategies
- Performance benchmarking or optimization beyond baseline correctness
- Web UI, REST API, or CLI commands specific to the graph store
- Migration tooling from other backends to the graph store
