<<<<<<< HEAD
# Architecture: Blocklist Purge History Detail Table

## 1. Purpose

This document defines the normalized child table that stores the detailed record-level data for each data purge run. It is designed to replace the previous JSONB-in-parent-table approach for phone, email, address, and name records.

The parent summary table remains `workflow.blocklist_purge_history`. This child table stores one row per processed item linked to a parent purge history row.

---

## 2. Why a Separate Detail Table

The previous JSONB design had two performance concerns:

1. Large runs would create very large JSONB columns in the parent row.
2. Reading and deserializing those JSONB blobs would add CPU and memory overhead in the application.

A normalized child table avoids both issues:

| Benefit | Explanation |
|---|---|
| Faster point lookups | Query only the detail rows needed for a run/type/page |
| Better pagination | Page through detail rows directly in SQL |
| Better filtering | Filter by `record_type`, `value`, `item_status`, `common_consumer_id` |
| Simpler DTO mapping | One row maps to one response item; no JSON blob parsing |
| Better indexing | Index exactly the access paths the UI/API needs |

---

## 3. New Child Table: `workflow.blocklist_purge_history_details`

### 3.1 DDL

```sql
CREATE TABLE IF NOT EXISTS workflow.blocklist_purge_history_details (
    blocklist_purge_history_detail_id UUID        NOT NULL DEFAULT gen_random_uuid(),
    blocklist_purge_history_id        UUID        NOT NULL,
    blocklist_schedule_id             UUID        NOT NULL,
    coorg_id                          VARCHAR(64) NOT NULL DEFAULT '',
    record_type                       VARCHAR(32) NOT NULL,
    common_consumer_id                VARCHAR(64) NOT NULL DEFAULT '',
    record_value                      TEXT        NULL,
    item_status                       VARCHAR(32) NOT NULL DEFAULT 'Unknown',
    result_payload                    JSONB       NULL,
    source_page_number                INT         NULL,
    page_number                       INT         NULL,
    created_utc                       TIMESTAMP   NOT NULL DEFAULT timezone('utc'::text, now()),
    updated_utc                       TIMESTAMP   NOT NULL DEFAULT timezone('utc'::text, now()),

    CONSTRAINT blocklist_purge_history_details_pkey
        PRIMARY KEY (blocklist_purge_history_detail_id),
    CONSTRAINT blocklist_purge_history_details_parent_fk
        FOREIGN KEY (blocklist_purge_history_id)
        REFERENCES workflow.blocklist_purge_history(blocklist_purge_history_id),
    CONSTRAINT blocklist_purge_history_details_type_chk
        CHECK (record_type IN ('phone', 'email', 'address', 'name'))
);
```

### 3.2 Indexes

```sql
CREATE INDEX IF NOT EXISTS idx_bphd_parent_id
    ON workflow.blocklist_purge_history_details (blocklist_purge_history_id);

CREATE INDEX IF NOT EXISTS idx_bphd_schedule_id
    ON workflow.blocklist_purge_history_details (blocklist_schedule_id);

CREATE INDEX IF NOT EXISTS idx_bphd_type_parent
    ON workflow.blocklist_purge_history_details (record_type, blocklist_purge_history_id);

CREATE INDEX IF NOT EXISTS idx_bphd_parent_status
    ON workflow.blocklist_purge_history_details (blocklist_purge_history_id, item_status);

CREATE INDEX IF NOT EXISTS idx_bphd_parent_value
    ON workflow.blocklist_purge_history_details (blocklist_purge_history_id, record_value);

CREATE INDEX IF NOT EXISTS idx_bphd_parent_consumer
    ON workflow.blocklist_purge_history_details (blocklist_purge_history_id, common_consumer_id);

CREATE INDEX IF NOT EXISTS idx_bphd_parent_page
    ON workflow.blocklist_purge_history_details (blocklist_purge_history_id, source_page_number, page_number);
```

### 3.3 Column Reference

