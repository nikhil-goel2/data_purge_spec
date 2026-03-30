# Existing Data Purge Implementation (Current State Only)

## DESIGN EVOLUTION NOTE
This document captures the baseline/current implementation state. The architecture is being enhanced to support normalized parent/child tables for purge history:
- **Previous Design:** JSONB `details_payload` embedded in parent purge history row (discussed in sections below)
- **New Design:** Normalized `blocklist_purge_history` (parent, summary-only) + `blocklist_purge_history_details` (child, detail rows) - see separate architecture documents
- **Rationale:** Separates summary queries from high-volume detail queries; enables paged detail drilldown without deserializing JSONB; maintains counts even after detail rows are purged

Read this document for **baseline context** on the existing rule-based purge flow; consult `data purge history table architecture.md` and `data purge history detail table architecture.md` for the enhanced design.

---

## Scope
This document captures only what is currently implemented in the codebase and database scripts. It does not include proposed design changes, new requirements, or future implementation plans.

**Domain definition:** In this system, "data purge" refers to the rule-driven blocklist processing flow that is triggered whenever a new blocklist rule is added. When the rule is submitted with `ScheduleDataPurge = true`, a schedule is created and a state machine works through evaluating and updating consumer data against that new rule. A separate background step (`BlocklistItemClean`) handles periodic retention-based deletion of old processed records and is distinct from the main purge flow.

---

## 1. Trigger: New Blocklist Rule Added

The data purge process is triggered from configuration-service when a new blocklist rule is added.

- File: `configuration-service/src/CoxAuto.Thunderbird.ConfigurationService.Core/Managers/BlocklistManager.cs`
- Method: `AddBlocklistItemAsync(AddBlockListRuleRequest addBlockListRuleRequest)`

Trigger conditions:
1. The inbound request has `ScheduleDataPurge = true`.
2. There is no existing `Scheduled`-status schedule for the same `coorg_id`.

If both conditions are met:
- `BlocklistSchedulePublisher.PublishAsync()` is called with a `ConfigurationBlocklistPurgeEventsMessages` event.
- Event fields: `CoOrgDealerId`, `RequestedBy`, `ScheduledUtc`.
- Event metadata: `Type = "schedule-purge"`, `Source = "configuration-service"`.
- The event is published to the AWS EventBridge message bus.

Deduplication guard in existing code:
```csharp
var scheduledBlocklist = _workflowResourceClient.BlocklistScheduleCoorgIdAsync(
    new GetBlocklistScheduleRequest {
        CoOrgDealerId = addBlockListRuleRequest.CommonOrgId,
        BlocklistScheduleStatus = BlocklistScheduleStatus.Scheduled });

if ((scheduledBlocklist.Result?.Count ?? 0) == 0)
{
    await _blocklistSchedulePublisher.PublishAsync(new ConfigurationBlocklistPurgeEventsMessages { ... });
}
```

---

## 2. Schedule Creation in `workflow.blocklist_schedules`

After the event is published, a consumer (handled by import-service or workflow-resource) creates a row in `workflow.blocklist_schedules` with status `Scheduled`.

Table: `workflow.blocklist_schedules`

| Column | Type | Notes |
|---|---|---|
| `blocklist_schedule_id` | uuid | PK |
| `coorg_id` | varchar(64) | Dealer/org identifier |
| `schedule_status` | varchar(32) | Enum string — see §6.1 |
| `requested_by` | varchar(32) | User who triggered rule add |
| `scheduled_utc` | timestamp | Time of request |
| `created_utc` | timestamp | Row creation time |
| `updated_utc` | timestamp | Last update time |
| `lease_expires_utc` | timestamp | Used by state machine for distributed locking |

Repository: `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/Repositories/BlocklistScheduleRepository.cs`

- `CreateAsync(blocklist_schedule)` — inserts a new schedule row.
- `UpdateScheduleStatusAsync(id, status)` — updates `schedule_status` and resets `lease_expires_utc`.

---

