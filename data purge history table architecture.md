<<<<<<< HEAD
# Architecture: Blocklist Purge History Table

## 1. Problem Statement

The existing data purge flow, triggered when a new blocklist rule is added with `ScheduleDataPurge = true`, does not persist run-level summary data. Specifically, the current implementation does not store:

1. Persisted history of purge runs for API/UI consumption.
2. Per-run totals for records processed.
3. Per-run breakdown totals by record type: phone, email, address, and name.
4. Retention cleanup metrics from the background cleanup step.

This document defines the architecture for a new summary table, `workflow.blocklist_purge_history`, that stores one row per purge run. Detailed per-record data is intentionally moved out of this table into a separate normalized child table documented in the companion architecture document.

---

## 2. Design Principles

| # | Principle | Rationale |
|---|---|---|
| 1 | One summary row per `blocklist_schedule_id` | Matches the existing schedule/run model |
| 2 | Summary table stays narrow | Fast list queries, lower IO, simpler mapping |
| 3 | Detailed records are normalized into a child table | Better query performance than large JSON payload columns |
| 4 | Summary writes are incremental and idempotent | State machine retries must not create duplicates |
| 5 | Parent and child are both owned by workflow-resource | Keeps writes near the execution flow |
| 6 | Core purge flow must not fail because history persistence fails | History is additive observability, not execution control |

---

## 3. New Summary Table: `workflow.blocklist_purge_history`

### 3.1 DDL

```sql
CREATE TABLE IF NOT EXISTS workflow.blocklist_purge_history (
    -- Identity
    blocklist_purge_history_id  UUID        NOT NULL DEFAULT gen_random_uuid(),
    blocklist_schedule_id       UUID        NOT NULL,

    -- Organization context
    coorg_id                    VARCHAR(64) NOT NULL DEFAULT '',

    -- Run lifecycle
    run_status                  VARCHAR(32) NOT NULL DEFAULT 'Scheduled',
    requested_by                VARCHAR(32) NOT NULL DEFAULT '',
    started_utc                 TIMESTAMP   NULL,
    completed_utc               TIMESTAMP   NULL,
    time_taken_seconds          INT         NULL,

    -- Summary metrics for rule-driven processing
    total_items_processed       INT         NOT NULL DEFAULT 0,
    phone_records_count         INT         NOT NULL DEFAULT 0,
    email_records_count         INT         NOT NULL DEFAULT 0,
    address_records_count       INT         NOT NULL DEFAULT 0,
    name_records_count          INT         NOT NULL DEFAULT 0,
    total_business_results      INT         NOT NULL DEFAULT 0,
    items_failed                INT         NOT NULL DEFAULT 0,

    -- Retention cleanup metrics
    retention_items_deleted     INT         NULL,
    retention_business_deleted  INT         NULL,
    retention_days_applied      INT         NULL,

    -- Metadata
    is_estimated                BOOLEAN     NOT NULL DEFAULT FALSE,
    error_message               TEXT        NULL,
    created_utc                 TIMESTAMP   NOT NULL DEFAULT timezone('utc'::text, now()),
    updated_utc                 TIMESTAMP   NOT NULL DEFAULT timezone('utc'::text, now()),

    CONSTRAINT blocklist_purge_history_pkey PRIMARY KEY (blocklist_purge_history_id),
    CONSTRAINT blocklist_purge_history_schedule_uq UNIQUE (blocklist_schedule_id),
    CONSTRAINT blocklist_purge_history_schedule_fk
        FOREIGN KEY (blocklist_schedule_id)
        REFERENCES workflow.blocklist_schedules(blocklist_schedule_id)
);
```

### 3.2 Indexes

```sql
CREATE INDEX IF NOT EXISTS idx_bph_coorg_id
    ON workflow.blocklist_purge_history (coorg_id);

CREATE INDEX IF NOT EXISTS idx_bph_run_status
    ON workflow.blocklist_purge_history (run_status);

CREATE INDEX IF NOT EXISTS idx_bph_started_utc
    ON workflow.blocklist_purge_history (started_utc DESC NULLS LAST);

CREATE INDEX IF NOT EXISTS idx_bph_completed_utc
    ON workflow.blocklist_purge_history (completed_utc DESC NULLS LAST);

CREATE INDEX IF NOT EXISTS idx_bph_coorg_status_started
    ON workflow.blocklist_purge_history (coorg_id, run_status, started_utc DESC NULLS LAST);
```

