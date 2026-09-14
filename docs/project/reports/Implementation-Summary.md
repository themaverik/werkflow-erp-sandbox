# Implementation Summary: Two Key Architecture Decisions

**Date**: 2026-04-06
**For**: werkflow-erp-sandbox independence and platform user integration

---

## The Two Key Questions (Answered)

### 1. How Do We Link Platform Users with HR Employees (Without Coupling)?

**The Pattern: Optional `platformUserId` Field**

werkflow-erp-sandbox does NOT manage platform users. Instead:

1. **HR Employee is managed by werkflow-erp-sandbox**
   ```java
   @Entity
   public class Employee {
       private Long id;
       private String firstName;
       private String lastName;
       private String email;
       private String departmentCode;
       private Long managerId;  // FK to another Employee

       @Nullable
       private String platformUserId;  // ← NEW: Optional link
   }
   ```

2. **External system (werkflow, SAP, Azure, etc.) links the two**
   ```
   External Platform creates/manages user
        (next step)
   Calls: PATCH /api/v1/hr/employees/{id}/platform-link
   Body: { platformUserId: "user-uuid-123" }
        (next step)
   werkflow-erp-sandbox stores the link
   ```

3. **Query by platform user ID**
   ```
   Later, when workflow needs employee data:
   GET /api/v1/hr/employees/platform/user-uuid-123
   Response: { id: 42, firstName: "Alice", departmentCode: "ENG", ... }
   ```

**Why This Works:**

| Aspect | Benefit |
|--------|---------|
| **No coupling** | werkflow-erp-sandbox never imports Keycloak, Azure AD, or any IAM library |
| **Works with any platform** | Keycloak, Azure AD, LDAP, custom system — all use the same API |
| **Works without platform** | `platformUserId = null` is valid. Employees can exist without platform access. |
| **Event-driven** | External system drives the linking, not werkflow-erp-sandbox |
| **Bidirectional queries** | Find employees by platformUserId OR find platformUserId by employee |

**Implementation Checklist:**

```
[ ] Add platformUserId field to Employee entity
[ ] Create endpoint: PATCH /api/v1/hr/employees/{id}/platform-link
[ ] Create endpoint: DELETE /api/v1/hr/employees/{id}/platform-link
[ ] Create query: GET /api/v1/hr/employees/platform/{platformUserId}
[ ] Create batch endpoint: POST /api/v1/hr/employees/platform-link-batch
[ ] Document: Example curl for linking
[ ] Test: Verify linking works, unlinking works, queries work
```

---

### 2. werkflow-erp-sandbox Must Be Completely Independent

**The Principle: Zero werkflow Dependencies**

werkflow-erp-sandbox is a **standalone business data service**. It:

[YES] **Works without werkflow:**
```
Company A doesn't use werkflow. They have their own HR app.
     (next step)
They call werkflow-erp-sandbox REST APIs
     (next step)
werkflow-erp-sandbox stores/retrieves data
     (next step)
Completely works.
```

[YES] **Can be used AS a test data provider for werkflow:**
```
werkflow Platform needs to test workflows.
     (next step)
werkflow starts werkflow-erp-sandbox (Docker)
     (next step)
werkflow calls werkflow-erp-sandbox REST APIs during test
     (next step)
werkflow-erp-sandbox has NO IDEA it's being used for testing
     (next step)
Both systems work correctly
```

[YES] **Works with any orchestrator (Zapier, SAP, custom scheduler):**
```
Multiple systems call the same werkflow-erp-sandbox APIs
     (next step)
werkflow-erp-sandbox doesn't know or care which system is calling
     (next step)
All systems get consistent data
```

**Implementation Checklist:**

```
[ ] Remove all imports of werkflow.* code
[ ] Remove all Keycloak admin client code
[ ] Remove all external service calls from critical paths
[ ] Ensure all user IDs are stored as opaque strings (no validation)
[ ] Ensure all status updates are pure state transitions (no business logic)
[ ] Add CI/CD check: Reject PRs with werkflow imports
[ ] Document: Independent deployment guide
[ ] Test: Verify werkflow-erp-sandbox works standalone (no Engine, Admin, Portal)
```

**CI/CD Guard:**
```bash
# Add to pre-commit hooks or CI pipeline
if grep -r "import com.werkflow" services/business/src/main/java ; then
  echo "ERROR: werkflow-erp-sandbox must not import werkflow code"
  exit 1
fi
echo "[YES] werkflow-erp-sandbox is independent"
```

---

## Architecture Summary

