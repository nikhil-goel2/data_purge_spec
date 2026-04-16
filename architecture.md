<<<<<<< HEAD
---
stepsCompleted: [1, 2]
inputDocuments:
  - c:/Code/Blocklist-Purge-History-Analysis.md
  - c:/Code/Blocklist-Purge-Data-Logic.md
  - c:/Code/PurgeBlocklistREADME.md
  - c:/Code/Dealer-Blocklist-End-to-End-Architecture.md
  - c:/Code/Dealer-Blocklist-UI-Records-Workflow.md
  - c:/Code/consumer-card/src/CoxAuto.ConsumerCard.Host/Controllers/BlocklistController.cs
  - c:/Code/configuration-service/src/CoxAuto.Thunderbird.ConfigurationService.Core/Orchestrators/BlockListController.cs
  - c:/Code/configuration-service/src/CoxAuto.Thunderbird.ConfigurationService.Core/Managers/BlocklistManager.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Host/Controllers/BlocklistController.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/BlocklistStateMachines/Steps/BlocklistItemClean.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/BlocklistStateMachines/Steps/BlocklistScheduleRequested.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/BlocklistStateMachines/Steps/BlocklistScheduleCompleted.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/StateMachineResourceAccess.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/IStateMachineResourceAccess.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/Repositories/BlocklistScheduleRepository.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/Repositories/BlocklistItemResultRepository.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/Repositories/Entities/blocklist_schedule.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/Repositories/Entities/blocklist_item_result.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/Repositories/Entities/blocklist_business_result.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.UnitTests/Manager/BlocklistStateMachines/Steps/BlocklistScheduleRequestedTests.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.UnitTests/Manager/BlocklistStateMachines/Steps/BlocklistScheduleCompletedTests.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.UnitTests/Manager/BlocklistStateMachines/Steps/BlocklistItemCleanTests.cs
workflowType: 'architecture'
project_name: 'Code'
user_name: 'NGOEL1'
date: '2026-03-19'
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Initialization Notes

This workflow is initialized using approved purge-related design and source references. PRD has not yet been provided.

## Seed PRD Baseline (Draft from User Requirement)

### Problem Statement

Developers currently do not have a single reliable API contract to retrieve data purge execution history with complete operational metrics. The system needs a dedicated, queryable purge history view including status, start and end timestamps, elapsed time, initiator, and records affected.

### Goal

Design and implement an API that fetches purge history records end-to-end so each record includes:

- status
- start date
- end date
- time taken
- number of records affected
- initiated by user

### Purge Activity Visibility Contract

Purge Activity Visibility is the operational ability to answer, for any purge run, who initiated it, when it started, when it ended, what final status it reached, and how many records were deleted by category and in total.

Visibility in this design is defined by these strict rules:

- One row represents one purge run keyed by `blocklist_schedule_id`.
- `startDateUtc` is the execution start timestamp for that run in UTC.
- `endDateUtc` is the execution completion timestamp for that run in UTC.
- `timeTakenSeconds` is derived as `max(0, endDateUtc - startDateUtc)` in seconds.
- `requestedBy` is the user identity captured at scheduling/request time.
- `itemResultsDeletedCount`, `businessResultsDeletedCount`, and `totalRecordsDeleted` are exact for runs executed after audit rollout.
- `isEstimatedCount = true` only for legacy runs without an audit-log row.
- All date/time values are returned as UTC (`Z`) and never as local time.
- API sorting and filtering must operate on persisted UTC timestamps, not client-local transforms.

### In Scope

- Read-only API for purge history retrieval.
- Data model and query strategy across workflow schema tables.
- Accurate derivation and persistence strategy for record count affected.
- Detailed developer-facing architecture report for implementation.

### Out of Scope

- Building UI components.
- Redesigning unrelated blocklist rule management endpoints.
- Historical backfill beyond available source data unless explicitly defined.

### Functional Requirements (FR)

- FR-001: API returns purge history list for a requested date range.
- FR-002: Each record includes blocklist_schedule_id, coorg/dealer identifier, status, requested_by, start_date_utc, end_date_utc, and time_taken_seconds.
- FR-003: Each record includes records_affected split by item/business and total.
- FR-004: API supports filtering by blocklist_schedule_status, coorg_id, requested_by, and UTC date ranges.
- FR-005: API supports paging and sorting by start_date_utc, end_date_utc, or total_records_deleted.
- FR-006: API supports detail endpoint by blocklist_schedule_id.
- FR-007: API returns stable schema even when some metrics are unavailable, with explicit null/zero semantics.
- FR-008: API exposes visibility state transitions using canonical schedule statuses only.

### Non-Functional Requirements (NFR)

- NFR-001: Query response should be optimized for operational dashboards and support index-backed filters.
- NFR-002: Metrics must be auditable and deterministic.
- NFR-003: API must enforce existing service authorization policy.
- NFR-004: API behavior must be backward-compatible and non-breaking to existing purge flow.
- NFR-005: Telemetry should include correlation id and query diagnostics.
- NFR-006: Visibility freshness target is near-real-time; completed purge runs should be queryable within 60 seconds of status completion.
- NFR-007: Visibility completeness target is 100 percent for all purge runs executed after audit-log rollout.

### Assumptions

- Primary source-of-truth schedule table remains workflow.blocklist_schedules.
- Current system does not persist exact deleted row counts for purge function execution.
- Enhancement will be needed to persist records_affected accurately.

### Finalized Decisions (Implementation Defaults)

These defaults remove ambiguity for implementation. Any deviation must be recorded as a versioned contract change.

- Decision D-001 (records_affected policy): Use exact counts for all runs after audit rollout. For pre-rollout runs, return counts as `0` with `isEstimatedCount = true`. No online estimate computation in read path.
- Decision D-002 (retention visibility): Expose `retentionDaysApplied` per row as nullable. Null means retention value is unavailable for that run.
- Decision D-003 (service exposure): Implement source endpoints in workflow-resource and expose passthrough endpoints in configuration-service and consumer-card.
- Decision D-004 (identity field): Use `requestedBy` as the canonical initiator field across all layers.
- Decision D-005 (detail not found behavior): Return `204 NoContent` for missing `blocklistScheduleId`.

### Visibility Status Vocabulary

The visibility API accepts and returns only these schedule statuses:

- Scheduled
- Requested
- InProgress
- Completed
- Cancelled
- Failed (if added by workflow state machine enhancement)

If `Failed` is not implemented in the state model yet, clients must not submit it as a filter value.

### Acceptance Criteria

- AC-001: Developer can call one API and retrieve purge history with all required fields.
- AC-002: Records include exact records_affected for newly executed purges after implementation.
- AC-003: Architecture and implementation report documents endpoint contract, data flow, storage, indexing, and migration strategy.

## Project Context Analysis

### Requirements Overview

**Functional Requirements (architectural interpretation):**

- The system needs a dedicated read API over purge history data.
- Purge rows must expose lifecycle timestamps and status.
- The API must include deterministic records_affected metrics.
- The API must be filterable, pageable, and sortable for operations use.

**Non-Functional Requirements (architectural impact):**

- Audit integrity drives persistence design (metrics must be stored, not inferred).
- Query performance drives index design on status/date/coorg/user.
- Existing auth and service boundaries must be preserved.
- Backward compatibility drives additive API strategy.

### Scale and Complexity Assessment

- Primary domain: backend API + data access + operational reporting.
- Complexity level: medium.
- Cross-cutting concerns: auditability, performance, security, backward compatibility, observability.

### Technical Constraints and Dependencies

- Current purge flow uses workflow.blocklist_schedules as orchestration source.
- Purge execution currently calls workflow.purge_blocklist_results_data_as_per_retention_days(...).
- Existing flow does not persist exact records_affected for each purge execution.
- Existing service chain for blocklist data spans consumer-card -> configuration-service -> workflow-resource.

