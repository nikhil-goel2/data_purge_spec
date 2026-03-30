# Data Purge Feature - Implementation Tasks for Backend Team

**Status:** Ready for Implementation  
**Target Delivery:** Multi-phase approach  
**Prerequisite Docs:**
- [data purge history table architecture.md](/c:/Code/_bmad-output/planning-artifacts/data purge history table architecture.md)
- [data purge history detail table architecture.md](/c:/Code/_bmad-output/planning-artifacts/data purge history detail table architecture.md)
- [data purge api endpoints.md](/c:/Code/_bmad-output/planning-artifacts/data purge api endpoints.md)

---

## Overview

This document breaks implementation into three phases aligned to the normalized architecture:
1. Keep the existing rule-triggered purge flow intact.
2. Add a parent summary table `workflow.blocklist_purge_history` and a child detail table `workflow.blocklist_purge_history_details`.
3. Expose lightweight history APIs plus paged detail drilldown APIs.

| Phase | Title | Scope | Duration Est. |
|---|---|---|---|
| Phase 1 | Schema and Data Model | New workflow tables, FK/delete behavior, indexes | 1-2 sprints |
| Phase 2 | Workflow-Resource Write Path | Parent/child persistence during schedule lifecycle | 2-3 sprints |
| Phase 3 | Read APIs, Testing, Deployment | Summary API, detail API, integration/perf hardening | 1-2 sprints |

---

## Phase 1: Schema and Data Model

### 1.1 Verify Existing Source Tables and Flow

**Task 1.1.1: Confirm current source-of-truth tables and columns**
- Verify `workflow.blocklist_schedules` columns:
  - `blocklist_schedule_id`, `coorg_id`, `schedule_status`, `requested_by`, `scheduled_utc`, `created_utc`, `updated_utc`, `lease_expires_utc`
- Verify `workflow.blocklist_item_results` columns:
  - `blocklist_item_result_id`, `blocklist_schedule_id`, `common_consumer_id`, `payload`, `result_payload`, `item_status`, `source_page_number`, `page_number`, `created_utc`, `updated_utc`, `lease_expires_utc`
- Verify `workflow.blocklist_business_results` exists and is keyed by `blocklist_schedule_id`
- Confirm there are no columns such as:
  - `phone_number`
  - `email_address`
  - `street_address`
  - `cleared_date_utc`

**Task 1.1.2: Confirm state-machine ordering assumptions**
- Validate that `BlocklistScheduleCompleted` runs before `BlocklistItemClean`
- Validate that schedule completion happens while `blocklist_item_results` rows still exist
- Confirm the write path can capture detail rows before retention cleanup removes source rows

### 1.2 Create Parent Summary Table

**Task 1.2.1: Add Liquibase changelog for `workflow.blocklist_purge_history`**
- Create directory:
  - `common-db/postgres/Databases/store/Schemas/workflow/Tables/16.BlocklistPurgeHistory/`
- Create `changelog-1.xml` with the following columns:
  - `blocklist_purge_history_id`
  - `blocklist_schedule_id`
  - `coorg_id`
  - `run_status`
  - `requested_by`
  - `started_utc`
  - `completed_utc`
  - `time_taken_seconds`
  - `total_items_processed`
  - `phone_records_count`
  - `email_records_count`
  - `address_records_count`
  - `name_records_count`
  - `total_business_results`
  - `items_failed`
  - `retention_items_deleted`
  - `retention_business_deleted`
  - `retention_days_applied`
  - `is_estimated`
  - `error_message`
  - `created_utc`
  - `updated_utc`
- Constraints:
  - PK on `blocklist_purge_history_id`
  - UNIQUE on `blocklist_schedule_id`
  - FK from `blocklist_schedule_id` to `workflow.blocklist_schedules(blocklist_schedule_id)`
- Indexes:
  - `coorg_id`
  - `run_status`
  - `started_utc`
  - `completed_utc`
  - `(coorg_id, run_status, started_utc)`

**Task 1.2.2: Validate parent summary migration**
- Tests:
  - Migration applies cleanly on empty DB
  - Migration is idempotent
  - Unique constraint prevents duplicate row per schedule
  - FK is enforced correctly

### 1.3 Create Child Detail Table

**Task 1.3.1: Add Liquibase changelog for `workflow.blocklist_purge_history_details`**
- Create directory:
  - `common-db/postgres/Databases/store/Schemas/workflow/Tables/17.BlocklistPurgeHistoryDetails/`
- Create `changelog-1.xml` with the following columns:
  - `blocklist_purge_history_detail_id`
  - `blocklist_purge_history_id`
  - `blocklist_schedule_id`
  - `coorg_id`
  - `record_type`
  - `common_consumer_id`
  - `record_value`
  - `item_status`
  - `result_payload`
  - `source_page_number`
  - `page_number`
  - `created_utc`
  - `updated_utc`
