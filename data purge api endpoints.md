<<<<<<< HEAD
# Data Purge Tab - API Endpoints Specification

## Overview

This document specifies the API endpoints required to support the Data Purge UI using the normalized purge-history design.

The endpoints support three capabilities:
1. **Manual purge trigger** — request execution of rule-driven blocklist data purge.
2. **Purge history summary retrieval** — fetch lightweight run-level rows from `workflow.blocklist_purge_history`.
3. **Purge history detail retrieval** — fetch paged record-level drilldown rows from `workflow.blocklist_purge_history_details`.

This specification intentionally removes the old JSON payload design. There is no `detailsPayload` field in the summary response.

---

## Service Boundary

Recommended service exposure:

| Layer | Purpose | Endpoints |
|---|---|---|
| workflow-resource | Source-of-truth read/write APIs | Summary + detail endpoints |
| configuration-service | Passthrough for internal consumers | Summary + detail passthrough |
| consumer-card BFF | UI-facing API surface | Summary + detail endpoints under `/api/blocklist/...` |

This document describes the UI-facing BFF contract first and then the source query model behind it.

---

## Endpoint 1: Manual Purge Trigger

### Request
- **Method:** `POST`
- **Route:** `/api/blocklist/purge`
- **Headers:**
  - `Authorization: Bearer {token}`
  - `Content-Type: application/json`

### Request Body
```json
{
  "commonOrgId": "string",
  "solutionId": "string",
  "solutionDealerId": "string"
}
```

### Response — `202 Accepted`
```json
{
  "scheduled": true,
  "publishId": "string",
  "message": "string"
}
```

### Response — `409 Conflict`
```json
{
  "scheduled": false,
  "message": "A data purge is already scheduled for this organization"
}
```

### Business Rules
- Check for existing active schedule for the org.
- If active schedule exists, return `409 Conflict`.
- If not, publish the existing schedule event and return `202 Accepted`.
- This endpoint remains unchanged by the history-table redesign.

---

## Endpoint 2: Purge History Summary

### Purpose

Returns a paged, lightweight list of purge runs. This endpoint reads only from `workflow.blocklist_purge_history` and does not load the detail table.

### Request
- **Method:** `GET`
- **Route:** `/api/blocklist/purge-history`
- **Headers:**
  - `Authorization: Bearer {token}`

### Query Parameters

| Parameter | Type | Required | Default | Notes |
|---|---|---|---|---|
| `commonOrgId` | string | Yes | — | Org identifier |
| `solutionId` | string | No | null | Solution context |
| `solutionDealerId` | string | No | null | Dealer context |
| `runStatus` | string | No | null | Filter on summary `run_status` |
| `requestedBy` | string | No | null | Filter on initiator |
| `startDateFromUtc` | datetime | No | null | Inclusive lower bound on `started_utc` |
| `startDateToUtc` | datetime | No | null | Inclusive upper bound on `started_utc` |
| `pageNumber` | integer | No | 1 | Min 1 |
| `pageSize` | integer | No | 25 | Min 1, max 200 |
| `sortBy` | string | No | `startedUtc` | `startedUtc`, `completedUtc`, `totalItemsProcessed` |
| `sortDirection` | string | No | `desc` | `asc` or `desc` |

### Example Request
```http
GET /api/blocklist/purge-history?commonOrgId=org123&pageNumber=1&pageSize=25
Authorization: Bearer {token}
```

### Response — `200 OK`
```json
{
  "items": [
    {
      "blocklistScheduleId": "uuid",
      "coOrgDealerId": "org123",
      "runStatus": "Completed",
      "requestedBy": "user@company.com",
      "startedUtc": "2026-03-25T10:30:00Z",
      "completedUtc": "2026-03-25T10:45:00Z",
      "timeTakenSeconds": 900,
      "totalItemsProcessed": 1842,
      "phoneRecordsCount": 920,
      "emailRecordsCount": 411,
      "addressRecordsCount": 301,
      "nameRecordsCount": 210,
      "totalBusinessResults": 650,
      "itemsFailed": 12,
      "retentionItemsDeleted": null,
      "retentionBusinessDeleted": null,
      "retentionDaysApplied": null,
      "isEstimated": false
    }
  ],
  "pageNumber": 1,
  "pageSize": 25,
  "totalCount": 120
}
```