### 3.3 Column Reference

| Column | Type | Source | Written At |
|---|---|---|---|
| `blocklist_purge_history_id` | uuid | Auto-generated | INSERT |
| `blocklist_schedule_id` | uuid | `blocklist_schedules.blocklist_schedule_id` | INSERT |
| `coorg_id` | varchar(64) | `blocklist_schedules.coorg_id` | INSERT |
| `run_status` | varchar(32) | Mirrors `blocklist_schedules.schedule_status` | Each status transition |
| `requested_by` | varchar(32) | `blocklist_schedules.requested_by` | INSERT |
| `started_utc` | timestamp | First move to `Requested` | UPDATE at Requested |
| `completed_utc` | timestamp | Terminal transition time | UPDATE at Completed/Cancelled/Failed |
| `time_taken_seconds` | int | `completed_utc - started_utc` | UPDATE at terminal |
| `total_items_processed` | int | Count of detail rows for the run | UPDATE at Completed |
| `phone_records_count` | int | Count of detail rows where `record_type = 'phone'` | UPDATE at Completed |
| `email_records_count` | int | Count of detail rows where `record_type = 'email'` | UPDATE at Completed |
| `address_records_count` | int | Count of detail rows where `record_type = 'address'` | UPDATE at Completed |
| `name_records_count` | int | Count of detail rows where `record_type = 'name'` | UPDATE at Completed |
| `total_business_results` | int | Count from `blocklist_business_results` for the run | UPDATE at Completed |
| `items_failed` | int | Count of detail rows where `item_status = 'Failed'` | UPDATE at Completed |
| `retention_items_deleted` | int | Optional cleanup metric | UPDATE during cleanup enhancement |
| `retention_business_deleted` | int | Optional cleanup metric | UPDATE during cleanup enhancement |
| `retention_days_applied` | int | `BlocklistItemsDeletionIntervalDays` | UPDATE during cleanup enhancement |
| `is_estimated` | boolean | `true` for legacy/backfilled rows | INSERT |
| `error_message` | text | Exception summary if history write fails | UPDATE on error |
| `created_utc` | timestamp | Auto-generated | INSERT |
| `updated_utc` | timestamp | Auto-updated | Every UPDATE |

---

## 4. Write Strategy

The summary row is written at specific points in the existing state machine. Detailed records are inserted into the child table first, then summary counts are computed from that child table for consistency.

### 4.1 Lifecycle Write Map

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Existing State Machine Steps                     │
├─────────────────────────┬────────────────────────────────────────────┤
│ Step                    │ Summary Table Action                       │
├─────────────────────────┼────────────────────────────────────────────┤
│ Schedule Created        │ INSERT row with Scheduled status           │
│ Requested               │ UPDATE run_status + started_utc            │
│ Completed               │ UPDATE run_status + completed_utc +        │
│                         │ time_taken_seconds + summary counts        │
│ Cancelled/Failed        │ UPDATE run_status + completed_utc          │
│ Cleanup enhancement     │ UPDATE retention metrics                    │
└─────────────────────────┴────────────────────────────────────────────┘
```

### 4.2 Completion Aggregation Strategy

At `BlocklistScheduleCompleted`:

1. Insert detailed rows into the child table from `workflow.blocklist_item_results`.
2. Aggregate counts from the child table.
3. Update the parent summary row with the aggregated counts.

Summary aggregation query:

```sql
SELECT
    COUNT(*) AS total_items_processed,
    COUNT(*) FILTER (WHERE record_type = 'phone') AS phone_records_count,
    COUNT(*) FILTER (WHERE record_type = 'email') AS email_records_count,
    COUNT(*) FILTER (WHERE record_type = 'address') AS address_records_count,
    COUNT(*) FILTER (WHERE record_type = 'name') AS name_records_count,
    COUNT(*) FILTER (WHERE item_status = 'Failed') AS items_failed