```
┌────────────────────────────────────────────────────────────────┐
│                     werkflow-erp-sandbox                       │
│                                                                │
│  Pure Business Data Service                                   │
│  • HR, Finance, Procurement, Inventory                        │
│  • No workflow concepts                                       │
│  • No Keycloak client code                                    │
│  • No werkflow imports                                        │
│  • No external service calls (except internal repos)          │
│                                                                │
│  Platform User Linking (WITHOUT coupling):                    │
│  • Employee.platformUserId is optional field                  │
│  • External system manages the linking                        │
│  • werkflow-erp-sandbox just stores it                        │
│                                                                │
│  Can be used by:                                              │
│  [YES] werkflow (for testing workflows)                          │
│  [YES] SAP (as data source)                                      │
│  [YES] Zapier (as REST backend)                                  │
│  [YES] Custom app (direct REST client)                           │
│  [YES] Standalone (no external system needed)                    │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## Testing Strategy

### Test 1: Standalone Operation
```bash
# Start only werkflow-erp-sandbox and PostgreSQL
docker compose up -d postgres werkflow-erp-sandbox

# No Engine, Admin, or Portal needed
# Test that it works
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8084/api/v1/hr/employees
# Response: 200 OK [YES]
```

### Test 2: werkflow Integration
```bash
# Start full werkflow platform including werkflow-erp-sandbox
docker compose -f docker-compose.yml \
             -f docker-compose.business.yml up -d

# Test that werkflow can call werkflow-erp-sandbox
# (werkflow's responsibility to verify this works)
```

### Test 3: No Coupling
```bash
# Verify no werkflow imports
grep -r "import com.werkflow" services/business/
# Output: (empty) [YES]

# Verify no Keycloak admin code
grep -r "KeycloakAdmin" services/business/
# Output: (empty) [YES]
```

---

## Migration Path (If You Currently Have Coupling)

**If werkflow-erp-sandbox currently has werkflow imports:**

### Step 1: Identify All werkflow Imports
```bash
grep -r "import com.werkflow" services/business/src/main/java
```

### Step 2: Replace with Generic Patterns

**BEFORE (Coupled):**
```java
// In AssetRequestService
public void approveRequest(Long assetId) {
    // Call Engine to continue BPMN
    engineClient.continueProcess(processInstanceId);  // [NO]
}
```

**AFTER (Uncoupled):**
```java
// In AssetRequestService
public AssetRequest updateStatus(Long assetId, String newStatus) {
    AssetRequest request = repository.findById(assetId);
    request.setStatus(newStatus);  // [YES] Pure state update
    return repository.save(request);

    // Caller (could be werkflow, Zapier, etc.) decides what to do next
}
```

### Step 3: Update Calling Code (werkflow side, not werkflow-erp-sandbox)

**werkflow Engine now orchestrates:**
```java
// In werkflow's ExternalApiCallDelegate
// Before: werkflow-erp-sandbox had the approval logic
// Now: werkflow Engine has the approval logic

1. Call werkflow-erp-sandbox: PATCH /asset-requests/{id}/status
   werkflow-erp-sandbox: Updates status, returns confirmation

2. werkflow-erp-sandbox returns the response

3. werkflow Engine evaluates: "Is status APPROVED?"

4. If YES: werkflow Engine continues with next step (notifications, routing)
   If NO: werkflow Engine handles rejection (retry, escalate)
```

---

## Documentation Files Created

| File | Purpose |
|------|---------|
| `docs/adr/ADR-001-Service-Boundary-Architecture.md` | Detailed architectural decisions |
| `docs/Architecture-Overview.md` | Visual explanation of independence and flow diagrams |
| `docs/Independence-Checklist.md` | Review checklist for PRs |
| `Roadmap.md` | Implementation priorities |
| `README.md` | Quick start and API reference |

---

## Next Steps

1. **Review the updated ADR-001**: Understand the platform user linking pattern
2. **Review Architecture-Overview.md**: See three deployment scenarios
3. **Review Independence-Checklist.md**: Use before every PR
4. **Implement P0 roadmap items**: Multi-tenancy, idempotency, API versioning
5. **Add platform user linking endpoints**: `PATCH /platform-link`, etc.
6. **Add CI/CD check**: Reject PRs with werkflow imports

---

## Key Takeaway

werkflow-erp-sandbox is a **business data service**, not a workflow component.

It provides **pure CRUD + validation**, no orchestration.

External systems (werkflow or others) orchestrate, decide business logic, and manage workflows.

werkflow-erp-sandbox **just stores and retrieves data**—and does it for anyone who calls it.

