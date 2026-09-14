# P0.6: Pagination on List Endpoints — Design Spec

**Date**: 2026-04-07
**Status**: Design Approved
**Phase**: P0.6 (Critical Path to MVP)

---

## Overview

Add Spring Data pagination to all GET list endpoints across 4 domains (HR, Finance, Procurement, Inventory).

**Scope**: ~18-20 list endpoints (one per controller, excluding single-resource GETs like `/{id}`)

**Result**: Clients can request pages via `?page=0&size=20&sort=createdAt,desc` and receive `Page<Dto>` responses with metadata (totalElements, totalPages, currentPage, size).

---

## Architecture & Implementation

### Return Type: Spring Data Page<T>

Use Spring's built-in `Page<T>` wrapper instead of custom response types.

```java
// Before
@GetMapping
public ResponseEntity<List<EmployeeResponse>> getAllEmployees() {
    return ResponseEntity.ok(employeeService.getAllEmployees());
}

// After
@GetMapping
public ResponseEntity<Page<EmployeeResponse>> getAllEmployees(
    @ParameterObject Pageable pageable) {
    return ResponseEntity.ok(employeeService.getAllEmployees(pageable));
}
```

**Benefits**:
- Springdoc OpenAPI auto-documents `Page` perfectly
- Spring Data provides metadata automatically (totalElements, totalPages, number, size)
- Zero custom serialization needed
- Standard Spring API contract

### Pageable Parameter

Use Springdoc's `@ParameterObject` annotation to auto-document pagination params in Swagger.

```java
@GetMapping
@Operation(summary = "Get all employees", parameters = {
    @Parameter(name = "page", description = "0-indexed page number"),
    @Parameter(name = "size", description = "Page size (max 1000)"),
    @Parameter(name = "sort", description = "Sort criteria (e.g., createdAt,desc)")
})
public ResponseEntity<Page<EmployeeResponse>> getAllEmployees(
    @ParameterObject Pageable pageable) {
    return ResponseEntity.ok(employeeService.getAllEmployees(pageable));
}
```

**Query examples**:
- `GET /api/v1/hr/employees`  (calls) Default: page=0, size=20, sort=createdAt DESC
- `GET /api/v1/hr/employees?page=1&size=50`  (calls) Page 2 (1-indexed in UI), 50 items per page
- `GET /api/v1/hr/employees?sort=firstName,asc&sort=createdAt,desc`  (calls) Sort by firstName ASC, then createdAt DESC
- `GET /api/v1/hr/employees?size=2000`  (calls) Capped at 1000 by config

### Configuration (application.yml)

```yaml
spring:
  data:
    web:
      pageable:
        default-page-size: 20
        max-page-size: 1000
        one-indexed-parameters: false  # URLs use 0-indexed page, not 1-indexed
```

These settings apply globally to all `@ParameterObject Pageable` parameters.

### Service Layer Changes

Services accept `Pageable` and return `Page<Dto>`.

```java
@Service
public class EmployeeService {

    @Autowired
    private EmployeeRepository repository;

    public Page<EmployeeResponse> getAllEmployees(Pageable pageable) {
        String tenantId = TenantContext.getTenantId();
        return repository.findByTenantId(tenantId, pageable)
            .map(this::toResponse);
    }
}
```

### Repository Layer

Repositories inherit `PagingAndSortingRepository` from Spring Data, which provides `Page<T> findAll(Pageable)`.

**No changes needed.** Update existing methods to accept Pageable:

```java
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    // Existing single-result methods (unchanged)
    Optional<Employee> findByIdAndTenantId(Long id, String tenantId);

    // List method: add Pageable parameter
    Page<Employee> findByTenantId(String tenantId, Pageable pageable);
}
```

---

## Endpoints Affected

**Total list endpoints**: ~18-20 across 4 domains

### HR Domain
- EmployeeController.getAllEmployees()
- DepartmentController.getAllDepartments()
- LeaveController.getAllLeaves()
- AttendanceController.getAllAttendance()
- PayrollController.getAllPayrolls()
- PerformanceReviewController.getAllPerformanceReviews()

### Finance Domain
- BudgetPlanController.getAllBudgetPlans()
- BudgetCategoryController.getAllBudgetCategories()
- BudgetLineItemController.getAllBudgetLineItems()
- ExpenseController.getAllExpenses()
- ApprovalThresholdController.getAllApprovalThresholds()

### Procurement Domain
- PurchaseRequestController.getAllPurchaseRequests()
- PurchaseOrderController.getAllPurchaseOrders()
- ReceiptController.getAllReceipts()
- VendorController.getAllVendors()

### Inventory Domain
- AssetRequestController.getAllAssetRequests()
- AssetInstanceController.getAllAssetInstances()
- CustodyRecordController.getAllCustodyRecords()
- TransferRequestController.getAllTransferRequests()
- MaintenanceRecordController.getAllMaintenanceRecords()
- AssetCategoryController.getAllAssetCategories()
- AssetDefinitionController.getAllAssetDefinitions()

