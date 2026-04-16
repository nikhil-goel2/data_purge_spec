<<<<<<< HEAD
# Data Purge UI Tab - Implementation Architecture

## 1. Goal

Add a new tab named **Data Purge** to the existing Blocklist UI so authorized users can:

1. manually trigger the rule-driven data purge flow for the current organization, and
2. view purge run history in a lightweight grid backed by the summary table,
3. open a detail popup that retrieves paged per-record drilldown data from the new child detail endpoint.

This document is implementation-ready and aligns the UI contract to the normalized backend design:

- parent summary table: `workflow.blocklist_purge_history`
- child detail table: `workflow.blocklist_purge_history_details`

---

## 2. Scope and Non-Goals

### In Scope
- Add a fourth tab in Blocklist: `Data Purge`.
- Add one `Start Data Purge` button aligned to the top-right corner of the tab.
- Add one purge activity summary grid below the button.
- Load grid data from the purge-history summary endpoint.
- Open a details popup that loads paged drilldown rows from the purge-history details endpoint.
- Trigger manual purge for the current organization context.
- Add unit and e2e coverage.
- Gate rollout behind the existing/appropriate feature flag.

### Out of Scope
- Changing purge business logic.
- Replacing existing add-rule auto-trigger behavior.
- Returning full detailed records in the grid payload.
- Building analytics or reporting beyond the summary grid and row-level drilldown.

---

## 3. Existing UI Baseline

Current tab host and orchestration:
- `libs/customer/blocklist/src/lib/blocklist-tabs/blocklist-tabs.tsx`

Current API utility pattern:
- `libs/customer/blocklist/src/api/getBlocklistData.ts`

Current tab test IDs:
- `libs/customer/blocklist/src/lib/testIds.ts`

Current e2e selectors/patterns:
- `apps/customer-e2e/src/e2e/BlockList/selectors.ts`
- existing Blocklist tab e2e suites in `apps/customer-e2e/src/e2e/BlockList/`

---

## 4. Target User Experience

### Primary User Story
As a Blocklist admin, I can open the Data Purge tab, manually request a purge run, view prior runs in a fast grid, and inspect record-level details for a selected run.

### UI Behavior
- New tab label: `Data Purge`
- Tab visible only when feature flag is enabled.
- Tab contains:
  - a top action row with a right-aligned `Start Data Purge` button
  - a summary history grid below the button
  - inline trigger status feedback
- Grid columns:
  - `Status`
  - `Start Date`
  - `End Date`
  - `Duration`
  - `Total Records Processed`
  - `Started By`
  - `Details`
- Grid loading behavior:
  - fetch summary data when the user enters the Data Purge tab
  - fetch again when the user leaves and re-enters the tab
  - fetch again after a successful manual trigger
- Details behavior:
  - `Details` column renders a clickable link with text `Details`
  - clicking opens a modal popup titled `Data Purge Details`
  - popup does not rely on preloaded row payloads
  - popup calls the detail endpoint on open and supports server-side paging/filtering
- Start button behavior:
  - opens a confirmation modal before execution
  - calls the trigger API only after confirmation
  - handles accepted/conflict/error outcomes deterministically

### Accessibility
- Button and details link are keyboard accessible.
- Status changes are exposed through visible alert text.
- Loading states disable controls and expose stable testable labels.

---

## 5. UI Data Contracts

### 5.1 Summary Grid Row Model

```ts
type DataPurgeActivityRow = {
  blocklistScheduleId: string;
  coOrgDealerId: string;
  runStatus: string;
  requestedBy: string;
  startedUtc?: string;
  completedUtc?: string;
  timeTakenSeconds?: number;
  totalItemsProcessed: number;
  phoneRecordsCount: number;
  emailRecordsCount: number;
  addressRecordsCount: number;
  nameRecordsCount: number;
  totalBusinessResults: number;
  itemsFailed: number;
  retentionItemsDeleted?: number | null;
  retentionBusinessDeleted?: number | null;
  retentionDaysApplied?: number | null;
  isEstimated: boolean;
};
```

### 5.2 Detail Grid Row Model

