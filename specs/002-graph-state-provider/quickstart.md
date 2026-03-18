# Quickstart: Git Graph State Provider

**Feature**: 002-graph-state-provider
**Date**: 2026-03-18

## Prerequisites

- Go toolchain (version from `go.mod`)
- A Git repository (any existing repo, or create a new one)
- Radius source code cloned

## 1. Configure Radius to Use the Graph Store

Create or edit the development config file (e.g., `cmd/ucpd/ucp-dev.yaml`):

```yaml
databaseProvider:
  provider: "graphstore"
  graphstore:
    repoPath: "/path/to/your/git/repo"
    graphName: "radius"
```

Do the same for `cmd/applications-rp/applications-rp-dev.yaml`:

```yaml
databaseProvider:
  provider: "graphstore"
  graphstore:
    repoPath: "/path/to/your/git/repo"
    graphName: "radius"
```

## 2. Run Radius Locally

Start the Radius components with the graph store backend:

```bash
make debug-start
```

## 3. Verify State Storage

After deploying a resource through the CLI (e.g., `rad deploy`), inspect the Git repository:

```bash
cd /path/to/your/git/repo

# View the graph ref
git log refs/infra/radius --oneline

# View the stored resources
git ls-tree -r refs/infra/radius
```

Each resource appears as a blob at a path matching its resource ID structure.

## 4. Run Conformance Tests

Run the shared conformance tests against the graph store:

```bash
go test ./pkg/components/database/graphstore/ -v -run TestGraphStoreClient
```

## 5. Inspect the Audit Trail

Every Save and Delete operation produces a Git commit:

```bash
git log refs/infra/radius --format="%h %s" | head -20
```

## Smoke Test Sequence

```bash
# 1. Initialize a test Git repository
mkdir /tmp/radius-test-repo && cd /tmp/radius-test-repo && git init

# 2. Create an initial commit (required by grif)
git commit --allow-empty -m "init"

# 3. Configure Radius to use this repo (edit YAML configs as above)

# 4. Run unit tests
go test ./pkg/components/database/graphstore/ -v

# 5. Run conformance tests
go test ./pkg/components/database/graphstore/ -v -run TestConformance
```