### Cross-Cutting Concerns

- Authorization policy consistency across service endpoints.
- Correlation-id propagation for traceability.
- Idempotency and race handling for schedule status transitions.
- Legacy row behavior for pre-enhancement purge runs.

## Target Architecture: Purge History API

### Architectural Decision

Use workflow-resource as the source API for purge history and expose it through existing service boundaries:

1. workflow-resource: source-of-truth query endpoint and persistence model.
2. configuration-service: passthrough/translation endpoint for platform-level consumers.
3. consumer-card BFF: optional external-facing endpoint for UI/client access.

This preserves existing layering while keeping data ownership where purge execution occurs.

### Why This Design

- Keeps write/read ownership close to purge execution logic.
- Avoids duplicated query logic in higher layers.
- Supports future direct ops tooling against workflow-resource if needed.

## API Design

### Endpoint 1: List Purge History

**Workflow Resource (source):**

- Method: GET
- Path: /blocklist-schedule/purge-history

**Query Parameters:**

- coOrgDealerId: string optional
- blocklistScheduleStatus: enum optional (Scheduled, Requested, InProgress, Completed, Cancelled)
- requestedBy: string optional (exact match or case-insensitive match based on DB collation)
- startDateFromUtc: datetime optional (inclusive)
- startDateToUtc: datetime optional (inclusive)
- endDateFromUtc: datetime optional (inclusive)
- endDateToUtc: datetime optional (inclusive)
- pageNumber: int default 1, minimum 1
- pageSize: int default 50, minimum 1, maximum 200
- sortBy: enum default EndDateUtc (EndDateUtc, StartDateUtc, TotalRecordsDeleted)
- sortDirection: enum default desc (asc, desc)

**Validation Rules:**

- Reject requests where `startDateFromUtc > startDateToUtc`.
- Reject requests where `endDateFromUtc > endDateToUtc`.
- Reject requests where `pageNumber < 1` or `pageSize < 1`.
- Clamp or reject `pageSize > 200` (recommend reject with 400 for deterministic behavior).

**Response DTO:**

```json
{
  "totalCount": 125,
  "pageNumber": 1,
  "pageSize": 50,
  "items": [
    {
      "blocklistScheduleId": "uuid",
      "coOrgDealerId": "string",
      "blocklistScheduleStatus": "Completed",
      "requestedBy": "user@company.com",
      "startDateUtc": "2026-03-19T10:00:00Z",
      "endDateUtc": "2026-03-19T10:06:30Z",
      "timeTakenSeconds": 390,
      "itemResultsDeletedCount": 1200,
      "businessResultsDeletedCount": 340,
      "totalRecordsDeleted": 1540,
      "retentionDaysApplied": 30,
      "isEstimatedCount": false
    }
  ]
}
```

### Endpoint 2: Purge History Detail

**Workflow Resource (source):**

- Method: GET
- Path: /blocklist-schedule/purge-history/id/{blocklistScheduleId}

**Response DTO:** one item with the same schema as list item.

**Detail behavior:**

- Return `204 NoContent` when the schedule id is not found.
- Return `400 BadRequest` for invalid GUID format where framework model binding does not already reject.

### Optional Upstream Exposure

- configuration-service: GET /blocklist/purge-history and GET /blocklist/purge-history/id/{blocklistScheduleId}
- consumer-card BFF: GET /api/blocklist/purge-history and GET /api/blocklist/purge-history/id/{blocklistScheduleId}

## Data Model and Persistence

### Existing Tables Used

- workflow.blocklist_schedules
- workflow.blocklist_item_results
- workflow.blocklist_business_results

### New Table Required

Add a dedicated audit table for exact per-run counts:

```sql
CREATE TABLE IF NOT EXISTS workflow.blocklist_purge_audit_log (
    purge_audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    blocklist_schedule_id UUID NOT NULL REFERENCES workflow.blocklist_schedules(blocklist_schedule_id),
    item_results_deleted_count INT NOT NULL DEFAULT 0,
    business_results_deleted_count INT NOT NULL DEFAULT 0,
    total_records_deleted INT NOT NULL DEFAULT 0,
    retention_days INT NULL,
    purge_operation_started_utc TIMESTAMP NOT NULL,
    purge_operation_completed_utc TIMESTAMP NOT NULL,
    purge_duration_seconds INT NOT NULL,
    error_occurred BOOLEAN NOT NULL DEFAULT false,
    error_message VARCHAR NULL,
    created_utc TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_utc TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_bpal_schedule_id
ON workflow.blocklist_purge_audit_log(blocklist_schedule_id);

CREATE INDEX IF NOT EXISTS idx_bpal_completed_utc
ON workflow.blocklist_purge_audit_log(purge_operation_completed_utc);

CREATE INDEX IF NOT EXISTS idx_bpal_total_deleted
ON workflow.blocklist_purge_audit_log(total_records_deleted);
```

### Optional Read View

```sql
CREATE OR REPLACE VIEW workflow.v_blocklist_purge_history AS
SELECT
    bs.blocklist_schedule_id,
    bs.coorg_id,
    bs.schedule_status,
    bs.requested_by,
    bs.scheduled_utc AS start_date_utc,
    bs.updated_utc AS end_date_utc,
    EXTRACT(EPOCH FROM (bs.updated_utc - bs.created_utc))::INT AS time_taken_seconds,
    COALESCE(bpa.item_results_deleted_count, 0) AS item_results_deleted_count,
    COALESCE(bpa.business_results_deleted_count, 0) AS business_results_deleted_count,
    COALESCE(bpa.total_records_deleted, 0) AS total_records_deleted,
    bpa.retention_days,
    CASE WHEN bpa.blocklist_schedule_id IS NULL THEN true ELSE false END AS is_estimated_count
FROM workflow.blocklist_schedules bs
LEFT JOIN workflow.blocklist_purge_audit_log bpa
  ON bpa.blocklist_schedule_id = bs.blocklist_schedule_id;
```

## Purge Execution Enhancement (Records Affected)

### Current Gap

Purge operation executes SQL function but discards delete counts.

### Proposed Change

1. Modify purge SQL function (or create new function) to return deleted counts by table.
2. Update repository call to capture returned counts.
3. Persist counts into workflow.blocklist_purge_audit_log within same execution path.

### Recommended SQL Function Contract

```sql
-- Return exact delete counts for this purge run
-- result: item_results_deleted_count, business_results_deleted_count, total_records_deleted
SELECT *
FROM workflow.purge_blocklist_results_data_as_per_retention_days_with_counts(@deletion_interval_days);
```

### Service Method Contract

```csharp
public async Task<PurgeCountsResult> PurgeBlocklistItemsAsync(int? deletionIntervalDays)
```

Where PurgeCountsResult includes item count, business count, total.

## Query Strategy for API

### Primary Query Pattern

- Query from workflow.v_blocklist_purge_history.
- Apply where-clause filters only on indexed columns.
- Apply stable sort then paging.
- Use a deterministic tiebreaker (`blocklist_schedule_id`) after primary sort column to prevent page drift.

### Legacy Data Handling

- For rows before audit table rollout, counts default to 0 and is_estimated_count=true.
- Optionally compute estimates for legacy rows in offline backfill job, not in hot read path.

## Security and Access Control

- Apply existing Scope policy at workflow-resource endpoint.
- Preserve authorization at higher layers for external exposure.
- Log correlation id, request filters, and caller identity.

## Observability and Diagnostics

Emit structured logs/metrics:

- purge_history_query_count
- purge_history_query_latency_ms
- purge_history_rows_returned
- purge_history_filter_usage
- purge_execution_deleted_total
- purge_execution_error_count
- purge_visibility_freshness_seconds
- purge_visibility_missing_audit_count

Include correlationId and blocklistScheduleId where applicable.