### Paging Semantics
- `pageNumber`: Current page, 1-indexed.
- `pageSize`: Number of items returned.
- `totalCount`: Total matching run-history rows for the organization/filter.

### Summary Response Field Definitions

| Field | Source | Notes |
|---|---|---|
| `blocklistScheduleId` | `workflow.blocklist_purge_history.blocklist_schedule_id` | Unique run/schedule identifier |
| `coOrgDealerId` | `coorg_id` | Organization identifier |
| `runStatus` | `run_status` | Mirrors lifecycle state |
| `requestedBy` | `requested_by` | Initiating user/system |
| `startedUtc` | `started_utc` | Null until Requested |
| `completedUtc` | `completed_utc` | Null for non-terminal runs |
| `timeTakenSeconds` | `time_taken_seconds` | Derived and persisted |
| `totalItemsProcessed` | `total_items_processed` | Count of detail rows for the run |
| `phoneRecordsCount` | `phone_records_count` | Count where detail `record_type = 'phone'` |
| `emailRecordsCount` | `email_records_count` | Count where detail `record_type = 'email'` |
| `addressRecordsCount` | `address_records_count` | Count where detail `record_type = 'address'` |
| `nameRecordsCount` | `name_records_count` | Count where detail `record_type = 'name'` |
| `totalBusinessResults` | `total_business_results` | Count from `blocklist_business_results` |
| `itemsFailed` | `items_failed` | Count where detail `item_status = 'Failed'` |
| `retentionItemsDeleted` | `retention_items_deleted` | Nullable until cleanup metric enhancement |
| `retentionBusinessDeleted` | `retention_business_deleted` | Nullable until cleanup metric enhancement |
| `retentionDaysApplied` | `retention_days_applied` | Nullable or configured value |
| `isEstimated` | `is_estimated` | `true` for legacy/backfilled rows |

### Business Rules
- Summary endpoint must not query `workflow.blocklist_purge_history_details` row-by-row during list rendering.
- Counts are read directly from parent summary columns.
- This endpoint is intended for grid/list use and must stay lightweight.

---

## Endpoint 3: Purge History Detail Drilldown

### Purpose

Returns paged record-level details for a single purge run from `workflow.blocklist_purge_history_details`.

### Request
- **Method:** `GET`
- **Route:** `/api/blocklist/purge-history/id/{blocklistScheduleId}/details`
- **Headers:**
  - `Authorization: Bearer {token}`

### Path Parameter

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `blocklistScheduleId` | uuid | Yes | Purge run / schedule identifier |

### Query Parameters

| Parameter | Type | Required | Default | Notes |
|---|---|---|---|---|
| `recordType` | string | No | null | `phone`, `email`, `address`, `name` |
| `itemStatus` | string | No | null | `Processed`, `Failed`, etc. |
| `searchValue` | string | No | null | Search on `recordValue` |
| `pageNumber` | integer | No | 1 | Min 1 |
| `pageSize` | integer | No | 100 | Min 1, max 500 |
| `sortBy` | string | No | `recordValue` | `recordValue`, `itemStatus`, `sourcePageNumber` |
| `sortDirection` | string | No | `asc` | `asc` or `desc` |

### Example Request
```http
GET /api/blocklist/purge-history/id/0d4f3d7a-1111-2222-3333-444455556666/details?recordType=phone&pageNumber=1&pageSize=100
Authorization: Bearer {token}
```

### Response — `200 OK`
```json
{
  "blocklistScheduleId": "0d4f3d7a-1111-2222-3333-444455556666",
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

### Response — `204 No Content`
- Returned when the `blocklistScheduleId` exists nowhere in purge history, or no matching detail data exists and the chosen service layer follows existing `NoContent` conventions.

### Detail Response Field Definitions

| Field | Source | Notes |
|---|---|---|
| `blocklistPurgeHistoryDetailId` | Child PK | Unique detail row identifier |
| `recordType` | `record_type` | One of phone/email/address/name |
| `commonConsumerId` | `common_consumer_id` | Consumer identifier |
| `recordValue` | `record_value` | Actual phone/email/address/name value |
| `itemStatus` | `item_status` | Processing status |
| `resultPayload` | `result_payload` | Existing JSON payload from source result row |
| `sourcePageNumber` | `source_page_number` | Optional source metadata |
| `pageNumber` | `page_number` | Optional source metadata |

### Business Rules
- Detail endpoint must filter and paginate directly in SQL.
- Detail endpoint must not load all rows in memory before paging.
- `recordType` filter is optional but expected for best UI performance.
- Summary counts shown in the main grid come from the parent table, not from recomputing this endpoint.

---

## Source Queries Behind the APIs

### Summary Query

```sql
SELECT
    blocklist_schedule_id,
    coorg_id,
    run_status,
    requested_by,
    started_utc,
    completed_utc,
    time_taken_seconds,
    total_items_processed,
    phone_records_count,
    email_records_count,
    address_records_count,
    name_records_count,
    total_business_results,
    items_failed,
    retention_items_deleted,
    retention_business_deleted,
    retention_days_applied,
    is_estimated