| Column | Type | Source | Notes |
|---|---|---|---|
| `blocklist_purge_history_detail_id` | uuid | Auto-generated | PK |
| `blocklist_purge_history_id` | uuid | Parent summary row | FK to `blocklist_purge_history` |
| `blocklist_schedule_id` | uuid | `blocklist_schedules.blocklist_schedule_id` | Denormalized for query convenience |
| `coorg_id` | varchar(64) | `blocklist_schedules.coorg_id` | Denormalized for query convenience |
| `record_type` | varchar(32) | `payload->>'type'` | One of phone/email/address/name |
| `common_consumer_id` | varchar(64) | `blocklist_item_results.common_consumer_id` | Consumer reference |
| `record_value` | text | `payload->>'value'` | Actual phone/email/address/name value |
| `item_status` | varchar(32) | `blocklist_item_results.item_status` | For filtering failed/processed |
| `result_payload` | jsonb | `blocklist_item_results.result_payload` | Existing result payload can stay JSONB |
| `source_page_number` | int | `blocklist_item_results.source_page_number` | Optional drilldown |
| `page_number` | int | `blocklist_item_results.page_number` | Optional drilldown |
| `created_utc` | timestamp | Copy from insert time | Audit |
| `updated_utc` | timestamp | Copy from insert time | Audit |

---

## 4. Data Flow and Write Strategy

### 4.1 Insert Timing

Detail rows are created when the schedule reaches `Completed`, before retention cleanup can remove source rows.

Write order:

1. Ensure parent row exists in `workflow.blocklist_purge_history`.
2. Read all matching rows from `workflow.blocklist_item_results` for the schedule.
3. Insert one child row per source item into `workflow.blocklist_purge_history_details`.
4. Aggregate counts from the child table.
5. Update the parent summary row with counts.

### 4.2 Insert Query

```sql
INSERT INTO workflow.blocklist_purge_history_details (
    blocklist_purge_history_id,
    blocklist_schedule_id,
    coorg_id,
    record_type,
    common_consumer_id,
    record_value,
    item_status,
    result_payload,
    source_page_number,
    page_number,
    created_utc,
    updated_utc
)
SELECT
    @blocklist_purge_history_id,
    bir.blocklist_schedule_id,
    @coorg_id,
    bir.payload->>'type',
    bir.common_consumer_id,
    bir.payload->>'value',
    bir.item_status,
    bir.result_payload,
    bir.source_page_number,
    bir.page_number,
    timezone('utc'::text, now()),
    timezone('utc'::text, now())
FROM workflow.blocklist_item_results bir
WHERE bir.blocklist_schedule_id = @blocklist_schedule_id;
```

### 4.3 Idempotency Requirement

Because state machine steps may retry, the child insert path must be idempotent. Recommended approach:

Add a uniqueness constraint:

```sql
ALTER TABLE workflow.blocklist_purge_history_details
ADD CONSTRAINT blocklist_purge_history_details_unique_item
UNIQUE (blocklist_purge_history_id, common_consumer_id, record_type, record_value);
```

Then insert with conflict protection:

```sql
ON CONFLICT (blocklist_purge_history_id, common_consumer_id, record_type, record_value)
DO NOTHING;
```

This assumes the tuple `(history, consumer, type, value)` is stable enough for a purge run. If the source system allows true duplicates for that tuple, use a source identifier column instead.

---

## 5. API Design for Detail Retrieval

### 5.1 List Detail Rows for a Run

| Property | Value |
|---|---|
| Method | `GET` |
| Path | `/blocklist-schedule/purge-history/id/{blocklistScheduleId}/details` |
| Service | workflow-resource |
| Auth | Existing `Scope` policy |

Query parameters:

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `recordType` | string | null | phone/email/address/name |
| `itemStatus` | string | null | Processed/Failed/etc. |
| `searchValue` | string | null | Prefix or contains search on `record_value` |
| `pageNumber` | int | 1 | Min 1 |
| `pageSize` | int | 100 | Min 1, max 500 |
| `sortBy` | string | `recordValue` | `recordValue`, `itemStatus`, `sourcePageNumber` |
| `sortDirection` | string | `asc` | `asc` or `desc` |

Response:

```json
{
  "totalCount": 400,
  "pageNumber": 1,
  "pageSize": 100,
  "items": [
    {
      "blocklistPurgeHistoryDetailId": "uuid",
      "recordType": "phone",
      "commonConsumerId": "abc-123",
      "recordValue": "+15551234567",
      "itemStatus": "Processed",
      "resultPayload": {},
      "sourcePageNumber": 2,
      "pageNumber": 2
    }
  ]
}
```

### 5.2 Aggregate Counts by Type

The parent summary table already stores type counts, so the detail endpoint does not need to recompute counts unless filtering is applied.

---

## 6. Query Patterns

