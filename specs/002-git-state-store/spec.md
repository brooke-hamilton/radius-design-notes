# Feature Specification: Git-Based State Store

**Feature Branch**: `002-git-state-store`
**Created**: February 7, 2026
**Status**: Draft
**Input**: User description: "Build a Git-based state store backend for Radius that implements the database.Client interface using Git's internal object model (blobs, trees, commits, refs)"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Store and Retrieve Application State (Priority: P1)

As a platform engineer, I want Radius to store my application state in a Git repository so that I have a familiar, version-controlled system managing my infrastructure state.

**Why this priority**: Core functionality—without the ability to store and retrieve resources, no other features can work. This enables Radius to function as a complete platform with Git as the backing store.

**Independent Test**: Can be fully tested by deploying a simple application and verifying the resources are persisted and can be retrieved across Radius restarts.

**Acceptance Scenarios**:

1. **Given** a Git repository is configured as the state store, **When** I deploy an application using Radius, **Then** the application resources are persisted in the Git repository
2. **Given** resources exist in the Git state store, **When** I query for those resources via Radius, **Then** I receive the correct resource data
3. **Given** a resource exists in the Git state store, **When** I delete that resource, **Then** it is removed from the current state and the deletion is recorded in history
4. **Given** multiple resources are deployed in a single operation, **When** the deployment completes, **Then** all resource changes are captured in a single atomic state update

---

### User Story 2 - View State History and Audit Trail (Priority: P2)

As a platform engineer, I want to view the complete history of all state changes so that I can audit who changed what and when, troubleshoot issues, and maintain compliance records.

**Why this priority**: Provides the key differentiating value of using Git as a state store—built-in version history without additional tooling.

**Independent Test**: Can be fully tested by making a series of changes and verifying the complete history is accessible with timestamps and change details.

**Acceptance Scenarios**:

1. **Given** multiple deployments have occurred, **When** I view the state history, **Then** I see a chronological list of all state changes with timestamps
2. **Given** a state change was recorded, **When** I inspect that change, **Then** I can see what resources were added, modified, or deleted
3. **Given** state changes are recorded, **When** I compare two points in history, **Then** I can see the differences between those states

---

### User Story 3 - Rollback to Previous State (Priority: P2)

As a platform engineer, I want to rollback my application state to a previous point in time so that I can recover from failed deployments or configuration errors.

**Why this priority**: A key benefit of version-controlled state—the ability to undo mistakes and recover quickly.

**Independent Test**: Can be fully tested by making changes, then rolling back to a previous state and verifying resources match the earlier configuration.

**Acceptance Scenarios**:

1. **Given** I have made changes that caused problems, **When** I rollback to a previous state, **Then** the system restores the resource configuration from that point in time
2. **Given** I select a historical state to restore, **When** the rollback completes, **Then** the restoration is recorded as a new state change (not erasing subsequent history)
3. **Given** I am viewing state history, **When** I select a point to rollback to, **Then** I can preview what changes will be made before confirming

---

### User Story 4 - Link State Changes to Source Code (Priority: P3)

As a developer, I want each state change to be linked to the source code commit that produced it so that I can trace exactly what code was deployed for any given state.

**Why this priority**: Enables powerful debugging and compliance capabilities but requires the core storage functionality to be working first.

**Independent Test**: Can be fully tested by deploying from a source repository and verifying the state change includes the source commit reference.

**Acceptance Scenarios**:

1. **Given** I deploy from a source repository, **When** the deployment completes, **Then** the state change includes a reference to my source commit
2. **Given** I am investigating a previous state, **When** I view that state's details, **Then** I can see which source code commit produced that state
3. **Given** a state change has a source reference, **When** I click that reference, **Then** I am directed to the source code at that commit

---

### User Story 5 - Git Remote Operations (Priority: P3)

As a platform engineer, I want to synchronize Radius state with a remote Git server so that I can backup state, share it across environments, and use familiar Git tools (push, pull) to manage state distribution.