FROM workflow.blocklist_purge_history_details
WHERE blocklist_purge_history_id = @blocklist_purge_history_id;
```

Business results count:

```sql
SELECT COUNT(*) AS total_business_results
FROM workflow.blocklist_business_results
WHERE blocklist_schedule_id = @blocklist_schedule_id;
```

### 4.3 Upsert Pattern

Create summary row:

```sql
INSERT INTO workflow.blocklist_purge_history (
    blocklist_schedule_id,
    coorg_id,
    requested_by,
    run_status,
    created_utc,
    updated_utc
)
VALUES (
    @blocklist_schedule_id,
    @coorg_id,
    @requested_by,
    'Scheduled',
    timezone('utc'::text, now()),
    timezone('utc'::text, now())
)
ON CONFLICT (blocklist_schedule_id) DO NOTHING;
```

Update status:

```sql
UPDATE workflow.blocklist_purge_history
SET run_status = @new_status,
    started_utc = CASE WHEN @new_status = 'Requested' AND started_utc IS NULL
        THEN timezone('utc'::text, now()) ELSE started_utc END,
    completed_utc = CASE WHEN @new_status IN ('Completed', 'Cancelled', 'Failed')
        THEN timezone('utc'::text, now()) ELSE completed_utc END,
    time_taken_seconds = CASE
        WHEN @new_status IN ('Completed', 'Cancelled', 'Failed') AND started_utc IS NOT NULL
        THEN EXTRACT(EPOCH FROM (timezone('utc'::text, now()) - started_utc))::INT
        ELSE time_taken_seconds
    END,
    updated_utc = timezone('utc'::text, now())
WHERE blocklist_schedule_id = @blocklist_schedule_id;
```

Update summary counts:

```sql
UPDATE workflow.blocklist_purge_history
SET total_items_processed = @total_items_processed,
    phone_records_count = @phone_records_count,
    email_records_count = @email_records_count,
    address_records_count = @address_records_count,
    name_records_count = @name_records_count,
    total_business_results = @total_business_results,
    items_failed = @items_failed,
    updated_utc = timezone('utc'::text, now())