### 6.1 Common Queries

All phone records for a run:

```sql
SELECT *
FROM workflow.blocklist_purge_history_details
WHERE blocklist_purge_history_id = @blocklist_purge_history_id
  AND record_type = 'phone'
ORDER BY record_value ASC
LIMIT @limit OFFSET @offset;
```

Failed records for a run:

```sql
SELECT *
FROM workflow.blocklist_purge_history_details
WHERE blocklist_purge_history_id = @blocklist_purge_history_id
  AND item_status = 'Failed';
```

Search by value:

```sql
SELECT *
FROM workflow.blocklist_purge_history_details
WHERE blocklist_purge_history_id = @blocklist_purge_history_id
  AND record_type = @record_type
  AND record_value ILIKE @search_prefix || '%';
```

---

## 7. Integration Points in Existing Code

Future implementation touchpoints:

| File | Planned Change |
|---|---|
| `BlocklistScheduleRepository.cs` | Add parent lookup + child insert SQL |
| `BlocklistScheduleCompleted.cs` | Insert detail rows before parent aggregation |
| `BlocklistPurgeHistoryController.cs` | Add detail endpoint |
| `BlocklistPurgeHistoryMapper.cs` | Map child rows to response DTO |
| `purge_data_by_blocklist_schedule_id.sql` | Delete child rows first |

### 7.1 Future Entity Class

```csharp
public class blocklist_purge_history_detail
{
    public Guid blocklist_purge_history_detail_id { get; set; }
    public Guid blocklist_purge_history_id { get; set; }
    public Guid blocklist_schedule_id { get; set; }
    public string coorg_id { get; set; }
    public string record_type { get; set; }
    public string common_consumer_id { get; set; }
    public string record_value { get; set; }
    public string item_status { get; set; }
    public string result_payload { get; set; }
    public int? source_page_number { get; set; }
    public int? page_number { get; set; }
    public DateTime created_utc { get; set; }
    public DateTime updated_utc { get; set; }
}
```

### 7.2 Future Liquibase Changelog

Create:

`common-db/postgres/Databases/store/Schemas/workflow/Tables/17.BlocklistPurgeHistoryDetails/changelog-1.xml`

The changelog should mirror the DDL in §3.1 plus the uniqueness constraint from §4.3.

---

## 8. Delete Behavior

When purging a schedule explicitly via `purge_data_by_blocklist_schedule_id`, delete in this order:

```sql
DELETE FROM workflow.blocklist_purge_history_details
WHERE blocklist_purge_history_id IN (
    SELECT blocklist_purge_history_id
    FROM workflow.blocklist_purge_history
    WHERE blocklist_schedule_id = p_blocklist_schedule_id
);

DELETE FROM workflow.blocklist_purge_history
WHERE blocklist_schedule_id = p_blocklist_schedule_id;
```

This avoids FK violations and keeps schedule cleanup deterministic.

---

## 9. Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Detail table grows large over time | Medium | Partition later if needed; start with indexes and retention policy review |
| Duplicate detail rows from retries | High | Add uniqueness constraint + `ON CONFLICT DO NOTHING` |
| Search on `record_value` becomes slow | Medium | Use targeted index, add trigram index only if needed later |
| Source rows are deleted before detail copy | High | Insert detail rows during `Completed` step before cleanup step runs |
| Result payload JSONB is still large | Low | Only one JSONB column remains, scoped to one detail row, not one whole run |

---

## 10. Testing Strategy

| Test | Validates |
|---|---|
| Child row insert copies source fields correctly | Mapping fidelity |
| Retry does not duplicate child rows | Uniqueness + conflict handling |
| Parent summary counts match child-table aggregates | Parent-child consistency |
| Detail endpoint pagination works | Scalable drilldown |
| Detail endpoint filters by type/status/value | Query correctness |

---

## 11. Rollout Plan

| Phase | Scope |
|---|---|
| 1 | Deploy parent summary table |
| 2 | Deploy child detail table |
| 3 | Wire child insert path at schedule completion |
| 4 | Wire parent aggregation from child table |
| 5 | Expose detail endpoint |

---

## 12. Relationship to Parent Document

This document defines the detailed child table only.

The parent document defines:

1. One row per purge run.
2. Run lifecycle timestamps and summary counts.
3. Lightweight list-query contract.
4. Parent-level retention metrics and error fields.
=======
# Architecture: Blocklist Purge History Detail Table