## Error Handling Contract

- 400: invalid filter values or date range.
- 401/403: unauthorized/forbidden.
- 204: blocklistScheduleId not found for detail endpoint.
- 500: unexpected persistence/query failure.

## Implementation Blueprint by Service

### Workflow Resource (mandatory)

- Add repository method(s) for purge history query and detail.
- Add DTO mapper from view/table rows.
- Add controller endpoints:
  - GET /blocklist-schedule/purge-history
  - GET /blocklist-schedule/purge-history/id/{blocklistScheduleId}
- Enhance purge execution path to persist exact counts into audit table.

### Configuration Service (recommended)

- Add passthrough endpoints to workflow-resource purge-history APIs.
- Keep contracts aligned with workflow-resource response DTO.
- Expose:
  - GET /blocklist/purge-history
  - GET /blocklist/purge-history/id/{blocklistScheduleId}

### Consumer Card BFF (recommended if external clients need it)

- Add external endpoint(s) under /api/blocklist/purge-history.
- Forward filters and pagination to configuration-service.
- Enforce Blocklist policy.
- Expose:
  - GET /api/blocklist/purge-history
  - GET /api/blocklist/purge-history/id/{blocklistScheduleId}

## Testing Strategy

### Unit Tests

- repository filter, pagination, and sort behavior.
- count mapping and timeTakenSeconds derivation.
- audit-log persistence from purge execution result.

### Integration Tests

- endpoint returns required fields.
- authorization and validation behavior.
- exact counts stored for new purges.

### Migration Validation

- schema migration applies cleanly.
- view returns rows for both with-audit and without-audit schedules.

## Rollout Plan

1. Deploy schema migration for audit table and indexes.
2. Deploy workflow-resource changes for write-path count capture.
3. Deploy workflow-resource read endpoints.
4. Optionally deploy configuration-service and BFF passthrough endpoints.
5. Enable dashboard/reporting consumers.

## Developer Quick Start

1. Create migration for workflow.blocklist_purge_audit_log.
2. Implement purge count return contract in SQL + repository.
3. Persist count audit rows during purge execution.
4. Add purge-history query endpoints in workflow-resource.
5. Add passthrough endpoints upstream as needed.
6. Validate with integration tests and sample data.

## Final Notes

This architecture intentionally separates:

- execution state (blocklist_schedules)
- execution outcomes (blocklist_purge_audit_log)
- query projection (v_blocklist_purge_history)

## Code Stubs and File Mapping (Exact Signatures)

This section provides implementation-ready DTOs and method signatures aligned with the current naming and layering patterns in this repository.

### Workflow Resource (Source of Truth)

#### Host Contracts

Target files to add under `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Host/Contracts/`:

1) `GetBlocklistPurgeHistoryRequest.cs`

```csharp
using CoxAuto.Thunderbird.WorkflowResource.Common.Models;
using System;

namespace CoxAuto.Thunderbird.WorkflowResource.Host.Contracts
{
  public class GetBlocklistPurgeHistoryRequest
  {
    public string CoOrgDealerId { get; set; }
    public BlocklistScheduleStatus? BlocklistScheduleStatus { get; set; }
    public string RequestedBy { get; set; }
    public DateTimeOffset? StartDateFromUtc { get; set; }
    public DateTimeOffset? StartDateToUtc { get; set; }
    public DateTimeOffset? EndDateFromUtc { get; set; }
    public DateTimeOffset? EndDateToUtc { get; set; }
    public int PageNumber { get; set; } = 1;
    public int PageSize { get; set; } = 50;
    public string SortBy { get; set; } = "EndDateUtc";
    public string SortDirection { get; set; } = "desc";
  }
}
```

2) `BlocklistPurgeHistoryResponse.cs`

```csharp
using System.Collections.Generic;

namespace CoxAuto.Thunderbird.WorkflowResource.Host.Contracts
{
  public class BlocklistPurgeHistoryResponse
  {
    public int TotalCount { get; set; }
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
    public List<BlocklistPurgeHistoryItemResponse> Items { get; set; } = new List<BlocklistPurgeHistoryItemResponse>();
  }
}
```

3) `BlocklistPurgeHistoryItemResponse.cs`

```csharp
using CoxAuto.Thunderbird.WorkflowResource.Common.Models;
using System;

namespace CoxAuto.Thunderbird.WorkflowResource.Host.Contracts
{
  public class BlocklistPurgeHistoryItemResponse
  {
    public Guid BlocklistScheduleId { get; set; }
    public string CoOrgDealerId { get; set; }
    public BlocklistScheduleStatus BlocklistScheduleStatus { get; set; }
    public string RequestedBy { get; set; }
    public DateTimeOffset StartDateUtc { get; set; }
    public DateTimeOffset EndDateUtc { get; set; }
    public int TimeTakenSeconds { get; set; }
    public int ItemResultsDeletedCount { get; set; }
    public int BusinessResultsDeletedCount { get; set; }
    public int TotalRecordsDeleted { get; set; }
    public int? RetentionDaysApplied { get; set; }
    public bool IsEstimatedCount { get; set; }
  }
}
```

#### Common Models

Target files to add under `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Common/Models/`:

1) `GetBlocklistPurgeHistoryFilter.cs`

```csharp
using System;

namespace CoxAuto.Thunderbird.WorkflowResource.Common.Models
{
  public class GetBlocklistPurgeHistoryFilter
  {
    public string CoOrgDealerId { get; set; }
    public BlocklistScheduleStatus? BlocklistScheduleStatus { get; set; }
    public string RequestedBy { get; set; }
    public DateTimeOffset? StartDateFromUtc { get; set; }
    public DateTimeOffset? StartDateToUtc { get; set; }
    public DateTimeOffset? EndDateFromUtc { get; set; }
    public DateTimeOffset? EndDateToUtc { get; set; }
    public int PageNumber { get; set; } = 1;
    public int PageSize { get; set; } = 50;
    public string SortBy { get; set; } = "EndDateUtc";
    public string SortDirection { get; set; } = "desc";
  }
}
```

2) `BlocklistPurgeHistoryResult.cs`

```csharp
using System;

namespace CoxAuto.Thunderbird.WorkflowResource.Common.Models
{
  public class BlocklistPurgeHistoryResult
  {
    public Guid BlocklistScheduleId { get; set; }
    public string CoOrgDealerId { get; set; }
    public BlocklistScheduleStatus BlocklistScheduleStatus { get; set; }
    public string RequestedBy { get; set; }
    public DateTimeOffset StartDateUtc { get; set; }
    public DateTimeOffset EndDateUtc { get; set; }
    public int TimeTakenSeconds { get; set; }
    public int ItemResultsDeletedCount { get; set; }
    public int BusinessResultsDeletedCount { get; set; }
    public int TotalRecordsDeleted { get; set; }
    public int? RetentionDaysApplied { get; set; }
    public bool IsEstimatedCount { get; set; }
  }
}
```

3) `BlocklistPurgeHistoryPageResult.cs`

```csharp
using System.Collections.Generic;

namespace CoxAuto.Thunderbird.WorkflowResource.Common.Models
{
  public class BlocklistPurgeHistoryPageResult
  {
    public int TotalCount { get; set; }
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
    public List<BlocklistPurgeHistoryResult> Items { get; set; } = new List<BlocklistPurgeHistoryResult>();
  }
}
```

#### Controller Signatures

Target file: `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Host/Controllers/BlocklistSchedulesController.cs`

