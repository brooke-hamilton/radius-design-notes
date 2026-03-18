# Implementation Plan: Git Graph State Provider

**Branch**: `002-graph-state-provider` | **Date**: 2026-03-18 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/002-graph-state-provider/spec.md`

## Summary

Implement a new `database.Client` backend for Radius that persists resource state as a versioned graph inside a Git repository using the `git-infra-graph` (grif) Go library. The implementation translates Radius CRUD and query operations into grif graph operations (Put, Get, DeleteNode, Commit), maps resource IDs to Git tree paths via direct segment mapping with lowercasing, computes ETags on JSON-marshaled data, and registers as a new provider type selectable via YAML configuration. The CLI detects Git-backed configuration and passes the remote URL to the control plane, which clones the repository on startup.

## Technical Context

**Language/Version**: Go (version specified in `radius/go.mod`)
**Primary Dependencies**: `github.com/brooke-hamilton/git-infra-graph/src/graph` (grif library), `github.com/radius-project/radius/pkg/components/database` (database.Client interface), `github.com/radius-project/radius/pkg/ucp/resources` (ID parsing), `github.com/radius-project/radius/pkg/ucp/util/etag` (ETag computation)
**Storage**: Git object database via grif library (blobs, trees, commits, refs) — no external database
**Testing**: `go test` with shared conformance tests from `test/ucp/storetest/`, plus unit and integration tests using live Git repositories (no mocks for Git)
**Target Platform**: Linux (Kubernetes containers and local development)
**Project Type**: Single Go module addition to existing multi-package Radius codebase
**Performance Goals**: Sub-second scope queries for 100 resources across 10 scopes (SC-005)
**Constraints**: Single-process deployment; correctness over performance; no remote sync beyond initial clone
**Scale/Scope**: Initial target is local development and testing; production deployment deferred

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. API-First Design | PASS | Implements existing `database.Client` interface — API contract already defined |
| II. Idiomatic Code Standards | PASS | Go implementation follows Effective Go; `gofmt` formatting; godoc on exports |
| III. Multi-Cloud Neutrality | PASS | Storage backend is cloud-agnostic (Git repository on any filesystem) |
| IV. Testing Pyramid (NON-NEGOTIABLE) | PASS | Shared conformance tests + unit tests + integration tests with live Git repos |
| V. Collaboration-Centric Design | PASS | Enables zero-infrastructure state storage for platform engineers; Git audit trail benefits operations |
| VI. Open Source and Community-First | PASS | Design spec authored in design-notes repo before implementation |
| VII. Simplicity Over Cleverness | PASS | Thin adapter over grif library; no new abstractions beyond required interface compliance |
| VIII. Separation of Concerns | PASS | New package `pkg/components/database/graphstore/`; no coupling to other backends |
| IX. Incremental Adoption | PASS | Opt-in via YAML config; no breaking changes to existing backends |
| XVI. Repository-Specific Standards | PASS | Follows existing `databaseprovider` factory pattern exactly |
| XVII. Polyglot Coherence | PASS | Uses same error types, ETag patterns, and ID normalization as other backends |

**Gate result: PASS — no violations.**

## Project Structure

### Documentation (this feature)

```text
specs/002-graph-state-provider/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   └── go-api.md        # Go API contracts
└── tasks.md             # Phase 2 output (/speckit.tasks command)
```

### Source Code (radius repository)

```text
pkg/components/database/
├── graphstore/                  # NEW: graph store client package
│   ├── client.go                # database.Client implementation
│   ├── client_test.go           # Unit tests
│   ├── pathmapper.go            # Resource ID ↔ tree path mapping
│   ├── pathmapper_test.go       # Path mapping tests
│   └── doc.go                   # Package documentation
├── databaseprovider/
│   ├── types.go                 # ADD: TypeGraphStore constant
│   ├── options.go               # ADD: GraphStoreOptions struct
│   ├── factory.go               # ADD: initGraphStoreClient factory function
│   └── storageprovider.go       # (no changes needed)
└── databaseutil/
    └── id.go                    # (existing — used for ID normalization)

test/ucp/storetest/
└── shared.go                    # (existing — conformance tests invoked from graphstore tests)

build/configs/
└── *.yaml                       # ADD: example graph store config entries
```

**Structure Decision**: Follows existing pattern — each database backend gets its own sub-package under `pkg/components/database/`. The `graphstore` package mirrors the structure of `inmemory/`, `postgres/`, and `apiserverstore/`. Provider registration follows the established factory pattern in `databaseprovider/`.

## Complexity Tracking

> No constitution violations. No complexity justification needed.
