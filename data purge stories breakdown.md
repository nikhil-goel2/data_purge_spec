---
stepsCompleted: ["Step 1: Requirements Extracted"]
inputDocuments: 
  - "data purge ui tab implementation.md"
  - "data purge api endpoints.md"
  - "data purge endpoint.md"
  - "data purge implementation tasks.md"
documentCreated: "2026-03-25"
status: "Ready for Story Creation"
---

# Data Purge Feature - Epic & Story Breakdown

## Overview

This document provides a complete epic and story breakdown for the **Data Purge Feature**, decomposing requirements from architecture specifications and implementation tasks into actionable stories for development teams.

**Feature Goal:** Enable authorized users to manually trigger blocklist data purge from a dedicated UI tab, with activity history and detailed purge results, while integrating seamlessly with existing blocklist management and purge business logic.

---

## Requirements Inventory

### Functional Requirements (FRs)

- **FR1:** System shall display a fourth tab labeled "Data Purge" in the Blocklist UI module
- **FR2:** Data Purge tab shall be conditionally visible based on LaunchDarkly feature flag `nx.cxm.show-data-purge-tab`
- **FR3:** Data Purge tab shall render action button "Start Data Purge" aligned to the top-right corner
- **FR4:** Data Purge tab shall display purge activity grid below the action button with fixed columns: Status, Start Date, End Date, Duration, Total Records Processed, Started By, Details
- **FR5:** Purge activity grid shall load data from `GET /api/blocklist/purge-history` endpoint
- **FR6:** Purge activity grid shall refresh whenever user navigates to (enters) the Data Purge tab
- **FR7:** Purge activity grid shall support pagination with default page size of 25 and maximum page size of 100
- **FR8:** Clicking "Start Data Purge" button shall open a confirmation modal before executing trigger
- **FR9:** Confirmation modal shall only call trigger API if user clicks "Confirm," not if user clicks "Cancel"
- **FR10:** System shall send `POST /api/blocklist/purge` request with commonOrgId, solutionId (optional), solutionDealerId (optional) when user confirms purge trigger
- **FR11:** System shall display "Accepted" message when purge trigger returns HTTP 202
- **FR12:** System shall display "Already Scheduled" warning when purge trigger returns HTTP 409
- **FR13:** System shall display generic error message when purge trigger fails (4xx/5xx)
- **FR14:** Details link in grid shall open modal popup titled "Data Purge Details"
- **FR15:** Details popup shall display a paged grid of processed records with columns: Record Type (phone/email/address/name), Value, Consumer ID, Status
- **FR16:** Details popup grid shall support pagination, filtering by record type and status, and sorting by value or status
- **FR17:** Details popup grid data shall be fetched from separate `GET /api/blocklist/purge-history/id/{blocklistScheduleId}/details` endpoint with paging parameters
- **FR18:** Backend shall expose `POST /api/blocklist/purge` endpoint to trigger manual purge
- **FR19:** Backend shall check for active schedule before allowing new purge trigger (dedup logic)
- **FR20:** Backend shall return 202 Accepted when purge is successfully scheduled
- **FR21:** Backend shall return 409 Conflict when an active purge is already scheduled for organization
- **FR22:** Backend shall expose `GET /api/blocklist/purge-history` endpoint to retrieve purge activity records
- **FR23:** Purge history endpoint shall accept query parameters: commonOrgId (required), solutionId, solutionDealerId, pageNumber, pageSize
- **FR24:** Purge history endpoint shall return paginated response with items array, pageNumber, pageSize, totalCount
- **FR25:** Backend shall persist `total_items_processed` count in parent purge history record, aggregated from child `workflow.blocklist_purge_history_details` table at completion
- **FR26:** Backend shall store individual processed record details (record_type, value, consumer_id, status) in child `workflow.blocklist_purge_history_details` table, one row per processed item
- **FR27:** Backend shall persist type-specific counts (phone_records_count, email_records_count, address_records_count, name_records_count) in parent table, derived from child rows at completion
- **FR28:** Internal configuration-service endpoint `POST /blocklist/purge-data` shall handle manual purge trigger logic
- **FR29:** Configuration-service shall create `blocklist_schedules` record using actual schema columns: `blocklist_schedule_id`, `coorg_id`, `schedule_status`, `requested_by`, `scheduled_utc`, `created_utc`, `updated_utc`
- **FR30:** Configuration-service shall publish purge event using existing purge publisher to maintain dedup behavior
- **FR31:** Configuration-service shall query blocklist_schedules to detect active schedules before accepting new trigger
- **FR32:** Manual purge trigger shall not modify existing auto-trigger behavior from add-rule flow
- **FR33:** Manual purge shall reuse existing workflow-resource consumer without changes to business logic

### Non-Functional Requirements (NFRs)

- **NFR1:** Purge history endpoint response time shall be < 500ms (p95) under normal load
- **NFR2:** Purge trigger endpoint response time shall be < 200ms (p95) under normal load
- **NFR3:** System shall prevent concurrent duplicate purge requests for same organization (via unique constraint on active status)
- **NFR4:** User must be authorized to view/trigger purge for the organization (no cross-org data leakage)
- **NFR5:** API endpoints shall validate Authorization header with bearer token
- **NFR6:** SQL queries shall be immune to injection attacks (parameterized queries)
- **NFR7:** Database indexes shall be created on workflow tables for query performance: `(coorg_id, run_status, started_utc DESC)` on `blocklist_purge_history` (parent); composite index `(blocklist_purge_history_id, record_type, item_status)` on `blocklist_purge_history_details` (child); uniqueness constraint on `(blocklist_purge_history_id, common_consumer_id, record_type, record_value)` for idempotent retries
- **NFR8:** All endpoints shall return proper HTTP status codes: 200, 202, 400, 401, 403, 409, 500, 503
- **NFR9:** Purge history summary endpoint shall query parent table only, returning lightweight summary rows with counts. Detail endpoint shall query child table with paging/filtering, supporting 1000+ records per purge run.
- **NFR10:** Feature shall have minimum 80% code coverage in unit tests
- **NFR11:** Feature shall pass all E2E tests (manual trigger, conflict, error scenarios, details popup)
- **NFR12:** Feature flag must default to OFF (disabled) until backend and frontend fully deployed
- **NFR13:** No changes to existing blocklist tabs or add-rule auto-trigger behavior
- **NFR14:** Feature shall support pagination for large datasets (1000+ records)
- **NFR15:** API responses shall include X-Request-Id correlation headers for troubleshooting

### Additional Technical Requirements

- **ATR1:** Configuration-service uses existing database table `blocklist_schedules` with actual columns: `blocklist_schedule_id` (PK), `coorg_id`, `schedule_status`, `requested_by`, `scheduled_utc`, `created_utc`, `updated_utc`, `lease_expires_utc`
- **ATR2:** Workflow database uses existing `blocklist_item_results` table with actual columns: `blocklist_item_result_id` (PK), `blocklist_schedule_id` (FK), `common_consumer_id`, `payload` (jsonb), `result_payload` (jsonb), `item_status`, `updated_utc`, `page_number`, `source_page_number`, `lease_expires_utc`
- **ATR3:** Workflow database requires two **new** tables:
  - `blocklist_purge_history` (parent): summary-only with columns `blocklist_purge_history_id`, `blocklist_schedule_id`, `coorg_id`, `run_status`, `requested_by`, `started_utc`, `completed_utc`, `total_items_processed`, `phone_records_count`, `email_records_count`, `address_records_count`, `name_records_count`, `total_business_results`, `items_failed`, timestamps, indexes
  - `blocklist_purge_history_details` (child): normalized detail rows with columns `blocklist_purge_history_detail_id`, `blocklist_purge_history_id`, `blocklist_schedule_id` (denorm), `coorg_id` (denorm), `record_type`, `common_consumer_id`, `record_value`, `item_status`, `result_payload`
- **ATR4:** Consumer-card BFF shall expose three endpoints: `POST /api/blocklist/purge`, `GET /api/blocklist/purge-history`, and `GET /api/blocklist/purge-history/id/{blocklistScheduleId}/details`
- **ATR5:** Frontend shall use existing StateManagement pattern (React Query) for API calls
- **ATR6:** Frontend shall use existing Interstate UI component library for buttons, modals, tables
- **ATR7:** Frontend shall reuse existing blocklist tab architecture pattern (lazy-loaded components, tab registry)
- **ATR8:** All components shall have unique test IDs for e2e testing
- **ATR9:** Feature flag integration shall use existing `useCxmFlags()` hook pattern

---

## Epic Structure

### Epic 1: Configuration-Service Backend Implementation
**Goal:** Implement internal `/blocklist/purge-data` endpoint in configuration-service with manual purge trigger logic, dedup checks, and event publishing without modifying existing purge business logic.
**Dependencies:** None (lowest priority in dependency chain)
**Duration:** 2-3 sprints
**Team:** Backend (Configuration-Service team)

### Epic 2: Consumer-Card BFF & Data Endpoints
**Goal:** Expose user-facing BFF endpoints in consumer-card: (1) `POST /api/blocklist/purge` to trigger, (2) `GET /api/blocklist/purge-history` for summary grid data, (3) `GET /api/blocklist/purge-history/id/{blocklistScheduleId}/details` for paged detail drilldown. All endpoints delegate to configuration-service and query normalized parent/child tables.
**Dependencies:** Epic 1 complete (configuration-service endpoint available)
**Duration:** 2-3 sprints
**Team:** Backend (Consumer-Card BFF team)