```ts
type DataPurgeDetailRow = {
  blocklistPurgeHistoryDetailId: string;
  recordType: 'phone' | 'email' | 'address' | 'name';
  commonConsumerId: string;
  recordValue: string;
  itemStatus: string;
  resultPayload?: unknown;
  sourcePageNumber?: number;
  pageNumber?: number;
};
```

### 5.3 Detail Modal Query Model

```ts
type DataPurgeDetailsQuery = {
  blocklistScheduleId: string;
  recordType?: 'phone' | 'email' | 'address' | 'name';
  itemStatus?: string;
  searchValue?: string;
  pageNumber: number;
  pageSize: number;
  sortBy?: 'recordValue' | 'itemStatus' | 'sourcePageNumber';
  sortDirection?: 'asc' | 'desc';
};
```

---

## 6. Integration Contract (UI Perspective)

This UI depends on the endpoints defined in [data purge api endpoints.md](/c:/Code/_bmad-output/planning-artifacts/data purge api endpoints.md).

### Endpoint Summary

1. `POST /api/blocklist/purge`
   - manual trigger

2. `GET /api/blocklist/purge-history`
   - summary grid data
   - backed only by `workflow.blocklist_purge_history`

3. `GET /api/blocklist/purge-history/id/{blocklistScheduleId}/details`
   - detail popup data
   - backed only by `workflow.blocklist_purge_history_details`

### Important Contract Changes from Old Design

- The summary grid no longer receives `detailsPayload`.
- The summary grid no longer receives embedded phone/email/address arrays.
- The detail popup must make its own API request when opened.
- `Total Records Cleared` is replaced in the UI by `Total Records Processed`, sourced from `totalItemsProcessed`.

---

## 7. Frontend Architecture Changes

### 7.1 Tab Model Expansion
File:
- `libs/customer/blocklist/src/lib/blocklist-tabs/blocklist-tabs.tsx`

Changes:
- add lazy import for `data-purge-tab`
- extend tab category model with Data Purge
- add tab entry without changing existing tabs

Recommended ordering:
1. Trending Duplicate Values
2. Dealer Blocklist
3. System Blocklist
4. Data Purge

### 7.2 New Data Purge Tab Component
Create:
- `libs/customer/blocklist/src/lib/tabs/data-purge-tab.tsx`

Responsibilities:
- render action row with right-aligned `Start Data Purge` button
- render summary history grid
- own confirmation modal lifecycle
- own detail modal open/close state
- call summary query on tab activation
- refresh summary query after successful trigger

Suggested props:

```ts
type DataPurgeTabProps = {
  environment: Environment;
  commonOrgId?: string;
  solutionId?: string;
  solutionDealerId?: string;
  isActive: boolean;
};
```

### 7.3 New Hook for Purge Tab Data
Create:
- `libs/customer/blocklist/src/lib/hooks/use-data-purge.ts`

Responsibilities:
- wrap `triggerBlocklistDataPurge`
- wrap `getBlocklistDataPurgeHistory`
- wrap `getBlocklistDataPurgeDetails`
- normalize accepted/conflict/error states
- expose summary reload and detail fetch helpers

Suggested returned shape:

```ts
type UseDataPurgeResult = {
  triggerPurge: () => Promise<void>;
  refreshActivity: () => Promise<void>;
  loadDetails: (query: DataPurgeDetailsQuery) => Promise<void>;
  isTriggerPending: boolean;
  isSummaryLoading: boolean;
  isDetailsLoading: boolean;
  triggerState: 'idle' | 'success' | 'conflict' | 'error';
  triggerMessage?: string;
  activityRows: DataPurgeActivityRow[];
  activityTotalCount: number;
  detailRows: DataPurgeDetailRow[];
  detailTotalCount: number;
};
```

### 7.4 API Utility Additions
Update:
- `libs/customer/blocklist/src/api/getBlocklistData.ts`

Add functions:

```ts
export async function triggerBlocklistDataPurge(...): Promise<...>;

export async function getBlocklistDataPurgeHistory(...): Promise<{
  items: DataPurgeActivityRow[];
  pageNumber: number;
  pageSize: number;
  totalCount: number;
}>;

export async function getBlocklistDataPurgeDetails(...): Promise<{
  blocklistScheduleId: string;
  items: DataPurgeDetailRow[];
  pageNumber: number;
  pageSize: number;
  totalCount: number;
}>;
```