```csharp
[HttpGet("blocklist-schedule/purge-history")]
[SwaggerResponse(StatusCodes.Status200OK, type: typeof(BlocklistPurgeHistoryResponse))]
[SwaggerResponse(StatusCodes.Status400BadRequest)]
public async Task<ActionResult<BlocklistPurgeHistoryResponse>> GetBlocklistPurgeHistory([FromQuery] GetBlocklistPurgeHistoryRequest request)

[HttpGet("blocklist-schedule/purge-history/id/{blocklistScheduleId}")]
[SwaggerResponse(StatusCodes.Status200OK, type: typeof(BlocklistPurgeHistoryItemResponse))]
[SwaggerResponse(StatusCodes.Status204NoContent, type: typeof(BlocklistPurgeHistoryItemResponse))]
[SwaggerResponse(StatusCodes.Status400BadRequest)]
public async Task<ActionResult<BlocklistPurgeHistoryItemResponse>> GetBlocklistPurgeHistoryById(Guid blocklistScheduleId)
```

#### Manager Signatures

Target files:

- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/IBlocklistScheduleManager.cs`
- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/BlocklistScheduleManager.cs`

```csharp
Task<BlocklistPurgeHistoryPageResult> GetBlocklistPurgeHistoryAsync(GetBlocklistPurgeHistoryFilter filter);
Task<BlocklistPurgeHistoryResult> GetBlocklistPurgeHistoryByIdAsync(Guid blocklistScheduleId);
```

#### Resource Access Signatures

Target files:

- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/IBlocklistScheduleResourceAccess.cs`
- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/BlocklistScheduleResourceAccess.cs`

```csharp
Task<BlocklistPurgeHistoryPageResult> GetBlocklistPurgeHistoryAsync(GetBlocklistPurgeHistoryFilter filter);
Task<BlocklistPurgeHistoryResult> GetBlocklistPurgeHistoryByIdAsync(Guid blocklistScheduleId);
```

#### Repository Signatures

Target file: `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/Repositories/BlocklistScheduleRepository.cs`

```csharp
Task<(List<blocklist_purge_history_row> rows, int totalCount)> GetBlocklistPurgeHistoryAsync(
  string coorgId,
  string scheduleStatus,
  string requestedBy,
  DateTimeOffset? startDateFromUtc,
  DateTimeOffset? startDateToUtc,
  DateTimeOffset? endDateFromUtc,
  DateTimeOffset? endDateToUtc,
  int pageNumber,
  int pageSize,
  string sortBy,
  string sortDirection);

Task<blocklist_purge_history_row> GetBlocklistPurgeHistoryByIdAsync(Guid blocklistScheduleId);
```

Target file to add: `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/Repositories/Entities/blocklist_purge_history_row.cs`

```csharp
using System;

namespace CoxAuto.Thunderbird.WorkflowResource.ResourceAccess.Repositories.Entities
{
  public class blocklist_purge_history_row
  {
    public Guid blocklist_schedule_id { get; set; }
    public string coorg_id { get; set; }
    public string schedule_status { get; set; }
    public string requested_by { get; set; }
    public DateTime start_date_utc { get; set; }
    public DateTime end_date_utc { get; set; }
    public int time_taken_seconds { get; set; }
    public int item_results_deleted_count { get; set; }
    public int business_results_deleted_count { get; set; }
    public int total_records_deleted { get; set; }
    public int? retention_days { get; set; }
    public bool is_estimated_count { get; set; }
  }
}
```

### Configuration Service (Passthrough)

#### Core Models

Target files to add under `configuration-service/src/CoxAuto.Thunderbird.ConfigurationService.Core/Models/`:

- `GetBlocklistPurgeHistoryRequest.cs`
- `BlocklistPurgeHistoryResponse.cs`
- `BlocklistPurgeHistoryItemResponse.cs`

Use the same property names as workflow-resource contract classes to avoid mapping complexity.

#### Manager Signatures

Target files:

- `configuration-service/src/CoxAuto.Thunderbird.ConfigurationService.Core/Managers/IBlocklistManager.cs`
- `configuration-service/src/CoxAuto.Thunderbird.ConfigurationService.Core/Managers/BlocklistManager.cs`

```csharp
Task<BlocklistPurgeHistoryResponse> GetBlocklistPurgeHistoryAsync(GetBlocklistPurgeHistoryRequest request);
Task<BlocklistPurgeHistoryItemResponse> GetBlocklistPurgeHistoryByIdAsync(Guid blocklistScheduleId);
```

#### Controller Signatures

Target file: `configuration-service/src/CoxAuto.Thunderbird.ConfigurationService.Core/Orchestrators/BlockListController.cs`

```csharp
[HttpGet("purge-history")]
[SwaggerResponse(StatusCodes.Status200OK, "The Message was completed successfully", typeof(BlocklistPurgeHistoryResponse))]
[SwaggerResponse(StatusCodes.Status400BadRequest, "The request is invalid")]
public async Task<ActionResult<BlocklistPurgeHistoryResponse>> GetPurgeHistory([FromQuery] GetBlocklistPurgeHistoryRequest request, [CorrelationHeader] string correlationId)

[HttpGet("purge-history/id/{blocklistScheduleId}")]
[SwaggerResponse(StatusCodes.Status200OK, "The Message was completed successfully", typeof(BlocklistPurgeHistoryItemResponse))]
[SwaggerResponse(StatusCodes.Status204NoContent, "No history record was found", typeof(BlocklistPurgeHistoryItemResponse))]
[SwaggerResponse(StatusCodes.Status400BadRequest, "The request is invalid")]
public async Task<ActionResult<BlocklistPurgeHistoryItemResponse>> GetPurgeHistoryById(Guid blocklistScheduleId, [CorrelationHeader] string correlationId)
```

### Consumer Card (External BFF)

#### Host Contracts

Target files to add under `consumer-card/src/CoxAuto.ConsumerCard.Host/Contracts/Blocklist/`:

- `GetBlocklistPurgeHistoryRequest.cs`
- `BlocklistPurgeHistoryResponse.cs`
- `BlocklistPurgeHistoryItemResponse.cs`

Use same property names as configuration-service models for direct pass-through.

#### Manager Signatures

Target files:

- `consumer-card/src/CoxAuto.ConsumerCard.Host/Services/Managers/IBlocklistManager.cs`
- `consumer-card/src/CoxAuto.ConsumerCard.Host/Services/Managers/BlocklistManager.cs`

```csharp
Task<(ResponseType ResponseType, BlocklistPurgeHistoryResponse purgeHistory, ValidationProblemDetails ProblemDetails)> GetBlocklistPurgeHistory(GetBlocklistPurgeHistoryRequest request);

Task<(ResponseType ResponseType, BlocklistPurgeHistoryItemResponse purgeHistoryItem, ValidationProblemDetails ProblemDetails)> GetBlocklistPurgeHistoryById(Guid blocklistScheduleId);
```

#### Controller Signatures

Target file: `consumer-card/src/CoxAuto.ConsumerCard.Host/Controllers/BlocklistController.cs`

```csharp
[Route("purge-history")]
[HttpGet]
[ProducesResponseType(typeof(BlocklistPurgeHistoryResponse), StatusCodes.Status200OK)]
public async Task<IActionResult> GetBlocklistPurgeHistory([FromQuery] GetBlocklistPurgeHistoryRequest request)

[Route("purge-history/id/{blocklistScheduleId}")]
[HttpGet]
[ProducesResponseType(typeof(BlocklistPurgeHistoryItemResponse), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status204NoContent)]
public async Task<IActionResult> GetBlocklistPurgeHistoryById(Guid blocklistScheduleId)
```

### Notes on Null and Legacy Semantics

- `ItemResultsDeletedCount`, `BusinessResultsDeletedCount`, and `TotalRecordsDeleted` should always be non-null in API responses (default `0`).
- `IsEstimatedCount = true` for legacy schedules without audit-log rows.
- `RetentionDaysApplied` may be null for legacy schedules.
- For detail API, return `204 NoContent` when `blocklistScheduleId` does not exist.