### Epic 3: Frontend Data Purge Tab Implementation
**Goal:** Build Data Purge tab UI component with purge activity grid (backed by summary endpoint), confirmation modal, and details modal with paged record grid (backed by separate detail endpoint). Implement tab refresh logic and navigate details on row click. Gated by feature flag.
**Dependencies:** Epic 2 complete (BFF endpoints available)
**Duration:** 2-3 sprints
**Team:** Frontend (React + Nx)

### Epic 4: Testing, Integration & Deployment
**Goal:** Comprehensive testing across all layers (unit, integration, E2E), performance validation, security testing, regression testing, and canary deployment with monitoring.
**Dependencies:** Epics 1, 2, 3 complete
**Duration:** 1-2 sprints
**Team:** QA + Backend + Frontend (cross-functional)

---

## Functional Requirement Coverage Map

| FR | Epic 1 | Epic 2 | Epic 3 | Epic 4 | Status |
|----|--------|--------|--------|--------|--------|
| FR1 - Display Data Purge tab | — | — | ✓ | Tested | Covered |
| FR2 - Feature flag visibility | — | — | ✓ | Tested | Covered |
| FR3 - Start Data Purge button | — | — | ✓ | Tested | Covered |
| FR4 - Grid columns display | — | — | ✓ | Tested | Covered |
| FR5 - Grid data from API | — | ✓ | ✓ | Tested | Covered |
| FR6 - Grid refresh on tab enter | — | — | ✓ | Tested | Covered |
| FR7 - Pagination support | — | ✓ | ✓ | Tested | Covered |
| FR8-9 - Confirmation modal workflow | — | — | ✓ | Tested | Covered |
| FR10-13 - Trigger API & responses | ✓ | ✓ | ✓ | Tested | Covered |
| FR14-17 - Details modal with paged grid | — | ✓ | ✓ | Tested | Covered |
| FR18-21 - Manual trigger endpoint | ✓ | ✓ | — | Tested | Covered |
| FR22-27 - Purge history endpoint | — | ✓ | — | Tested | Covered |
| FR28-31 - Config-service implementation | ✓ | — | — | Tested | Covered |
| FR32-33 - Backward compatibility | ✓ | ✓ | — | Verified | Covered |

---

## Epic 1: Configuration-Service Backend Implementation

### Story 1.1: Create Purge Trigger Request/Response Models

**User Story:**  
As a Backend Developer,
I want to create strongly-typed request and response models for manual purge trigger,
So that the endpoint has validated contracts and proper serialization.

**Acceptance Criteria:**

**Given** a new configuration-service code branch
**When** I create `TriggerBlocklistPurgeRequest.cs` model
**Then** the model has properties: CommonOrgId (string, required), RequestedBy (string, required)
**And** the model includes validation attributes that enforce non-empty strings
**And** unit tests verify valid and invalid request payloads

**Given** the request model is complete
**When** I create `TriggerBlocklistPurgeResponse.cs` model
**Then** the model has properties: Scheduled (bool), PublishId (string, nullable), Message (string, nullable)
**And** the model serializes/deserializes correctly to/from JSON
**And** unit tests verify success (202) and conflict (409) response variants

**Tasks:**
- Create request model with validation attributes
- Create response model with proper nullability
- Add unit tests for both models (5-8 tests)
- Ensure models are compatible with existing configuration-service patterns

**Story Points:** 3  
**Priority:** High  
**Testing:** Unit (Jest/NUnit)

---

### Story 1.2: Extend IBlocklistManager with TriggerManualPurgeAsync Method

**User Story:**  
As a Backend Architect,
I want to extend the IBlocklistManager interface with a new TriggerManualPurgeAsync method,
So that the manager layer has a contract for manual purge logic implementation.

**Acceptance Criteria:**

**Given** the IBlocklistManager interface exists
**When** I add `TriggerManualPurgeAsync(commonOrgId, requestedBy, cancellationToken)` method signature
**Then** the method returns `Task<TriggerBlocklistPurgeResponse>`
**And** the method includes XML documentation with behavior description
**And** documentation specifies 202 success and 409 conflict paths
**And** documentation lists exception handling expectations

**Tasks:**
- Add method signature to interface with proper async pattern
- Write comprehensive XML documentation
- Document both success and conflict scenarios
- Ensure method signature matches implementation in Story 1.3

**Story Points:** 2  
**Priority:** High  
**Testing:** Code review verification

---

### Story 1.3: Implement TriggerManualPurgeAsync Business Logic

**User Story:**  
As a Backend Developer,
I want to implement the TriggerManualPurgeAsync method with dedup and event publishing logic,
So that manual purge triggers are handled atomically with proper conflict detection.

**Acceptance Criteria:**

**Given** an organization with no active purge schedule
**When** I call TriggerManualPurgeAsync with valid parameters
**Then** the method queries blocklist_schedules table
**And** no active schedule is found (`schedule_status` not in `Scheduled`, `Requested`, `Uploaded`)
**And** a new schedule record is created with: `blocklist_schedule_id` (GUID), `coorg_id`, `schedule_status='Scheduled'`, `requested_by`, `scheduled_utc`, `created_utc`, `updated_utc`
**And** a new parent `blocklist_purge_history` row is created with `run_status='Scheduled'` and linked `blocklist_schedule_id`
**And** the existing IPurgePublisher.PublishPurgeEventAsync() is called with schedule details
**And** a response with scheduled=true, publishId, and message is returned
**And** HTTP 202 would be returned by the caller

**Given** an organization with an active purge schedule
**When** I call TriggerManualPurgeAsync with the same commonOrgId
**Then** the method finds the active schedule
**And** no new schedule is created
**And** no event is published
**And** a response with scheduled=false and appropriate message is returned
**And** HTTP 409 Conflict would be returned by the caller

**Given** concurrent requests for the same organization
**When** two TriggerManualPurgeAsync calls occur simultaneously
**Then** database dedup logic on (`coorg_id`, active `schedule_status`) prevents duplicate schedules
**And** only one succeeds with 202
**And** one receives 409

**Given** a database or publish operation fails
**When** an error occurs during schedule creation or event publishing
**Then** the transaction is rolled back
**And** an exception is thrown (caller returns 500)

**Tasks:**
- Query blocklist_schedules for active schedules
- Create schedule record with correct fields
- Call existing purge publisher (verify no changes to publisher)
- Implement transaction atomicity for create + publish
- Add error handling for database and publish failures
- Add logging (INFO for success, WARN for conflict, ERROR for failures)
- Write 8-10 unit tests covering success, conflict, concurrent, and error scenarios
- Write 2-3 integration tests with real database

**Story Points:** 8  
**Priority:** High  
**Testing:** Unit + Integration

---

### Story 1.4: Create POST /blocklist/purge-data Controller Endpoint

**User Story:**  
As a Backend Developer,
I want to create the POST /blocklist/purge-data controller endpoint,
So that internal callers can request manual purge triggers via HTTP.

**Acceptance Criteria:**

**Given** the TriggerManualPurgeAsync implementation is complete
**When** I create the BlocklistController endpoint [HttpPost("/blocklist/purge-data")]
**Then** the endpoint is protected with [Authorize] attribute
**And** the endpoint accepts TriggerBlocklistPurgeRequest in request body
**And** the endpoint extracts requestedBy from JWT claims (e.g., 'email' claim)
**And** the endpoint calls blocklistManager.TriggerManualPurgeAsync()
**And** the endpoint returns 202 Accepted with response body for success
**And** the endpoint returns 409 Conflict with response body for conflict
**And** the endpoint returns 400 Bad Request for validation failures
**And** the endpoint returns 500 Internal Server Error for server errors

**Given** a valid request
**When** I POST to /blocklist/purge-data with commonOrgId, requestedBy in body
**Then** response status is 202
**And** response body contains { scheduled: true, publishId: "...", message: "..." }
**And** Content-Type header is application/json

**Given** a conflict scenario
**When** I POST with commonOrgId that already has active schedule
**Then** response status is 409
**And** response body contains { scheduled: false, message: "..." }

**Tasks:**
- Create or extend BlocklistController
- Add HttpPost endpoint with Authorize attribute
- Extract requestedBy from JWT claims
- Call manager method and map responses to HTTP status codes
- Handle errors and return appropriate status codes
- Add Produces/ProducesResponseType attributes for Swagger
- Write 5+ unit tests covering happy path, conflict, auth failure, validation failure

**Story Points:** 5  
**Priority:** High  
**Testing:** Unit

---

### Story 1.5: Verify Existing Workflow Tables and Prepare Normalized Purge History Storage

**User Story:**  
As a Database Administrator,
I want to verify the existing workflow tables and prepare normalized purge history storage,
So that purge summary and detail data remain queryable after source records are hard-deleted.

**Acceptance Criteria:**

**Given** the workflow database
**When** I verify `blocklist_schedules` and `blocklist_item_results`
**Then** I confirm the actual schemas already exist and are used as-is:
  - `blocklist_schedules`: `blocklist_schedule_id`, `coorg_id`, `schedule_status`, `requested_by`, `scheduled_utc`, `created_utc`, `updated_utc`, `lease_expires_utc`
  - `blocklist_item_results`: `blocklist_item_result_id`, `blocklist_schedule_id`, `common_consumer_id`, `payload` (jsonb), `result_payload` (jsonb), `item_status`, `updated_utc`, `page_number`, `source_page_number`, `lease_expires_utc`
