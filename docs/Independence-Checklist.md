# werkflow-erp-sandbox Independence Checklist

**Purpose**: Ensure werkflow-erp-sandbox remains a standalone, workflow-agnostic data service.

Use this checklist:
- Before creating new features
- In pull request reviews
- Before commits to main branch

---

## Code Review Checklist

### Forbidden Patterns (REJECT if found)

- [ ] **Imports from werkflow**
  ```java
  // FORBIDDEN
  import com.werkflow.engine.*;
  import com.werkflow.admin.*;
  import com.werkflow.portal.*;

  // Search and reject any PR with these imports
  ```

- [ ] **Hardcoded Keycloak client code**
  ```java
  // FORBIDDEN
  new KeycloakAdmin(...);
  keycloakClient.createUser(...);
  realmClient.assignRole(...);

  // Only acceptable: Spring Security JWT validation via application.yml
  ```

- [ ] **External service calls in core logic**
  ```java
  // FORBIDDEN
  adminServiceClient.validateDepartment(deptId);
  keycloakAdminApi.checkUserExists(userId);
  engageServiceRegistry.registerEndpoint(...);

  // Only acceptable: internal repository calls, caches (optional)
  ```

- [ ] **Workflow-specific naming or logic**
  ```java
  // FORBIDDEN
  public void onWorkflowApprove() { ... }
  private boolean isEngineCallback() { ... }
  String processInstanceId = generateFlowableId();

  // Use instead:
  public void updateStatus() { ... }
  String externalProcessId; // generic
  ```

- [ ] **Assumptions about orchestration**
  ```java
  // FORBIDDEN
  if (this.isApprovedByManager()) { notifyProcurement(); }
  applyBudgetCheck(); // embedded business logic
  routeToApprover(requesterManager);

  // Return data only: { available: true, amount: 5000 }
  // Caller decides what to do next
  ```

---

### Required Patterns (REQUIRE in PRs)

- [ ] **Opaque user/tenant ID handling**
  ```java
  // CORRECT
  private String platformUserId;  // Just store it
  // Never validate against external system
  // Never assume format (UUID, email, DN, etc.)
  // Trust the caller
  ```

- [ ] **Optional external process tracking**
  ```java
  // CORRECT
  @Nullable
  private String externalProcessId;  // OPTIONAL
  // For caller's tracking, werkflow-erp-sandbox doesn't use it
  ```

- [ ] **Status enums for state machine**
  ```java
  // CORRECT
  @Enumerated(EnumType.STRING)
  private AssetRequestStatus status;  // PENDING, APPROVED, etc.
  // Status updates are pure state transitions, no business logic
  ```

- [ ] **Multi-tenant scoping on all queries**
  ```java
  // CORRECT
  public List<Employee> findByDepartment(String tenantId, String dept) {
      return repository.findByTenantIdAndDepartmentCode(tenantId, dept);
  }
  // Every query includes tenantId filter
  ```

- [ ] **Idempotency key support on mutations**
  ```java
  // CORRECT
  @PostMapping("/purchase-requests")
  public PrDto create(
    @RequestBody PrCreateRequest request,
    @RequestHeader("X-Idempotency-Key") String idempotencyKey
  ) { ... }
  // Caller can safely retry
  ```

---

## Architecture Decision Review

### Before Implementing a Feature, Answer These

#### 1. Does it require calling an external service?

```
Question: Does this feature need to call Keycloak, Admin, Engine, SAP, etc.?

REJECT if YES: werkflow-erp-sandbox must be self-contained

OK to proceed if NO: Pure data operation
```

#### 2. Does it assume a specific orchestration system?

```
Question: Does this code only work with BPMN / Engine / workflows?

REJECT if YES: werkflow-erp-sandbox must be orchestration-agnostic
  Rewrite as generic status updates

Examples of rejectable code:
  public void onBpmnApproval() { ... }
  if (workflowDefinitionKey == "asset-request") { ... }
  routeToManagerViaWorkflow(...);

OK to proceed if NO: Works with any caller
```

#### 3. Does it expose opaque identifiers?