WHERE blocklist_purge_history_id = @blocklist_purge_history_id;
```

---

## 5. Integration Points in Existing Code

No code changes are made now; this section identifies future implementation touchpoints only.

| File | Planned Change |
|---|---|
| `BlocklistScheduleResourceAccess.cs` | Add summary history create/update methods |
| `IBlocklistScheduleResourceAccess.cs` | Add summary history interface methods |
| `BlocklistScheduleRepository.cs` | Add Dapper SQL for summary history |
| `BlocklistScheduleRequested.cs` | Write Requested transition to summary row |
| `BlocklistScheduleCompleted.cs` | Trigger detail-row insert + summary aggregation |
| `purge_data_by_blocklist_schedule_id.sql` | Delete child rows, then summary row, then schedule row |

### 5.1 Future Entity Class

```csharp
public class blocklist_purge_history
{
    public Guid blocklist_purge_history_id { get; set; }
    public Guid blocklist_schedule_id { get; set; }
    public string coorg_id { get; set; }
    public string run_status { get; set; }
    public string requested_by { get; set; }
    public DateTime? started_utc { get; set; }
    public DateTime? completed_utc { get; set; }
    public int? time_taken_seconds { get; set; }
    public int total_items_processed { get; set; }
    public int phone_records_count { get; set; }
    public int email_records_count { get; set; }
    public int address_records_count { get; set; }
    public int name_records_count { get; set; }
    public int total_business_results { get; set; }
    public int items_failed { get; set; }
    public int? retention_items_deleted { get; set; }
    public int? retention_business_deleted { get; set; }
    public int? retention_days_applied { get; set; }
    public bool is_estimated { get; set; }
    public string error_message { get; set; }
    public DateTime created_utc { get; set; }
    public DateTime updated_utc { get; set; }
}
```

### 5.2 Future Liquibase Changelog

Create:

`common-db/postgres/Databases/store/Schemas/workflow/Tables/16.BlocklistPurgeHistory/changelog-1.xml`

The changelog should mirror the DDL in §3.1.

---

## 6. API Design

### 6.1 List Purge History

| Property | Value |
|---|---|
| Method | `GET` |
| Path | `/blocklist-schedule/purge-history` |
| Service | workflow-resource |
| Auth | Existing `Scope` policy |

Response item:

```json
{
  "blocklistScheduleId": "uuid",
  "coOrgDealerId": "string",
  "runStatus": "Completed",
  "requestedBy": "user@company.com",
  "startedUtc": "2026-03-19T10:00:00Z",
  "completedUtc": "2026-03-19T10:06:30Z",
  "timeTakenSeconds": 390,
  "totalItemsProcessed": 1200,
  "phoneRecordsCount": 400,
  "emailRecordsCount": 350,
  "addressRecordsCount": 300,
  "nameRecordsCount": 150,
  "totalBusinessResults": 340,
  "itemsFailed": 5,
  "retentionItemsDeleted": null,
  "retentionBusinessDeleted": null,
  "retentionDaysApplied": null,
  "isEstimated": false
}
```

### 6.2 Detail Strategy

The summary endpoint remains lightweight. Full per-record drilldown is served from the child-table design documented in `data purge history detail table architecture.md`.

---

## 7. Concurrency and Safety

| Concern | Decision |
|---|---|
| Distributed locking | Reuse existing `lease_expires_utc` scheduling model |
| Parent duplicate writes | Prevent with `UNIQUE (blocklist_schedule_id)` + `ON CONFLICT` |
| Count consistency | Aggregate from child detail rows after insert |
| Failure isolation | History write failures are logged and swallowed |

---

## 8. Failure Handling

| Scenario | Behavior |
|---|---|
| Summary row insert fails | Log and continue purge execution |
| Count aggregation fails | Log, set `error_message`, keep run status update |
| Detail row insert partially fails | Treat as detail persistence failure, do not block purge flow |
| Cleanup metrics unavailable | Leave retention fields null |

---

## 9. Migration and Legacy Data

### 9.1 Schema Rollout

Deploy the parent table first, then the child table, then the read APIs.

### 9.2 Legacy Backfill

Optional backfill can create summary rows for completed schedules with `is_estimated = true` and zeroed counts when source rows are already gone.

---

## 10. Testing Strategy

| Test | Validates |
|---|---|
| Summary row upsert is idempotent | One row per schedule |
| Summary count update works after child inserts | Parent metrics stay consistent |
| List endpoint returns narrow payload | No heavy detail payload in summary response |
| Purge flow still succeeds if history write fails | History is non-blocking |

---

## 11. Rollout Plan

| Phase | Scope |
|---|---|
| 1 | Deploy `blocklist_purge_history` table |
| 2 | Deploy child detail table from companion design |
| 3 | Wire write path in workflow-resource |
| 4 | Expose summary endpoint |
| 5 | Expose drilldown endpoint from child table |

---

## 12. Required Change to Existing Purge Delete Function

Because `blocklist_purge_history` references `blocklist_schedules`, the schedule purge function must delete history rows before deleting the schedule row. If the child table is also implemented, delete child rows first, then parent summary, then schedule.

Planned deletion order:

```sql
DELETE FROM workflow.blocklist_purge_history_details
WHERE blocklist_purge_history_id IN (
    SELECT blocklist_purge_history_id
    FROM workflow.blocklist_purge_history
    WHERE blocklist_schedule_id = p_blocklist_schedule_id
);

DELETE FROM workflow.blocklist_purge_history
WHERE blocklist_schedule_id = p_blocklist_schedule_id;

