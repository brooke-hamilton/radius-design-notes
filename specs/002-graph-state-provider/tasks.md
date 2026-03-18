# Tasks: Git Graph State Provider

**Input**: Design documents from `/specs/002-graph-state-provider/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/go-api.md

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story?] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2)
- Exact file paths included in descriptions

**Note**: Within each phase, follow test-first (red-green-refactor) workflow. Write tests before implementation regardless of listed order.

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Add the grif dependency, create the `graphstore` package skeleton, and register the new provider type.

- [ ] T001 Add `github.com/brooke-hamilton/git-infra-graph` dependency to go.mod
- [ ] T002 Create package directory and doc.go at pkg/components/database/graphstore/doc.go
- [ ] T003 [P] Add `TypeGraphStore` constant to pkg/components/database/databaseprovider/types.go
- [ ] T004 [P] Add `GraphStoreOptions` struct and `GraphStore` field on `Options` to pkg/components/database/databaseprovider/options.go

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Implement the path-mapping layer and the `storedObject` envelope used by every CRUD method. These must be complete before any user story work.

**CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T005 Implement `idToTreePath` and `idToGrifPath` path-mapping functions in pkg/components/database/graphstore/pathmapper.go
- [ ] T006 Implement `scopeToGrifPath` scope-to-path mapping in pkg/components/database/graphstore/pathmapper.go
- [ ] T007 [P] Write unit tests for all path-mapping functions, including special characters and empty segments, in pkg/components/database/graphstore/pathmapper_test.go
- [ ] T008 Implement `storedObject` type and marshal/unmarshal helpers in pkg/components/database/graphstore/client.go
- [ ] T009 Implement `NewClient` constructor (graph init-if-not-exists, default graph name) in pkg/components/database/graphstore/client.go
- [ ] T010 Add compile-time interface check `var _ database.Client = (*Client)(nil)` in pkg/components/database/graphstore/client.go
- [ ] T011 Implement test helper `setupTestRepo` that creates a temp Git repository with an initial commit in pkg/components/database/graphstore/client_test.go

**Checkpoint**: Foundation ready — `NewClient` can create a graph store client against a live Git repository.

---

## Phase 3: User Story 1 — Store and Retrieve Resource State in Git (Priority: P1) MVP

**Goal**: Implement `Save`, `Get`, and `Delete` on the graph store client so resources can be persisted as Git blobs and retrieved by ID. Each mutation produces a Git commit. This is the core value proposition.

**Independent Test**: Save a resource, Get it back, Delete it, verify commit history reflects the mutations.

### Implementation for User Story 1

- [ ] T012 [US1] Implement `Get` method — parse ID, call grif `Get`, unmarshal `storedObject`, return `database.Object` — in pkg/components/database/graphstore/client.go
- [ ] T013 [US1] Implement `Save` method — compute ETag, build `storedObject`, call grif `Put` + `Commit`, handle rollback on commit failure — in pkg/components/database/graphstore/client.go
- [ ] T014 [US1] Implement `Delete` method — core logic (not-found handling, stage + commit); ETag enforcement added in T028 — in pkg/components/database/graphstore/client.go
- [ ] T015 [US1] Add staging-ref rollback logic: capture pre-operation staging ref hash in Save/Delete, restore on commit error in pkg/components/database/graphstore/client.go
- [ ] T016 [US1] Write unit tests for `Get` (success, not-found, invalid ID) in pkg/components/database/graphstore/client_test.go
- [ ] T017 [US1] Write unit tests for `Save` (create new, update existing, verify ETag returned) in pkg/components/database/graphstore/client_test.go
- [ ] T018 [US1] Write unit tests for `Delete` (success, not-found) in pkg/components/database/graphstore/client_test.go
- [ ] T019 [US1] Write integration test verifying Git commit is created per Save/Delete (inspect graph ref log) in pkg/components/database/graphstore/client_test.go

**Checkpoint**: Save/Get/Delete work end-to-end. Each mutation creates a Git commit. User Story 1 is independently testable.

---

## Phase 4: User Story 2 — Query Resources by Scope and Type (Priority: P2)

**Goal**: Implement `Query` with recursive tree walking, scope filtering, resource type filtering, property filters, and pagination so the control plane can list resources.

**Independent Test**: Save multiple resources across scopes and types, query by scope and type, verify correct results with pagination.

### Implementation for User Story 2

- [ ] T020 [US2] Implement recursive tree-walk helper (`collectBlobsUnderPath`) that descends grif tree nodes to collect all leaf blobs in pkg/components/database/graphstore/client.go
- [ ] T021 [US2] Implement `Query` method — validate query, walk scope subtree, deserialize blobs, filter by `databaseutil.IDMatchesQuery` and `MatchesFilters` — in pkg/components/database/graphstore/client.go
- [ ] T022 [US2] Implement pagination logic — index-based continuation tokens with base64 encoding — in pkg/components/database/graphstore/client.go
- [ ] T023 [US2] Handle scope queries (`IsScopeQuery`) using `databaseutil.ConvertScopeTypeToResourceType` in Query method in pkg/components/database/graphstore/client.go
- [ ] T024 [US2] Write unit tests for `Query` — scope filtering, type filtering, property filtering — in pkg/components/database/graphstore/client_test.go
- [ ] T025 [US2] Write unit tests for pagination — page size limits, continuation tokens, exhaustion — in pkg/components/database/graphstore/client_test.go
- [ ] T026 [US2] Write unit tests for scope queries (`IsScopeQuery`, `ScopeRecursive`) in pkg/components/database/graphstore/client_test.go

**Checkpoint**: Query returns filtered, paginated results. User Story 2 is independently testable.

---

## Phase 5: User Story 3 — Optimistic Concurrency Control (Priority: P2)

**Goal**: Enforce ETag-based OCC on `Save` and `Delete` so concurrent writers cannot silently overwrite each other.

**Independent Test**: Save a resource, get its ETag, save again with correct ETag (succeeds), then try with stale ETag (fails with `ErrConcurrency`).

### Implementation for User Story 3

- [ ] T027 [US3] Add ETag comparison logic to `Save` — read existing blob, compare stored ETag with provided ETag, return `ErrConcurrency` on mismatch, return `ErrConcurrency` (not `ErrNotFound`) when resource deleted but ETag was provided — in pkg/components/database/graphstore/client.go
- [ ] T028 [US3] Add ETag comparison logic to `Delete` — read existing blob, compare stored ETag with provided ETag, return `ErrConcurrency` on mismatch or if resource deleted — in pkg/components/database/graphstore/client.go
- [ ] T029 [US3] Write unit tests for `Save` with ETag — correct ETag succeeds, stale ETag returns `ErrConcurrency`, deleted resource with ETag returns `ErrConcurrency` — in pkg/components/database/graphstore/client_test.go
- [ ] T030 [US3] Write unit tests for `Delete` with ETag — correct ETag succeeds, stale ETag returns `ErrConcurrency`, deleted resource with ETag returns `ErrConcurrency` — in pkg/components/database/graphstore/client_test.go

**Checkpoint**: OCC is enforced. User Story 3 is independently testable.

---

## Phase 6: User Story 4 — Provider Registration and Configuration (Priority: P3)

**Goal**: Register the graph store in the Radius provider/factory pattern so operators can select it via YAML configuration. Include remote clone on startup and Git credential support.

**Independent Test**: Configure the graph store in a YAML config, initialize the provider, verify the graph store client is returned.

### Implementation for User Story 4

- [ ] T031 [US4] Implement `initGraphStoreClient` factory function in pkg/components/database/databaseprovider/factory.go — handle default graph name, validate repo path, call `graphstore.NewClient`
- [ ] T032 [US4] Add `TypeGraphStore: initGraphStoreClient` entry to `databaseClientFactory` map in pkg/components/database/databaseprovider/factory.go
- [ ] T033 [US4] Implement Git clone logic in factory function — if `RemoteURL` is set and `RepoPath` does not exist, clone using go-git with optional credential support (`GIT_TOKEN`, `GIT_SSH_KEY` env vars) in pkg/components/database/databaseprovider/factory.go
- [ ] T034 [US4] Write unit test for factory — valid config initializes client, missing repo path errors, invalid graph name errors — in pkg/components/database/databaseprovider/factory_test.go
- [ ] T035 [P] [US4] Add example graph store configuration entries to build/configs/ucp-dev.yaml and build/configs/applications-rp-dev.yaml (commented out)
- [ ] T036 [US4] Implement CLI remote URL detection — when `provider: graphstore`, read the Git remote/origin URL from the local repository at `repoPath` using go-git and expose it for propagation to the control plane — in pkg/cli/ (location TBD based on CLI config flow)
- [ ] T037 [US4] Write unit test for CLI remote URL detection — valid repo returns URL, missing remote returns empty, invalid repo path errors — in pkg/cli/ (location TBD)

**Checkpoint**: Provider is selectable via YAML config. Control plane can clone from remote URL on startup.

---

## Phase 7: User Story 5 — Full Audit Trail via Git History (Priority: P3)

**Goal**: Verify that the commit-per-mutation behavior from US1 creates meaningful commit messages and a browsable audit trail. This story validates a byproduct of the core storage mechanism.

**Independent Test**: Perform save and delete operations, inspect git log for the graph ref, verify each mutation is a distinct commit with a descriptive message.

### Implementation for User Story 5

- [ ] T038 [US5] Enhance commit messages in `Save` and `Delete` methods to include the resource path and operation type (e.g., "Save: planes/radius/local/...") in pkg/components/database/graphstore/client.go
- [ ] T039 [US5] Write integration test verifying commit messages reference affected resource paths and operations in pkg/components/database/graphstore/client_test.go

**Checkpoint**: Audit trail is human-readable via standard Git tooling.

---

## Phase 8: Conformance Tests & Validation

**Purpose**: Integrate with the shared conformance test suite and validate success criteria that require dedicated tests.

- [ ] T040 Wire up shared conformance tests from test/ucp/storetest by calling `storetest.RunTest(t, client)` in pkg/components/database/graphstore/client_test.go
- [ ] T041 Fix any conformance test failures identified by the shared test suite in pkg/components/database/graphstore/client.go
- [ ] T042 [P] Write integration test for SC-003 (clone recovery): save resources, clone the repo to a new temp directory, create a new client against the clone, verify all resources are retrievable in pkg/components/database/graphstore/client_test.go
- [ ] T043 [P] Write benchmark test for SC-005 (query performance): insert 100 resources across 10 scopes, measure query latency, assert p95 < 1 second in pkg/components/database/graphstore/client_test.go
- [ ] T044 [P] Write concurrency test for SC-006: launch two goroutines saving the same resource concurrently, assert exactly one succeeds and one gets `ErrConcurrency` in pkg/components/database/graphstore/client_test.go
- [ ] T045 Run full test suite (`go test ./pkg/components/database/graphstore/ -v -bench=.`) and verify all tests pass

**Checkpoint**: All conformance tests pass — behavioral parity with existing backends is proven (SC-001).

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: Documentation, config examples, and cleanup.

- [ ] T046 [P] Add godoc comments to all exported types and functions in pkg/components/database/graphstore/
- [ ] T047 [P] Validate quickstart.md instructions work end-to-end against a real Git repository (also validates SC-007: no external DB required)
- [ ] T048 Run `make lint` and fix any linting issues in new code
- [ ] T049 Run `make format-check` and fix any formatting issues in new code

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 (go.mod, package skeleton)
- **US1 (Phase 3)**: Depends on Phase 2 (path mapper, storedObject, NewClient)
- **US2 (Phase 4)**: Depends on Phase 2 + US1's `Save` (need data to query)
- **US3 (Phase 5)**: Depends on Phase 2 + US1's `Save`/`Delete` (need operations to add ETag logic to)
- **US4 (Phase 6)**: Depends on Phase 2 + US1's `NewClient` (need constructor to call from factory)
- **US5 (Phase 7)**: Depends on US1 (need commit-producing operations)
- **Conformance (Phase 8)**: Depends on US1 + US2 + US3 (need all CRUD + Query + OCC)
- **Polish (Phase 9)**: Depends on all user stories being complete

### User Story Dependencies

- **US1 (P1)**: Depends only on Foundational — no cross-story dependencies
- **US2 (P2)**: Depends on US1 `Save` (need stored data to query against)
- **US3 (P2)**: Depends on US1 `Save`/`Delete` (adds ETag checks to existing methods)
- **US4 (P3)**: Depends on US1 `NewClient` (factory calls constructor)
- **US5 (P3)**: Depends on US1 commit behavior (validates audit trail)

### Parallel Opportunities

- T003 and T004 can run in parallel (different files in databaseprovider/)
- T005/T006 and T007 can overlap (write mapper then immediately test)
- US2 and US3 can run in parallel after US1 completes (different concerns, mostly different code sections)
- T035 can run in parallel with other US4 tasks (config files are independent)
- T042, T043, T044 can run in parallel with each other (independent test scenarios)
- T046 and T047 can run in parallel with each other (docs vs. testing)

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001–T004)
2. Complete Phase 2: Foundational (T005–T011)
3. Complete Phase 3: US1 — Save/Get/Delete (T012–T019)
4. **STOP and VALIDATE**: Run `go test ./pkg/components/database/graphstore/ -v`
5. Resources are persistable in Git — core value delivered

### Incremental Delivery

1. Setup + Foundational → Package skeleton ready
2. US1 → Save/Get/Delete work → MVP!
3. US2 → Query works → Control plane can list resources
4. US3 → OCC enforced → Concurrent correctness
5. US4 → Provider registered + CLI remote URL detection → YAML-configurable
6. US5 → Audit trail validated → Differentiated feature proven
7. Conformance → Parity proven, SC-003/SC-005/SC-006 verified → Production-ready

### Task Counts

| Phase | Tasks | Parallel |
|-------|-------|----------|
| Setup | 4 | 2 |
| Foundational | 7 | 1 |
| US1 (P1) | 8 | 0 |
| US2 (P2) | 7 | 0 |
| US3 (P2) | 4 | 0 |
| US4 (P3) | 7 | 1 |
| US5 (P3) | 2 | 0 |
| Conformance | 6 | 3 |
| Polish | 4 | 2 |
| **Total** | **49** | **9** |