```
Question: Do user IDs, process IDs, etc. remain opaque strings?

REJECT if you parse/validate them:
  You're making assumptions about format

Examples of rejectable code:
  if (userId.startsWith("keycloak-")) { ... }
  UUID.fromString(platformUserId);  // Assumes UUID format
  validateKeycloakUserExists(userId);

OK to proceed if you just store/return them:
  Caller is responsible for interpretation
```

#### 4. Could this code be tested without werkflow?

```
Question: Can I test this feature without running Engine, Admin, Portal?

REJECT if NO: There's a hidden dependency

Examples of untestable code:
  Constructor requires: EngineClient, AdminClient, KeycloakAdminApi
  Calls external service in the critical path
  Hardcoded assumption about environment (e.g., KEYCLOAK_ADMIN_URL)

OK to proceed if YES: Just provide Repository, database connection
```

#### 5. Would this code need changes if we switched from werkflow to another orchestrator?

```
Question: If client uses Zapier instead of werkflow, would I need to modify this code?

REJECT if YES: You've hardcoded werkflow assumptions

Examples:
  public void handleFlowableCallback() { ... }
  if (source == WorkflowEngine) { ... }

OK to proceed if NO: Code is orchestrator-agnostic
```

---

## Deployment Verification

### Before Releasing to Production

#### 1. Run Import Check
```bash
#!/bin/bash
echo "Checking for forbidden imports..."

FORBIDDEN=(
  "com.werkflow.engine"
  "com.werkflow.admin"
  "com.werkflow.portal"
  "org.keycloak.admin.client"
)

FOUND_FORBIDDEN=0
for pattern in "${FORBIDDEN[@]}"; do
  if grep -r "import.*$pattern" services/business/src/main/java ; then
    echo "FORBIDDEN: Found import of $pattern"
    FOUND_FORBIDDEN=1
  fi
done

if [ $FOUND_FORBIDDEN -eq 0 ]; then
  echo "PASS: No forbidden imports found"
else
  echo "FAIL: Remove all forbidden imports before release"
  exit 1
fi
```

#### 2. Verify Standalone Operation
```bash
# Start only werkflow-erp-sandbox and PostgreSQL (no werkflow platform)
docker compose up -d postgres werkflow-erp-sandbox

# Wait for startup
sleep 10

# Test without JWT (should 401)
curl http://localhost:8084/api/v1/hr/employees
# Expected: {"code": "UNAUTHORIZED", ...}

# Test with valid JWT (should work)
TOKEN=$(curl -s -X POST http://localhost:8090/realms/werkflow/protocol/openid-connect/token \
  --data-urlencode "grant_type=client_credentials" \
  --data-urlencode "client_id=werkflow-api" \
  --data-urlencode "client_secret=secret" | jq -r '.access_token')

curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8084/api/v1/hr/employees
# Expected: {"content": [...], "pageable": {...}}

echo "PASS: werkflow-erp-sandbox works standalone"
```

#### 3. Verify Multi-Tenant Isolation
```bash
# Create employee in tenant A
curl -H "Authorization: Bearer $TENANT_A_TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -X POST http://localhost:8084/api/v1/hr/employees \
  -d '{"firstName": "Alice", "lastName": "Smith", ...}'

# Attempt to read from tenant B
curl -H "Authorization: Bearer $TENANT_B_TOKEN" \
  -H "X-Tenant-ID: other-corp" \
  http://localhost:8084/api/v1/hr/employees

# Expected: Empty list or 403 Forbidden (not Alice's data)
# FAIL if Alice appears: multi-tenant isolation is broken
```

#### 4. Verify Idempotency
```bash
# POST with idempotency key
curl -X POST http://localhost:8084/api/v1/procurement/purchase-requests \
  -H "X-Idempotency-Key: abc-123" \
  -H "X-Tenant-ID: acme-corp" \
  -d '{"priority": "HIGH", ...}'
# Expected: 201 Created, response body X

# Retry with same key
curl -X POST http://localhost:8084/api/v1/procurement/purchase-requests \
  -H "X-Idempotency-Key: abc-123" \
  -H "X-Tenant-ID: acme-corp" \
  -d '{"priority": "HIGH", ...}'
# Expected: 200 OK, same response body X (not 201, not duplicated)

echo "PASS: Idempotency works correctly"
```