- Constraints:
  - PK on `blocklist_purge_history_detail_id`
  - FK from `blocklist_purge_history_id` to `workflow.blocklist_purge_history(blocklist_purge_history_id)`
  - CHECK `record_type IN ('phone','email','address','name')`

**Task 1.3.2: Add detail-table indexes and idempotency constraint**
- Indexes:
  - `blocklist_purge_history_id`
  - `blocklist_schedule_id`
  - `(record_type, blocklist_purge_history_id)`
  - `(blocklist_purge_history_id, item_status)`
  - `(blocklist_purge_history_id, record_value)`
  - `(blocklist_purge_history_id, common_consumer_id)`
  - `(blocklist_purge_history_id, source_page_number, page_number)`
- Add uniqueness constraint for retry-safe inserts:
```sql
UNIQUE (blocklist_purge_history_id, common_consumer_id, record_type, record_value)
```

**Task 1.3.3: Validate detail-table migration**
- Tests:
  - Migration applies cleanly
  - Retry-safe uniqueness works
  - `record_type` check blocks unsupported types
  - FK to parent row is enforced

### 1.4 Update Explicit Schedule Purge Function Design

**Task 1.4.1: Update deletion plan for `purge_data_by_blocklist_schedule_id`**
- Required delete order:
  1. `workflow.blocklist_item_results`
  2. `workflow.blocklist_business_results`
  3. `workflow.blocklist_purge_history_details`
  4. `workflow.blocklist_purge_history`
  5. `workflow.blocklist_schedules`
- Add migration/update task for the function definition

**Task 1.4.2: Validate delete-order behavior**
- Tests:
  - No FK violations when explicit purge runs
  - Parent/child history rows are deleted for the schedule

---

## Phase 2: Workflow-Resource Write Path

### 2.1 Add Parent Summary Models and Repositories

**Task 2.1.1: Add entity/model for `blocklist_purge_history`**
- Add repository entity class under workflow-resource ResourceAccess entities
- Add mapping model if service-layer model is required

**Task 2.1.2: Add repository methods for parent summary row**
- Required operations:
  - Create summary row if missing
  - Update run status
  - Update summary counts
  - Update failure state and `error_message`
  - Read parent row by `blocklist_schedule_id`

**Task 2.1.3: Add resource-access interface methods**
- Add methods to:
  - `CreatePurgeHistoryIfMissingAsync(...)`
  - `UpdatePurgeHistoryStatusAsync(...)`
  - `UpdatePurgeHistorySummaryAsync(...)`
  - `GetPurgeHistoryByScheduleIdAsync(...)`

### 2.2 Add Child Detail Models and Repositories

**Task 2.2.1: Add entity/model for `blocklist_purge_history_details`**
- Add repository entity class under workflow-resource ResourceAccess entities
- Ensure `result_payload` maps correctly as JSONB

**Task 2.2.2: Add repository method to bulk-insert detail rows from item results**
- Insert source rows from `workflow.blocklist_item_results` into `workflow.blocklist_purge_history_details`
- Use `payload->>'type'` for `record_type`
- Use `payload->>'value'` for `record_value`
- Add `ON CONFLICT DO NOTHING` for idempotency

**Task 2.2.3: Add repository method to aggregate summary counts from child rows**
- Aggregate:
  - `total_items_processed`
  - `phone_records_count`
  - `email_records_count`
  - `address_records_count`
  - `name_records_count`
  - `items_failed`

### 2.3 Hook Into Existing Schedule Lifecycle

**Task 2.3.1: Create parent summary row at schedule creation/requested stage**
- Primary path: when schedule row is created by the event consumer
- Fallback path: when `BlocklistScheduleRequested` runs, create summary row if still missing
- Use `ON CONFLICT DO NOTHING`

**Task 2.3.2: Update summary row when status changes to `Requested`**
- Persist:
  - `run_status = 'Requested'`
  - `started_utc` if null

**Task 2.3.3: Copy item-result rows into child detail table at `Completed`**
- In `BlocklistScheduleCompleted`:
  - Load parent summary row ID
  - Insert all detail rows from `workflow.blocklist_item_results` for the schedule
  - Ensure this happens before `BlocklistItemClean`

**Task 2.3.4: Aggregate and persist summary counts at `Completed`**
- Aggregate child-table counts
- Count `workflow.blocklist_business_results` for `total_business_results`
- Update parent summary row with:
  - `run_status = 'Completed'`
  - `completed_utc`
  - `time_taken_seconds`
  - all count columns

**Task 2.3.5: Persist terminal error/cancel states**
- For `Cancelled`, `Paused`, and future `Failed` flows:
  - Update parent `run_status`
  - Set `completed_utc` for terminal states
  - Populate `error_message` when relevant

