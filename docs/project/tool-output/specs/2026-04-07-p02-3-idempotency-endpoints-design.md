# P0.2.3 Design: Document Idempotency-Key Header in POST Endpoints

**Date:** 2026-04-07
**Status:** Approved
**Phase:** P0 — Critical Path to Production
**Scope:** Single-object creation endpoints only (exclude bulk/batch)

---

## Overview

Add Idempotency-Key header documentation to all single-object creation (@PostMapping) endpoints across all 25 REST controllers. This makes idempotency discoverable in Swagger/OpenAPI docs and encourages safe retry practices by clients, with no behavioral changes (the IdempotencyFilter already handles caching).

---

## Goals

1. **Discoverability** — Clients can see idempotency support in auto-generated API docs
2. **Best practices** — Clear documentation on how to use Idempotency-Key for safe retries
3. **Optional adoption** — Header is optional; existing clients continue to work; new clients can opt in
4. **Zero behavior change** — No changes to business logic, only annotations and documentation

---

## What Stays the Same

The **IdempotencyFilter** (P0.2.2) already handles all caching logic:
- Detects `Idempotency-Key` header
- Checks cache for duplicate requests
- Returns cached response on hit (after payload validation)
- Stores successful (2xx) responses
- Returns 409 Conflict if payload differs

This phase adds **documentation only**, no filter/service changes.

---

## Changes Required

### Pattern: Every Single-Object POST Endpoint

**Before:**
```java
@PostMapping
@Operation(summary = "Create new purchase request")
@PreAuthorize("isAuthenticated()")
public ResponseEntity<PurchaseRequestResponse> createPurchaseRequest(@Valid @RequestBody PurchaseRequestRequest request) {
    return ResponseEntity.status(HttpStatus.CREATED).body(prService.createPurchaseRequest(request));
}
```

**After:**
```java
@PostMapping
@Operation(
    summary = "Create new purchase request",
    description = "Supports idempotent creation via Idempotency-Key header. " +
        "Provide a unique idempotency key to safely retry failed requests without duplicating the resource. " +
        "If the key is omitted, each request is processed independently. " +
        "If the same key is used with different payloads, a 409 Conflict is returned."
)
@PreAuthorize("isAuthenticated()")
public ResponseEntity<PurchaseRequestResponse> createPurchaseRequest(
    @Valid @RequestBody PurchaseRequestRequest request,
    @RequestHeader(name = "Idempotency-Key", required = false) String idempotencyKey) {
    return ResponseEntity.status(HttpStatus.CREATED).body(prService.createPurchaseRequest(request));
}
```

**Key modifications:**
1. Expand `@Operation` summary with `description` parameter explaining idempotency behavior
2. Add `@RequestHeader(name = "Idempotency-Key", required = false)` parameter to method signature
3. Parameter name `idempotencyKey` — Spring maps header automatically to method parameter
4. **No changes to method body** — parameter is unused; IdempotencyFilter reads the header directly

### Why Add the Parameter?

Even though the parameter is unused in the method body, adding it to the signature has two benefits:
1. **Swagger documentation** — OpenAPI generator includes the parameter in the API spec
2. **Code clarity** — Readers understand this endpoint supports idempotency

---

## Endpoints to Update

### Scope Definition

**Include:**
- All `@PostMapping` methods that create a **single object** and return a 201 Created response

**Exclude:**
- Bulk/batch POST endpoints (create multiple items in one request) — none identified in current codebase
- GET, DELETE, PATCH endpoints
- Endpoints that return paginated lists

### Expected Controller Count: ~15-20 Single-Object POST Endpoints

**Procurement Domain:**
- `PurchaseRequestController.createPurchaseRequest()`
- `PurchaseOrderController.createPurchaseOrder()`
- `ReceiptController.createReceipt()`
- `VendorController.createVendor()`

**Inventory Domain:**
- `AssetRequestController.createAssetRequest()`
- `AssetInstanceController.createAssetInstance()`
- `CustodyRecordController.createCustodialRecord()` (or equivalent)
- `TransferRequestController.createTransferRequest()`
- `MaintenanceRecordController.createMaintenanceRecord()`
- `AssetCategoryController.createAssetCategory()`
- `AssetDefinitionController.createAssetDefinition()`