**And** no migration is created to alter those existing source tables for this feature

**Given** normalized storage is required
**When** I prepare migrations for purge history persistence
**Then** parent table `blocklist_purge_history` includes summary columns such as:
  - `blocklist_purge_history_id` (uuid, PK)
  - `blocklist_schedule_id` (uuid, FK)
  - `coorg_id` (varchar(64), not null)
  - `run_status` (varchar(32), mirrors schedule lifecycle values such as `Scheduled`, `Requested`, `Completed`, `Failed`)
  - `requested_by`, `started_utc`, `completed_utc`
  - `total_items_processed`, `phone_records_count`, `email_records_count`, `address_records_count`, `name_records_count`, `items_failed`
**And** child table `blocklist_purge_history_details` stores one detail row per processed item
**And** indexes are created for summary list and detail paging/filtering lookups

**Given** migration scripts are ready
**When** I run migration on test environment
**Then** both `blocklist_purge_history` and `blocklist_purge_history_details` are created with constraints and indexes
**And** data can be inserted, updated, and queried successfully via parent and child queries

**Tasks:**
- Verify actual schema state of existing workflow source tables
- Create migration scripts for parent and child purge history tables
- Validate summary and detail index strategies
- Test migrations on test database
- Ensure migrations are reversible (rollback scripts)

**Story Points:** 5  
**Priority:** High  
**Testing:** Database validation

---

### Story 1.6: Unit & Integration Tests for Configuration-Service

**User Story:**  
As a QA Engineer,
I want comprehensive unit and integration tests for the purge trigger functionality,
So that configuration-service changes are reliable and regressions are caught.

**Acceptance Criteria:**

**Given** implementation of Stories 1.1-1.5 is complete
**When** I write unit tests for models and manager logic
**Then** tests cover:
  - Valid request validation
  - Invalid/empty field validation
  - Successful purge trigger (202 path)
  - Conflict scenario (409 path)
  - Concurrent request handling
  - Database errors and rollback
  - Publisher call verification
  - No modifications to existing publisher/consumer
**And** minimum 80% code coverage on purge-related code
**And** all tests pass

**Given** unit tests are complete
**When** I write integration tests
**Then** tests cover:
  - Successful trigger via controller endpoint
  - Conflict detection with real database
  - Concurrent trigger testing
  - Bad request validation
  - Authorization failure
  - Schedule record creation with correct values
**And** all integration tests use test database and clean up after execution

**Tasks:**
- Write 12-15 unit tests for models and business logic
- Write 5-7 integration tests with database
- Verify publisher is called (without modifying it)
- Verify no regressions to existing add-rule auto-trigger
- Achieve > 80% code coverage
- Document test scenarios and expected outcomes

**Story Points:** 8  
**Priority:** High  
**Testing:** Unit + Integration

---

### Story 1.7: API Documentation & Swagger Schema for /blocklist/purge-data

**User Story:**  
As a Backend Developer,
I want API documentation and Swagger schema for the /blocklist/purge-data endpoint,
So that frontend and BFF teams understand the contract clearly.

**Acceptance Criteria:**

**Given** the endpoint implementation is complete
**When** I add Swagger/OpenAPI attributes to the controller
**Then** Swagger shows:
  - Endpoint: POST /blocklist/purge-data
  - Authorization: Bearer token required
  - Request body schema with CommonOrgId, RequestedBy
  - 202 response schema with example
  - 409 response schema with example
  - 400 response for validation errors
  - 500 response for server errors
**And** example request and response payloads are provided
**And** request/response properties are documented

**Given** Swagger schema is created
**When** I generate client SDK from OpenAPI spec
**Then** SDK is compatible with .NET/C# consumers

**Tasks:**
- Add ProducesResponseType attributes for each status code
- Add example request/response bodies in Swagger UI
- Document each property with descriptions
- Generate SDK or verify existing SDK is updated
- Test Swagger UI displays correctly
- Share Swagger JSON with BFF team

**Story Points:** 2  
**Priority:** Medium  
**Testing:** Swagger UI verification

---

## Epic 2: Consumer-Card BFF & Grid Data Endpoint

### Story 2.1: Create Parent & Child Purge History Tables

**User Story:**  
As a Database Administrator,
I want to create normalized parent and child tables for purge history,
So that summary and detail queries are independent and scalable even with 1000+ records per run.

**Acceptance Criteria:**

**Given** the workflow database
**When** I create `workflow.blocklist_purge_history` (parent summary table)
**Then** the table contains:
  - `blocklist_purge_history_id` (PK, uuid)
  - `blocklist_schedule_id` (FK → `blocklist_schedules.blocklist_schedule_id`)
  - `coorg_id` (varchar, indexed)
  - `run_status` (varchar, aligned with schedule lifecycle values such as `Scheduled`, `Requested`, `Completed`, `Failed`)
  - `requested_by` (varchar)
  - `started_utc`, `completed_utc` (timestamps)
  - `total_items_processed`, `phone_records_count`, `email_records_count`, `address_records_count`, `name_records_count` (integers)
  - `total_business_results`, `items_failed` (integers)
  - Composite index on `(coorg_id, run_status, started_utc DESC)`
**And** the table can be queried by org for summary grid; counts are persistent, not calculated

**Given** the detail storage requirement
**When** I create `workflow.blocklist_purge_history_details` (child detail table)
**Then** the table contains:
  - `blocklist_purge_history_detail_id` (PK, uuid)
  - `blocklist_purge_history_id` (FK → parent)
  - `blocklist_schedule_id` (denorm, uuid)
  - `coorg_id` (denorm, varchar)
  - `record_type` (varchar, CHECK IN ('phone', 'email', 'address', 'name'))
  - `common_consumer_id` (varchar)
  - `record_value` (varchar)
  - `item_status` (varchar, CHECK IN ('Processed', 'Failed', 'Skipped'))
  - `result_payload` (jsonb, nullable)
  - `created_utc` (timestamp)
**And** uniqueness constraint on `(blocklist_purge_history_id, coorg_id, record_type, record_value, common_consumer_id)` for idempotent retries
**And** composite indexes: `(blocklist_purge_history_id, record_type, item_status)`, `(blocklist_purge_history_id, coorg_id)`, `(blocklist_purge_history_id, record_value, common_consumer_id)`

**Tasks:**
- Create Liquibase migration for parent table with all columns and indexes
- Create Liquibase migration for child table with all columns, constraints, indexes
- Validate index names follow naming convention
- Document schema rationale in migration comments

**Story Points:** 5  
**Priority:** High  
**Testing:** Database verification

---

### Story 2.2: Create BFF Request/Response Models

**User Story:**  
As a Backend Developer,
I want to create BFF request/response models for trigger and paginated history endpoints,
So that consumer-card has clean contracts aligned to normalized parent/child query design.

**Acceptance Criteria:**

**Given** configuration-service models from Epic 1
**When** I create consumer-card models: BlocklistPurgeRequest, BlocklistPurgeResponse
**Then** models match configuration-service structure for serialization compatibility

**Given** the summary grid data requirement
**When** I create BlocklistPurgeHistoryResponse and BlocklistPurgeActivityRow models
**Then** BlocklistPurgeHistoryResponse has: Items[], PageNumber, PageSize, TotalCount
**And** BlocklistPurgeActivityRow has: BlocklistScheduleId, RunStatus, RequestedBy, StartDateUtc, EndDateUtc, Duration, TotalItemsProcessed, PhoneRecordsCount, EmailRecordsCount, AddressRecordsCount, NameRecordsCount, TotalBusinessResults, ItemsFailed
**And** all fields map directly from parent table columns (or simple calculations like Duration)

**Given** the detail grid data requirement
**When** I create BlocklistPurgeDetailsResponse and BlocklistPurgeDetailRow models
**Then** BlocklistPurgeDetailsResponse has: Items[], PageNumber, PageSize, TotalCount, BlocklistScheduleId
**And** BlocklistPurgeDetailRow has: Id, RecordType, RecordValue, ConsumerId, ItemStatus, ResultPayload
**And** all fields map directly from child table columns
**And** all models serialize/deserialize correctly to/from JSON

**Tasks:**
- Create summary/grid models without details payload
- Create detail/drilldown models for child table rows
- Add JSON serialization attributes (JsonProperty names)
- Write unit tests for serialization (6-8 tests)
- Ensure models follow consumer-card naming conventions

**Story Points:** 3  
**Priority:** High  
**Testing:** Unit

---

### Story 2.3: Create Blocklist Purge Repository for Parent & Child Tables

**User Story:**  
As a Backend Developer,
I want to create a repository layer for both summary and detail queries,
So that BFF can efficiently load independent summary grid and paged detail drilldown.

**Acceptance Criteria:**

**Given** `workflow.blocklist_purge_history` (parent) exists
**When** I implement `IBlocklistPurgeHistoryRepository.GetPurgeHistoryAsync(commonOrgId, pageNumber, pageSize)`
**Then** the method queries parent table filtered by `coorg_id`
**And** results are paginated with pageNumber (1-based) and pageSize (default 25, max 100)
**And** total count is returned for UI pagination
**And** each row maps directly from parent columns: blocklist_purge_history_id, blocklist_schedule_id, run_status, started_utc, completed_utc, total_items_processed, phone_records_count, etc.
**And** `Duration` is calculated as `(completed_utc - started_utc)` in milliseconds or human-readable format
**And** results are ordered by `started_utc DESC` (most recent first)
**And** query executes in < 100ms for normal org sizes
**And** queries use parameterized SQL