Requirements:
- include bearer token header
- parse `202` and `409` for trigger endpoint
- treat empty summary/detail lists as valid successful responses
- throw only for transport/server failures outside expected business outcomes

### 7.5 Summary Grid Design

Grid columns and display mapping:

| UI Column | Source Field | Display Rule |
|---|---|---|
| `Status` | `runStatus` | render as text badge if design system supports |
| `Start Date` | `startedUtc` | formatted UTC date/time |
| `End Date` | `completedUtc` | formatted UTC date/time or em dash |
| `Duration` | `timeTakenSeconds` | format into `HH:MM:SS` |
| `Total Records Processed` | `totalItemsProcessed` | integer |
| `Started By` | `requestedBy` | raw string |
| `Details` | `blocklistScheduleId` | clickable `Details` link |

Note:
- typed counts such as `phoneRecordsCount` are not shown as separate grid columns in the base layout unless product asks for them later
- they are available for future tooltip/detail summary use

### 7.6 Details Modal Design

Create modal component, for example:
- `libs/customer/blocklist/src/lib/components/data-purge-details-modal.tsx`

Modal behavior:
- title: `Data Purge Details`
- open on click of summary row `Details`
- fetch detail rows on open
- show paged grid of record-level rows
- optionally expose quick filters for:
  - `Record Type`
  - `Item Status`
  - `Search Value`

Recommended detail grid columns:
- `Record Type`
- `Record Value`
- `Common Consumer Id`
- `Item Status`
- `Source Page`
- `Page Number`

Why this changed:
- the old design assumed three pre-grouped arrays returned inline in the row payload
- the new backend design is normalized and optimized for query performance
- therefore the UI should consume a paged detail table, not three embedded payload arrays

### 7.7 Test IDs and Selectors
Update:
- `libs/customer/blocklist/src/lib/testIds.ts`

Add IDs:
- `dataPurgeTab`
- `triggerDataPurgeButton`
- `dataPurgeStatusAlert`
- `dataPurgeActivityGrid`
- `dataPurgeActivityGridRow`
- `dataPurgeStartConfirmModal`
- `dataPurgeStartConfirmSubmit`
- `dataPurgeStartConfirmCancel`
- `dataPurgeDetailsLink`
- `dataPurgeDetailsModal`
- `dataPurgeDetailsGrid`
- `dataPurgeDetailsFilterRecordType`
- `dataPurgeDetailsFilterItemStatus`

---

## 8. UI States

### 8.1 Summary Grid States
- initial loading
- loaded with rows
- loaded empty
- request failed

### 8.2 Trigger States
- idle
- submitting
- accepted/success
- conflict
- unexpected failure

### 8.3 Detail Modal States
- closed
- open + loading
- open + rows
- open + empty
- open + request failure

---

## 9. Testing Strategy

### Unit Tests
- tab is rendered only when feature flag is enabled
- entering active tab triggers summary refresh
- clicking trigger button opens confirmation modal
- confirming calls trigger API
- accepted and conflict states render correctly
- clicking `Details` opens modal and calls detail endpoint
- detail grid paging/filter state updates correctly

### E2E Tests
- user navigates to Data Purge tab and sees summary grid
- summary grid reloads when tab is revisited
- manual trigger success flow
- manual trigger conflict flow
- detail modal opens and shows paged drilldown rows
- detail filter by `recordType` works

---

## 10. Acceptance Criteria

1. Data Purge tab is added without altering existing tab behavior.
2. Tab contains a right-aligned `Start Data Purge` button and a summary history grid.
3. Grid loads from the summary endpoint each time the tab becomes active.
4. Grid refreshes after a successful trigger.
5. Grid columns are `Status`, `Start Date`, `End Date`, `Duration`, `Total Records Processed`, `Started By`, and `Details`.
6. Clicking `Details` opens a modal and fetches paged row-level data from the details endpoint.
7. UI does not expect or use inline `detailsPayload` on summary rows.
8. Confirmation modal appears before manual trigger execution.
9. Success, conflict, and failure trigger outcomes are deterministic and testable.
=======
# Data Purge UI Tab - Implementation Architecture