DELETE FROM workflow.blocklist_schedules
WHERE blocklist_schedule_id = p_blocklist_schedule_id;
```

---

## 13. Document Relationship

This document defines the summary/history parent table only.

The companion document defines:

1. The normalized child table for phone/email/address/name records.
2. Detail-row indexing and drilldown query patterns.
3. Parent-child write order.
4. Detail API contract.
=======
# Architecture: Blocklist Purge History Table

## 1. Problem Statement

The existing data purge flow, triggered when a new blocklist rule is added with `ScheduleDataPurge = true`, does not persist run-level summary data. Specifically, the current implementation does not store:

1. Persisted history of purge runs for API/UI consumption.
2. Per-run totals for records processed.
3. Per-run breakdown totals by record type: phone, email, address, and name.
4. Retention cleanup metrics from the background cleanup step.

This document defines the architecture for a new summary table, `workflow.blocklist_purge_history`, that stores one row per purge run. Detailed per-record data is intentionally moved out of this table into a separate normalized child table documented in the companion architecture document.

---

## 2. Design Principles

| # | Principle | Rationale |
|---|---|---|
| 1 | One summary row per `blocklist_schedule_id` | Matches the existing schedule/run model |
| 2 | Summary table stays narrow | Fast list queries, lower IO, simpler mapping |
| 3 | Detailed records are normalized into a child table | Better query performance than large JSON payload columns |
| 4 | Summary writes are incremental and idempotent | State machine retries must not create duplicates |
| 5 | Parent and child are both owned by workflow-resource | Keeps writes near the execution flow |
| 6 | Core purge flow must not fail because history persistence fails | History is additive observability, not execution control |

---

## 3. New Summary Table: `workflow.blocklist_purge_history`

### 3.1 DDL

```sql
CREATE TABLE IF NOT EXISTS workflow.blocklist_purge_history (
    -- Identity
    blocklist_purge_history_id  UUID        NOT NULL DEFAULT gen_random_uuid(),
    blocklist_schedule_id       UUID        NOT NULL,

    -- Organization context
    coorg_id                    VARCHAR(64) NOT NULL DEFAULT '',

    -- Run lifecycle
    run_status                  VARCHAR(32) NOT NULL DEFAULT 'Scheduled',
    requested_by                VARCHAR(32) NOT NULL DEFAULT '',
    started_utc                 TIMESTAMP   NULL,
    completed_utc               TIMESTAMP   NULL,
    time_taken_seconds          INT         NULL,

    -- Summary metrics for rule-driven processing
    total_items_processed       INT         NOT NULL DEFAULT 0,
    phone_records_count         INT         NOT NULL DEFAULT 0,
    email_records_count         INT         NOT NULL DEFAULT 0,
    address_records_count       INT         NOT NULL DEFAULT 0,
    name_records_count          INT         NOT NULL DEFAULT 0,
    total_business_results      INT         NOT NULL DEFAULT 0,
    items_failed                INT         NOT NULL DEFAULT 0,

    -- Retention cleanup metrics
    retention_items_deleted     INT         NULL,
    retention_business_deleted  INT         NULL,
    retention_days_applied      INT         NULL,

    -- Metadata
    is_estimated                BOOLEAN     NOT NULL DEFAULT FALSE,
    error_message               TEXT        NULL,
    created_utc                 TIMESTAMP   NOT NULL DEFAULT timezone('utc'::text, now()),
    updated_utc                 TIMESTAMP   NOT NULL DEFAULT timezone('utc'::text, now()),

    CONSTRAINT blocklist_purge_history_pkey PRIMARY KEY (blocklist_purge_history_id),
    CONSTRAINT blocklist_purge_history_schedule_uq UNIQUE (blocklist_schedule_id),
    CONSTRAINT blocklist_purge_history_schedule_fk
        FOREIGN KEY (blocklist_schedule_id)
        REFERENCES workflow.blocklist_schedules(blocklist_schedule_id)
);
```

### 3.2 Indexes

```sql
CREATE INDEX IF NOT EXISTS idx_bph_coorg_id
    ON workflow.blocklist_purge_history (coorg_id);

CREATE INDEX IF NOT EXISTS idx_bph_run_status
    ON workflow.blocklist_purge_history (run_status);

CREATE INDEX IF NOT EXISTS idx_bph_started_utc
    ON workflow.blocklist_purge_history (started_utc DESC NULLS LAST);