That separation gives developers a clear implementation path and gives operations a reliable API contract for purge history analytics.
=======
---
stepsCompleted: [1, 2]
inputDocuments:
  - c:/Code/Blocklist-Purge-History-Analysis.md
  - c:/Code/Blocklist-Purge-Data-Logic.md
  - c:/Code/PurgeBlocklistREADME.md
  - c:/Code/Dealer-Blocklist-End-to-End-Architecture.md
  - c:/Code/Dealer-Blocklist-UI-Records-Workflow.md
  - c:/Code/consumer-card/src/CoxAuto.ConsumerCard.Host/Controllers/BlocklistController.cs
  - c:/Code/configuration-service/src/CoxAuto.Thunderbird.ConfigurationService.Core/Orchestrators/BlockListController.cs
  - c:/Code/configuration-service/src/CoxAuto.Thunderbird.ConfigurationService.Core/Managers/BlocklistManager.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Host/Controllers/BlocklistController.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/BlocklistStateMachines/Steps/BlocklistItemClean.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/BlocklistStateMachines/Steps/BlocklistScheduleRequested.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/BlocklistStateMachines/Steps/BlocklistScheduleCompleted.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/StateMachineResourceAccess.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/IStateMachineResourceAccess.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/Repositories/BlocklistScheduleRepository.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/Repositories/BlocklistItemResultRepository.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/Repositories/Entities/blocklist_schedule.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/Repositories/Entities/blocklist_item_result.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/Repositories/Entities/blocklist_business_result.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.UnitTests/Manager/BlocklistStateMachines/Steps/BlocklistScheduleRequestedTests.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.UnitTests/Manager/BlocklistStateMachines/Steps/BlocklistScheduleCompletedTests.cs
  - c:/Code/workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.UnitTests/Manager/BlocklistStateMachines/Steps/BlocklistItemCleanTests.cs
workflowType: 'architecture'
project_name: 'Code'
user_name: 'NGOEL1'
date: '2026-03-19'
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Initialization Notes

This workflow is initialized using approved purge-related design and source references. PRD has not yet been provided.

## Seed PRD Baseline (Draft from User Requirement)

### Problem Statement

Developers currently do not have a single reliable API contract to retrieve data purge execution history with complete operational metrics. The system needs a dedicated, queryable purge history view including status, start and end timestamps, elapsed time, initiator, and records affected.

### Goal

Design and implement an API that fetches purge history records end-to-end so each record includes:

- status
- start date
- end date
- time taken
- number of records affected
- initiated by user

### Purge Activity Visibility Contract

Purge Activity Visibility is the operational ability to answer, for any purge run, who initiated it, when it started, when it ended, what final status it reached, and how many records were deleted by category and in total.

Visibility in this design is defined by these strict rules:

- One row represents one purge run keyed by `blocklist_schedule_id`.
- `startDateUtc` is the execution start timestamp for that run in UTC.
- `endDateUtc` is the execution completion timestamp for that run in UTC.
- `timeTakenSeconds` is derived as `max(0, endDateUtc - startDateUtc)` in seconds.
- `requestedBy` is the user identity captured at scheduling/request time.
- `itemResultsDeletedCount`, `businessResultsDeletedCount`, and `totalRecordsDeleted` are exact for runs executed after audit rollout.
- `isEstimatedCount = true` only for legacy runs without an audit-log row.
- All date/time values are returned as UTC (`Z`) and never as local time.
- API sorting and filtering must operate on persisted UTC timestamps, not client-local transforms.

### In Scope

- Read-only API for purge history retrieval.
- Data model and query strategy across workflow schema tables.
- Accurate derivation and persistence strategy for record count affected.
- Detailed developer-facing architecture report for implementation.

### Out of Scope

- Building UI components.
- Redesigning unrelated blocklist rule management endpoints.
- Historical backfill beyond available source data unless explicitly defined.

### Functional Requirements (FR)

- FR-001: API returns purge history list for a requested date range.
- FR-002: Each record includes blocklist_schedule_id, coorg/dealer identifier, status, requested_by, start_date_utc, end_date_utc, and time_taken_seconds.
- FR-003: Each record includes records_affected split by item/business and total.
- FR-004: API supports filtering by blocklist_schedule_status, coorg_id, requested_by, and UTC date ranges.
- FR-005: API supports paging and sorting by start_date_utc, end_date_utc, or total_records_deleted.
- FR-006: API supports detail endpoint by blocklist_schedule_id.
- FR-007: API returns stable schema even when some metrics are unavailable, with explicit null/zero semantics.
- FR-008: API exposes visibility state transitions using canonical schedule statuses only.

### Non-Functional Requirements (NFR)

- NFR-001: Query response should be optimized for operational dashboards and support index-backed filters.
- NFR-002: Metrics must be auditable and deterministic.
- NFR-003: API must enforce existing service authorization policy.
- NFR-004: API behavior must be backward-compatible and non-breaking to existing purge flow.
- NFR-005: Telemetry should include correlation id and query diagnostics.
- NFR-006: Visibility freshness target is near-real-time; completed purge runs should be queryable within 60 seconds of status completion.
- NFR-007: Visibility completeness target is 100 percent for all purge runs executed after audit-log rollout.

### Assumptions

- Primary source-of-truth schedule table remains workflow.blocklist_schedules.
- Current system does not persist exact deleted row counts for purge function execution.
- Enhancement will be needed to persist records_affected accurately.

### Finalized Decisions (Implementation Defaults)

These defaults remove ambiguity for implementation. Any deviation must be recorded as a versioned contract change.

- Decision D-001 (records_affected policy): Use exact counts for all runs after audit rollout. For pre-rollout runs, return counts as `0` with `isEstimatedCount = true`. No online estimate computation in read path.
- Decision D-002 (retention visibility): Expose `retentionDaysApplied` per row as nullable. Null means retention value is unavailable for that run.
- Decision D-003 (service exposure): Implement source endpoints in workflow-resource and expose passthrough endpoints in configuration-service and consumer-card.
- Decision D-004 (identity field): Use `requestedBy` as the canonical initiator field across all layers.
- Decision D-005 (detail not found behavior): Return `204 NoContent` for missing `blocklistScheduleId`.

### Visibility Status Vocabulary

The visibility API accepts and returns only these schedule statuses:

- Scheduled
- Requested
- InProgress
- Completed
- Cancelled
- Failed (if added by workflow state machine enhancement)

If `Failed` is not implemented in the state model yet, clients must not submit it as a filter value.

### Acceptance Criteria

- AC-001: Developer can call one API and retrieve purge history with all required fields.
- AC-002: Records include exact records_affected for newly executed purges after implementation.
- AC-003: Architecture and implementation report documents endpoint contract, data flow, storage, indexing, and migration strategy.

## Project Context Analysis

### Requirements Overview

**Functional Requirements (architectural interpretation):**

- The system needs a dedicated read API over purge history data.
- Purge rows must expose lifecycle timestamps and status.
- The API must include deterministic records_affected metrics.
- The API must be filterable, pageable, and sortable for operations use.

**Non-Functional Requirements (architectural impact):**

- Audit integrity drives persistence design (metrics must be stored, not inferred).
- Query performance drives index design on status/date/coorg/user.
- Existing auth and service boundaries must be preserved.
- Backward compatibility drives additive API strategy.

### Scale and Complexity Assessment

- Primary domain: backend API + data access + operational reporting.
- Complexity level: medium.
- Cross-cutting concerns: auditability, performance, security, backward compatibility, observability.

### Technical Constraints and Dependencies

- Current purge flow uses workflow.blocklist_schedules as orchestration source.
- Purge execution currently calls workflow.purge_blocklist_results_data_as_per_retention_days(...).
- Existing flow does not persist exact records_affected for each purge execution.
- Existing service chain for blocklist data spans consumer-card -> configuration-service -> workflow-resource.

### Cross-Cutting Concerns