---

## Common Pitfalls (Anti-Patterns)

### Pitfall 1: Calling External Services in Service Layer

```java
// WRONG
@Service
public class EmployeeService {

    public Employee create(EmployeeRequest request) {
        // Validate department exists in Admin Service
        adminClient.validateDepartment(request.getDeptId());  // COUPLING

        Employee emp = new Employee();
        emp.setDepartmentId(request.getDeptId());
        return repository.save(emp);
    }
}

// RIGHT
@Service
public class EmployeeService {

    public Employee create(EmployeeRequest request) {
        // FK validation only (internal repository)
        if (!departmentRepository.existsById(request.getDeptId())) {
            throw new EntityNotFoundException("Department not found");
        }

        Employee emp = new Employee();
        emp.setDepartmentId(request.getDeptId());
        return repository.save(emp);
    }
}
```

### Pitfall 2: Embedding Business Logic

```java
// WRONG
public void approveAssetRequest(Long assetRequestId) {
    AssetRequest request = repository.findById(assetRequestId);

    // Business logic embedded here — this is NOT werkflow-erp-sandbox's concern
    if (shouldCheckBudget(request)) {         // NOT OUR CONCERN
        if (!budgetAvailable(request)) {       // BUSINESS RULE
            notifyManager(request);            // NOTIFICATION - NOT OUR CONCERN
            return;                            // CONDITIONAL LOGIC
        }
    }

    request.setStatus(APPROVED);
    repository.save(request);
}

// RIGHT
public void updateStatus(Long assetRequestId, String newStatus) {
    AssetRequest request = repository.findById(assetRequestId);

    // Pure status update
    request.setStatus(newStatus);
    repository.save(request);

    // That's it. Caller decides what happens next.
}
```

### Pitfall 3: Hardcoding Workflow References

```java
// WRONG
@Nullable
private String processInstanceId;  // Workflow-specific naming

public void onBpmnCompletion() { ... }  // Workflow-specific method name

// RIGHT
@Nullable
private String externalProcessId;  // Generic, works with any system

public void updateStatus(String newStatus) { ... }  // Generic method
```

### Pitfall 4: User ID Validation

```java
// WRONG
public CustodyRecord assign(String custodianUserId) {
    // Validate user exists in Keycloak
    if (!keycloakAdmin.userExists(custodianUserId)) {  // COUPLING
        throw new EntityNotFoundException("User not found in Keycloak");
    }
    // ... rest of code
}

// RIGHT
public CustodyRecord assign(String custodianUserId) {
    // Trust the caller. User ID is opaque.
    CustodyRecord record = new CustodyRecord();
    record.setCustodianUserId(custodianUserId);  // Just store it
    return repository.save(record);

    // Caller (werkflow, SAP, etc.) is responsible for validating user ID
}
```

---

## Summary

| Aspect | Allowed | Forbidden |
|--------|---------|----------|
| **Imports** | Spring, Lombok, JPA | werkflow.*, keycloak.admin.* |
| **External calls** | None (internal repos only) | Admin, Engine, Keycloak APIs |
| **Business logic** | FK validation, state machines | Approvals, budget checks, routing |
| **User IDs** | Store as opaque strings | Validate, parse, enrich, query external systems |
| **Status updates** | Simple state transitions | Conditional logic, workflows, notifications |
| **Testing** | No external dependencies needed | Requires specific system (Engine, Keycloak) |

---

## Questions?

If you're unsure about a design, ask:

1. **"Does this code need to be removed if we replace werkflow with Zapier?"**
   - If YES: it has forbidden coupling

2. **"Can this code be tested without starting the Engine service?"**
   - If NO: it has forbidden dependencies

3. **"Does this make assumptions about who's calling this API?"**
   - If YES: it has forbidden coupling

If the answer to any of these is "no", the design needs to change.