CREATE INDEX IF NOT EXISTS idx_bph_completed_utc
    ON workflow.blocklist_purge_history (completed_utc DESC NULLS LAST);

CREATE INDEX IF NOT EXISTS idx_bph_coorg_status_started
    ON workflow.blocklist_purge_history (coorg_id, run_status, started_utc DESC NULLS LAST);
```

### 3.3 Column Reference

| Column | Type | Source | Written At |
|---|---|---|---|
| `blocklist_purge_history_id` | uuid | Auto-generated | INSERT |
| `blocklist_schedule_id` | uuid | `blocklist_schedules.blocklist_schedule_id` | INSERT |
| `coorg_id` | varchar(64) | `blocklist_schedules.coorg_id` | INSERT |
| `run_status` | varchar(32) | Mirrors `blocklist_schedules.schedule_status` | Each status transition |
| `requested_by` | varchar(32) | `blocklist_schedules.requested_by` | INSERT |
| `started_utc` | timestamp | First move to `Requested` | UPDATE at Requested |
| `completed_utc` | timestamp | Terminal transition time | UPDATE at Completed/Cancelled/Failed |
| `time_taken_seconds` | int | `completed_utc - started_utc` | UPDATE at terminal |
| `total_items_processed` | int | Count of detail rows for the run | UPDATE at Completed |
| `phone_records_count` | int | Count of detail rows where `record_type = 'phone'` | UPDATE at Completed |
| `email_records_count` | int | Count of detail rows where `record_type = 'email'` | UPDATE at Completed |
| `address_records_count` | int | Count of detail rows where `record_type = 'address'` | UPDATE at Completed |
| `name_records_count` | int | Count of detail rows where `record_type = 'name'` | UPDATE at Completed |
| `total_business_results` | int | Count from `blocklist_business_results` for the run | UPDATE at Completed |
| `items_failed` | int | Count of detail rows where `item_status = 'Failed'` | UPDATE at Completed |
| `retention_items_deleted` | int | Optional cleanup metric | UPDATE during cleanup enhancement |
| `retention_business_deleted` | int | Optional cleanup metric | UPDATE during cleanup enhancement |
| `retention_days_applied` | int | `BlocklistItemsDeletionIntervalDays` | UPDATE during cleanup enhancement |
| `is_estimated` | boolean | `true` for legacy/backfilled rows | INSERT |
| `error_message` | text | Exception summary if history write fails | UPDATE on error |
| `created_utc` | timestamp | Auto-generated | INSERT |
| `updated_utc` | timestamp | Auto-updated | Every UPDATE |

---

## 4. Write Strategy

The summary row is written at specific points in the existing state machine. Detailed records are inserted into the child table first, then summary counts are computed from that child table for consistency.

### 4.1 Lifecycle Write Map

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Existing State Machine Steps                     │
├─────────────────────────┬────────────────────────────────────────────┤
│ Step                    │ Summary Table Action                       │
├─────────────────────────┼────────────────────────────────────────────┤
│ Schedule Created        │ INSERT row with Scheduled status           │
│ Requested               │ UPDATE run_status + started_utc            │
│ Completed               │ UPDATE run_status + completed_utc +        │
│                         │ time_taken_seconds + summary counts        │
│ Cancelled/Failed        │ UPDATE run_status + completed_utc          │
│ Cleanup enhancement     │ UPDATE retention metrics                    │
└─────────────────────────┴────────────────────────────────────────────┘
```

### 4.2 Completion Aggregation Strategy

At `BlocklistScheduleCompleted`:

1. Insert detailed rows into the child table from `workflow.blocklist_item_results`.
2. Aggregate counts from the child table.
3. Update the parent summary row with the aggregated counts.

Summary aggregation query:

```sql
SELECT
    COUNT(*) AS total_items_processed,
    COUNT(*) FILTER (WHERE record_type = 'phone') AS phone_records_count,
    COUNT(*) FILTER (WHERE record_type = 'email') AS email_records_count,
    COUNT(*) FILTER (WHERE record_type = 'address') AS address_records_count,
    COUNT(*) FILTER (WHERE record_type = 'name') AS name_records_count,
    COUNT(*) FILTER (WHERE item_status = 'Failed') AS items_failed
FROM workflow.blocklist_purge_history_details
WHERE blocklist_purge_history_id = @blocklist_purge_history_id;
```