- Authorization policy consistency across service endpoints.
- Correlation-id propagation for traceability.
- Idempotency and race handling for schedule status transitions.
- Legacy row behavior for pre-enhancement purge runs.

## Target Architecture: Purge History API

### Architectural Decision

Use workflow-resource as the source API for purge history and expose it through existing service boundaries:

1. workflow-resource: source-of-truth query endpoint and persistence model.
2. configuration-service: passthrough/translation endpoint for platform-level consumers.
3. consumer-card BFF: optional external-facing endpoint for UI/client access.

This preserves existing layering while keeping data ownership where purge execution occurs.

### Why This Design

- Keeps write/read ownership close to purge execution logic.
- Avoids duplicated query logic in higher layers.
- Supports future direct ops tooling against workflow-resource if needed.

## API Design

### Endpoint 1: List Purge History

**Workflow Resource (source):**

- Method: GET
- Path: /blocklist-schedule/purge-history

**Query Parameters:**

- coOrgDealerId: string optional
- blocklistScheduleStatus: enum optional (Scheduled, Requested, InProgress, Completed, Cancelled)
- requestedBy: string optional (exact match or case-insensitive match based on DB collation)
- startDateFromUtc: datetime optional (inclusive)
- startDateToUtc: datetime optional (inclusive)
- endDateFromUtc: datetime optional (inclusive)
- endDateToUtc: datetime optional (inclusive)
- pageNumber: int default 1, minimum 1
- pageSize: int default 50, minimum 1, maximum 200
- sortBy: enum default EndDateUtc (EndDateUtc, StartDateUtc, TotalRecordsDeleted)
- sortDirection: enum default desc (asc, desc)

**Validation Rules:**

- Reject requests where `startDateFromUtc > startDateToUtc`.
- Reject requests where `endDateFromUtc > endDateToUtc`.
- Reject requests where `pageNumber < 1` or `pageSize < 1`.
- Clamp or reject `pageSize > 200` (recommend reject with 400 for deterministic behavior).

**Response DTO:**

```json
{
  "totalCount": 125,
  "pageNumber": 1,
  "pageSize": 50,
  "items": [
    {
      "blocklistScheduleId": "uuid",
      "coOrgDealerId": "string",
      "blocklistScheduleStatus": "Completed",
      "requestedBy": "user@company.com",
      "startDateUtc": "2026-03-19T10:00:00Z",
      "endDateUtc": "2026-03-19T10:06:30Z",
      "timeTakenSeconds": 390,
      "itemResultsDeletedCount": 1200,
      "businessResultsDeletedCount": 340,
      "totalRecordsDeleted": 1540,
      "retentionDaysApplied": 30,
      "isEstimatedCount": false
    }
  ]
}
```

### Endpoint 2: Purge History Detail

**Workflow Resource (source):**

- Method: GET
- Path: /blocklist-schedule/purge-history/id/{blocklistScheduleId}

**Response DTO:** one item with the same schema as list item.

**Detail behavior:**

- Return `204 NoContent` when the schedule id is not found.
- Return `400 BadRequest` for invalid GUID format where framework model binding does not already reject.

### Optional Upstream Exposure

- configuration-service: GET /blocklist/purge-history and GET /blocklist/purge-history/id/{blocklistScheduleId}
- consumer-card BFF: GET /api/blocklist/purge-history and GET /api/blocklist/purge-history/id/{blocklistScheduleId}

## Data Model and Persistence

### Existing Tables Used

- workflow.blocklist_schedules
- workflow.blocklist_item_results
- workflow.blocklist_business_results

### New Table Required

Add a dedicated audit table for exact per-run counts:

```sql
CREATE TABLE IF NOT EXISTS workflow.blocklist_purge_audit_log (
    purge_audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    blocklist_schedule_id UUID NOT NULL REFERENCES workflow.blocklist_schedules(blocklist_schedule_id),
    item_results_deleted_count INT NOT NULL DEFAULT 0,
    business_results_deleted_count INT NOT NULL DEFAULT 0,
    total_records_deleted INT NOT NULL DEFAULT 0,
    retention_days INT NULL,
    purge_operation_started_utc TIMESTAMP NOT NULL,
    purge_operation_completed_utc TIMESTAMP NOT NULL,
    purge_duration_seconds INT NOT NULL,
    error_occurred BOOLEAN NOT NULL DEFAULT false,
    error_message VARCHAR NULL,
    created_utc TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_utc TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_bpal_schedule_id
ON workflow.blocklist_purge_audit_log(blocklist_schedule_id);

CREATE INDEX IF NOT EXISTS idx_bpal_completed_utc
ON workflow.blocklist_purge_audit_log(purge_operation_completed_utc);

CREATE INDEX IF NOT EXISTS idx_bpal_total_deleted
ON workflow.blocklist_purge_audit_log(total_records_deleted);
```

### Optional Read View

```sql
CREATE OR REPLACE VIEW workflow.v_blocklist_purge_history AS
SELECT
    bs.blocklist_schedule_id,
    bs.coorg_id,
    bs.schedule_status,
    bs.requested_by,
    bs.scheduled_utc AS start_date_utc,
    bs.updated_utc AS end_date_utc,
    EXTRACT(EPOCH FROM (bs.updated_utc - bs.created_utc))::INT AS time_taken_seconds,
    COALESCE(bpa.item_results_deleted_count, 0) AS item_results_deleted_count,
    COALESCE(bpa.business_results_deleted_count, 0) AS business_results_deleted_count,
    COALESCE(bpa.total_records_deleted, 0) AS total_records_deleted,
    bpa.retention_days,
    CASE WHEN bpa.blocklist_schedule_id IS NULL THEN true ELSE false END AS is_estimated_count
FROM workflow.blocklist_schedules bs
LEFT JOIN workflow.blocklist_purge_audit_log bpa
  ON bpa.blocklist_schedule_id = bs.blocklist_schedule_id;
```

## Purge Execution Enhancement (Records Affected)

### Current Gap

Purge operation executes SQL function but discards delete counts.

### Proposed Change

1. Modify purge SQL function (or create new function) to return deleted counts by table.
2. Update repository call to capture returned counts.
3. Persist counts into workflow.blocklist_purge_audit_log within same execution path.

### Recommended SQL Function Contract

```sql
-- Return exact delete counts for this purge run
-- result: item_results_deleted_count, business_results_deleted_count, total_records_deleted
SELECT *
FROM workflow.purge_blocklist_results_data_as_per_retention_days_with_counts(@deletion_interval_days);
```

### Service Method Contract

```csharp
public async Task<PurgeCountsResult> PurgeBlocklistItemsAsync(int? deletionIntervalDays)
```

Where PurgeCountsResult includes item count, business count, total.

## Query Strategy for API

### Primary Query Pattern

- Query from workflow.v_blocklist_purge_history.
- Apply where-clause filters only on indexed columns.
- Apply stable sort then paging.
- Use a deterministic tiebreaker (`blocklist_schedule_id`) after primary sort column to prevent page drift.

### Legacy Data Handling

- For rows before audit table rollout, counts default to 0 and is_estimated_count=true.
- Optionally compute estimates for legacy rows in offline backfill job, not in hot read path.

## Security and Access Control

- Apply existing Scope policy at workflow-resource endpoint.
- Preserve authorization at higher layers for external exposure.
- Log correlation id, request filters, and caller identity.

## Observability and Diagnostics

Emit structured logs/metrics:

- purge_history_query_count
- purge_history_query_latency_ms
- purge_history_rows_returned
- purge_history_filter_usage
- purge_execution_deleted_total
- purge_execution_error_count
- purge_visibility_freshness_seconds
- purge_visibility_missing_audit_count

Include correlationId and blocklistScheduleId where applicable.

## Error Handling Contract

- 400: invalid filter values or date range.
- 401/403: unauthorized/forbidden.
- 204: blocklistScheduleId not found for detail endpoint.
- 500: unexpected persistence/query failure.