## 1. Purpose

This document defines the normalized child table that stores the detailed record-level data for each data purge run. It is designed to replace the previous JSONB-in-parent-table approach for phone, email, address, and name records.

The parent summary table remains `workflow.blocklist_purge_history`. This child table stores one row per processed item linked to a parent purge history row.

---

## 2. Why a Separate Detail Table

The previous JSONB design had two performance concerns:

1. Large runs would create very large JSONB columns in the parent row.
2. Reading and deserializing those JSONB blobs would add CPU and memory overhead in the application.

A normalized child table avoids both issues:

| Benefit | Explanation |
|---|---|
| Faster point lookups | Query only the detail rows needed for a run/type/page |
| Better pagination | Page through detail rows directly in SQL |
| Better filtering | Filter by `record_type`, `value`, `item_status`, `common_consumer_id` |
| Simpler DTO mapping | One row maps to one response item; no JSON blob parsing |
| Better indexing | Index exactly the access paths the UI/API needs |

---

## 3. New Child Table: `workflow.blocklist_purge_history_details`

### 3.1 DDL

```sql
CREATE TABLE IF NOT EXISTS workflow.blocklist_purge_history_details (
    blocklist_purge_history_detail_id UUID        NOT NULL DEFAULT gen_random_uuid(),
    blocklist_purge_history_id        UUID        NOT NULL,
    blocklist_schedule_id             UUID        NOT NULL,
    coorg_id                          VARCHAR(64) NOT NULL DEFAULT '',
    record_type                       VARCHAR(32) NOT NULL,
    common_consumer_id                VARCHAR(64) NOT NULL DEFAULT '',
    record_value                      TEXT        NULL,
    item_status                       VARCHAR(32) NOT NULL DEFAULT 'Unknown',
    result_payload                    JSONB       NULL,
    source_page_number                INT         NULL,
    page_number                       INT         NULL,
    created_utc                       TIMESTAMP   NOT NULL DEFAULT timezone('utc'::text, now()),
    updated_utc                       TIMESTAMP   NOT NULL DEFAULT timezone('utc'::text, now()),

    CONSTRAINT blocklist_purge_history_details_pkey
        PRIMARY KEY (blocklist_purge_history_detail_id),
    CONSTRAINT blocklist_purge_history_details_parent_fk
        FOREIGN KEY (blocklist_purge_history_id)
        REFERENCES workflow.blocklist_purge_history(blocklist_purge_history_id),
    CONSTRAINT blocklist_purge_history_details_type_chk
        CHECK (record_type IN ('phone', 'email', 'address', 'name'))
);
```

### 3.2 Indexes

```sql
CREATE INDEX IF NOT EXISTS idx_bphd_parent_id
    ON workflow.blocklist_purge_history_details (blocklist_purge_history_id);

CREATE INDEX IF NOT EXISTS idx_bphd_schedule_id
    ON workflow.blocklist_purge_history_details (blocklist_schedule_id);

CREATE INDEX IF NOT EXISTS idx_bphd_type_parent
    ON workflow.blocklist_purge_history_details (record_type, blocklist_purge_history_id);

CREATE INDEX IF NOT EXISTS idx_bphd_parent_status
    ON workflow.blocklist_purge_history_details (blocklist_purge_history_id, item_status);

CREATE INDEX IF NOT EXISTS idx_bphd_parent_value
    ON workflow.blocklist_purge_history_details (blocklist_purge_history_id, record_value);

CREATE INDEX IF NOT EXISTS idx_bphd_parent_consumer
    ON workflow.blocklist_purge_history_details (blocklist_purge_history_id, common_consumer_id);

CREATE INDEX IF NOT EXISTS idx_bphd_parent_page
    ON workflow.blocklist_purge_history_details (blocklist_purge_history_id, source_page_number, page_number);
```

### 3.3 Column Reference