Business results count:

```sql
SELECT COUNT(*) AS total_business_results
FROM workflow.blocklist_business_results
WHERE blocklist_schedule_id = @blocklist_schedule_id;
```

### 4.3 Upsert Pattern

Create summary row:

```sql
INSERT INTO workflow.blocklist_purge_history (
    blocklist_schedule_id,
    coorg_id,
    requested_by,
    run_status,
    created_utc,
    updated_utc
)
VALUES (
    @blocklist_schedule_id,
    @coorg_id,
    @requested_by,
    'Scheduled',
    timezone('utc'::text, now()),
    timezone('utc'::text, now())
)
ON CONFLICT (blocklist_schedule_id) DO NOTHING;
```

Update status:

```sql
UPDATE workflow.blocklist_purge_history
SET run_status = @new_status,
    started_utc = CASE WHEN @new_status = 'Requested' AND started_utc IS NULL
        THEN timezone('utc'::text, now()) ELSE started_utc END,
    completed_utc = CASE WHEN @new_status IN ('Completed', 'Cancelled', 'Failed')
        THEN timezone('utc'::text, now()) ELSE completed_utc END,
    time_taken_seconds = CASE
        WHEN @new_status IN ('Completed', 'Cancelled', 'Failed') AND started_utc IS NOT NULL
        THEN EXTRACT(EPOCH FROM (timezone('utc'::text, now()) - started_utc))::INT
        ELSE time_taken_seconds
    END,
    updated_utc = timezone('utc'::text, now())
WHERE blocklist_schedule_id = @blocklist_schedule_id;
```

Update summary counts:

```sql
UPDATE workflow.blocklist_purge_history
SET total_items_processed = @total_items_processed,
    phone_records_count = @phone_records_count,
    email_records_count = @email_records_count,
    address_records_count = @address_records_count,
    name_records_count = @name_records_count,
    total_business_results = @total_business_results,
    items_failed = @items_failed,
    updated_utc = timezone('utc'::text, now())
WHERE blocklist_purge_history_id = @blocklist_purge_history_id;
```

---

## 5. Integration Points in Existing Code

No code changes are made now; this section identifies future implementation touchpoints only.

| File | Planned Change |
|---|---|
| `BlocklistScheduleResourceAccess.cs` | Add summary history create/update methods |
| `IBlocklistScheduleResourceAccess.cs` | Add summary history interface methods |
| `BlocklistScheduleRepository.cs` | Add Dapper SQL for summary history |
| `BlocklistScheduleRequested.cs` | Write Requested transition to summary row |
| `BlocklistScheduleCompleted.cs` | Trigger detail-row insert + summary aggregation |
| `purge_data_by_blocklist_schedule_id.sql` | Delete child rows, then summary row, then schedule row |

### 5.1 Future Entity Class

```csharp
public class blocklist_purge_history
{
    public Guid blocklist_purge_history_id { get; set; }
    public Guid blocklist_schedule_id { get; set; }
    public string coorg_id { get; set; }
    public string run_status { get; set; }
    public string requested_by { get; set; }
    public DateTime? started_utc { get; set; }
    public DateTime? completed_utc { get; set; }
    public int? time_taken_seconds { get; set; }
    public int total_items_processed { get; set; }
    public int phone_records_count { get; set; }
    public int email_records_count { get; set; }
    public int address_records_count { get; set; }
    public int name_records_count { get; set; }
    public int total_business_results { get; set; }
    public int items_failed { get; set; }
    public int? retention_items_deleted { get; set; }
    public int? retention_business_deleted { get; set; }
    public int? retention_days_applied { get; set; }
    public bool is_estimated { get; set; }
    public string error_message { get; set; }
    public DateTime created_utc { get; set; }
    public DateTime updated_utc { get; set; }
}
```

### 5.2 Future Liquibase Changelog

Create:

`common-db/postgres/Databases/store/Schemas/workflow/Tables/16.BlocklistPurgeHistory/changelog-1.xml`