## Implementation Blueprint by Service

### Workflow Resource (mandatory)

- Add repository method(s) for purge history query and detail.
- Add DTO mapper from view/table rows.
- Add controller endpoints:
  - GET /blocklist-schedule/purge-history
  - GET /blocklist-schedule/purge-history/id/{blocklistScheduleId}
- Enhance purge execution path to persist exact counts into audit table.

### Configuration Service (recommended)

- Add passthrough endpoints to workflow-resource purge-history APIs.
- Keep contracts aligned with workflow-resource response DTO.
- Expose:
  - GET /blocklist/purge-history
  - GET /blocklist/purge-history/id/{blocklistScheduleId}

### Consumer Card BFF (recommended if external clients need it)

- Add external endpoint(s) under /api/blocklist/purge-history.
- Forward filters and pagination to configuration-service.
- Enforce Blocklist policy.
- Expose:
  - GET /api/blocklist/purge-history
  - GET /api/blocklist/purge-history/id/{blocklistScheduleId}

## Testing Strategy

### Unit Tests

- repository filter, pagination, and sort behavior.
- count mapping and timeTakenSeconds derivation.
- audit-log persistence from purge execution result.

### Integration Tests

- endpoint returns required fields.
- authorization and validation behavior.
- exact counts stored for new purges.

### Migration Validation

- schema migration applies cleanly.
- view returns rows for both with-audit and without-audit schedules.

## Rollout Plan

1. Deploy schema migration for audit table and indexes.
2. Deploy workflow-resource changes for write-path count capture.
3. Deploy workflow-resource read endpoints.
4. Optionally deploy configuration-service and BFF passthrough endpoints.
5. Enable dashboard/reporting consumers.

## Developer Quick Start

1. Create migration for workflow.blocklist_purge_audit_log.
2. Implement purge count return contract in SQL + repository.
3. Persist count audit rows during purge execution.
4. Add purge-history query endpoints in workflow-resource.
5. Add passthrough endpoints upstream as needed.
6. Validate with integration tests and sample data.

## Final Notes

This architecture intentionally separates:

- execution state (blocklist_schedules)
- execution outcomes (blocklist_purge_audit_log)
- query projection (v_blocklist_purge_history)

## Code Stubs and File Mapping (Exact Signatures)

This section provides implementation-ready DTOs and method signatures aligned with the current naming and layering patterns in this repository.

### Workflow Resource (Source of Truth)

#### Host Contracts

Target files to add under `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Host/Contracts/`:

1) `GetBlocklistPurgeHistoryRequest.cs`

```csharp
using CoxAuto.Thunderbird.WorkflowResource.Common.Models;
using System;

namespace CoxAuto.Thunderbird.WorkflowResource.Host.Contracts
{
  public class GetBlocklistPurgeHistoryRequest
  {
    public string CoOrgDealerId { get; set; }
    public BlocklistScheduleStatus? BlocklistScheduleStatus { get; set; }
    public string RequestedBy { get; set; }
    public DateTimeOffset? StartDateFromUtc { get; set; }
    public DateTimeOffset? StartDateToUtc { get; set; }
    public DateTimeOffset? EndDateFromUtc { get; set; }
    public DateTimeOffset? EndDateToUtc { get; set; }
    public int PageNumber { get; set; } = 1;
    public int PageSize { get; set; } = 50;
    public string SortBy { get; set; } = "EndDateUtc";
    public string SortDirection { get; set; } = "desc";
  }
}
```

2) `BlocklistPurgeHistoryResponse.cs`

```csharp
using System.Collections.Generic;

namespace CoxAuto.Thunderbird.WorkflowResource.Host.Contracts
{
  public class BlocklistPurgeHistoryResponse
  {
    public int TotalCount { get; set; }
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
    public List<BlocklistPurgeHistoryItemResponse> Items { get; set; } = new List<BlocklistPurgeHistoryItemResponse>();
  }
}
```

3) `BlocklistPurgeHistoryItemResponse.cs`

```csharp
using CoxAuto.Thunderbird.WorkflowResource.Common.Models;
using System;

namespace CoxAuto.Thunderbird.WorkflowResource.Host.Contracts
{
  public class BlocklistPurgeHistoryItemResponse
  {
    public Guid BlocklistScheduleId { get; set; }
    public string CoOrgDealerId { get; set; }
    public BlocklistScheduleStatus BlocklistScheduleStatus { get; set; }
    public string RequestedBy { get; set; }
    public DateTimeOffset StartDateUtc { get; set; }
    public DateTimeOffset EndDateUtc { get; set; }
    public int TimeTakenSeconds { get; set; }
    public int ItemResultsDeletedCount { get; set; }
    public int BusinessResultsDeletedCount { get; set; }
    public int TotalRecordsDeleted { get; set; }
    public int? RetentionDaysApplied { get; set; }
    public bool IsEstimatedCount { get; set; }
  }
}
```

#### Common Models

Target files to add under `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Common/Models/`:

1) `GetBlocklistPurgeHistoryFilter.cs`

```csharp
using System;

namespace CoxAuto.Thunderbird.WorkflowResource.Common.Models
{
  public class GetBlocklistPurgeHistoryFilter
  {
    public string CoOrgDealerId { get; set; }
    public BlocklistScheduleStatus? BlocklistScheduleStatus { get; set; }
    public string RequestedBy { get; set; }
    public DateTimeOffset? StartDateFromUtc { get; set; }
    public DateTimeOffset? StartDateToUtc { get; set; }
    public DateTimeOffset? EndDateFromUtc { get; set; }
    public DateTimeOffset? EndDateToUtc { get; set; }
    public int PageNumber { get; set; } = 1;
    public int PageSize { get; set; } = 50;
    public string SortBy { get; set; } = "EndDateUtc";
    public string SortDirection { get; set; } = "desc";
  }
}
```

2) `BlocklistPurgeHistoryResult.cs`

```csharp
using System;

namespace CoxAuto.Thunderbird.WorkflowResource.Common.Models
{
  public class BlocklistPurgeHistoryResult
  {
    public Guid BlocklistScheduleId { get; set; }
    public string CoOrgDealerId { get; set; }
    public BlocklistScheduleStatus BlocklistScheduleStatus { get; set; }
    public string RequestedBy { get; set; }
    public DateTimeOffset StartDateUtc { get; set; }
    public DateTimeOffset EndDateUtc { get; set; }
    public int TimeTakenSeconds { get; set; }
    public int ItemResultsDeletedCount { get; set; }
    public int BusinessResultsDeletedCount { get; set; }
    public int TotalRecordsDeleted { get; set; }
    public int? RetentionDaysApplied { get; set; }
    public bool IsEstimatedCount { get; set; }
  }
}
```

3) `BlocklistPurgeHistoryPageResult.cs`

```csharp
using System.Collections.Generic;

namespace CoxAuto.Thunderbird.WorkflowResource.Common.Models
{
  public class BlocklistPurgeHistoryPageResult
  {
    public int TotalCount { get; set; }
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
    public List<BlocklistPurgeHistoryResult> Items { get; set; } = new List<BlocklistPurgeHistoryResult>();
  }
}
```

#### Controller Signatures

Target file: `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Host/Controllers/BlocklistSchedulesController.cs`

```csharp
[HttpGet("blocklist-schedule/purge-history")]
[SwaggerResponse(StatusCodes.Status200OK, type: typeof(BlocklistPurgeHistoryResponse))]
[SwaggerResponse(StatusCodes.Status400BadRequest)]
public async Task<ActionResult<BlocklistPurgeHistoryResponse>> GetBlocklistPurgeHistory([FromQuery] GetBlocklistPurgeHistoryRequest request)

[HttpGet("blocklist-schedule/purge-history/id/{blocklistScheduleId}")]
[SwaggerResponse(StatusCodes.Status200OK, type: typeof(BlocklistPurgeHistoryItemResponse))]
[SwaggerResponse(StatusCodes.Status204NoContent, type: typeof(BlocklistPurgeHistoryItemResponse))]
[SwaggerResponse(StatusCodes.Status400BadRequest)]
public async Task<ActionResult<BlocklistPurgeHistoryItemResponse>> GetBlocklistPurgeHistoryById(Guid blocklistScheduleId)
```