| Column | Type | Source | Notes |
|---|---|---|---|
| `blocklist_purge_history_detail_id` | uuid | Auto-generated | PK |
| `blocklist_purge_history_id` | uuid | Parent summary row | FK to `blocklist_purge_history` |
| `blocklist_schedule_id` | uuid | `blocklist_schedules.blocklist_schedule_id` | Denormalized for query convenience |
| `coorg_id` | varchar(64) | `blocklist_schedules.coorg_id` | Denormalized for query convenience |
| `record_type` | varchar(32) | `payload->>'type'` | One of phone/email/address/name |
| `common_consumer_id` | varchar(64) | `blocklist_item_results.common_consumer_id` | Consumer reference |
| `record_value` | text | `payload->>'value'` | Actual phone/email/address/name value |
| `item_status` | varchar(32) | `blocklist_item_results.item_status` | For filtering failed/processed |
| `result_payload` | jsonb | `blocklist_item_results.result_payload` | Existing result payload can stay JSONB |
| `source_page_number` | int | `blocklist_item_results.source_page_number` | Optional drilldown |
| `page_number` | int | `blocklist_item_results.page_number` | Optional drilldown |
| `created_utc` | timestamp | Copy from insert time | Audit |
| `updated_utc` | timestamp | Copy from insert time | Audit |

---

## 4. Data Flow and Write Strategy

### 4.1 Insert Timing

Detail rows are created when the schedule reaches `Completed`, before retention cleanup can remove source rows.

Write order:

1. Ensure parent row exists in `workflow.blocklist_purge_history`.
2. Read all matching rows from `workflow.blocklist_item_results` for the schedule.
3. Insert one child row per source item into `workflow.blocklist_purge_history_details`.
4. Aggregate counts from the child table.
5. Update the parent summary row with counts.

### 4.2 Insert Query

```sql
INSERT INTO workflow.blocklist_purge_history_details (
    blocklist_purge_history_id,
    blocklist_schedule_id,
    coorg_id,
    record_type,
    common_consumer_id,
    record_value,
    item_status,
    result_payload,
    source_page_number,
    page_number,
    created_utc,
    updated_utc
)
SELECT
    @blocklist_purge_history_id,
    bir.blocklist_schedule_id,
    @coorg_id,
    bir.payload->>'type',
    bir.common_consumer_id,
    bir.payload->>'value',
    bir.item_status,
    bir.result_payload,
    bir.source_page_number,
    bir.page_number,
    timezone('utc'::text, now()),
    timezone('utc'::text, now())
FROM workflow.blocklist_item_results bir
WHERE bir.blocklist_schedule_id = @blocklist_schedule_id;
```

### 4.3 Idempotency Requirement

Because state machine steps may retry, the child insert path must be idempotent. Recommended approach:

Add a uniqueness constraint:

```sql
ALTER TABLE workflow.blocklist_purge_history_details
ADD CONSTRAINT blocklist_purge_history_details_unique_item
UNIQUE (blocklist_purge_history_id, common_consumer_id, record_type, record_value);
```

Then insert with conflict protection:

```sql
ON CONFLICT (blocklist_purge_history_id, common_consumer_id, record_type, record_value)
DO NOTHING;
```

This assumes the tuple `(history, consumer, type, value)` is stable enough for a purge run. If the source system allows true duplicates for that tuple, use a source identifier column instead.

---

## 5. API Design for Detail Retrieval

### 5.1 List Detail Rows for a Run

| Property | Value |
|---|---|
| Method | `GET` |
| Path | `/blocklist-schedule/purge-history/id/{blocklistScheduleId}/details` |
| Service | workflow-resource |
| Auth | Existing `Scope` policy |

Query parameters:

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `recordType` | string | null | phone/email/address/name |
| `itemStatus` | string | null | Processed/Failed/etc. |
| `searchValue` | string | null | Prefix or contains search on `record_value` |
| `pageNumber` | int | 1 | Min 1 |
| `pageSize` | int | 100 | Min 1, max 500 |
| `sortBy` | string | `recordValue` | `recordValue`, `itemStatus`, `sourcePageNumber` |
| `sortDirection` | string | `asc` | `asc` or `desc` |

Response:

```json
{
  "totalCount": 400,
  "pageNumber": 1,
  "pageSize": 100,
  "items": [
    {
      "blocklistPurgeHistoryDetailId": "uuid",
      "recordType": "phone",
      "commonConsumerId": "abc-123",
      "recordValue": "+15551234567",
      "itemStatus": "Processed",
      "resultPayload": {},
      "sourcePageNumber": 2,
      "pageNumber": 2
    }
  ]
}
```

### 5.2 Aggregate Counts by Type

The parent summary table already stores type counts, so the detail endpoint does not need to recompute counts unless filtering is applied.

---

## 6. Query Patterns

### 6.1 Common Queries

All phone records for a run:

```sql
SELECT *
FROM workflow.blocklist_purge_history_details
WHERE blocklist_purge_history_id = @blocklist_purge_history_id
  AND record_type = 'phone'
ORDER BY record_value ASC
LIMIT @limit OFFSET @offset;
```

Failed records for a run:

```sql
SELECT *
FROM workflow.blocklist_purge_history_details
WHERE blocklist_purge_history_id = @blocklist_purge_history_id
  AND item_status = 'Failed';
```

Search by value:

```sql
SELECT *
FROM workflow.blocklist_purge_history_details
WHERE blocklist_purge_history_id = @blocklist_purge_history_id
  AND record_type = @record_type
  AND record_value ILIKE @search_prefix || '%';
```

---

## 7. Integration Points in Existing Code

Future implementation touchpoints:

| File | Planned Change |
|---|---|
| `BlocklistScheduleRepository.cs` | Add parent lookup + child insert SQL |
| `BlocklistScheduleCompleted.cs` | Insert detail rows before parent aggregation |
| `BlocklistPurgeHistoryController.cs` | Add detail endpoint |
| `BlocklistPurgeHistoryMapper.cs` | Map child rows to response DTO |
| `purge_data_by_blocklist_schedule_id.sql` | Delete child rows first |

### 7.1 Future Entity Class

```csharp
public class blocklist_purge_history_detail
{
    public Guid blocklist_purge_history_detail_id { get; set; }
    public Guid blocklist_purge_history_id { get; set; }
    public Guid blocklist_schedule_id { get; set; }
    public string coorg_id { get; set; }
    public string record_type { get; set; }
    public string common_consumer_id { get; set; }
    public string record_value { get; set; }
    public string item_status { get; set; }
    public string result_payload { get; set; }
    public int? source_page_number { get; set; }
    public int? page_number { get; set; }
    public DateTime created_utc { get; set; }
    public DateTime updated_utc { get; set; }
}
```

### 7.2 Future Liquibase Changelog

Create:

`common-db/postgres/Databases/store/Schemas/workflow/Tables/17.BlocklistPurgeHistoryDetails/changelog-1.xml`

The changelog should mirror the DDL in §3.1 plus the uniqueness constraint from §4.3.

---

## 8. Delete Behavior

When purging a schedule explicitly via `purge_data_by_blocklist_schedule_id`, delete in this order:

```sql
DELETE FROM workflow.blocklist_purge_history_details
WHERE blocklist_purge_history_id IN (
    SELECT blocklist_purge_history_id
    FROM workflow.blocklist_purge_history
    WHERE blocklist_schedule_id = p_blocklist_schedule_id
);

DELETE FROM workflow.blocklist_purge_history
WHERE blocklist_schedule_id = p_blocklist_schedule_id;
```

This avoids FK violations and keeps schedule cleanup deterministic.

---

## 9. Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Detail table grows large over time | Medium | Partition later if needed; start with indexes and retention policy review |
| Duplicate detail rows from retries | High | Add uniqueness constraint + `ON CONFLICT DO NOTHING` |
| Search on `record_value` becomes slow | Medium | Use targeted index, add trigram index only if needed later |
| Source rows are deleted before detail copy | High | Insert detail rows during `Completed` step before cleanup step runs |
| Result payload JSONB is still large | Low | Only one JSONB column remains, scoped to one detail row, not one whole run |

---

## 10. Testing Strategy

| Test | Validates |
|---|---|
| Child row insert copies source fields correctly | Mapping fidelity |
| Retry does not duplicate child rows | Uniqueness + conflict handling |
| Parent summary counts match child-table aggregates | Parent-child consistency |
| Detail endpoint pagination works | Scalable drilldown |
| Detail endpoint filters by type/status/value | Query correctness |

---

## 11. Rollout Plan

| Phase | Scope |
|---|---|
| 1 | Deploy parent summary table |
| 2 | Deploy child detail table |
| 3 | Wire child insert path at schedule completion |
| 4 | Wire parent aggregation from child table |
| 5 | Expose detail endpoint |

---

## 12. Relationship to Parent Document

This document defines the detailed child table only.

The parent document defines:

1. One row per purge run.
2. Run lifecycle timestamps and summary counts.
3. Lightweight list-query contract.
4. Parent-level retention metrics and error fields.
>>>>>>> f5e283b04bc9ba168be46150be990bc62dd47697