**Given** `workflow.blocklist_purge_history_details` (child) exists
**When** I implement `IBlocklistPurgeHistoryDetailsRepository.GetDetailsAsync(blocklistScheduleId, pageNumber, pageSize, filterRecordType, filterStatus, searchValue)`
**Then** the method queries child table filtered by `blocklist_schedule_id`
**And** optional filters: `record_type`, `item_status`, `record_value` (LIKE search)
**And** results are paginated with pageNumber and pageSize (default 25, max 100)
**And** total count is returned
**And** each row maps to: id, record_type, record_value, common_consumer_id, item_status, result_payload
**And** results are ordered by `created_utc DESC` (newest first)
**And** query executes in < 200ms for 1000-row detail set
**And** queries use parameterized SQL

**Tasks:**
- Create `IBlocklistPurgeHistoryRepository` interface for parent table queries
- Implement paginated history retrieval against parent table
- Create `IBlocklistPurgeHistoryDetailsRepository` interface for child table queries
- Implement paginated detail retrieval with optional filtering
- Compute Duration safely for both completed and in-progress runs
- Use parameterized SQL queries for security
- Add unit and integration tests for both repositories
- Benchmark both query paths

**Story Points:** 8  
**Priority:** High  
**Testing:** Unit + Integration + Performance

---

### Story 2.4: Create Blocklist Purge Service Layer

**User Story:**  
As a Backend Developer,
I want to create a service layer that orchestrates repository and configuration-service calls,
So that business logic is centralized and testable.

**Acceptance Criteria:**

**Given** the repository and configuration-service endpoint are ready
**When** I implement IBlocklistPurgeService.TriggerManualPurgeAsync()
**Then** the method validates the request (CommonOrgId required)
**And** calls configuration-service POST /blocklist/purge-data via HTTP client or SDK
**And** returns BlocklistPurgeResponse (delegates to config-service)
**And** handles configuration-service errors gracefully (500, timeouts, etc.)
**And** logs trigger attempts and outcomes

**Given** the history retrieval requirement
**When** I implement GetPurgeHistoryAsync()
**Then** the method validates user has access to commonOrgId (via auth context)
**And** calls repository.GetPurgeHistoryAsync()
**And** returns BlocklistPurgeHistoryResponse
**And** handles repository errors gracefully
**And** logs history queries (non-sensitive org id only)

**Tasks:**
- Create IBlocklistPurgeService interface
- Implement TriggerManualPurgeAsync (delegates to config-service)
- Implement GetPurgeHistoryAsync (delegates to repository)
- Add error handling for both success and failure paths
- Add logging for debugging
- Write 8-10 unit tests with mocked repository and HTTP client
- Verify config-service call is made correctly

**Story Points:** 5  
**Priority:** High  
**Testing:** Unit

---

### Story 2.5: Create BFF Controller Endpoints (Summary & Details)

**User Story:**  
As a Backend Developer,
I want to create BFF controller endpoints for manual trigger, history summary, and details drilldown,
So that frontend can efficiently load summary grid and paged detail records separately.

**Acceptance Criteria:**

**Given** the service layer is complete
**When** I create [HttpPost("/api/blocklist/purge")] endpoint
**Then** endpoint is protected with [Authorize] attribute
**And** accepts BlocklistPurgeRequest in body
**And** validates commonOrgId is required
**And** extracts requestedBy from JWT claims
**And** calls service.TriggerManualPurgeAsync()
**And** returns 202 Accepted with response body for success
**And** returns 409 Conflict with response body for conflict
**And** returns 400 Bad Request for validation failures
**And** returns 401 Unauthorized for auth failures
**And** returns 500 Internal Server Error for server errors

**Given** the summary list requirement
**When** I create [HttpGet("/api/blocklist/purge-history")] endpoint
**Then** endpoint is protected with [Authorize] attribute
**And** accepts query parameters: commonOrgId (required), pageNumber, pageSize
**And** validates commonOrgId is required
**And** validates pageNumber >= 1, pageSize > 0 and <= 100
**And** calls service.GetPurgeHistoryAsync(commonOrgId, pageNumber, pageSize)
**And** returns 200 OK with BlocklistPurgeHistoryResponse (summary grid rows)
**And** response contains: Items[], PageNumber, PageSize, TotalCount
**And** returns 400 Bad Request for validation failures
**And** returns 401 Unauthorized for auth failures
**And** returns 500 Internal Server Error for server errors

**Given** the detail drilldown requirement
**When** I create [HttpGet("/api/blocklist/purge-history/id/{blocklistScheduleId}/details")] endpoint
**Then** endpoint is protected with [Authorize] attribute
**And** accepts path parameter blocklistScheduleId (required)
**And** accepts query parameters: pageNumber, pageSize, recordType (optional), status (optional), searchValue (optional)
**And** validates blocklistScheduleId is valid GUID
**And** validates pageNumber >= 1, pageSize > 0 and <= 100
**And** calls service.GetPurgeDetailsAsync(blocklistScheduleId, pageNumber, pageSize, filters)
**And** returns 200 OK with BlocklistPurgeDetailsResponse (detail records)
**And** response contains: Items[], PageNumber, PageSize, TotalCount, BlocklistScheduleId
**And** returns 404 Not Found if blocklistScheduleId does not exist
**And** returns 400 Bad Request for validation failures
**And** returns 401 Unauthorized for auth failures
**And** returns 500 Internal Server Error for server errors

**Tasks:**
- Create or extend BlocklistController
- Add POST /api/blocklist/purge endpoint
- Add GET /api/blocklist/purge-history endpoint for summary
- Add GET /api/blocklist/purge-history/id/{blocklistScheduleId}/details endpoint for details
- Extract requestedBy from JWT claims for POST
- Verify user org access for all endpoints
- Add Produces/ProducesResponseType attributes for Swagger
- Write 12+ unit tests covering all endpoints, validation, auth, errors
- Manual testing with frontend team

**Story Points:** 8  
**Priority:** High  
**Testing:** Unit

---

### Story 2.6: Update Service Layer for Summary & Details

**User Story:**  
As a Backend Developer,
I want to extend the service layer with separate methods for summary and detail queries,
So that front-end can independently load grid data and modal content.

**Acceptance Criteria:**

**Given** the summary requirement
**When** I implement IBlocklistPurgeService.GetPurgeHistoryAsync(commonOrgId, pageNumber, pageSize)
**Then** method validates user has access to commonOrgId (via auth context)
**And** calls repository.GetPurgeHistoryAsync()
**And** returns BlocklistPurgeHistoryResponse
**And** handles repository errors gracefully
**And** logs history queries

**Given** the detail requirement
**When** I implement IBlocklistPurgeService.GetPurgeDetailsAsync(blocklistScheduleId, pageNumber, pageSize, filters)
**Then** method validates blocklistScheduleId exists and user has org access
**And** calls detailsRepository.GetDetailsAsync() with filters
**And** returns BlocklistPurgeDetailsResponse
**And** handles repository errors and not-found scenarios gracefully
**And** logs detail queries

**Tasks:**
- Extend IBlocklistPurgeService interface with both methods
- Implement GetPurgeHistoryAsync delegating to parent repository
- Implement GetPurgeDetailsAsync delegating to details repository
- Add error handling and logging for both paths
- Write 10+ unit tests with mocked repositories
- Test filter parameter passing to repository

**Story Points:** 4  
**Priority:** High  
**Testing:** Unit

---

### Story 2.7: Database Seed Data for Parent & Child Tables

**User Story:**  
As a QA Engineer,
I want test and demo seed data for parent and child purge history tables,
So that frontend and integration testing can proceed with realistic normalized data.

**Acceptance Criteria:**

**Given** both `blocklist_purge_history` (parent) and `blocklist_purge_history_details` (child) tables exist
**When** I create seed script
**Then** script inserts test data:
  - 5-10 test parent purge history records with varying status (`Scheduled`, `Requested`, `Completed`, `Failed`)
  - Realistic timestamps (recent dates, varying durations)
  - Parent row summary counts: phone_records_count, email_records_count, address_records_count, name_records_count
  - For each parent row: 50-500 child detail rows with varied record_type, record_value, item_status
  - Child rows span different pages (pagination testing with 25+ items per page)
**And** parent totals match count() aggregations from child rows
**And** seed script can be run on test database without impact to production
**And** seed data is repeatable (can be truncated and reloaded)

**Tasks:**
- Create seed script (SQL with INSERT statements for parent and child tables)
- Insert 5-10 parent purge history rows with realistic data
- Insert 200-500 child detail rows across parents (mix of record types, statuses)
- Ensure child row counts match parent summary counts
- Ensure data covers pagination testing (25+ items per parent), filtering scenarios
- Document how to run seed script
- Verify seed data loads successfully and parent/child relationships are valid

**Story Points:** 4  
**Priority:** Medium  
**Testing:** Database verification

---

### Story 2.8: API Documentation & Swagger for All Endpoints

**User Story:**  
As a Backend Developer,
I want API documentation and Swagger schema for all BFF endpoints and config-service endpoint,
So that all teams have clear reference for request/response contracts.

**Acceptance Criteria:**