## 3. State Machine Processing (workflow-resource)

The `BlocklistStateMachine` is a recurring background job in workflow-resource that runs a fixed sequence of steps on every tick.

- File: `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/BlocklistStateMachines/BlocklistStateMachine.cs`
- Base class: `RecurringJobService` (runs continuously with `DelaySeconds` between iterations)
- Steps are registered via dependency injection and executed in order.

### 3.1 Step: `BlocklistScheduleRequested`

- File: `...BlocklistStateMachines/Steps/BlocklistScheduleRequested.cs`
- Calls: `workflow.lease_blocklist_scheduled(...)` — leases one schedule in `Scheduled` status.
- If a lease is obtained:
  - Publishes `WorkflowStateEventMessage`: `Scheduled → Requested`.
  - Updates schedule status to `Requested` via `BlocklistScheduleResourceAccess.UpdateBlocklistScheduleStatusAsync`.

### 3.2 Step: `BlocklistItemReceived`

- File: `...BlocklistStateMachines/Steps/BlocklistItemReceived.cs`
- Calls: `workflow.lease_blocklist_item_results_received(...)` — leases batch of item result rows in `Saved` status.
- For each leased row:
  - Publishes `WorkflowStateEventMessage`: `Saved → Received`.
  - Updates item status to `Received` via `BlocklistItemResultsResourceAccess.UpdateBlocklistItemStatusAsync`.

### 3.3 Step: `BlocklistBusinessEntitiesReceived`

- File: `...BlocklistStateMachines/Steps/BlocklistBusinessEntitiesReceived.cs`
- Calls: `workflow.lease_blocklist_business_entities_results_received(...)` — leases batch of business result rows in `Saved` status.
- For each leased row:
  - Publishes `WorkflowStateEventMessage`: `Saved → Received`.
  - Updates business entity status to `Received`.

### 3.4 Step: `BlocklistScheduleCompleted`

- File: `...BlocklistStateMachines/Steps/BlocklistScheduleCompleted.cs`
- Calls: `workflow.lease_blocklist_schedule_completed(...)` — leases schedules where all items have been fully processed.
- For each leased schedule:
  - Updates schedule status to `Completed`.

### 3.5 Step: `BlocklistItemClean` (Background Retention Cleanup — Separate Concern)

- File: `...BlocklistStateMachines/Steps/BlocklistItemClean.cs`
- This step is **operational hygiene**, not part of the main rule-driven data purge flow.
- Calls: `stateMachineResourceAccess.PurgeBlocklistItemResultsAsync(deletionIntervalDays)`
- Which executes: `workflow.purge_blocklist_results_data_as_per_retention_days(deletion_interval_days)`
- Deletes rows from `workflow.blocklist_item_results` and `workflow.blocklist_business_results` where `updated_utc` is older than the configured interval (default 15 days).
- Batched: up to 1000 rows per table per call.
- Returns `void` — no deleted count surfaced.

---

## 4. Item Results Table

Consumer data items evaluated against the blocklist rule are stored in `workflow.blocklist_item_results`.

Table: `workflow.blocklist_item_results`

| Column | Type | Notes |
|---|---|---|
| `blocklist_item_result_id` | uuid | PK |
| `blocklist_schedule_id` | uuid | FK → `blocklist_schedules` |
| `common_consumer_id` | varchar(64) | Consumer being evaluated |
| `payload` | jsonb | Input data: `{"type":"phone\|email\|address\|name","value":"..."}` |
| `result_payload` | jsonb | Processing result data |
| `item_status` | varchar(32) | See §6.2 |
| `source_page_number` | int | Source page reference |
| `page_number` | int | Page reference |
| `created_utc` | timestamp | Row creation time |
| `updated_utc` | timestamp | Last update time (used for retention filter) |
| `lease_expires_utc` | timestamp | Distributed lock field |

Constraints:
- Unique index on `(blocklist_schedule_id, common_consumer_id)` — prevents duplicate evaluation per run.
- Bulk insert uses PostgreSQL binary copy via temp table.