**Finance Domain:**
- `BudgetPlanController.createBudgetPlan()`
- `ExpenseController.createExpense()`
- `BudgetCategoryController.createBudgetCategory()` (if exists)

**HR Domain:**
- `EmployeeController.createEmployee()`
- `DepartmentController.createDepartment()`
- `LeaveController.createLeave()` (or equivalent)
- `AttendanceController.createAttendance()` (or equivalent)
- `PerformanceReviewController.createPerformanceReview()` (if exists)
- `PayrollController.createPayroll()` (if exists)

---

## Behavior: No Changes

### Idempotent Request (With Header)
```
POST /api/v1/procurement/purchase-requests
Idempotency-Key: abc-123-def
Content-Type: application/json

{"vendorId": "v1", "amount": 1000}

 (calls) Response 201 Created, cached

[Client retries with same key and payload]

 (calls) Response 201 Created (from cache, no duplicate PR created)

[Client retries with same key but different payload]

 (calls) Response 409 Conflict
```

### Non-Idempotent Request (Without Header)
```
POST /api/v1/procurement/purchase-requests
Content-Type: application/json

{"vendorId": "v1", "amount": 1000}

 (calls) Response 201 Created, NOT cached

[Client retries without header]

 (calls) Response 201 Created (new PR created — duplicate)
```

**No behavior change** — IdempotencyFilter handles all caching; this phase only adds documentation.

---

## Testing Strategy

### No Unit/Integration Tests Required
- IdempotencyFilterTest (P0.2) already validates caching behavior
- No new business logic to test

### Swagger Validation (Manual)
1. Run application: `mvn spring-boot:run`
2. Open Swagger UI: `http://localhost:8084/swagger-ui.html`
3. Verify updated endpoints display:
   - Idempotency-Key parameter in request section
   - Description text explains idempotency support
4. Test endpoint calls with/without header to verify caching still works

### Example Swagger Check
- Endpoint: `POST /api/v1/procurement/purchase-requests`
- Should show:
  - **Parameter**: `Idempotency-Key` (header, optional, string)
  - **Description**: "Supports idempotent creation via Idempotency-Key header. Provide a unique..."

---

## Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Missing an endpoint — some POST methods not updated | Systematic scan: grep for `@PostMapping` in all controllers, verify each manually |
| Parameter name mismatch — Spring can't map header if name is wrong | Use exact name `Idempotency-Key` in annotation; parameter name `idempotencyKey` follows convention |
| Swagger rendering issues — docs don't show parameter | Test on actual Swagger UI after build; common issue if annotation syntax is wrong |
| Confusion about optional parameter — clients think it's required | Clear description: "If key is omitted, each request is processed independently" |

---

## Effort Estimate

- **Scanning & identifying endpoints:** 30 minutes
- **Updating ~15-20 endpoints:** 1-2 hours (formulaic changes)
- **Swagger validation:** 15 minutes
- **Testing:** 15 minutes
- **Total:** ~2-2.5 hours

---

## Files to Modify

All controller files across 4 domains:

**Procurement:**
- `PurchaseRequestController.java`
- `PurchaseOrderController.java`
- `ReceiptController.java`
- `VendorController.java`

**Inventory:**
- `AssetRequestController.java`
- `AssetInstanceController.java`
- `CustodyRecordController.java`
- `TransferRequestController.java`
- `MaintenanceRecordController.java`
- `AssetCategoryController.java`
- `AssetDefinitionController.java`

**Finance:**
- `BudgetPlanController.java`
- `ExpenseController.java`
- (Additional finance controllers as identified)

**HR:**
- `EmployeeController.java`
- `DepartmentController.java`
- (Additional HR controllers as identified)

---

## Success Criteria

[YES] All single-object POST endpoints include:
- `@RequestHeader(name = "Idempotency-Key", required = false)` parameter
- Enhanced `@Operation` description with idempotency explanation

[YES] Swagger/OpenAPI docs render the parameter correctly

[YES] No behavioral changes — existing clients unaffected

[YES] Idempotency still works — filter intercepts header, caches responses

---

## Related Documents

- `docs/superpowers/specs/2026-04-07-p02-idempotency-design.md` — Full P0.2 idempotency spec
- `ROADMAP.md` — P0.2.3 task definition