**Given** all endpoints are implemented (config-service trigger, BFF summary, BFF details)
**When** I add Swagger/OpenAPI attributes to controllers
**Then** Swagger shows:
  - POST /api/blocklist/purge (BFF trigger) with request/response schemas and examples
  - GET /api/blocklist/purge-history (BFF summary list) with query parameters, response schema, pagination example
  - GET /api/blocklist/purge-history/id/{blocklistScheduleId}/details (BFF detail drilldown) with path parameter, query parameters, response schema
  - POST /blocklist/purge-data (config-service, internal) with request/response schemas
  - 200, 202, 204, 400, 401, 404, 409, 500 status codes documented
  - Request/response properties with descriptions
  - Authorization: Bearer token required (shown in UI)
**And** Swagger UI displays correctly and example requests work
**And** Swagger JSON can be downloaded and shared with frontend team

**Tasks:**
- Add ProducesResponseType, Consumes, Produces attributes to all endpoints
- Add example request/response bodies for each endpoint
- Document each parameter and property with descriptions
- Verify pagination parameters are documented (pageNumber, pageSize defaults)
- Verify detail endpoint filter parameters are documented (recordType, status, searchValue)
- Test Swagger UI functionality
- Share Swagger JSON with frontend and config-service teams

**Story Points:** 3  
**Priority:** Medium  
**Testing:** Swagger UI verification
- Test Swagger UI functionality
- Share Swagger JSON with frontend team

**Story Points:** 2  
**Priority:** Medium  
**Testing:** Swagger UI verification

---

## Epic 3: Frontend Data Purge Tab Implementation

### Story 3.1: Create Tab Component & Feature Flag Integration

**User Story:**  
As a Frontend Developer,
I want to create the Data Purge tab component and integrate feature flag logic,
So that the tab appears only when flag is enabled via LaunchDarkly.

**Acceptance Criteria:**

**Given** the Data Purge feature requirements
**When** I create new file `libs/customer/blocklist/src/lib/tabs/data-purge-tab.tsx`
**Then** component accepts props: environment, commonOrgId, solutionId (optional), solutionDealerId (optional)

**Given** feature flag integration
**When** component loads
**Then** it calls useCxmFlags() hook to check `nx.cxm.show-data-purge-tab` flag value
**And** if flag is OFF, component returns null (tab not rendered)
**And** if flag is ON, component renders the tab content

**Given** the tab is enabled
**When** component renders
**Then** it displays a header with right-aligned "Start Data Purge" button
**And** it displays purge activity grid below the button
**And** it displays status alert for trigger results
**And** it has test ID `dataPurgeTab` for e2e testing

**Tasks:**
- Create component file with proper React/Nx structure
- Import and use useCxmFlags hook
- Verify flag name matches LaunchDarkly config: `nx.cxm.show-data-purge-tab`
- Add feature flag conditional rendering
- Export component from `libs/customer/blocklist/src/lib/tabs/index.ts`
- Write unit tests for flag logic (flag ON, flag OFF)
- Add test IDs for e2e testing

**Story Points:** 5  
**Priority:** High  
**Testing:** Unit + E2E

---

### Story 3.2: Build Purge Activity Grid Component

**User Story:**  
As a Frontend Developer,
I want to build the purge activity grid that displays history with 7 columns,
So that users can see all purge runs with key metrics.

**Acceptance Criteria:**

**Given** the purge activity grid requirements
**When** I create the grid component
**Then** it displays columns in exact order: Status, Start Date, End Date, Duration, Total Records Cleared, Started By, Details

**Given** the data loading requirement
**When** component mounts or tab becomes active
**Then** it calls `getBlocklistDataPurgeHistory(environment, commonOrgId, solutionId, solutionDealerId, 1, 25)`
**And** awaits response and populates grid with items

**Given** a grid row is returned
**When** grid renders the row
**Then** each cell displays the correct value:
  - Status: displays row.status (e.g., `Completed`, `Requested`)
  - Start Date: formats row.startDateUtc as locale-specific date (e.g., "Mar 25, 2026")
  - End Date: formats row.endDateUtc as locale-specific date (nullable, shows empty if null)
  - Duration: displays row.duration formatted as HH:MM:SS
  - Total Records Processed: displays row.totalItemsProcessed as integer
  - Started By: displays row.startedBy (email)
  - Details: renders as clickable link with text "Details" (see Story 3.4)

**Given** the grid has many rows
**When** grid is displayed
**Then** pagination controls appear at bottom showing: current page, page size, total count
**And** pagination allows navigation to next/previous pages (default 25 items per page)
**And** pagination validates page number >= 1

**Given** user navigates to different page
**When** pagination is updated
**Then** component calls API with new pageNumber and refreshes grid

**Given** grid data is loading
**When** API request is in flight
**Then** grid shows loading spinner or skeleton state
**And** interactions are disabled until data loads

**Given** grid API fails
**When** error occurs
**Then** error alert is displayed with message
**And** user can retry by clicking refresh button

**Tasks:**
- Create grid component using Interstate Table component
- Map DataPurgeActivityRow to table columns (7 columns, fixed order)
- Format dates and durations appropriately
- Implement pagination UI with page navigation
- Add loading and error states
- Add test ID `dataPurgeActivityGrid` for e2e
- Write unit tests for grid rendering, pagination, loading states (10+ tests)
- Add column test IDs for e2e: dataPurgeColumnStatus, dataPurgeColumnStartDate, etc.

**Story Points:** 8  
**Priority:** High  
**Testing:** Unit + E2E

---

### Story 3.3: Implement Start Data Purge Button & Confirmation Modal

**User Story:**  
As a Frontend Developer,
I want to implement the "Start Data Purge" button and confirmation modal workflow,
So that users must confirm before triggering a purge.

**Acceptance Criteria:**

**Given** the Start Data Purge button is required
**When** component renders the button
**Then** button is aligned to top-right corner
**And** button text is "Start Data Purge"
**And** button has test ID `triggerDataPurgeButton`
**And** button is disabled while API request is in flight
**And** button has Interstate Button component styling

**Given** user clicks the button
**When** button click event fires
**Then** confirmation modal opens with:
  - Title: "Confirm Purge"
  - Message: "Are you sure you want to start a data purge? This action cannot be undone."
  - Two buttons: "Cancel" and "Confirm"
  - Test ID `dataPurgeStartConfirmModal`
**And** Cancel button (test ID `dataPurgeStartConfirmCancel`) closes modal WITHOUT calling API
**And** Confirm button (test ID `dataPurgeStartConfirmSubmit`) calls API

**Given** user clicks Confirm in modal
**When** API call is made
**Then** it calls `triggerBlocklistDataPurge(environment, commonOrgId, solutionId, solutionDealerId)`
**And** button is disabled and shows loading state
**And** modal closes

**Given** API returns 202 Accepted
**When** response is received
**Then** status alert displays success message: "Purge request accepted. It will run overnight."
**And** success message is shown for 5 seconds
**And** grid automatically refreshes to show new schedule
**And** alert has test ID `dataPurgeStatusAlert` with type="success"

**Given** API returns 409 Conflict
**When** response is received
**Then** status alert displays warning message: "A purge is already scheduled for this organization."
**And** warning message is shown until user dismisses
**And** alert has test ID `dataPurgeStatusAlert` with type="warning"

**Given** API returns error (500, timeout, etc.)
**When** error occurs
**Then** status alert displays error message: "Unable to trigger purge. Please try again."
**And** user can retry by clicking button again
**And** alert has test ID `dataPurgeStatusAlert` with type="error"

**Tasks:**
- Create button component with right-alignment styling
- Create confirmation modal using Interstate Modal component
- Implement confirmation workflow (open modal, handle confirm/cancel)
- Call triggerBlocklistDataPurge hook on confirm
- Implement status alert for 202, 409, 500 responses
- Add appropriate messages and styling for each response type
- Add loading/disabled states for button
- Write 10+ unit tests for button behavior, modal workflow, API calls
- Add e2e tests for user interactions (click button, confirm, see alert)

**Story Points:** 8  
**Priority:** High  
**Testing:** Unit + E2E

---

### Story 3.4: Create Details Modal with Paged Detail Grid

**User Story:**  
As a Frontend Developer,
I want to create a Details modal that displays a paged grid of processed records from the detail endpoint,
So that users can drill down into specific items processed during a purge run.

**Acceptance Criteria:**

**Given** Details link is clicked in grid row
**When** link click event fires
**Then** modal opens with:
  - Title: "Data Purge Details"
  - Subtitle: showing the purge run date and status
  - Close button (test ID `dataPurgeDetailsModalClose`)
  - Modal test ID: `dataPurgeDetailsModal`
**And** modal displays a single paged detail grid

**Given** the details grid is displayed
**When** modal is open
**Then** grid titled "Processed Records" renders with columns: Record Type, Value, Consumer ID, Status
**And** grid displays paginated results from `GET /api/blocklist/purge-history/id/{blocklistScheduleId}/details`
**And** each row shows: record_type (phone/email/address/name), record_value, common_consumer_id, item_status (Processed/Failed/Skipped)
**And** grid is read-only (no edit capability)
**And** grid test ID: `dataPurgeDetailsGrid`
**And** column test IDs: `dataPurgeDetailsTypeColumn`, `dataPurgeDetailsValueColumn`, `dataPurgeDetailsConsumerColumn`, `dataPurgeDetailsStatusColumn`