FROM workflow.blocklist_purge_history
WHERE coorg_id = @coorg_id
  AND (@run_status IS NULL OR run_status = @run_status)
  AND (@requested_by IS NULL OR requested_by = @requested_by)
  AND (@start_from IS NULL OR started_utc >= @start_from)
  AND (@start_to IS NULL OR started_utc <= @start_to)
ORDER BY started_utc DESC, blocklist_schedule_id DESC
LIMIT @page_size OFFSET ((@page_number - 1) * @page_size);
```

Count query:

```sql
SELECT COUNT(*)
FROM workflow.blocklist_purge_history
WHERE coorg_id = @coorg_id
  AND (@run_status IS NULL OR run_status = @run_status)
  AND (@requested_by IS NULL OR requested_by = @requested_by)
  AND (@start_from IS NULL OR started_utc >= @start_from)
  AND (@start_to IS NULL OR started_utc <= @start_to);
```

### Detail Query

```sql
SELECT
    blocklist_purge_history_detail_id,
    record_type,
    common_consumer_id,
    record_value,
    item_status,
    result_payload,
    source_page_number,
    page_number
FROM workflow.blocklist_purge_history_details
WHERE blocklist_schedule_id = @blocklist_schedule_id
  AND (@record_type IS NULL OR record_type = @record_type)
  AND (@item_status IS NULL OR item_status = @item_status)
  AND (@search_value IS NULL OR record_value ILIKE @search_value || '%')
ORDER BY record_value ASC, blocklist_purge_history_detail_id ASC
LIMIT @page_size OFFSET ((@page_number - 1) * @page_size);
```

Detail count query:

```sql
SELECT COUNT(*)
FROM workflow.blocklist_purge_history_details
WHERE blocklist_schedule_id = @blocklist_schedule_id
  AND (@record_type IS NULL OR record_type = @record_type)
  AND (@item_status IS NULL OR item_status = @item_status)
  AND (@search_value IS NULL OR record_value ILIKE @search_value || '%');