### 2.4 Retention Cleanup Metrics

**Task 2.4.1: Leave retention metrics nullable in first implementation**
- Do not change existing cleanup function yet
- Parent table fields remain:
  - `retention_items_deleted = NULL`
  - `retention_business_deleted = NULL`
  - `retention_days_applied = NULL` or configured value if cheaply available

**Task 2.4.2: Document future enhancement**
- Future iteration may change:
  - `workflow.purge_blocklist_results_data_as_per_retention_days(...)`
- To return exact delete counts for cleanup metrics

### 2.5 Workflow-Resource Unit Tests

**Task 2.5.1: Parent row tests**
- Create-if-missing is idempotent
- Status updates are correct
- Summary count update persists expected values

**Task 2.5.2: Child row tests**
- One detail row inserted per source item
- `record_type` extracted correctly from JSONB payload
- Retry does not duplicate rows

**Task 2.5.3: Schedule completion tests**
- Child rows inserted before cleanup step could delete source rows
- Parent counts match child-row aggregate

---

## Phase 3: Read APIs, Testing, and Deployment

### 3.1 Workflow-Resource Summary API

**Task 3.1.1: Add request/response contracts for summary history list**
- Request supports:
  - `coOrgDealerId`
  - `runStatus`
  - `requestedBy`
  - `startDateFromUtc`
  - `startDateToUtc`
  - `pageNumber`
  - `pageSize`
  - `sortBy`
  - `sortDirection`
- Response item includes only summary fields and counts

**Task 3.1.2: Add repository query for parent table list endpoint**
- Query `workflow.blocklist_purge_history`
- Filter by parent-table columns only
- Support paging and sorting

**Task 3.1.3: Add controller endpoint**
- `GET /blocklist-schedule/purge-history`

### 3.2 Workflow-Resource Detail API

**Task 3.2.1: Add request/response contracts for detail drilldown**
- Query/filter support:
  - `recordType`
  - `itemStatus`
  - `searchValue`
  - `pageNumber`
  - `pageSize`
  - `sortBy`
  - `sortDirection`

**Task 3.2.2: Add repository query for child detail rows**
- Query `workflow.blocklist_purge_history_details`
- Filter by parent schedule/history ID plus detail filters
- Support paging and sorting

**Task 3.2.3: Add controller endpoint**
- `GET /blocklist-schedule/purge-history/id/{blocklistScheduleId}/details`

### 3.3 Upstream Passthrough APIs

**Task 3.3.1: Update configuration-service passthrough plan**
- Summary endpoint:
  - `GET /blocklist/purge-history`
- Detail endpoint:
  - `GET /blocklist/purge-history/id/{blocklistScheduleId}/details`

**Task 3.3.2: Update consumer-card/BFF passthrough plan**
- Summary endpoint:
  - `GET /api/blocklist/purge-history`
- Detail endpoint:
  - `GET /api/blocklist/purge-history/id/{blocklistScheduleId}/details`

### 3.4 Integration and Performance Tests

**Task 3.4.1: End-to-end run history test**
- Seed blocklist schedule + item results
- Run completion step
- Assert parent summary row and child detail rows are created

**Task 3.4.2: Summary endpoint behavior**
- Filters, paging, sorting, empty result set

**Task 3.4.3: Detail endpoint behavior**
- Filter by `recordType`
- Filter by `itemStatus`
- Search by `recordValue`
- Verify paging across large runs

**Task 3.4.4: Large-run performance test**
- Seed high-volume detail rows
- Verify summary endpoint stays fast because it only hits the parent table
- Verify detail endpoint remains queryable using child-table indexes

### 3.5 Deployment and Rollout

**Task 3.5.1: Rollout order**
1. Deploy parent summary table migration
2. Deploy child detail table migration
3. Deploy workflow-resource write path
4. Deploy workflow-resource read APIs
5. Deploy configuration-service/BFF passthroughs

**Task 3.5.2: Monitoring and telemetry**
- Add logs/metrics for:
  - parent summary row creation/update
  - child detail row bulk insert
  - detail insert conflict counts
  - summary/detail API latency

**Task 3.5.3: Optional legacy backfill**
- Evaluate whether to backfill parent summary rows with `is_estimated = true`
- Do not backfill child detail rows if source rows are already gone

---

## Definition of Done

- `workflow.blocklist_purge_history` exists and is populated during purge lifecycle
- `workflow.blocklist_purge_history_details` exists and stores one row per processed item
- Parent summary counts are derived from child rows and remain consistent
- Summary API returns lightweight run history without heavy payloads
- Detail API returns paged/filterable per-record results
- Explicit schedule purge deletes child rows before parent rows
- Unit, integration, and performance tests cover the normalized design