**Given** the details grid has many records (100+)
**When** modal is open
**Then** pagination controls appear showing: current page, page size, total count
**And** pagination defaults to 25 items per page
**And** pagination allows navigation to next/previous pages

**Given** user changes pagination page
**When** page is navigated
**Then** component calls API with new pageNumber and updates grid
**And** loading spinner shows while fetching
**And** grid updates with new data

**Given** optional filter controls are added
**When** user selects filter option
**Then** component supports filtering by:
  - Record Type: (phone, email, address, name)
  - Status: (Processed, Failed, Skipped)
  - Search Value: (text search on record_value LIKE)
**And** filter changes call API with filter parameters
**And** pagination resets to page 1 on filter change

**Given** user closes modal
**When** Close button or outside click occurs
**Then** modal closes and grid is visible again
**And** previously selected row is still highlighted

**Tasks:**
- Create Details modal component using Interstate Modal
- Create paged grid component for detail records
- Implement data fetching from /api/blocklist/purge-history/id/{id}/details endpoint
- Implement pagination UI with page navigation
- Add optional filter inputs (record type, status, search)
- Implement loading and error states
- Add empty state handling
- Add test IDs for modal, grid, filters
- Write 10+ unit tests for modal opening, grid rendering, pagination, filtering
- Add e2e tests for Details link click and paging workflow

**Story Points:** 10  
**Priority:** High  
**Testing:** Unit + E2E

---

### Story 3.5: Implement Tab Refresh on Entry & Data Synchronization

**User Story:**  
As a Frontend Developer,
I want to implement automatic grid refresh when user enters Data Purge tab,
So that users always see the latest purge history when switching tabs.

**Acceptance Criteria:**

**Given** user navigates to Data Purge tab from another tab
**When** tab becomes active
**Then** `useEffect` hook detects tab activation
**And** getBlocklistDataPurgeHistory API is called to refresh grid data
**And** grid is updated with latest records
**And** loading state shows while data is fetching

**Given** user navigates away from Data Purge tab and comes back
**When** tab is re-activated
**Then** API is called again to refresh (ensures latest data)
**And** previous API request is cancelled if in flight (abort controller)

**Given** user manually clicks a refresh button (if added)
**When** button is clicked
**Then** API is called to refresh grid immediately

**Given** grid is refreshed after successful purge trigger
**When** trigger API returns 202
**Then** grid automatically refreshes without user action
**And** new schedule appears at top of grid

**Tasks:**
- Implement useEffect hook to detect tab activation
- Store selected tab in state/context to track changes
- Call getBlocklistDataPurgeHistory on each tab entry
- Implement abort controller to cancel in-flight requests
- Add refresh button to grid (optional enhancement)
- Write 5+ unit tests for tab activation, refresh logic, abort handling
- Add e2e test for tab switching and data refresh

**Story Points:** 5  
**Priority:** High  
**Testing:** Unit + E2E

---

### Story 3.6: Create use-data-purge Custom Hook

**User Story:**  
As a Frontend Developer,
I want to create a use-data-purge hook that encapsulates data purge business logic,
So that the component remains focused on UI and the hook handles state management.

**Acceptance Criteria:**

**Given** React Query is used for state management
**When** I create the hook `libs/customer/blocklist/src/lib/hooks/use-data-purge.ts`
**Then** hook exports:
  - `triggerPurge()` function to trigger manual purge
  - `isPending` boolean (true while API request in flight)
  - `result` enum ('success' | 'conflict' | 'error' | 'idle')
  - `message` string (user-facing message from API)
  - `activityRows` array of DataPurgeActivityRow
  - `refreshActivity()` function to manually refresh grid

**Given** triggerPurge() is called
**When** function executes
**Then** it calls triggerBlocklistDataPurge() via React Query mutation
**And** isPending is set to true during request
**And** result is set based on response (success, conflict, error)
**And** message is set from API response
**And** isPending is set to false when complete

**Given** refreshActivity() is called
**When** function executes
**Then** it calls getBlocklistDataPurgeHistory() via React Query query
**And** activityRows is updated with response data

**Given** error occurs during trigger or refresh
**When** error is caught
**Then** result is set to 'error'
**And** message contains error description
**And** UI can display error alert

**Tasks:**
- Create hook file with TypeScript types
- Use useMutation for triggerPurge
- Use useQuery for getBlocklistDataPurgeHistory
- Export all required values and functions
- Handle loading, error, and success states
- Implement React Query retry logic (3 retries by default)
- Write 8+ unit tests for hook logic (with mocked API calls)
- Verify hook follows React hooks best practices

**Story Points:** 5  
**Priority:** High  
**Testing:** Unit

---

### Story 3.7: Update blocklist-tabs.tsx to Include Data Purge Tab

**User Story:**  
As a Frontend Developer,
I want to update the blocklist-tabs component to register and display Data Purge tab,
So that the new tab appears alongside existing tabs.

**Acceptance Criteria:**

**Given** the blocklist-tabs component exists
**When** I update `libs/customer/blocklist/src/lib/blocklist-tabs/blocklist-tabs.tsx`
**Then** component adds lazy import: `const DataPurgeTab = lazy(() => import('../tabs/data-purge-tab'))`
**And** component defines new tab constant/enum for Data Purge category
**And** component adds Data Purge to tabs array with label "Data Purge", component reference, and test ID

**Given** tab ordering requirement
**When** tabs are rendered
**Then** order is:
  1. Trending Duplicate Values (if flag enabled)
  2. Dealer Blocklist
  3. System Blocklist
  4. Data Purge

**Given** tab activation event
**When** user clicks Data Purge tab
**Then** Suspense boundary handles lazy loading
**And** Data Purge component renders
**And** tab is marked as active

**Given** existing tab logic
**When** Data Purge tab is added
**Then** existing tabs continue to work unchanged
**And** no regressions to Trending, Dealer, System blocklist tabs
**And** existing tab activation map is preserved

**Tasks:**
- Add lazy import for DataPurgeTab
- Define new tab category constant
- Add Data Purge to tabs array
- Ensure correct ordering in tabs array
- Verify Suspense boundary wraps lazy component
- Write unit tests for tab registration (3-5 tests)
- Add e2e test to verify tab appears when flag enabled
- Regression test existing tabs still work

**Story Points:** 3  
**Priority:** High  
**Testing:** Unit + E2E

---

### Story 3.8: Update testIds.ts with All New Test IDs

**User Story:**  
As a Frontend Developer,
I want to add all new test IDs for Data Purge feature to the centralized testIds file,
So that e2e tests have consistent, maintainable references.

**Acceptance Criteria:**

**Given** the testIds file exists at `libs/customer/blocklist/src/lib/testIds.ts`
**When** I update the file
**Then** I add all test IDs:
  - dataPurgeTab
  - triggerDataPurgeButton
  - dataPurgeStatusAlert
  - dataPurgeActivityGrid
  - dataPurgeActivityGridRow
  - dataPurgeStartConfirmModal
  - dataPurgeStartConfirmSubmit
  - dataPurgeStartConfirmCancel
  - dataPurgeDetailsLink
  - dataPurgeDetailsModal
  - dataPurgeDetailsModalClose
  - dataPurgeDetailsPhoneNumbersGrid
  - dataPurgeDetailsEmailAddressesGrid
  - dataPurgeDetailsStreetAddressesGrid
  - dataPurgeDetailsPhoneNumberColumn
  - dataPurgeDetailsPhoneOccurrencesColumn
  - dataPurgeDetailsEmailAddressColumn
  - dataPurgeDetailsEmailOccurrencesColumn
  - dataPurgeDetailsStreetAddressColumn
  - dataPurgeDetailsStreetOccurrencesColumn
  - dataPurgeColumnStatus
  - dataPurgeColumnStartDate
  - dataPurgeColumnEndDate
  - dataPurgeColumnDuration
  - dataPurgeColumnTotalItemsProcessed
  - dataPurgeColumnStartedBy
  - dataPurgeColumnDetails
**And** all IDs follow naming convention: dataPurge[ComponentName]
**And** IDs are exported and available for component imports

**Tasks:**
- Add all test IDs to testIds.ts
- Verify naming convention consistency
- Export all IDs as named exports
- Update constants to use these IDs in components (from Stories 3.1-3.4)
- Add comment grouping IDs by component (tab, grid, modal, etc.)
- Write unit test to verify all IDs are unique (no duplicates)

**Story Points:** 2  
**Priority:** Medium  
**Testing:** Unit

---

### Story 3.9: Add Unit & E2E Tests for Data Purge Tab

**User Story:**  
As a QA Engineer,
I want comprehensive unit and e2e tests for the Data Purge feature,
So that UI behavior is reliable and regressions are caught before production.

**Acceptance Criteria:**

**Given** all Data Purge components are implemented
**When** I write unit tests
**Then** tests cover:
  - **Tab Component:**
    - Feature flag ON → tab renders
    - Feature flag OFF → tab returns null
    - Test IDs present and correct
  - **Grid Component:**
    - Grid renders with 7 columns and correct headers
    - Data from API populates grid correctly
    - Loading state shows while fetching
    - Error state shows error message with retry
    - Pagination controls work (next/previous page)
    - Column formatting (dates, duration)
    - Clicking Details link opens modal
  - **Confirmation Modal:**
    - Modal opens on button click
    - Cancel button closes modal without API call
    - Confirm button calls trigger API
    - Button disabled during request
  - **Details Modal:**
    - Modal opens with correct title and purge run info
    - Detail grid renders with correct columns: Record Type, Value, Consumer ID, Status
    - Data from detail API endpoint populates grid
    - Pagination controls work (displays page info and page navigation)
    - Optional filters work (record type, status, search value)
    - Grid is read-only (no edit controls)
    - Loading state shows while fetching detail data
    - Empty state handles no-data scenarios
  - **Hook (use-data-purge):**
    - triggerPurge calls mutation
    - refreshActivity calls query
    - isPending, result, message state updates correctly
    - Handles success, conflict, error responses