## 1. Goal

Add a new tab named **Data Purge** to the existing Blocklist UI so authorized users can:

1. manually trigger the rule-driven data purge flow for the current organization, and
2. view purge run history in a lightweight grid backed by the summary table,
3. open a detail popup that retrieves paged per-record drilldown data from the new child detail endpoint.

This document is implementation-ready and aligns the UI contract to the normalized backend design:

- parent summary table: `workflow.blocklist_purge_history`
- child detail table: `workflow.blocklist_purge_history_details`

---

## 2. Scope and Non-Goals

### In Scope
- Add a fourth tab in Blocklist: `Data Purge`.
- Add one `Start Data Purge` button aligned to the top-right corner of the tab.
- Add one purge activity summary grid below the button.
- Load grid data from the purge-history summary endpoint.
- Open a details popup that loads paged drilldown rows from the purge-history details endpoint.
- Trigger manual purge for the current organization context.
- Add unit and e2e coverage.
- Gate rollout behind the existing/appropriate feature flag.

### Out of Scope
- Changing purge business logic.
- Replacing existing add-rule auto-trigger behavior.
- Returning full detailed records in the grid payload.
- Building analytics or reporting beyond the summary grid and row-level drilldown.

---

## 3. Existing UI Baseline

Current tab host and orchestration:
- `libs/customer/blocklist/src/lib/blocklist-tabs/blocklist-tabs.tsx`

Current API utility pattern:
- `libs/customer/blocklist/src/api/getBlocklistData.ts`

Current tab test IDs:
- `libs/customer/blocklist/src/lib/testIds.ts`

Current e2e selectors/patterns:
- `apps/customer-e2e/src/e2e/BlockList/selectors.ts`
- existing Blocklist tab e2e suites in `apps/customer-e2e/src/e2e/BlockList/`

---

## 4. Target User Experience

### Primary User Story
As a Blocklist admin, I can open the Data Purge tab, manually request a purge run, view prior runs in a fast grid, and inspect record-level details for a selected run.

### UI Behavior
- New tab label: `Data Purge`
- Tab visible only when feature flag is enabled.
- Tab contains:
  - a top action row with a right-aligned `Start Data Purge` button
  - a summary history grid below the button
  - inline trigger status feedback
- Grid columns:
  - `Status`
  - `Start Date`
  - `End Date`
  - `Duration`
  - `Total Records Processed`
  - `Started By`
  - `Details`
- Grid loading behavior:
  - fetch summary data when the user enters the Data Purge tab
  - fetch again when the user leaves and re-enters the tab
  - fetch again after a successful manual trigger
- Details behavior:
  - `Details` column renders a clickable link with text `Details`
  - clicking opens a modal popup titled `Data Purge Details`
  - popup does not rely on preloaded row payloads
  - popup calls the detail endpoint on open and supports server-side paging/filtering
- Start button behavior:
  - opens a confirmation modal before execution
  - calls the trigger API only after confirmation
  - handles accepted/conflict/error outcomes deterministically

### Accessibility
- Button and details link are keyboard accessible.
- Status changes are exposed through visible alert text.
- Loading states disable controls and expose stable testable labels.

---

## 5. UI Data Contracts

### 5.1 Summary Grid Row Model

```ts
type DataPurgeActivityRow = {
  blocklistScheduleId: string;
  coOrgDealerId: string;
  runStatus: string;
  requestedBy: string;
  startedUtc?: string;
  completedUtc?: string;
  timeTakenSeconds?: number;
  totalItemsProcessed: number;
  phoneRecordsCount: number;
  emailRecordsCount: number;
  addressRecordsCount: number;
  nameRecordsCount: number;
  totalBusinessResults: number;
  itemsFailed: number;
  retentionItemsDeleted?: number | null;
  retentionBusinessDeleted?: number | null;
  retentionDaysApplied?: number | null;
  isEstimated: boolean;
};
```

### 5.2 Detail Grid Row Model

```ts
type DataPurgeDetailRow = {
  blocklistPurgeHistoryDetailId: string;
  recordType: 'phone' | 'email' | 'address' | 'name';
  commonConsumerId: string;
  recordValue: string;
  itemStatus: string;
  resultPayload?: unknown;
  sourcePageNumber?: number;
  pageNumber?: number;
};
```

