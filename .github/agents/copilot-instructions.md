# design-notes Development Guidelines

Auto-generated from all feature plans. Last updated: 2025-12-15

## Active Technologies
- Go (version specified in `radius/go.mod`) + `github.com/brooke-hamilton/git-infra-graph/src/graph` (grif library), `github.com/radius-project/radius/pkg/components/database` (database.Client interface), `github.com/radius-project/radius/pkg/ucp/resources` (ID parsing), `github.com/radius-project/radius/pkg/ucp/util/etag` (ETag computation) (002-graph-state-provider)
- Git object database via grif library (blobs, trees, commits, refs) — no external database (002-graph-state-provider)

- YAML (GitHub Actions), Bash + Radius CLI (installed via official installer), `rad version`, `rad upgrade kubernetes`, `rad install kubernetes` (001-lrt-current-release)

## Project Structure

```text
src/
tests/
```

## Commands

# Add commands for YAML (GitHub Actions), Bash

## Code Style

YAML (GitHub Actions), Bash: Follow standard conventions

## Recent Changes
- 002-graph-state-provider: Added Go (version specified in `radius/go.mod`) + `github.com/brooke-hamilton/git-infra-graph/src/graph` (grif library), `github.com/radius-project/radius/pkg/components/database` (database.Client interface), `github.com/radius-project/radius/pkg/ucp/resources` (ID parsing), `github.com/radius-project/radius/pkg/ucp/util/etag` (ETag computation)

- 001-lrt-current-release: Added YAML (GitHub Actions), Bash + Radius CLI (installed via official installer), `rad version`, `rad upgrade kubernetes`, `rad install kubernetes`

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