The changelog should mirror the DDL in §3.1.

---

## 6. API Design

### 6.1 List Purge History

| Property | Value |
|---|---|
| Method | `GET` |
| Path | `/blocklist-schedule/purge-history` |
| Service | workflow-resource |
| Auth | Existing `Scope` policy |

Response item:

```json
{
  "blocklistScheduleId": "uuid",
  "coOrgDealerId": "string",
  "runStatus": "Completed",
  "requestedBy": "user@company.com",
  "startedUtc": "2026-03-19T10:00:00Z",
  "completedUtc": "2026-03-19T10:06:30Z",
  "timeTakenSeconds": 390,
  "totalItemsProcessed": 1200,
  "phoneRecordsCount": 400,
  "emailRecordsCount": 350,
  "addressRecordsCount": 300,
  "nameRecordsCount": 150,
  "totalBusinessResults": 340,
  "itemsFailed": 5,
  "retentionItemsDeleted": null,
  "retentionBusinessDeleted": null,
  "retentionDaysApplied": null,
  "isEstimated": false
}
```

### 6.2 Detail Strategy

The summary endpoint remains lightweight. Full per-record drilldown is served from the child-table design documented in `data purge history detail table architecture.md`.

---

## 7. Concurrency and Safety

| Concern | Decision |
|---|---|
| Distributed locking | Reuse existing `lease_expires_utc` scheduling model |
| Parent duplicate writes | Prevent with `UNIQUE (blocklist_schedule_id)` + `ON CONFLICT` |
| Count consistency | Aggregate from child detail rows after insert |
| Failure isolation | History write failures are logged and swallowed |

---

## 8. Failure Handling

| Scenario | Behavior |
|---|---|
| Summary row insert fails | Log and continue purge execution |
| Count aggregation fails | Log, set `error_message`, keep run status update |
| Detail row insert partially fails | Treat as detail persistence failure, do not block purge flow |
| Cleanup metrics unavailable | Leave retention fields null |

---

## 9. Migration and Legacy Data

### 9.1 Schema Rollout

Deploy the parent table first, then the child table, then the read APIs.

### 9.2 Legacy Backfill

Optional backfill can create summary rows for completed schedules with `is_estimated = true` and zeroed counts when source rows are already gone.

---

## 10. Testing Strategy

| Test | Validates |
|---|---|
| Summary row upsert is idempotent | One row per schedule |
| Summary count update works after child inserts | Parent metrics stay consistent |
| List endpoint returns narrow payload | No heavy detail payload in summary response |
| Purge flow still succeeds if history write fails | History is non-blocking |

---

## 11. Rollout Plan

| Phase | Scope |
|---|---|
| 1 | Deploy `blocklist_purge_history` table |
| 2 | Deploy child detail table from companion design |
| 3 | Wire write path in workflow-resource |
| 4 | Expose summary endpoint |
| 5 | Expose drilldown endpoint from child table |

---

## 12. Required Change to Existing Purge Delete Function

Because `blocklist_purge_history` references `blocklist_schedules`, the schedule purge function must delete history rows before deleting the schedule row. If the child table is also implemented, delete child rows first, then parent summary, then schedule.

Planned deletion order:

```sql
DELETE FROM workflow.blocklist_purge_history_details
WHERE blocklist_purge_history_id IN (
    SELECT blocklist_purge_history_id
    FROM workflow.blocklist_purge_history
    WHERE blocklist_schedule_id = p_blocklist_schedule_id
);

DELETE FROM workflow.blocklist_purge_history
WHERE blocklist_schedule_id = p_blocklist_schedule_id;

DELETE FROM workflow.blocklist_schedules
WHERE blocklist_schedule_id = p_blocklist_schedule_id;
```

---

## 13. Document Relationship

This document defines the summary/history parent table only.

The companion document defines:

1. The normalized child table for phone/email/address/name records.
2. Detail-row indexing and drilldown query patterns.
3. Parent-child write order.
4. Detail API contract.
>>>>>>> f5e283b04bc9ba168be46150be990bc62dd47697