### 5.3 Detail Modal Query Model

```ts
type DataPurgeDetailsQuery = {
  blocklistScheduleId: string;
  recordType?: 'phone' | 'email' | 'address' | 'name';
  itemStatus?: string;
  searchValue?: string;
  pageNumber: number;
  pageSize: number;
  sortBy?: 'recordValue' | 'itemStatus' | 'sourcePageNumber';
  sortDirection?: 'asc' | 'desc';
};
```

---

## 6. Integration Contract (UI Perspective)

This UI depends on the endpoints defined in [data purge api endpoints.md](/c:/Code/_bmad-output/planning-artifacts/data purge api endpoints.md).

### Endpoint Summary

1. `POST /api/blocklist/purge`
   - manual trigger

2. `GET /api/blocklist/purge-history`
   - summary grid data
   - backed only by `workflow.blocklist_purge_history`

3. `GET /api/blocklist/purge-history/id/{blocklistScheduleId}/details`
   - detail popup data
   - backed only by `workflow.blocklist_purge_history_details`

### Important Contract Changes from Old Design

- The summary grid no longer receives `detailsPayload`.
- The summary grid no longer receives embedded phone/email/address arrays.
- The detail popup must make its own API request when opened.
- `Total Records Cleared` is replaced in the UI by `Total Records Processed`, sourced from `totalItemsProcessed`.

---

## 7. Frontend Architecture Changes

### 7.1 Tab Model Expansion
File:
- `libs/customer/blocklist/src/lib/blocklist-tabs/blocklist-tabs.tsx`

Changes:
- add lazy import for `data-purge-tab`
- extend tab category model with Data Purge
- add tab entry without changing existing tabs

Recommended ordering:
1. Trending Duplicate Values
2. Dealer Blocklist
3. System Blocklist
4. Data Purge

### 7.2 New Data Purge Tab Component
Create:
- `libs/customer/blocklist/src/lib/tabs/data-purge-tab.tsx`

Responsibilities:
- render action row with right-aligned `Start Data Purge` button
- render summary history grid
- own confirmation modal lifecycle
- own detail modal open/close state
- call summary query on tab activation
- refresh summary query after successful trigger

Suggested props:

```ts
type DataPurgeTabProps = {
  environment: Environment;
  commonOrgId?: string;
  solutionId?: string;
  solutionDealerId?: string;
  isActive: boolean;
};
```

### 7.3 New Hook for Purge Tab Data
Create:
- `libs/customer/blocklist/src/lib/hooks/use-data-purge.ts`

Responsibilities:
- wrap `triggerBlocklistDataPurge`
- wrap `getBlocklistDataPurgeHistory`
- wrap `getBlocklistDataPurgeDetails`
- normalize accepted/conflict/error states
- expose summary reload and detail fetch helpers

Suggested returned shape:

```ts
type UseDataPurgeResult = {
  triggerPurge: () => Promise<void>;
  refreshActivity: () => Promise<void>;
  loadDetails: (query: DataPurgeDetailsQuery) => Promise<void>;
  isTriggerPending: boolean;
  isSummaryLoading: boolean;
  isDetailsLoading: boolean;
  triggerState: 'idle' | 'success' | 'conflict' | 'error';
  triggerMessage?: string;
  activityRows: DataPurgeActivityRow[];
  activityTotalCount: number;
  detailRows: DataPurgeDetailRow[];
  detailTotalCount: number;
};
```

### 7.4 API Utility Additions
Update:
- `libs/customer/blocklist/src/api/getBlocklistData.ts`

Add functions:

```ts
export async function triggerBlocklistDataPurge(...): Promise<...>;

export async function getBlocklistDataPurgeHistory(...): Promise<{
  items: DataPurgeActivityRow[];
  pageNumber: number;
  pageSize: number;
  totalCount: number;
}>;

export async function getBlocklistDataPurgeDetails(...): Promise<{
  blocklistScheduleId: string;
  items: DataPurgeDetailRow[];
  pageNumber: number;
  pageSize: number;
  totalCount: number;
}>;
```

