# Make Commands for GitHub Workflows

* **Author**: Brooke Hamilton (@brooke-hamilton)

## Overview

This design proposes a collection of standardized make commands to streamline GitHub workflow management for developers working with repository forks in the Radius project.

## Terms and definitions

| Term | Definition |
|------|------------|
| **Make** | A build automation tool that uses Makefiles to define and execute sets of commands. In this context, Make will be used to create standardized commands for GitHub workflow operations. |
| **GitHub Workflows** | Automated processes defined using GitHub Actions that run in response to specific events in a repository, such as push, pull request, or scheduled triggers. |
| **GitHub Repository Forks** | Personal copies of a repository that allow developers to make changes without affecting the original repository, commonly used in open-source development for contributing changes via pull requests. |

## Objectives

The objective is to define standardized make commands to automate common GitHub workflow operations developers have to perform on forks of the Radius project.

### Goals

The goal of the make commands is to improve developer productivity and reduce manual errors.

## Proposed Make Commands

### `make-workflow-dispatch`

Alias: `make-workflow`

Invokes a GitHub workflow by yaml file name. This command will be used to trigger workflows defined in the repository.

```bash
make-workflow NAME=<workflow file name> BRANCH=<(optional)branch name, defaults to current branch>
```

### `make-workflow-list`

Lists all workflows defined in the repository.

### `make-workflow-disable`

Invokes a GitHub workflow disable command. This command will be used to disable a workflow defined in the repository.

```bash
make-workflow-disable NAME=<workflow_name>
```

### `make-workflow-disable-triggered`

Disables all workflows in the repository that are triggered by events such as push or pull request, and all workflows invoked by timers.

### `make-workflow-enable`

Enables a workflow in the repository.

```bash
make-workflow-enable NAME=<workflow_name>
```

### `make-workflow-delete-all-runs`

Deletes all workflow runs in a GitHub repository.

## Design

A Makefile named `workflow` will be added to the `./build` folder.

The all `.PHONY` commands win the makefile will start with `worflow-*`, with `*` representing the name of the operation to run.

If logic is complex and difficult to implement within the Makefile, a separate shell script will be added to the `./build` folder and the Makefile will call the shell script.

## Security

Commands that modify workflow properties, like disabling a workflow, will be hard coded to prevent them from running on repos that belong to the `radius-project` GitHub organization.

<!--
## Design Review Notes
-->