**Why this priority**: Leverages the Git-native format to enable collaboration and backup via remote repositories.

**Independent Test**: Can be fully tested by pushing state changes to a remote repository and pulling them from another Radius instance.

**Acceptance Scenarios**:

1. **Given** a remote Git repository is configured, **When** I sync state, **Then** changes are pushed to or pulled from the remote repository
2. **Given** I have state in a local Git repository, **When** I connect a new Radius instance to that repository, **Then** the instance can read and use the existing state
3. **Given** multiple Radius instances share a remote repository, **When** one instance makes changes, **Then** other instances can pull those changes

---

### Edge Cases

- What happens when two Radius instances attempt to update the same resource simultaneously? System should detect the conflict and reject the second update with a clear error message.
- How does the system handle a corrupted Git repository? System should fail gracefully with actionable error messages and preserve the last-known-good state.
- What happens when storage space is exhausted? System should fail writes gracefully and alert the operator before state is lost.
- How does the system handle very large resources (e.g., resources with large output data)? System should gracefully handle resources up to a defined size limit and provide clear errors for oversized resources.
- What happens when the source commit cannot be determined during deployment? System should record "unknown" for the source reference and log a warning.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST store each Radius resource as a distinct, addressable unit within the Git repository
- **FR-002**: System MUST support creating, reading, updating, and deleting resources
- **FR-003**: System MUST organize resources hierarchically by scope (planes, resource groups, providers)
- **FR-004**: System MUST provide query capabilities to list resources by scope and/or type
- **FR-005**: System MUST record every state change as a versioned entry with timestamp
- **FR-006**: System MUST maintain optimistic concurrency control to prevent conflicting updates
- **FR-007**: System MUST ensure atomic updates—all resources changed in a single operation succeed or fail together
- **FR-008**: System MUST capture source code commit reference for each state change when available
- **FR-009**: System MUST support viewing history of all state changes
- **FR-010**: System MUST support rolling back to any previous state
- **FR-011**: System MUST support querying resources within a specific historical state
- **FR-012**: System MUST support synchronizing state with a remote repository
- **FR-013**: System MUST provide meaningful error messages when operations fail
- **FR-014**: System MUST support configurable repository location as a local filesystem path
- **FR-015**: System SHOULD support optional synchronization with a remote repository for users who enable this feature
- **FR-016**: System MUST be selectable as an opt-in configuration option (not changing the default)

### Key Entities

- **Resource**: A Radius-managed entity (application, container, connection, etc.) with identity, type, scope, and data. Represents a single unit of state.
- **State Snapshot**: A point-in-time capture of all resources, with metadata including timestamp, source commit reference, and operation description.
- **Scope**: A hierarchical namespace for resources (plane > resource group > provider > resource type > resource). Used for organization and access control.
- **Version Label**: A marker pointing to a specific state snapshot, used for rollback and reference.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can deploy, update, and delete resources with the Git state store just as they can with the default state store
- **SC-002**: Users can view the complete history of state changes and identify when specific changes occurred
- **SC-003**: Users can rollback to any previous state within 2 user actions (select state + confirm)
- **SC-004**: 95% of resource operations complete within 2 seconds for repositories with up to 1,000 resources
- **SC-005**: State changes from deployments include source commit references when deploying from a Git-managed source directory
- **SC-006**: Users can configure the Git state store in under 5 minutes using documentation
- **SC-007**: System handles concurrent operations by the same user without data loss or corruption
- **SC-008**: System maintains at least 90 days of state history by default

## Assumptions

- All Radius resource data can be serialized to a text-based format suitable for Git storage
- The target environment has Git installed and accessible
- Users have familiarity with basic Git concepts (commits, history, branches)
- Standard web/mobile application performance expectations apply (operations complete in seconds, not minutes)
- Error handling follows user-friendly patterns with actionable messages
- Initial release focuses on single-instance usage; multi-instance coordination is a future enhancement
