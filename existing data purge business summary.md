# Existing Data Purge - Business Summary (Current State Only)

## Purpose
This summary explains how data purge works today in production code and database scripts.
It includes only existing implementation behavior.

**What "data purge" means here:** Data purge is the process triggered whenever a new blocklist rule is added. The system schedules a run to evaluate all consumer data for the affected organization against the new rule, then updates the results accordingly. A secondary background process handles routine cleanup of old processed records.

---

## 1. What starts the data purge

A data purge run is started when a new blocklist rule is added and the caller includes `ScheduleDataPurge = true` in the request.

Step-by-step:
1. A new rule is submitted to configuration-service.
2. Configuration-service checks whether there is already an active (`Scheduled`) run for that organization. If one exists, no new run is started (deduplication).
3. If no active run exists, configuration-service publishes a scheduling event to the message bus.
4. A new purge schedule is created in the database (status: `Scheduled`) for that organization.

---

## 2. How the system processes the purge run

Once the schedule is created, the workflow-resource service runs a background state machine that picks it up and drives it through a sequence of steps:

| Step | What happens |
|---|---|
| **Schedule Requested** | State machine locks the `Scheduled` record, publishes a state event, and marks it `Requested` |
| **Items Received** | State machine picks up consumer data items linked to the schedule (status `Saved`) and marks them `Received` for processing |
| **Business Entities Received** | State machine picks up related business entity results (status `Saved`) and marks them `Received` |
| **Schedule Completed** | Once all linked items are fully processed, the schedule is marked `Completed` |

This state machine runs continuously in the background and handles these steps in order on every tick.

---

## 3. Tables involved in the existing flow

| Table | Role |
|---|---|
| `workflow.blocklist_schedules` | One row per purge run; tracks status (`Scheduled` to `Requested` to `Completed`), which organization, who triggered it, and when |
| `workflow.blocklist_item_results` | One row per consumer item evaluated during the run; holds input data (type/value in JSONB), result, and processing status |
| `workflow.blocklist_business_results` | Related business-level results processed in the same run |

---

## 4. How data is updated during the purge

During an active purge run:
- Each consumer item (in `blocklist_item_results`) progresses through statuses: `Saved` to `Received` to `Processed` (or `Failed`).
- The state machine uses distributed locking (`lease_expires_utc`) to ensure only one worker processes each item at a time.
- State changes are published as events to the message bus for downstream consumers.
- The originating schedule progresses: `Scheduled` to `Requested` to `Uploaded` to `Completed`.

---

## 5. Background retention cleanup (separate process)

In addition to the rule-driven purge, the same state machine includes a background cleanup step (`BlocklistItemClean`) that runs on every tick:

- Deletes rows from `blocklist_item_results` and `blocklist_business_results` where `updated_utc` is older than the configured interval (default: 15 days).
- Removes up to 1000 rows per table per tick.
- This is independent operational hygiene - it is not triggered by adding a rule and does not carry business meaning on its own.

---

## 6. Explicit schedule cleanup endpoint

There is also an existing API endpoint to manually delete all data for a specific purge run:
- `DELETE /blocklist-schedule/purge-data/id/{scheduleId}`
- Hard-deletes all item results, business results, and the schedule row for that run.
- Used for operational management (e.g., cancelling or removing a specific run''s data).

---

## 7. What is not tracked in current implementation

In today''s implementation:
- No persisted history of purge runs for UI display (run count, results breakdown).
- No per-run total of how many records were evaluated or cleared.
- No per-run breakdown by data type (phone, email, address, name).
- The background retention delete function returns no count - the number of rows removed is not stored anywhere.

---

## 8. Business takeaway

Today, each time a new blocklist rule is added, the system automatically schedules a processing run to evaluate consumer data for that organization against the rule. The state machine in workflow-resource drives that run through to completion. A separate background step cleans up old processed records after a retention window. Neither the rule-driven run nor the retention cleanup currently produce any persisted summary history that could be shown in a UI.