---

## Error Handling & Edge Cases

**Invalid size (exceeds max):**
- `?size=5000`  (calls) Capped to 1000 (handled by `max-page-size` config, no error)

**Page out of bounds:**
- `?page=999`  (calls) Returns empty `Page` with correct metadata (totalPages, totalElements still accurate)

**Invalid sort field:**
- `?sort=invalidField`  (calls) Returns HTTP 400 Bad Request (Spring default, handled by `PropertyReferenceException`)

**Negative page/size:**
- `?page=-1` or `?size=-1`  (calls) Spring validation rejects (400 Bad Request)

No special handling code needed — Spring handles these by default.

---

## Swagger/OpenAPI Documentation

Springdoc's `@ParameterObject Pageable` generates:

```json
{
  "page": {
    "type": "integer",
    "format": "int32",
    "default": 0,
    "description": "0-indexed page number"
  },
  "size": {
    "type": "integer",
    "format": "int32",
    "default": 20,
    "description": "Page size (max 1000)"
  },
  "sort": {
    "type": "array",
    "items": { "type": "string" },
    "description": "Sort criteria (e.g., createdAt,desc)"
  }
}
```

Response type becomes `Page<EmployeeResponse>`:

```json
{
  "content": [
    { "id": 1, "firstName": "Alice", ... },
    { "id": 2, "firstName": "Bob", ... }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 20,
    "offset": 0,
    "paged": true,
    "unpaged": false
  },
  "totalElements": 245,
  "totalPages": 13,
  "last": false,
  "size": 20,
  "number": 0,
  "numberOfElements": 20,
  "first": true,
  "empty": false
}
```

---

## Testing Strategy

### Unit Tests (Service Layer)

Test that services return correct Page metadata:

```java
@Test
void testGetAllEmployeesReturnsPage() {
    Pageable pageable = PageRequest.of(0, 20, Sort.by("createdAt").descending());

    Page<EmployeeResponse> result = service.getAllEmployees(pageable);

    assertEquals(0, result.getNumber());  // page number
    assertEquals(20, result.getSize());   // page size
    assertEquals(1, result.getTotalPages());
    assertFalse(result.isEmpty());
}

@Test
void testGetAllEmployeesWithTenantIsolation() {
    TenantContext.setTenantId("tenant-1");
    Pageable pageable = PageRequest.of(0, 10);

    Page<EmployeeResponse> result = service.getAllEmployees(pageable);

    assertTrue(result.getContent().stream()
        .allMatch(e -> e.getTenantId().equals("tenant-1")));
}
```

### Integration Tests (Controller + Repository)

Test HTTP pagination params:

```java
@Test
void testGetAllEmployeesWithPagination() {
    mvc.perform(get("/api/v1/hr/employees")
        .param("page", "0")
        .param("size", "10")
        .param("sort", "firstName,asc")
        .header("Authorization", "Bearer " + validToken))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.totalElements").exists())
        .andExpect(jsonPath("$.content", hasSize(10)));
}

@Test
void testGetAllEmployeesCapsSizeAt1000() {
    mvc.perform(get("/api/v1/hr/employees")
        .param("size", "5000")
        .header("Authorization", "Bearer " + validToken))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.size", is(1000)));
}
```

---

## Implementation Order

1. **Configuration**: Add `spring.data.web.pageable.*` to `application.yml`
2. **Repositories**: Update `findBy*` methods to accept `Pageable` parameter (may already be there)
3. **Services**: Update all `getAll*` methods to accept `Pageable` and return `Page<Dto>`
4. **Controllers**: Update all list endpoints:
   - Add `@ParameterObject Pageable pageable` parameter
   - Change return type to `Page<Dto>`
   - Add `@Operation` documentation
5. **Tests**: Write unit + integration tests for pagination behavior
6. **Verification**: Run full test suite (`mvn clean test`)

---

## Migration Path (Backward Compatibility)

**Breaking change**: List endpoints now return `Page<Dto>` instead of `List<Dto>`.

Clients must update:
```javascript
// Before
const employees = response;  // Array
employees.forEach(e => ...)

// After
const employees = response.content;  // Array inside Page
employees.forEach(e => ...)
```

This is acceptable for P0.6 since:
- API versioning (`/api/v1`) allows breaking changes within a major version
- Clients are internal (werkflow, not public API)
- Full documentation provided in OpenAPI schema

---

## Future Enhancements (Post-MVP)

- [ ] **Projection queries**: `?fields=id,firstName,email` to reduce payload size
- [ ] **Advanced filtering**: `?firstName=Alice&department=ENG` on specific endpoints
- [ ] **Cursor-based pagination**: For high-cardinality datasets (alternative to offset-based)
- [ ] **Default sort by lastModified DESC**: Better for real-time UI updates

---