**And** minimum 80% code coverage on Data Purge files
**And** all unit tests pass

**Given** unit tests pass
**When** I write e2e tests (Cypress)
**Then** tests cover:
  - **Tab Visibility:**
    - Tab appears when feature flag enabled
    - Tab hidden when feature flag disabled
  - **Grid Functionality:**
    - Grid loads and displays data on tab entry
    - Grid refreshes when user re-enters tab
    - Grid displays correct column values
    - Pagination works (navigate pages)
    - Clicking Details link opens modal
  - **Purge Trigger:**
    - Clicking "Start Data Purge" opens confirmation modal
    - Clicking Cancel closes modal without API call
    - Clicking Confirm calls trigger API
    - 202 response shows success alert
    - 409 response shows conflict alert
    - 500 response shows error alert
    - Grid refreshes after successful trigger
  - **Details Popup:**
    - Clicking Details link opens Details modal
    - Details modal displays paged detail grid with columns: Record Type, Value, Consumer ID, Status
    - Grid data loads from details endpoint for selected schedule id
    - Pagination and optional filters work correctly
    - Closing modal returns to grid
**And** all e2e tests pass with API mocks and real database

**Tasks:**
- Write 30+ unit tests (components and hook)
- Write 10+ e2e tests (Cypress)
- Achieve > 80% code coverage
- Use Cypress fixtures for API mocking
- Mock LaunchDarkly flag values in tests
- Test loading, error, and success states
- Test accessibility (keyboard navigation, ARIA attributes)
- Document test scenarios and expected outcomes

**Story Points:** 10  
**Priority:** High  
**Testing:** Unit + E2E

---

### Story 3.10: API Integration Testing & Developer Handoff

**User Story:**  
As a Frontend Developer,
I want end-to-end integration testing with real BFF endpoints,
So that frontend and backend contract is verified before production.

**Acceptance Criteria:**

**Given** BFF endpoints are deployed to test environment (from Epic 2)
**When** I set up test environment configuration
**Then** frontend app points to test BFF base URL
**And** LaunchDarkly flag is enabled via toggle

**Given** frontend is pointing to test BFF
**When** I run e2e tests with real API calls (not mocked)
**Then** tests cover:
  - Grid loads data from real GET /api/blocklist/purge-history endpoint
  - Trigger button calls real POST /api/blocklist/purge endpoint
  - 202 response shows success (real schedule created in database)
  - 409 response shows conflict (real constraint checked)
  - Details popup loads data from real API payload
  - Pagination works with real data
  - All field values match API response contracts
**And** no errors or mismatches between frontend and API

**Given** integration testing is complete
**When** frontend implementation is ready
**Then** handoff document is created with:
  - API endpoint URLs and query parameters
  - Request/response examples
  - Error codes and messages
  - Test account IDs for manual testing
  - Feature flag toggle instructions
  - Known limitations or edge cases
  - Performance baseline (response times)

**Tasks:**
- Configure test environment URLs in frontend
- Run e2e tests against real BFF endpoints
- Capture screenshots of working UI with real data
- Document any contract mismatches found
- Create handoff document for product owner and QA
- Verify grid, button, modal, details all work end-to-end
- Perform manual testing with real data
- Create demo/training video if needed

**Story Points:** 5  
**Priority:** Medium  
**Testing:** Integration

---

## Epic 4: Testing, Integration & Deployment

### Story 4.1: Performance & Load Testing

**User Story:**  
As a QA Engineer,
I want to conduct performance and load testing,
So that the feature can handle expected user load without degradation.

**Acceptance Criteria:**

**Purge History Endpoint Performance:**

**Given** load test with 100 concurrent users
**When** each user calls GET /api/blocklist/purge-history
**Then** response time p95 < 500ms
**And** response time p99 < 1000ms
**And** error rate < 0.1%
**And** database connection pool utilization < 80%

**Purge Trigger Endpoint Performance:**

**Given** load test with 50 concurrent users triggering purge
**When** each user calls POST /api/blocklist/purge
**Then** response time p95 < 200ms
**And** response time p99 < 500ms
**And** error rate < 0.1%
**And** configuration-service remains responsive

**Database Query Performance:**

**Given** parent table with 1000+ purge histories and child table with 500K+ detail rows
**When** summary list query runs (GET /api/blocklist/purge-history)
**Then** query on parent table only executes in < 100ms
**And** query plan shows index seek on `(coorg_id, run_status, started_utc DESC)`
**And** no joins to child table required

**Given** 1000 purge schedules with 500 detail rows per schedule
**When** detail drilldown query runs with `GET /api/blocklist/purge-history/id/{id}/details?pageNumber=1&pageSize=25`
**Then** execution time < 50ms

**Tasks:**
- Set up JMeter or similar load testing tool
- Create load test scenarios (100 concurrent history, 50 concurrent trigger)
- Run load tests and capture metrics
- Analyze query execution plans
- Optimize indexes if needed (from Story 2.1)
- Document performance baselines
- Create performance report with recommendations

**Story Points:** 8  
**Priority:** High  
**Testing:** Performance

---

### Story 4.2: End-to-End Integration Testing

**User Story:**  
As a QA Engineer,
I want end-to-end integration tests that verify flow from UI through all backend layers,
So that the complete feature works as designed.

**Acceptance Criteria:**

**Given** all components are deployed to test environment
**When** user triggers purge from UI
**Then** request flows: UI → BFF → Configuration-Service → Event Publisher → Workflow Consumer
**And** 202 response returned to UI
**And** Schedule created in blocklist_schedules table
**And** Event published to queue (verifiable via event store or logs)
**And** Workflow consumer receives and processes event

**Given** purge is being executed
**When** user refreshes grid on UI
**Then** request flows: UI → BFF → Repository (queries blocklist_purge_history summary table)
**And** Grid displays schedule status reflecting actual workflow progress

**Given** purge completes
**When** user views Details and detail grid loads
**Then** request flows: UI → BFF → Repository (queries blocklist_purge_history_details child table)
**And** Detail grid displays paged records from workflow.blocklist_purge_history_details
**And** Parent summary counts (total_items_processed, phone_records_count, etc.) are persisted and accurate
**And** Child detail rows match parent summary counts through aggregation at completion

**Given** concurrent purge attempts
**When** two users click Start Purge within seconds
**Then** First request succeeds with 202
**And** Second request returns 409 (conflict)
**And** Only one schedule is created (constraint enforced)

**Tasks:**
- Deploy all services to test environment
- Create end-to-end test scenarios
- Verify complete request/response flow
- Check database state after each operation
- Verify event publishing and consumption
- Test concurrent/conflict scenarios
- Document any integration issues found

**Story Points:** 5  
**Priority:** High  
**Testing:** Integration

---

### Story 4.3: Security Testing

**User Story:**  
As a QA Engineer / Security Lead,
I want security testing to ensure the feature prevents unauthorized access and attacks,
So that user data and system integrity are protected.

**Acceptance Criteria:**

**Authorization Testing:**

**Given** User A with access to Org 1 only
**When** User A attempts to trigger or view purge for Org 2
**Then** both requests return 403 Forbidden
**And** User A cannot see Org 2's purge history
**And** User A cannot trigger purge for Org 2

**Given** Unauthenticated request
**When** request is sent without bearer token
**Then** endpoint returns 401 Unauthorized

**SQL Injection Testing:**

**Given** attacker attempts SQL injection in query parameters
**When** payload like `commonOrgId: "'; DROP TABLE users; --"` is sent
**Then** request fails safely (no command execution)
**And** error message does not leak SQL details
**And** database integrity is maintained

**Given** POST request with injection payload
**When** request body contains malicious SQL or code
**Then** request fails safely with validation error
**And** No code execution occurs

**CSRF & XSS Testing:**

**Given** frontend forms and links
**When** CSRF tokens are checked
**Then** tokens are present and validated
**And** state-changing operations (POST) require CSRF token

**Given** user input fields (search, filters)
**When** malicious script is entered
**Then** script is escaped/sanitized before display
**And** No XSS vulnerability exists

**Rate Limiting:**

**Given** rapid trigger requests from single user
**When** user clicks trigger button repeatedly
**Then** requests are rate limited after threshold (e.g., 5 per minute)
**And** subsequent requests return 429 Too Many Requests

**Tasks:**
- Create security test plan
- Test cross-org access prevention
- Test bearer token validation
- Test SQL injection payloads
- Test XSS/CSRF vulnerabilities
- Test rate limiting
- Document any vulnerabilities found and fixes applied
- Perform manual penetration testing if needed

**Story Points:** 8  
**Priority:** High  
**Testing:** Security

---

### Story 4.4: Regression Testing

**User Story:**  
As a QA Engineer,
I want regression tests to verify that new Data Purge feature doesn't break existing blocklist functionality,
So that users continue to use existing features without interruption.