```

---

## Validation Rules

### Shared
- Reject `pageNumber < 1`.
- Reject `pageSize < 1`.
- Reject `pageSize > 200` for summary endpoint.
- Reject `pageSize > 500` for detail endpoint.

### Summary Endpoint
- Reject `startDateFromUtc > startDateToUtc`.
- Reject unsupported `sortBy` values.
- Reject unsupported `sortDirection` values.

### Detail Endpoint
- Reject unsupported `recordType` values.
- Reject unsupported `sortBy` values.
- Reject unsupported `sortDirection` values.

---

## Error Handling

| Status Code | When |
|---|---|
| `200 OK` | Request succeeds and returns rows |
| `202 Accepted` | Manual trigger accepted |
| `204 No Content` | No matching purge history/detail row for requested identifier |
| `400 Bad Request` | Validation failure |
| `401 Unauthorized` | Missing/invalid auth |
| `403 Forbidden` | Authenticated but not allowed |
| `409 Conflict` | Active purge schedule already exists |
| `500 Internal Server Error` | Unexpected server/storage error |

---

## Performance Expectations

| Endpoint | Performance Principle |
|---|---|
| Manual trigger | Reuse existing schedule/publish flow |
| Summary history | Query only parent summary table |
| Detail drilldown | Query only child detail table with paging and indexes |

This separation is the core reason for the normalized design: the history grid stays fast even when a purge run contains a very large number of detailed records.

---

## Out of Scope

- Returning all detail rows inline inside summary history response.
- Reconstructing detail payloads from deleted rows at read time.
- Changing existing blocklist processing logic as part of this API contract.
=======
# Data Purge Tab - API Endpoints Specification

## Overview

This document specifies the API endpoints required to support the Data Purge UI using the normalized purge-history design.

The endpoints support three capabilities:
1. **Manual purge trigger** — request execution of rule-driven blocklist data purge.
2. **Purge history summary retrieval** — fetch lightweight run-level rows from `workflow.blocklist_purge_history`.
3. **Purge history detail retrieval** — fetch paged record-level drilldown rows from `workflow.blocklist_purge_history_details`.

This specification intentionally removes the old JSON payload design. There is no `detailsPayload` field in the summary response.

---

## Service Boundary

Recommended service exposure:

| Layer | Purpose | Endpoints |
|---|---|---|
| workflow-resource | Source-of-truth read/write APIs | Summary + detail endpoints |
| configuration-service | Passthrough for internal consumers | Summary + detail passthrough |
| consumer-card BFF | UI-facing API surface | Summary + detail endpoints under `/api/blocklist/...` |

This document describes the UI-facing BFF contract first and then the source query model behind it.

---

## Endpoint 1: Manual Purge Trigger

### Request
- **Method:** `POST`
- **Route:** `/api/blocklist/purge`
- **Headers:**
  - `Authorization: Bearer {token}`
  - `Content-Type: application/json`

### Request Body
```json
{
  "commonOrgId": "string",
  "solutionId": "string",
  "solutionDealerId": "string"
}
```

### Response — `202 Accepted`
```json
{
  "scheduled": true,
  "publishId": "string",
  "message": "string"
}
```

### Response — `409 Conflict`
```json
{
  "scheduled": false,
  "message": "A data purge is already scheduled for this organization"
}
```

### Business Rules
- Check for existing active schedule for the org.
- If active schedule exists, return `409 Conflict`.
- If not, publish the existing schedule event and return `202 Accepted`.
- This endpoint remains unchanged by the history-table redesign.

---

## Endpoint 2: Purge History Summary

### Purpose

Returns a paged, lightweight list of purge runs. This endpoint reads only from `workflow.blocklist_purge_history` and does not load the detail table.

### Request
- **Method:** `GET`
- **Route:** `/api/blocklist/purge-history`
- **Headers:**
  - `Authorization: Bearer {token}`

### Query Parameters

| Parameter | Type | Required | Default | Notes |
|---|---|---|---|---|
| `commonOrgId` | string | Yes | — | Org identifier |
| `solutionId` | string | No | null | Solution context |
| `solutionDealerId` | string | No | null | Dealer context |
| `runStatus` | string | No | null | Filter on summary `run_status` |
| `requestedBy` | string | No | null | Filter on initiator |
| `startDateFromUtc` | datetime | No | null | Inclusive lower bound on `started_utc` |
| `startDateToUtc` | datetime | No | null | Inclusive upper bound on `started_utc` |
| `pageNumber` | integer | No | 1 | Min 1 |
| `pageSize` | integer | No | 25 | Min 1, max 200 |
| `sortBy` | string | No | `startedUtc` | `startedUtc`, `completedUtc`, `totalItemsProcessed` |
| `sortDirection` | string | No | `desc` | `asc` or `desc` |

### Example Request
```http
GET /api/blocklist/purge-history?commonOrgId=org123&pageNumber=1&pageSize=25
Authorization: Bearer {token}
```

### Response — `200 OK`
```json
{
  "items": [
    {
      "blocklistScheduleId": "uuid",
      "coOrgDealerId": "org123",
      "runStatus": "Completed",
      "requestedBy": "user@company.com",
      "startedUtc": "2026-03-25T10:30:00Z",
      "completedUtc": "2026-03-25T10:45:00Z",
      "timeTakenSeconds": 900,
      "totalItemsProcessed": 1842,
      "phoneRecordsCount": 920,
      "emailRecordsCount": 411,
      "addressRecordsCount": 301,
      "nameRecordsCount": 210,
      "totalBusinessResults": 650,
      "itemsFailed": 12,
      "retentionItemsDeleted": null,
      "retentionBusinessDeleted": null,
      "retentionDaysApplied": null,
      "isEstimated": false
    }
  ],
  "pageNumber": 1,
  "pageSize": 25,
  "totalCount": 120
}
```

### Paging Semantics
- `pageNumber`: Current page, 1-indexed.
- `pageSize`: Number of items returned.
- `totalCount`: Total matching run-history rows for the organization/filter.

### Summary Response Field Definitions

| Field | Source | Notes |
|---|---|---|
| `blocklistScheduleId` | `workflow.blocklist_purge_history.blocklist_schedule_id` | Unique run/schedule identifier |
| `coOrgDealerId` | `coorg_id` | Organization identifier |
| `runStatus` | `run_status` | Mirrors lifecycle state |
| `requestedBy` | `requested_by` | Initiating user/system |
| `startedUtc` | `started_utc` | Null until Requested |
| `completedUtc` | `completed_utc` | Null for non-terminal runs |
| `timeTakenSeconds` | `time_taken_seconds` | Derived and persisted |
| `totalItemsProcessed` | `total_items_processed` | Count of detail rows for the run |
| `phoneRecordsCount` | `phone_records_count` | Count where detail `record_type = 'phone'` |
| `emailRecordsCount` | `email_records_count` | Count where detail `record_type = 'email'` |
| `addressRecordsCount` | `address_records_count` | Count where detail `record_type = 'address'` |
| `nameRecordsCount` | `name_records_count` | Count where detail `record_type = 'name'` |
| `totalBusinessResults` | `total_business_results` | Count from `blocklist_business_results` |
| `itemsFailed` | `items_failed` | Count where detail `item_status = 'Failed'` |
| `retentionItemsDeleted` | `retention_items_deleted` | Nullable until cleanup metric enhancement |
| `retentionBusinessDeleted` | `retention_business_deleted` | Nullable until cleanup metric enhancement |
| `retentionDaysApplied` | `retention_days_applied` | Nullable or configured value |
| `isEstimated` | `is_estimated` | `true` for legacy/backfilled rows |

### Business Rules
- Summary endpoint must not query `workflow.blocklist_purge_history_details` row-by-row during list rendering.
- Counts are read directly from parent summary columns.
- This endpoint is intended for grid/list use and must stay lightweight.

---

## Endpoint 3: Purge History Detail Drilldown

### Purpose

Returns paged record-level details for a single purge run from `workflow.blocklist_purge_history_details`.

### Request
- **Method:** `GET`
- **Route:** `/api/blocklist/purge-history/id/{blocklistScheduleId}/details`
- **Headers:**
  - `Authorization: Bearer {token}`

### Path Parameter

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `blocklistScheduleId` | uuid | Yes | Purge run / schedule identifier |

### Query Parameters

| Parameter | Type | Required | Default | Notes |
|---|---|---|---|---|
| `recordType` | string | No | null | `phone`, `email`, `address`, `name` |
| `itemStatus` | string | No | null | `Processed`, `Failed`, etc. |
| `searchValue` | string | No | null | Search on `recordValue` |
| `pageNumber` | integer | No | 1 | Min 1 |
| `pageSize` | integer | No | 100 | Min 1, max 500 |
| `sortBy` | string | No | `recordValue` | `recordValue`, `itemStatus`, `sourcePageNumber` |
| `sortDirection` | string | No | `asc` | `asc` or `desc` |

### Example Request
```http
GET /api/blocklist/purge-history/id/0d4f3d7a-1111-2222-3333-444455556666/details?recordType=phone&pageNumber=1&pageSize=100
Authorization: Bearer {token}
```

### Response — `200 OK`
```json
{
  "blocklistScheduleId": "0d4f3d7a-1111-2222-3333-444455556666",
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

### Response — `204 No Content`
- Returned when the `blocklistScheduleId` exists nowhere in purge history, or no matching detail data exists and the chosen service layer follows existing `NoContent` conventions.

### Detail Response Field Definitions

| Field | Source | Notes |
|---|---|---|
| `blocklistPurgeHistoryDetailId` | Child PK | Unique detail row identifier |
| `recordType` | `record_type` | One of phone/email/address/name |
| `commonConsumerId` | `common_consumer_id` | Consumer identifier |
| `recordValue` | `record_value` | Actual phone/email/address/name value |
| `itemStatus` | `item_status` | Processing status |
| `resultPayload` | `result_payload` | Existing JSON payload from source result row |
| `sourcePageNumber` | `source_page_number` | Optional source metadata |
| `pageNumber` | `page_number` | Optional source metadata |

### Business Rules
- Detail endpoint must filter and paginate directly in SQL.
- Detail endpoint must not load all rows in memory before paging.
- `recordType` filter is optional but expected for best UI performance.
- Summary counts shown in the main grid come from the parent table, not from recomputing this endpoint.

---

## Source Queries Behind the APIs

### Summary Query

```sql
SELECT
    blocklist_schedule_id,
    coorg_id,
    run_status,
    requested_by,
    started_utc,
    completed_utc,
    time_taken_seconds,
    total_items_processed,
    phone_records_count,
    email_records_count,
    address_records_count,
    name_records_count,
    total_business_results,
    items_failed,
    retention_items_deleted,
    retention_business_deleted,
    retention_days_applied,
    is_estimated
FROM workflow.blocklist_purge_history
WHERE coorg_id = @coorg_id
  AND (@run_status IS NULL OR run_status = @run_status)
  AND (@requested_by IS NULL OR requested_by = @requested_by)
  AND (@start_from IS NULL OR started_utc >= @start_from)
  AND (@start_to IS NULL OR started_utc <= @start_to)
ORDER BY started_utc DESC, blocklist_schedule_id DESC
LIMIT @page_size OFFSET ((@page_number - 1) * @page_size);
```

Count query:

```sql
SELECT COUNT(*)
FROM workflow.blocklist_purge_history
WHERE coorg_id = @coorg_id
  AND (@run_status IS NULL OR run_status = @run_status)
  AND (@requested_by IS NULL OR requested_by = @requested_by)
  AND (@start_from IS NULL OR started_utc >= @start_from)
  AND (@start_to IS NULL OR started_utc <= @start_to);
```

### Detail Query

```sql
SELECT
    blocklist_purge_history_detail_id,
    record_type,
    common_consumer_id,
    record_value,
    item_status,
    result_payload,
    source_page_number,
    page_number
FROM workflow.blocklist_purge_history_details
WHERE blocklist_schedule_id = @blocklist_schedule_id
  AND (@record_type IS NULL OR record_type = @record_type)
  AND (@item_status IS NULL OR item_status = @item_status)
  AND (@search_value IS NULL OR record_value ILIKE @search_value || '%')
ORDER BY record_value ASC, blocklist_purge_history_detail_id ASC
LIMIT @page_size OFFSET ((@page_number - 1) * @page_size);
```

Detail count query:

```sql
SELECT COUNT(*)
FROM workflow.blocklist_purge_history_details
WHERE blocklist_schedule_id = @blocklist_schedule_id
  AND (@record_type IS NULL OR record_type = @record_type)
  AND (@item_status IS NULL OR item_status = @item_status)
  AND (@search_value IS NULL OR record_value ILIKE @search_value || '%');
```

---

## Validation Rules

### Shared
- Reject `pageNumber < 1`.
- Reject `pageSize < 1`.
- Reject `pageSize > 200` for summary endpoint.
- Reject `pageSize > 500` for detail endpoint.

### Summary Endpoint
- Reject `startDateFromUtc > startDateToUtc`.
- Reject unsupported `sortBy` values.
- Reject unsupported `sortDirection` values.

### Detail Endpoint
- Reject unsupported `recordType` values.
- Reject unsupported `sortBy` values.
- Reject unsupported `sortDirection` values.

---

## Error Handling

| Status Code | When |
|---|---|
| `200 OK` | Request succeeds and returns rows |
| `202 Accepted` | Manual trigger accepted |
| `204 No Content` | No matching purge history/detail row for requested identifier |
| `400 Bad Request` | Validation failure |
| `401 Unauthorized` | Missing/invalid auth |
| `403 Forbidden` | Authenticated but not allowed |
| `409 Conflict` | Active purge schedule already exists |
| `500 Internal Server Error` | Unexpected server/storage error |

---

## Performance Expectations

| Endpoint | Performance Principle |
|---|---|
| Manual trigger | Reuse existing schedule/publish flow |
| Summary history | Query only parent summary table |
| Detail drilldown | Query only child detail table with paging and indexes |

This separation is the core reason for the normalized design: the history grid stays fast even when a purge run contains a very large number of detailed records.

---

## Out of Scope

- Returning all detail rows inline inside summary history response.
- Reconstructing detail payloads from deleted rows at read time.
- Changing existing blocklist processing logic as part of this API contract.
>>>>>>> f5e283b04bc9ba168be46150be990bc62dd47697
