# P0.6: Pagination on List Endpoints Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Spring Data pagination to all ~18-20 GET list endpoints across 4 domains (HR, Finance, Procurement, Inventory), allowing clients to request pages via `?page=0&size=20&sort=createdAt,desc`.

**Architecture:** Configure Spring Data pagination defaults  (calls) update repositories to accept `Pageable`  (calls) update services to accept and return `Page<Dto>`  (calls) update controllers to expose `Pageable` parameter and return `Page<Dto>`  (calls) write integration tests for pagination behavior.

**Tech Stack:** Spring Data JPA (PagingAndSortingRepository), Springdoc OpenAPI (@ParameterObject), JUnit 5, MockMvc

---

## File Structure

**Configuration:**
- Modify: `services/business/src/main/resources/application.yml` — add spring.data.web.pageable config

**Repositories (no new files, modify existing):**
- Modify: `services/business/src/main/java/com/werkflow/business/hr/repository/*.java` — update findBy* methods (5 repos)
- Modify: `services/business/src/main/java/com/werkflow/business/finance/repository/*.java` — update (4 repos)
- Modify: `services/business/src/main/java/com/werkflow/business/procurement/repository/*.java` — update (4 repos)
- Modify: `services/business/src/main/java/com/werkflow/business/inventory/repository/*.java` — update (5 repos)

**Services (no new files, modify existing):**
- Modify: `services/business/src/main/java/com/werkflow/business/hr/service/*.java` — 6 services
- Modify: `services/business/src/main/java/com/werkflow/business/finance/service/*.java` — 5 services
- Modify: `services/business/src/main/java/com/werkflow/business/procurement/service/*.java` — 4 services
- Modify: `services/business/src/main/java/com/werkflow/business/inventory/service/*.java` — 5 services

**Controllers (no new files, modify existing):**
- Modify: `services/business/src/main/java/com/werkflow/business/hr/controller/*.java` — 6 controllers
- Modify: `services/business/src/main/java/com/werkflow/business/finance/controller/*.java` — 5 controllers
- Modify: `services/business/src/main/java/com/werkflow/business/procurement/controller/*.java` — 4 controllers
- Modify: `services/business/src/main/java/com/werkflow/business/inventory/controller/*.java` — 5 controllers

**Tests (modify existing, add new as needed):**
- Modify/Create: `services/business/src/test/java/com/werkflow/business/*/controller/*ControllerIntegrationTest.java` — pagination tests for all domains

---

## Task Clusters

### Cluster 1: Configuration

**Task 1: Configure Spring Data Pagination Defaults**

Configure `application.yml` with Spring Data pagination defaults (size=20, max=1000, 0-indexed pages).

---

### Cluster 2: Repositories (Parallel)

**Task 2: Update HR Repositories**
Update 6 HR repositories (Employee, Department, Leave, Attendance, Payroll, PerformanceReview) to accept `Pageable` parameter and return `Page<Entity>`.

**Task 3: Update Finance Repositories**
Update 5 Finance repositories (BudgetPlan, BudgetCategory, BudgetLineItem, Expense, ApprovalThreshold) to accept `Pageable` parameter and return `Page<Entity>`.

**Task 4: Update Procurement Repositories**
Update 4 Procurement repositories (PurchaseRequest, PurchaseOrder, Receipt, Vendor) to accept `Pageable` parameter and return `Page<Entity>`.

**Task 5: Update Inventory Repositories**
Update 7 Inventory repositories (AssetRequest, AssetInstance, CustodyRecord, TransferRequest, MaintenanceRecord, AssetCategory, AssetDefinition) to accept `Pageable` parameter and return `Page<Entity>`.

---

### Cluster 3: Services (Parallel)

**Task 6: Update HR Services**
Update 6 HR services (Employee, Department, Leave, Attendance, Payroll, PerformanceReview) to accept `Pageable` and return `Page<Dto>`.

**Task 7: Update Finance Services**
Update 5 Finance services (BudgetPlan, BudgetCategory, BudgetLineItem, Expense, ApprovalThreshold) to accept `Pageable` and return `Page<Dto>`.

**Task 8: Update Procurement Services**
Update 4 Procurement services (PurchaseRequest, PurchaseOrder, Receipt, Vendor) to accept `Pageable` and return `Page<Dto>`.

**Task 9: Update Inventory Services**
Update 7 Inventory services (AssetRequest, AssetInstance, CustodyRecord, TransferRequest, MaintenanceRecord, AssetCategory, AssetDefinition) to accept `Pageable` and return `Page<Dto>`.

---

### Cluster 4: Controllers (Parallel)

**Task 10: Update HR Controllers**
Update 6 HR controllers with @ParameterObject Pageable, return Page<Dto>, and @Operation documentation.

**Task 11: Update Finance Controllers**
Update 5 Finance controllers with @ParameterObject Pageable, return Page<Dto>, and @Operation documentation.

**Task 12: Update Procurement Controllers**
Update 4 Procurement controllers with @ParameterObject Pageable, return Page<Dto>, and @Operation documentation.

**Task 13: Update Inventory Controllers**
Update 7 Inventory controllers with @ParameterObject Pageable, return Page<Dto>, and @Operation documentation.

---

### Cluster 5: Testing & Verification

**Task 14: Write Integration Tests**
Add pagination integration tests for all domains (HR, Finance, Procurement, Inventory) covering default pagination, custom page size, page bounds, size capping, and custom sort.

**Task 15: Final Verification**
Run full test suite, verify Swagger documentation, and update ROADMAP.md to mark P0.6 complete.

---

## Context for Implementers

**Project:** werkflow-erp — standalone ERP microservice (Spring Boot 3.x, Java 17, PostgreSQL, JPA)

**Current Status:**
- P0.1-P0.5 complete (multi-tenancy, idempotency, FK validation, API versioning)
- All GET list endpoints currently return `List<Dto>` (no pagination)
- Services use `TenantContext` to extract tenantId from JWT

**Pattern to follow:**
```java
// Repository: findByTenantId returns Page instead of List
Page<Employee> findByTenantId(String tenantId, Pageable pageable);

// Service: accept Pageable, return Page
public Page<EmployeeResponse> getAllEmployees(Pageable pageable) {
    String tenantId = TenantContext.getTenantId();
    return employeeRepository.findByTenantId(tenantId, pageable)
        .map(this::toResponse);
}

// Controller: use @ParameterObject, return Page
@GetMapping
@Operation(summary = "Get all", parameters = {
    @Parameter(name = "page", description = "0-indexed page number"),
    @Parameter(name = "size", description = "Page size (max 1000)"),
    @Parameter(name = "sort", description = "Sort criteria (e.g., createdAt,desc)")
})
public ResponseEntity<Page<EmployeeResponse>> getAll(@ParameterObject Pageable pageable) {
    return ResponseEntity.ok(service.getAll(pageable));
}
```

**Test expectations:**
- Default: 20 items per page, 0-indexed
- Max size capped at 1000
- Supports custom sort (e.g., ?sort=firstName,asc)
- Page metadata includes totalElements, totalPages, number, size, first, last

**Verification:**
- All tests pass: `mvn clean test`
- Swagger shows Pageable parameters and Page response schema
- No breaking changes to single-resource endpoints (e.g., GET /{id})

---