---

## 5. Schedule-Level Purge (Explicit Cleanup Endpoint)

When a specific schedule's data needs to be removed, a dedicated delete endpoint exists:

- Controller: `workflow-resource/.../Controllers/BlocklistSchedulesController.cs`
- Route: `DELETE /blocklist-schedule/purge-data/id/{blocklistScheduleId}`
- Manager method: `BlocklistScheduleManager.PurgeDataByBlocklistScheduleIdAsync(blocklistScheduleId)`
- Repository method: `BlocklistScheduleRepository.PurgeDataAsync(blocklistScheduleId)`
- DB function: `workflow.purge_data_by_blocklist_schedule_id(p_blocklist_schedule_id uuid)`

DB function behavior (hard-delete):
```sql
DELETE FROM workflow.blocklist_item_results WHERE blocklist_schedule_id = p_blocklist_schedule_id;
DELETE FROM workflow.blocklist_business_results WHERE blocklist_schedule_id = p_blocklist_schedule_id;
DELETE FROM workflow.blocklist_schedules WHERE blocklist_schedule_id = p_blocklist_schedule_id;
```

This removes all item results, business results, and the schedule row itself for a given run.

---

## 6. Status Enumerations

### 6.1 `BlocklistScheduleStatus` (schedule lifecycle)
From `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Common/Models/BlocklistScheduleStatus.cs`:
- `Unknown`
- `Scheduled` — schedule created, waiting to be picked up
- `Requested` — state machine has picked it up and is processing
- `Uploaded` — consumer data has been loaded / items saved
- `Completed` — all items processed
- `Cancelled`
- `Paused`

### 6.2 `BlocklistItemStatus` (per-item lifecycle)
From `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Common/Models/BlocklistItemStatus.cs`:
- `Unknown`
- `Saved` — item written, awaiting processing
- `Received` — state machine has picked up the item
- `Notified` — downstream notification sent
- `Failed`
- `Processed` — item fully evaluated against the rule

---

## 7. Existing Behavior That Is Not Implemented Today

Strictly based on current implementation:
- No persisted purge-run history table (total records processed per run, breakdown by type)
- No API endpoint to query historical purge run results for UI display
- Per-run record counts are not captured or stored at any point in the current flow
- The `BlocklistItemClean` retention delete function returns `void` — no count is surfaced

---

## 8. Source File References

- `configuration-service/src/CoxAuto.Thunderbird.ConfigurationService.Core/Managers/BlocklistManager.cs`
- `configuration-service/src/CoxAuto.Thunderbird.ConfigurationService.Core/Publishers/BlocklistSchedulePublisher.cs`
- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/BlocklistStateMachines/BlocklistStateMachine.cs`
- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/BlocklistStateMachines/Steps/BlocklistScheduleRequested.cs`
- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/BlocklistStateMachines/Steps/BlocklistItemReceived.cs`
- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/BlocklistStateMachines/Steps/BlocklistBusinessEntitiesReceived.cs`
- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/BlocklistStateMachines/Steps/BlocklistScheduleCompleted.cs`
- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/BlocklistStateMachines/Steps/BlocklistItemClean.cs`
- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/Repositories/BlocklistScheduleRepository.cs`
- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Host/Controllers/BlocklistSchedulesController.cs`
- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/IBlocklistScheduleManager.cs`
- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Common/Models/BlocklistScheduleStatus.cs`
- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Common/Models/BlocklistItemStatus.cs`
- `common-db/postgres/Databases/store/Schemas/workflow/Functions/purge_data_by_blocklist_schedule_id.sql`
- `common-db/postgres/Databases/store/Schemas/workflow/Functions/purge_blocklist_results_data_as_per_retention_days.sql`
- `common-db/postgres/Databases/store/Schemas/workflow/Tables/14.BlocklistSchedules/changelog-1.xml`
- `common-db/postgres/Databases/store/Schemas/workflow/Tables/15.BlocklistItemResults/changelog-1.xml`