Requirements:
- include bearer token header
- parse `202` and `409` for trigger endpoint
- treat empty summary/detail lists as valid successful responses
- throw only for transport/server failures outside expected business outcomes

### 7.5 Summary Grid Design

Grid columns and display mapping:

| UI Column | Source Field | Display Rule |
|---|---|---|
| `Status` | `runStatus` | render as text badge if design system supports |
| `Start Date` | `startedUtc` | formatted UTC date/time |
| `End Date` | `completedUtc` | formatted UTC date/time or em dash |
| `Duration` | `timeTakenSeconds` | format into `HH:MM:SS` |
| `Total Records Processed` | `totalItemsProcessed` | integer |
| `Started By` | `requestedBy` | raw string |
| `Details` | `blocklistScheduleId` | clickable `Details` link |

Note:
- typed counts such as `phoneRecordsCount` are not shown as separate grid columns in the base layout unless product asks for them later
- they are available for future tooltip/detail summary use

### 7.6 Details Modal Design

Create modal component, for example:
- `libs/customer/blocklist/src/lib/components/data-purge-details-modal.tsx`

Modal behavior:
- title: `Data Purge Details`
- open on click of summary row `Details`
- fetch detail rows on open
- show paged grid of record-level rows
- optionally expose quick filters for:
  - `Record Type`
  - `Item Status`
  - `Search Value`

Recommended detail grid columns:
- `Record Type`
- `Record Value`
- `Common Consumer Id`
- `Item Status`
- `Source Page`
- `Page Number`

Why this changed:
- the old design assumed three pre-grouped arrays returned inline in the row payload
- the new backend design is normalized and optimized for query performance
- therefore the UI should consume a paged detail table, not three embedded payload arrays

### 7.7 Test IDs and Selectors
Update:
- `libs/customer/blocklist/src/lib/testIds.ts`

Add IDs:
- `dataPurgeTab`
- `triggerDataPurgeButton`
- `dataPurgeStatusAlert`
- `dataPurgeActivityGrid`
- `dataPurgeActivityGridRow`
- `dataPurgeStartConfirmModal`
- `dataPurgeStartConfirmSubmit`
- `dataPurgeStartConfirmCancel`
- `dataPurgeDetailsLink`
- `dataPurgeDetailsModal`
- `dataPurgeDetailsGrid`
- `dataPurgeDetailsFilterRecordType`
- `dataPurgeDetailsFilterItemStatus`

---

## 8. UI States

### 8.1 Summary Grid States
- initial loading
- loaded with rows
- loaded empty
- request failed

### 8.2 Trigger States
- idle
- submitting
- accepted/success
- conflict
- unexpected failure

### 8.3 Detail Modal States
- closed
- open + loading
- open + rows
- open + empty
- open + request failure

---

## 9. Testing Strategy

### Unit Tests
- tab is rendered only when feature flag is enabled
- entering active tab triggers summary refresh
- clicking trigger button opens confirmation modal
- confirming calls trigger API
- accepted and conflict states render correctly
- clicking `Details` opens modal and calls detail endpoint
- detail grid paging/filter state updates correctly

### E2E Tests
- user navigates to Data Purge tab and sees summary grid
- summary grid reloads when tab is revisited
- manual trigger success flow
- manual trigger conflict flow
- detail modal opens and shows paged drilldown rows
- detail filter by `recordType` works

---

## 10. Acceptance Criteria

1. Data Purge tab is added without altering existing tab behavior.
2. Tab contains a right-aligned `Start Data Purge` button and a summary history grid.
3. Grid loads from the summary endpoint each time the tab becomes active.
4. Grid refreshes after a successful trigger.
5. Grid columns are `Status`, `Start Date`, `End Date`, `Duration`, `Total Records Processed`, `Started By`, and `Details`.
6. Clicking `Details` opens a modal and fetches paged row-level data from the details endpoint.
7. UI does not expect or use inline `detailsPayload` on summary rows.
8. Confirmation modal appears before manual trigger execution.
9. Success, conflict, and failure trigger outcomes are deterministic and testable.
>>>>>>> f5e283b04bc9ba168be46150be990bc62dd47697