#### Manager Signatures

Target files:

- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/IBlocklistScheduleManager.cs`
- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.Manager/BlocklistScheduleManager.cs`

```csharp
Task<BlocklistPurgeHistoryPageResult> GetBlocklistPurgeHistoryAsync(GetBlocklistPurgeHistoryFilter filter);
Task<BlocklistPurgeHistoryResult> GetBlocklistPurgeHistoryByIdAsync(Guid blocklistScheduleId);
```

#### Resource Access Signatures

Target files:

- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/IBlocklistScheduleResourceAccess.cs`
- `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/BlocklistScheduleResourceAccess.cs`

```csharp
Task<BlocklistPurgeHistoryPageResult> GetBlocklistPurgeHistoryAsync(GetBlocklistPurgeHistoryFilter filter);
Task<BlocklistPurgeHistoryResult> GetBlocklistPurgeHistoryByIdAsync(Guid blocklistScheduleId);
```

#### Repository Signatures

Target file: `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/Repositories/BlocklistScheduleRepository.cs`

```csharp
Task<(List<blocklist_purge_history_row> rows, int totalCount)> GetBlocklistPurgeHistoryAsync(
  string coorgId,
  string scheduleStatus,
  string requestedBy,
  DateTimeOffset? startDateFromUtc,
  DateTimeOffset? startDateToUtc,
  DateTimeOffset? endDateFromUtc,
  DateTimeOffset? endDateToUtc,
  int pageNumber,
  int pageSize,
  string sortBy,
  string sortDirection);

Task<blocklist_purge_history_row> GetBlocklistPurgeHistoryByIdAsync(Guid blocklistScheduleId);
```

Target file to add: `workflow-resource/src/CoxAuto.Thunderbird.WorkflowResource.ResourceAccess/Repositories/Entities/blocklist_purge_history_row.cs`

```csharp
using System;

namespace CoxAuto.Thunderbird.WorkflowResource.ResourceAccess.Repositories.Entities
{
  public class blocklist_purge_history_row
  {
    public Guid blocklist_schedule_id { get; set; }
    public string coorg_id { get; set; }
    public string schedule_status { get; set; }
    public string requested_by { get; set; }
    public DateTime start_date_utc { get; set; }
    public DateTime end_date_utc { get; set; }
    public int time_taken_seconds { get; set; }
    public int item_results_deleted_count { get; set; }
    public int business_results_deleted_count { get; set; }
    public int total_records_deleted { get; set; }
    public int? retention_days { get; set; }
    public bool is_estimated_count { get; set; }
  }
}
```

### Configuration Service (Passthrough)

#### Core Models

Target files to add under `configuration-service/src/CoxAuto.Thunderbird.ConfigurationService.Core/Models/`:

- `GetBlocklistPurgeHistoryRequest.cs`
- `BlocklistPurgeHistoryResponse.cs`
- `BlocklistPurgeHistoryItemResponse.cs`

Use the same property names as workflow-resource contract classes to avoid mapping complexity.

#### Manager Signatures

Target files:

- `configuration-service/src/CoxAuto.Thunderbird.ConfigurationService.Core/Managers/IBlocklistManager.cs`
- `configuration-service/src/CoxAuto.Thunderbird.ConfigurationService.Core/Managers/BlocklistManager.cs`

```csharp
Task<BlocklistPurgeHistoryResponse> GetBlocklistPurgeHistoryAsync(GetBlocklistPurgeHistoryRequest request);
Task<BlocklistPurgeHistoryItemResponse> GetBlocklistPurgeHistoryByIdAsync(Guid blocklistScheduleId);
```

#### Controller Signatures

Target file: `configuration-service/src/CoxAuto.Thunderbird.ConfigurationService.Core/Orchestrators/BlockListController.cs`

```csharp
[HttpGet("purge-history")]
[SwaggerResponse(StatusCodes.Status200OK, "The Message was completed successfully", typeof(BlocklistPurgeHistoryResponse))]
[SwaggerResponse(StatusCodes.Status400BadRequest, "The request is invalid")]
public async Task<ActionResult<BlocklistPurgeHistoryResponse>> GetPurgeHistory([FromQuery] GetBlocklistPurgeHistoryRequest request, [CorrelationHeader] string correlationId)

[HttpGet("purge-history/id/{blocklistScheduleId}")]
[SwaggerResponse(StatusCodes.Status200OK, "The Message was completed successfully", typeof(BlocklistPurgeHistoryItemResponse))]
[SwaggerResponse(StatusCodes.Status204NoContent, "No history record was found", typeof(BlocklistPurgeHistoryItemResponse))]
[SwaggerResponse(StatusCodes.Status400BadRequest, "The request is invalid")]
public async Task<ActionResult<BlocklistPurgeHistoryItemResponse>> GetPurgeHistoryById(Guid blocklistScheduleId, [CorrelationHeader] string correlationId)
```

### Consumer Card (External BFF)

#### Host Contracts

Target files to add under `consumer-card/src/CoxAuto.ConsumerCard.Host/Contracts/Blocklist/`:

- `GetBlocklistPurgeHistoryRequest.cs`
- `BlocklistPurgeHistoryResponse.cs`
- `BlocklistPurgeHistoryItemResponse.cs`

Use same property names as configuration-service models for direct pass-through.

#### Manager Signatures

Target files:

- `consumer-card/src/CoxAuto.ConsumerCard.Host/Services/Managers/IBlocklistManager.cs`
- `consumer-card/src/CoxAuto.ConsumerCard.Host/Services/Managers/BlocklistManager.cs`

```csharp
Task<(ResponseType ResponseType, BlocklistPurgeHistoryResponse purgeHistory, ValidationProblemDetails ProblemDetails)> GetBlocklistPurgeHistory(GetBlocklistPurgeHistoryRequest request);

Task<(ResponseType ResponseType, BlocklistPurgeHistoryItemResponse purgeHistoryItem, ValidationProblemDetails ProblemDetails)> GetBlocklistPurgeHistoryById(Guid blocklistScheduleId);
```

#### Controller Signatures

Target file: `consumer-card/src/CoxAuto.ConsumerCard.Host/Controllers/BlocklistController.cs`

```csharp
[Route("purge-history")]
[HttpGet]
[ProducesResponseType(typeof(BlocklistPurgeHistoryResponse), StatusCodes.Status200OK)]
public async Task<IActionResult> GetBlocklistPurgeHistory([FromQuery] GetBlocklistPurgeHistoryRequest request)

[Route("purge-history/id/{blocklistScheduleId}")]
[HttpGet]
[ProducesResponseType(typeof(BlocklistPurgeHistoryItemResponse), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status204NoContent)]
public async Task<IActionResult> GetBlocklistPurgeHistoryById(Guid blocklistScheduleId)
```

### Notes on Null and Legacy Semantics

- `ItemResultsDeletedCount`, `BusinessResultsDeletedCount`, and `TotalRecordsDeleted` should always be non-null in API responses (default `0`).
- `IsEstimatedCount = true` for legacy schedules without audit-log rows.
- `RetentionDaysApplied` may be null for legacy schedules.
- For detail API, return `204 NoContent` when `blocklistScheduleId` does not exist.

That separation gives developers a clear implementation path and gives operations a reliable API contract for purge history analytics.
>>>>>>> f5e283b04bc9ba168be46150be990bc62dd47697
