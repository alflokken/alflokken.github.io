---
title: "Microsoft Graph Delta Query in PowerShell"
date: 2026-02-26T17:10:40+02:00
description: Learn how to use Microsoft Graph delta queries with PowerShell and PSGraphToolbox to perform incremental sync of Entra ID users, groups, and devices.
categories: [PowerShell, Microsoft Graph]
tags:
  - PowerShell
  - Microsoft Graph
  - Entra ID
  - Automation
---

This article explains how to use `Sync-GtGraphResourceDelta` from [PSGraphToolbox](https://github.com/alflokken/PSGraphToolbox) to build stateful synchronization on top of Microsoft Graph delta queries.

On the first run, the command performs a full synchronization and stores the returned delta token along with the current resource state. On subsequent runs, it uses the stored delta link to retrieve only new, updated, or deleted objects.

The module manages paging, state persistence, merging, relationship annotations, and token expiration handling automatically. If a delta token expires, it performs a full resynchronization.

This approach enables reliable and scalable incremental synchronization without manually handling delta links, skip tokens, or replay scenarios.

## How Delta Queries Work

The process runs in two phases. The first run returns the full dataset. Later runs return only the changes.

1. **Initial Request (Full Sync)**
   - Returns a full snapshot of the resource collection
   - The response includes an `@odata.deltaLink` token
   - You store this token for subsequent requests

2. **Subsequent Requests (Delta Sync)**
   - Send requests using the stored `@odata.deltaLink`
   - Returns only changes since the previous call
   - Includes additions, updates, and deletions

## Running Delta Queries with PSGraphToolbox

If you haven't already, install PSGraphToolbox and connect to Microsoft Graph:

```powershell
Install-Module PsGraphToolbox -Scope CurrentUser
Connect-MgGraph -Scopes "User.Read.All", "Group.Read.All"
```

> PSGraphToolbox is maintained by me. You can find it on [GitHub](https://github.com/alflokken/PSGraphToolbox).
{: .prompt-info }

### Initial Sync

The first time you run a delta query, the function performs a full sync:

```powershell
$syncParams = @{
    ResourcePath     = "users/delta"
    SelectProperties = "id,userPrincipalName,accountEnabled,displayName"
    StorageProvider  = "JsonFile"
    StoragePath      = ".\userDeltaState.json"
}

$result = Sync-GtGraphResourceDelta @syncParams
VERBOSE: 2026-01-29 08:09:27: No previous state found: performing full sync
VERBOSE: 2026-01-29 08:09:27: Performing initial (full) sync - users/delta
VERBOSE: 2026-01-29 08:10:36: Initial sync complete: 7976 items retrieved
VERBOSE: 2026-01-29 08:10:38: State written to .\userDeltaState.json
```

The state file contains the full dataset plus the delta token for future queries.

The function uses a pluggable storage provider. Currently, only `JsonFile` is supported. For custom storage providers, see the [storage provider implementation](https://github.com/alflokken/PSGraphToolbox/blob/main/src/core/helpers/storageProviders/storageProviders.ps1).

### Incremental Sync

Run the same command again and you receive only changes:

```powershell
$result = Sync-GtGraphResourceDelta @syncParams
VERBOSE: 2026-01-29 10:04:21: Performing incremental delta query (last sync: 2026-01-29T08:10:22Z)
VERBOSE: 2026-01-29 10:04:22: Processing 168 changed entities (current entity count: 7976)
VERBOSE: 2026-01-29 10:04:23: Incremental sync complete: UPDATE: 165, ADD: 3
VERBOSE: 2026-01-29 10:04:23: State written to .\userDeltaState.json
```

The function reads the stored delta token, queries for changes, and updates the state file. `CurrentState` contains the full, up-to-date dataset. Only the changed objects were retrieved from Graph.

### Other Resources

The same pattern works for any Microsoft Graph resource that supports delta queries. See [Microsoft's documentation](https://learn.microsoft.com/en-us/graph/delta-query-overview#supported-resources) for the full list.

```powershell
Sync-GtGraphResourceDelta -ResourcePath "devices/delta" -StoragePath ".\devicesDeltaState.json"
Sync-GtGraphResourceDelta -ResourcePath "groups/delta" -StoragePath ".\groupsDeltaState.json"
```

## Understanding the Output

`Sync-GtGraphResourceDelta` returns a result object and persists the current state to your configured storage for the next sync.

```text
$syncResult
├── ChangeLog[]             Array of changes since last sync
│   ├── entityId            Entity GUID
│   ├── entityName          Best-effort name (UPN or displayName)
│   ├── change              ADD | UPDATE | REMOVE
│   └── description         Human-readable summary
├── CurrentState
│   ├── value[]             All entities, merged and up-to-date
│   ├── deltaLink           (internal) Next sync URL with token
│   ├── dateRetrieved       (internal) Last sync timestamp
│   └── queryId             (internal) Parameter fingerprint
└── ItemCount               Total entity count
```

Properties marked `(internal)` are managed automatically. The `queryId` detects parameter changes: if you modify `SelectProperties` or `Filter`, the function triggers a full resync to prevent inconsistent state.

### Quick Usage

```powershell
# Full current dataset
$result.CurrentState.value

# Change log since last sync
$result.ChangeLog

# Discover available properties
$result.CurrentState.value | Get-Member -MemberType NoteProperty

# Select explicitly for output such as export, gridview, etc.
$result.CurrentState.value | Select-Object id, userPrincipalName, displayName
```

> Delta responses are sparse: Graph might only return properties with values. When working with collections, use `Select-Object` with explicit properties rather than `Select-Object *`, which formats columns based on the first object.
{: .prompt-warning }

## Handling Delta Queries for Large Group Memberships

> **Known Limitation:** Groups with large membership lists can return incomplete results during the initial full sync. To ensure a reliable starting state, use the hybrid approach described below.
{: .prompt-info }

The workaround is to seed the state manually using a standard GET for the initial snapshot, then let delta queries track changes from that point forward.

```powershell
$groupId = "d21ee78c-5bb4-499e-881c-55f77898ade8"

$queryParameters = @{
    SelectProperties = "id,displayName,members"
    Filter           = "id eq '$groupId'"
    ResourcePath     = "beta/groups/delta"
    StoragePath      = ".\groupMembersDeltaState.json"
}

# Step 1: Run delta sync to get a deltaLink, but skip writing state
$groupMembersDelta = Sync-GtGraphResourceDelta @queryParameters -NoWrite

# Step 2: Fetch the current group and members via standard GET
$groupObject = Invoke-GtGraphRequest -resource "groups/$groupId" -select "displayName"
$groupMembers = Invoke-GtGraphRequest -resource "groups/$groupId/members" -select "id"

# Step 3: Build the correct initial state and persist it
$state = [PSCustomObject]@{
    id              = $groupId
    displayName     = $groupObject.displayName
    "members@delta" = $groupMembers
}

$groupMembersDelta.CurrentState.value = $state
Set-GtDeltaStateToJsonFile -path $queryParameters.StoragePath -state $groupMembersDelta.CurrentState
```

After initialization, run delta sync normally:

```powershell
$groupMembersDelta = Sync-GtGraphResourceDelta @queryParameters
```

## Considerations and Limitations

- **Delta tokens expire after 30 days of inactivity.** The function handles expiration by performing a full resync, but running syncs more frequently avoids the overhead.
- **Not all resources support delta queries.** Check [Microsoft's documentation](https://learn.microsoft.com/en-us/graph/delta-query-overview) for supported resources.
- **`$select` affects what changes are tracked.** Graph only tracks changes to the properties you select. Changes to non-selected properties do not appear in delta results.
- **`$filter` has limited support in delta queries.** It only supports scoping by object ID.

## Resources

PSGraphToolbox also includes `Invoke-GtGraphBatchRequest` for bulk API operations. See the repo for details.

- [PSGraphToolbox on GitHub](https://github.com/alflokken/PSGraphToolbox)
- [Microsoft Graph Delta Query Documentation](https://learn.microsoft.com/en-us/graph/delta-query-overview)