**Acceptance Criteria:**

**Existing Tabs Unchanged:**

**Given** Trending Duplicate Values, Dealer Blocklist, System Blocklist tabs exist
**When** user navigates between tabs
**Then** all existing tabs work as before
**And** no errors or visual regressions
**And** tab data loads and displays correctly

**Add-Rule Auto-Purge Unchanged:**

**Given** user creates a blocklist rule with ScheduleDataPurge=true
**When** rule is submitted
**Then** automatic purge is triggered (existing behavior)
**And** blocklist_schedules record is created as before
**And** purge event is published to queue
**And** Workflow-resource consumer processes it normally

**Purge Business Logic Unchanged:**

**Given** workflow-resource consumer processes purge event
**When** purge runs
**Then** existing business logic executes unchanged
**And** blocklist items are cleared as designed
**And** blocklist_item_results records are created
**And** No new libraries or dependencies break existing functionality

**Database Backward Compatibility:**

**Given** existing blocklist_schedules records from auto-trigger
**When** migration adds new columns
**Then** existing data remains intact and accessible
**And** manual purge coexists with existing schedules
**And** queries still work for both old and new schedules

**Tasks:**
- Run existing blocklist e2e tests (Trending, Dealer, System, add-rule)
- Verify all existing tests still pass
- Run existing database schema tests
- Test auto-trigger flow unchanged
- Check for any breaking changes to APIs
- Document any issues found
- Run full regression suite before canary deployment

**Story Points:** 5  
**Priority:** High  
**Testing:** Regression

---

### Story 4.5: Canary Deployment & Monitoring

**User Story:**  
As a DevOps/Release Engineer,
I want to deploy Data Purge feature with canary strategy and monitoring,
So that risks are minimized and issues are caught quickly in production.

**Acceptance Criteria:**

**Canary Phase 1 (5% Traffic):**

**Given** production environment
**When** 5% of users are routed to new feature
**Then** feature flag controls rollout to 5%
**And** error rate, latency, and business metrics are monitored
**And** monitoring dashboard shows real-time metrics
**Duration:** 24 hours

**If** error rate > 1% or latency doubled:
**Then** rollback is initiated automatically
**And** root cause is investigated

**Canary Phase 2 (25% Traffic):**

**Given** Phase 1 successful
**When** feature is rolled out to 25% of users
**Then** similar monitoring as Phase 1
**And** configuration-service and consumer-card have no performance degradation
**Duration:** 48 hours

**Canary Phase 3 (100% Traffic):**

**Given** Phase 2 successful
**When** feature is rolled out to all users
**Then** feature flag default remains OFF
**And** feature is enabled per-customer or globally via flag

**Monitoring Setup:**

**Metrics Monitored:**
- Endpoint error rate (target < 0.1%)
- Response time p95 < 500ms (history), < 200ms (trigger)
- Database connection pool usage
- Event queue depth
- Workflow purge execution duration
- Feature flag evaluation time
- Backend service availability

**Alerts Triggered On:**
- Error rate > 1%
- Response time p95 > 1000ms
- Database unavailable
- Event queue backed up
- Workflow consumer failures
- Feature flag failures

**Tasks:**
- Set up monitoring dashboard (Prometheus, Grafana, DataDog, etc.)
- Configure canary deployment via feature flag
- Create alerting rules for critical metrics
- Document runbook for incident response
- Prepare rollback procedure
- Test canary deployment in staging first
- Monitor Phase 1, 2, 3 carefully
- Document post-deployment metrics

**Story Points:** 6  
**Priority:** High  
**Testing:** Deployment

---

### Story 4.6: Documentation & Release Notes

**User Story:**  
As a Tech Writer,
I want comprehensive documentation and release notes,
So that product owners, support, and users understand the feature.

**Acceptance Criteria:**

**User-Facing Documentation:**

**Given** the Data Purge feature is ready
**When** documentation is created
**Then** it includes:
  - What is Data Purge? (overview)
  - When to use manual purge vs. auto-purge
  - How to trigger purge (step-by-step)
  - Understanding purge status (`Scheduled`, `Requested`, `Completed`, `Failed`)
  - Viewing purge history and details
  - Troubleshooting common issues
  - FAQ: What happens during purge, how long does it take, etc.

**API Documentation:**

**Given** BFF and configuration-service endpoints
**When** API docs are created
**Then** they include:
  - POST /api/blocklist/purge (with request/response examples)
  - GET /api/blocklist/purge-history (with pagination examples)
  - Status codes and error messages
  - Authorization requirements (bearer token)
  - Rate limiting and quotas
  - Expected response times

**Release Notes:**

**Given** Data Purge feature is deployed
**When** release notes are created
**Then** they include:
  - Feature description and benefits
  - How to enable (feature flag `nx.cxm.show-data-purge-tab`)
  - What's new: Data Purge tab, manual trigger, history grid, details popup
  - System requirements (no breaking changes)
  - Known limitations (e.g., one active purge per org)
  - Performance characteristics
  - Migration guide (if any database changes)
  - Troubleshooting tips
  - Support contact info

**Tasks:**
- Write user guide with screenshots
- Write API documentation
- Write release notes for product release
- Create FAQ document
- Create troubleshooting guide
- Create admin/support documentation if needed
- Get review from product manager and documentation team

**Story Points:** 5  
**Priority:** Medium  
**Testing:** Content review

---

### Story 4.7: Rollback Plan & Disaster Recovery

**User Story:**  
As a DevOps/Release Engineer,
I want a documented rollback plan and disaster recovery procedures,
So that issues can be resolved quickly with minimal downtime.

**Acceptance Criteria:**

**Rollback Procedure:**

**Given** feature is deployed but critical issue arises
**When** rollback is needed
**Then** following steps are executed:
  1. Feature flag `nx.cxm.show-data-purge-tab` is set to OFF globally
  2. All new purge triggers are blocked (feature unavailable to users)
  3. Existing in-progress purges continue to completion
  4. BFF endpoints `/api/blocklist/purge` and `/api/blocklist/purge-history` remain available but feature flag gates them
  5. Frontend is cached, so cache invalidation may be needed
  6. Root cause is investigated

**Data Rollback (if database corruption):**

**Given** database becomes inconsistent
**When** data rollback is needed
**Then** following options exist:
  1. Truncate blocklist_purge_history_details, blocklist_purge_history, blocklist_schedules, and blocklist_item_results, then restore from backup
  2. Or, manually fix corrupted records via SQL scripts
  **Prerequisite:** Post-implementation, establish backup and restore procedures

**Disaster Recovery:**

**Given** production outage
**When** disaster recovery is activated
**Then** following procedures are followed:
  1. Switch to standby environment if available
  2. Restore databases from latest backup
  3. Verify feature flag is OFF (to prevent cascading failures)
  4. Restore configuration from version control
  5. Activate incident response team
  6. Post-incident, root cause analysis is performed

**Tasks:**
- Document rollback procedure step-by-step
- Document who approves rollback (escalation path)
- Create runbook for incident response
- Test rollback procedure in staging
- Document backup and restore procedures
- Create disaster recovery plan with RTO/RPO targets
- Share procedures with operations team

**Story Points:** 3  
**Priority:** Medium  
**Testing:** Procedure review

---

## Summary

**Total Stories:** 47 (7 prep + 7 Epic 1 + 8 Epic 2 + 10 Epic 3 + 15 Epic 4)  
**Total Story Points:** ~240 (allocation varies by team)  
**Timeline:** 8-10 sprints for complete delivery  
**Teams:** Backend (2 teams), Frontend (1 team), QA (1 team), DevOps (1 team)

**Dependencies:**
- Epic 1 → Epic 2 → Epic 3 → Epic 4 (sequential)
- Some stories within each epic can be parallelized

**Key Risks:**
- Database query performance (mitigated by Story 2.1 performance testing)
- Cross-org data leakage (mitigated by Story 4.3 security testing)
- Breaking existing auto-trigger (mitigated by Story 4.4 regression testing)
- Canary deployment issues (mitigated by Story 4.5 monitoring)

---

## For Product Owner: How to Create Stories

**Using This Document:**

1. **Review Each Epic** — Understand the goals and scope
2. **Review Each Story** — Read acceptance criteria thoroughly
3. **Create Jira/Story Issue** — Use story title and user story statement
4. **Acceptance Criteria** — Copy and adapt "Given/When/Then" statements to your testing framework
5. **Tasks/Subtasks** — Use the "Tasks:" section to create subtasks
6. **Story Points** — Validate story points with your team based on capacity
7. **Dependencies** — Link stories to show Epic dependencies (Epic 1 → Epic 2 → Epic 3 → Epic 4)
8. **Estimation** — Team can refine estimates after Sprint 0
9. **Assign Stories** — Assign by team (Backend, Frontend, QA, DevOps)
10. **Track Progress** — Use Burndown and stories board for visibility

**Recommended Sprint Allocation:**
- **Sprint 1-3:** Epic 1 (Backend) — Configuration-Service
- **Sprint 4-6:** Epic 2 (Backend) — Consumer-Card BFF + Epic 3 (Frontend) in parallel
- **Sprint 7-8:** Epic 3 (Frontend) completion + Epic 4 (QA/DevOps) — Testing & Release

---

**Document Status:** ✅ Ready for Product Owner Review  
**Next Step:** PO creates Jira epics and stories from this breakdown
