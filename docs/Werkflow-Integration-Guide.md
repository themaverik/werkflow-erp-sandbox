# Werkflow Integration Guide

**Note**: This guide is required only if you are integrating Werkflow-ERP with the Werkflow orchestration platform. If you are using Werkflow-ERP as a standalone service, see the [API Usage Guide](./API-Usage-Guide.md) instead.

---

## Overview

Werkflow-ERP is designed to be called from Werkflow BPMN workflows. This guide covers:
- Connector registration in Werkflow Admin
- BPMN service task configuration with complete examples
- Error handling and retry patterns
- User identity resolution and display names
- Multi-tenancy and authentication
- ProcessInstanceId linking patterns

---

## Prerequisites

- Werkflow platform running (Engine 8081, Admin 8083, Portal 4000)
- Werkflow-ERP service running (8084)
- Admin access to Werkflow Admin Portal
- JWT token generation (handled automatically by Werkflow)

---

## Step 1: Register Connector in Werkflow Admin

### Navigate to Connectors

1. Open Werkflow Admin Portal: `http://localhost:4000/admin`
2. Go to **Admin  (calls) Connectors**
3. Click **Add Connector**

### Fill in Details

| Field | Value |
|-------|-------|
| **Key** | `business-service` |
| **Display Name** | Werkflow-ERP Business Service |
| **Endpoint URL** | `http://localhost:8084/api/v1` |
| **Authentication Type** | Bearer Token (JWT) |
| **Timeout (seconds)** | 30 |
| **Retry On Failure** | Yes (3 attempts) |

Click **Save**.

---

## Step 2: Configure BPMN Service Tasks

### Example 1: Create Purchase Request

```xml
<bpmn:process id="procurement_workflow" name="Procurement Workflow">
  <bpmn:startEvent id="StartEvent" name="PR Submitted" />
  <bpmn:serviceTask
    id="Task_CreatePR"
    name="Create Purchase Request"
    camunda:type="http"
    camunda:method="POST"
    camunda:url="http://localhost:8084/api/v1/procurement/purchase-requests">

    <bpmn:extensionElements>
      <camunda:inputOutput>
        <!-- Request Headers -->
        <camunda:inputParameter name="headers">
          <camunda:map>
            <camunda:entry key="Authorization">Bearer ${auth.token}</camunda:entry>
            <camunda:entry key="X-Tenant-ID">${tenantId}</camunda:entry>
            <camunda:entry key="X-Idempotency-Key">${processInstanceId}</camunda:entry>
            <camunda:entry key="Content-Type">application/json</camunda:entry>
          </camunda:map>
        </camunda:inputParameter>

        <!-- Request Payload -->
        <camunda:inputParameter name="payload">
          <camunda:script scriptFormat="JavaScript">
            ({
              requestingDeptId: parseInt(requestingDeptId),
              priority: priority,
              lineItems: [
                {
                  description: itemDescription,
                  quantity: parseInt(quantity),
                  estimatedUnitCost: parseFloat(estimatedCost)
                }
              ],
              justification: justification
            })
          </camunda:script>
        </camunda:inputParameter>

        <!-- Response Handling -->
        <camunda:outputParameter name="prId">
          ${response.id}
        </camunda:outputParameter>
        <camunda:outputParameter name="prNumber">
          ${response.requestNumber}
        </camunda:outputParameter>
        <camunda:outputParameter name="prStatus">
          ${response.status}
        </camunda:outputParameter>
      </camunda:inputOutput>
    </bpmn:extensionElements>
  </bpmn:serviceTask>

  <!-- Continue workflow based on response -->
  <bpmn:sequenceFlow id="flow1" sourceRef="StartEvent" targetRef="Task_CreatePR" />
</bpmn:process>
```

### Example 2: Check Budget & Create Expense

```xml
<bpmn:serviceTask
  id="Task_CheckBudget"
  name="Check Budget Availability"
  camunda:type="http"
  camunda:method="POST"
  camunda:url="http://localhost:8084/api/v1/finance/budget-check">

  <bpmn:extensionElements>
    <camunda:inputOutput>
      <camunda:inputParameter name="headers">
        <camunda:map>
          <camunda:entry key="Authorization">Bearer ${auth.token}</camunda:entry>
          <camunda:entry key="X-Tenant-ID">${tenantId}</camunda:entry>
          <camunda:entry key="Content-Type">application/json</camunda:entry>
        </camunda:map>
      </camunda:inputParameter>

      <camunda:inputParameter name="payload">
        <camunda:script scriptFormat="JavaScript">
          ({
            departmentId: parseInt(departmentId),
            amount: parseFloat(expenseAmount),
            fiscalYear: parseInt(fiscalYear)
          })
        </camunda:script>
      </camunda:inputParameter>

      <camunda:outputParameter name="budgetAvailable">
        ${response.available}
      </camunda:outputParameter>
      <camunda:outputParameter name="availableAmount">
        ${response.availableAmount}
      </camunda:outputParameter>
    </camunda:inputOutput>
  </bpmn:extensionElements>
</bpmn:serviceTask>

<!-- If budget available, create expense -->
<bpmn:exclusiveGateway id="Gateway_BudgetCheck" name="Budget Available?" />

<bpmn:serviceTask
  id="Task_CreateExpense"
  name="Create Expense"
  camunda:type="http"
  camunda:method="POST"
  camunda:url="http://localhost:8084/api/v1/finance/expenses">

  <bpmn:extensionElements>
    <camunda:inputOutput>
      <camunda:inputParameter name="headers">
        <camunda:map>
          <camunda:entry key="Authorization">Bearer ${auth.token}</camunda:entry>
          <camunda:entry key="X-Tenant-ID">${tenantId}</camunda:entry>
          <camunda:entry key="X-Idempotency-Key">${processInstanceId}</camunda:entry>
          <camunda:entry key="Content-Type">application/json</camunda:entry>
        </camunda:map>
      </camunda:inputParameter>

      <camunda:inputParameter name="payload">
        <camunda:script scriptFormat="JavaScript">
          ({
            departmentId: parseInt(departmentId),
            budgetCategoryId: parseInt(budgetCategoryId),
            description: expenseDescription,
            amount: parseFloat(expenseAmount),
            invoiceDate: invoiceDate,
            paymentDueDate: paymentDueDate
          })
        </camunda:script>
      </camunda:inputParameter>

      <camunda:outputParameter name="expenseId">
        ${response.id}
      </camunda:outputParameter>
    </camunda:inputOutput>
  </bpmn:extensionElements>
</bpmn:serviceTask>

<bpmn:sequenceFlow
  id="flow_yes"
  sourceRef="Gateway_BudgetCheck"
  targetRef="Task_CreateExpense"
  conditionExpression="${budgetAvailable == true}" />

<bpmn:sequenceFlow
  id="flow_no"
  sourceRef="Gateway_BudgetCheck"
  targetRef="Task_RejectExpense"
  conditionExpression="${budgetAvailable == false}" />
```

### Example 3: Process Asset Request Callbacks

```xml
<!-- After Manager Approval -->
<bpmn:serviceTask
  id="Task_ApproveAssetRequest"
  name="Approve Asset Request"
  camunda:type="http"
  camunda:method="POST"
  camunda:url="http://localhost:8084/api/v1/inventory/asset-requests/callback/approve">

  <bpmn:extensionElements>
    <camunda:inputOutput>
      <camunda:inputParameter name="headers">
        <camunda:map>
          <camunda:entry key="Authorization">Bearer ${auth.token}</camunda:entry>
          <camunda:entry key="X-Tenant-ID">${tenantId}</camunda:entry>
          <camunda:entry key="Content-Type">application/json</camunda:entry>
        </camunda:map>
      </camunda:inputParameter>

      <camunda:inputParameter name="payload">
        <camunda:script scriptFormat="JavaScript">
          ({
            assetRequestId: parseInt(assetRequestId),
            approverUserId: currentUser.email
          })
        </camunda:script>
      </camunda:inputParameter>

      <camunda:outputParameter name="assetStatus">
        ${response.status}
      </camunda:outputParameter>
    </camunda:inputOutput>
  </bpmn:extensionElements>
</bpmn:serviceTask>

<!-- If Procurement Needed -->
<bpmn:serviceTask
  id="Task_TriggerProcurement"
  name="Trigger Procurement"
  camunda:type="http"
  camunda:method="POST"
  camunda:url="http://localhost:8084/api/v1/inventory/asset-requests/callback/procurement">

  <bpmn:extensionElements>
    <camunda:inputOutput>
      <camunda:inputParameter name="headers">
        <camunda:map>
          <camunda:entry key="Authorization">Bearer ${auth.token}</camunda:entry>
          <camunda:entry key="X-Tenant-ID">${tenantId}</camunda:entry>
          <camunda:entry key="Content-Type">application/json</camunda:entry>
        </camunda:map>
      </camunda:inputParameter>

      <camunda:inputParameter name="payload">
        <camunda:script scriptFormat="JavaScript">
          ({
            assetRequestId: parseInt(assetRequestId),
            vendorId: parseInt(vendorId),
            expectedDeliveryDays: 14
          })
        </camunda:script>
      </camunda:inputParameter>
    </camunda:inputOutput>
  </bpmn:extensionElements>
</bpmn:serviceTask>
```

---

## Step 3: Error Handling and Retries

### Automatic Retries

Werkflow automatically retries failed calls (3 attempts, exponential backoff). These fail:
- Network timeouts (> 30 seconds)
- HTTP 5xx errors (server errors)
- Connection refused

These do NOT retry:
- HTTP 400 Bad Request (validation error, won't be fixed by retrying)
- HTTP 401 Unauthorized (auth issue)
- HTTP 403 Forbidden (permission issue)

### Error Handling in BPMN

```xml
<bpmn:serviceTask id="Task_CreateAsset">
  <!-- ... task config ... -->
</bpmn:serviceTask>

<!-- Catch validation errors -->
<bpmn:boundaryEvent id="BoundaryEvent_ValidationError"
  attachedToRef="Task_CreateAsset"
  camunda:errorCodeVariable="errorCode">
  <bpmn:errorEventDefinition />
</bpmn:boundaryEvent>

<bpmn:sequenceFlow
  sourceRef="BoundaryEvent_ValidationError"
  targetRef="Task_LogError" />

<bpmn:serviceTask id="Task_LogError" name="Log Validation Error">
  <bpmn:extensionElements>
    <camunda:inputOutput>
      <camunda:inputParameter name="logMessage">
        Asset creation failed: ${errorCode}
      </camunda:inputParameter>
    </camunda:inputOutput>
  </bpmn:extensionElements>
</bpmn:serviceTask>
```

---

## Step 4: User Identity and Display Names

All audit-relevant API responses include user display names resolved from the OIDC provider:

```json
{
  "id": 123,
  "createdAt": "2026-04-10T10:00:00Z",
  "createdByDisplayName": "Jane Smith",
  "updatedAt": "2026-04-10T11:00:00Z",
  "updatedByDisplayName": "Jane Smith"
}
```

### Why Display Names in Responses?

- UI never needs extra calls to fetch user names
- Werkflow-ERP resolves names from the OIDC `/userinfo` endpoint
- Each service instance caches names locally (no shared state required)

### For Werkflow Integration

- Pass the user's JWT bearer token to Werkflow-ERP
- Werkflow-ERP calls `/userinfo` with the same token
- Names are cached independently per service instance (Caffeine, TTL 10 min)
- No data sharing between Werkflow and Werkflow-ERP is needed

### Affected Response Types (13 total)

- HR: EmployeeResponse, DepartmentResponse, LeaveResponse (3)
- Finance: BudgetPlanResponse, BudgetLineItemResponse, ExpenseResponse (3)
- Procurement: PurchaseRequestResponse, PurchaseOrderResponse, ReceiptResponse (3)
- Inventory: AssetRequestResponse, CustodyRecordResponse, MaintenanceRecordResponse, TransferRequestResponse (4)

---

## Step 5: Multi-Tenancy & Authentication

### Tenant ID Resolution

Werkflow-ERP requires `X-Tenant-ID` in every request. Set it from:

1. **Process Variable** (recommended):
```xml
<camunda:entry key="X-Tenant-ID">${tenantId}</camunda:entry>
```

2. **User Context**:
```xml
<camunda:entry key="X-Tenant-ID">${currentUser.organization}</camunda:entry>
```

### JWT Token

Werkflow automatically includes a JWT token with:
- `sub` — User ID
- `roles` — User's roles
- `organization_id` — Tenant ID
- `exp` — Expiration time

**Do NOT hardcode tokens.** Use `${auth.token}` variable:

```xml
<camunda:entry key="Authorization">Bearer ${auth.token}</camunda:entry>
```

Example JWT claim:
```json
{
  "sub": "user123",
  "organization_id": "acme-corp",
  "iat": 1712506800
}
```

### Idempotency Keys

Always use `X-Idempotency-Key` to prevent duplicate resource creation:

```xml
<camunda:entry key="X-Idempotency-Key">${processInstanceId}</camunda:entry>
```

This ensures:
- First request  (calls) 201 Created, resource created
- Retry with same key  (calls) 200 OK, cached response (no duplicate)

---

## ProcessInstanceId Linking

### Pattern: Generate First, Then Create

The recommended pattern for Werkflow is to generate `processInstanceId` in the workflow engine **before** making the API call to create an asset request, purchase request, or purchase order.

**Benefits:**
- Single API call — no need for subsequent updates
- Immediate linking of business entity to workflow instance
- Eliminates race conditions between API creation and workflow callback

#### Asset Request Example

```bash
# 1. Workflow engine generates processInstanceId
PROCESS_ID="arn:aws:bpm:us-east-1:123456789:process/asset-approval/2026-04-07-12345"

# 2. Call POST /api/v1/inventory/asset-requests with processInstanceId
curl -X POST http://localhost:8084/api/v1/inventory/asset-requests \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "X-Idempotency-Key: $(uuidgen)" \
  -d '{
    "requesterUserId": "user123",
    "requesterName": "John Doe",
    "requesterEmail": "john@acme.com",
    "officeLocation": "NEW_YORK",
    "assetCategoryId": 42,
    "quantity": 1,
    "processInstanceId": "'$PROCESS_ID'"
  }'

# 3. Asset request created with processInstanceId set
# Response includes id (e.g., 789) and processInstanceId
```

#### Purchase Request Example

```bash
# Workflow engine generates processInstanceId
PROCESS_ID="arn:aws:bpm:us-east-1:123456789:process/pr-approval/2026-04-07-54321"

# Call POST /api/v1/purchase-requests with processInstanceId
curl -X POST http://localhost:8084/api/v1/procurement/purchase-requests \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "X-Idempotency-Key: $(uuidgen)" \
  -d '{
    "requestingDeptId": 101,
    "priority": "HIGH",
    "lineItems": [
      {
        "description": "Mechanical Keyboards",
        "quantity": 10,
        "estimatedUnitCost": 150
      }
    ],
    "processInstanceId": "'$PROCESS_ID'"
  }'

# Purchase request created with processInstanceId set
```

### Fallback Pattern: Create First, Then Update

If the workflow engine cannot generate `processInstanceId` before API creation, use the fallback:

1. Create the request without `processInstanceId` (optional field)
2. After workflow instance is created, update using the dedicated endpoint

#### Asset Request Fallback

```bash
# 1. Create asset request without processInstanceId
RESPONSE=$(curl -X POST http://localhost:8084/api/v1/inventory/asset-requests \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "X-Idempotency-Key: $(uuidgen)" \
  -d '{
    "requesterUserId": "user123",
    "requesterName": "John Doe",
    "requesterEmail": "john@acme.com",
    "officeLocation": "NEW_YORK",
    "assetCategoryId": 42,
    "quantity": 1
  }')

ASSET_REQUEST_ID=$(echo $RESPONSE | jq -r '.id')
echo "Created asset request: $ASSET_REQUEST_ID"

# 2. Workflow engine processes and generates processInstanceId
PROCESS_ID="arn:aws:bpm:us-east-1:123456789:process/asset-approval/2026-04-07-12345"

# 3. Update asset request with processInstanceId
curl -X POST http://localhost:8084/api/v1/inventory/asset-requests/$ASSET_REQUEST_ID/process-instance \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -d "processInstanceId=$PROCESS_ID"

echo "Linked asset request $ASSET_REQUEST_ID to process $PROCESS_ID"
```

**Fallback applies to:**
- Asset Requests: `POST /api/v1/inventory/asset-requests/{id}/process-instance?processInstanceId=...`
- Purchase Requests: Similar endpoint (to be implemented)
- Purchase Orders: Similar endpoint (to be implemented)

---

## Common Workflows

### Workflow 1: Simple Purchase Order

```
1. User submits PR form
2. Manager approves (human task)
3. System creates Purchase Request in Werkflow-ERP
4. Finance reviews budget
5. If approved: Create Purchase Order
6. If rejected: Reject PR
```

### Workflow 2: Asset Request with Procurement

```
1. Employee requests asset
2. Manager approves
3. System checks asset availability
   - If in stock: Create custody record
   - If not: Create purchase request for procurement
4. Procurement creates PO and receives goods
5. System creates asset instance
6. System assigns to employee
```

### Workflow 3: Expense Approval with Budget Check

```
1. Employee submits expense
2. System checks budget availability
   - If available: Create expense, route to manager
   - If not available: Auto-reject
3. Manager approves/rejects
4. Finance records payment
```

---

## Configuration Best Practices

### 1. Error Handling

Always plan for failures:

```xml
<bpmn:serviceTask id="Task_Create" ... />

<bpmn:boundaryEvent id="Event_Timeout"
  attachedToRef="Task_Create"
  timerEventDefinition>
  <bpmn:timerEventDefinition>
    <bpmn:timeDuration>PT30S</bpmn:timeDuration>
  </bpmn:timerEventDefinition>
</bpmn:boundaryEvent>

<bpmn:sequenceFlow
  sourceRef="Event_Timeout"
  targetRef="Task_NotifyAdmin" />
```

### 2. Data Validation

Validate before calling Werkflow-ERP:

```xml
<bpmn:exclusiveGateway id="Gateway_ValidateInput">
  <bpmn:conditionExpression>
    ${quantity > 0 AND departmentId > 0}
  </bpmn:conditionExpression>
</bpmn:exclusiveGateway>
```

### 3. Logging

Use logging to debug integrations:

```xml
<camunda:executionListener event="start">
  <camunda:script scriptFormat="JavaScript">
    console.log('Creating PR with amount: ' + amount);
  </camunda:script>
</camunda:executionListener>
```

### 4. Timeouts

Set reasonable timeouts based on operation:

```xml
<camunda:property name="camunda:asyncAfter" value="true" />
<camunda:property name="camunda:timeout" value="PT1M" />
```

---

## Testing Integration

### Manual Test

```bash
# 1. Start Werkflow-ERP
cd werkflow-erp-sandbox
mvn spring-boot:run

# 2. Get JWT token
TOKEN=$(curl -s http://localhost:8090/realms/werkflow/protocol/openid-connect/token \
  -d "grant_type=client_credentials&client_id=werkflow-api&client_secret=secret" \
  | jq -r '.access_token')

# 3. Test API call
curl -X POST http://localhost:8084/api/v1/hr/employees \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: test-tenant" \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Test",
    "lastName": "User",
    "email": "test@example.com",
    "departmentId": 1,
    "status": "ACTIVE",
    "joinDate": "2026-04-10"
  }'
```

### BPMN Test

Deploy a test workflow:

1. Open Werkflow Designer: `http://localhost:4000/designer`
2. Create simple test workflow
3. Use Test API Connector task
4. Deploy and execute
5. Check Werkflow-ERP logs for requests/responses

---

## API Endpoints

### Asset Request Lifecycle

```
POST   /api/v1/inventory/asset-requests                         Create (processInstanceId optional)
GET    /api/v1/inventory/asset-requests/{id}                    Retrieve
POST   /api/v1/inventory/asset-requests/{id}/process-instance   Update processInstanceId
POST   /api/v1/inventory/asset-requests/callback/approve        Approve (via workflow callback)
POST   /api/v1/inventory/asset-requests/callback/reject         Reject (via workflow callback)
POST   /api/v1/inventory/asset-requests/callback/procurement    Initiate procurement after approval
```

---

## Troubleshooting

### 401 Unauthorized

**Cause**: JWT token missing or expired

**Solution**: Ensure `${auth.token}` is used, not hardcoded token. Token is automatically refreshed by Werkflow.

### 403 Forbidden - Cross-tenant Access

**Cause**: Tenant ID in header doesn't match JWT claim

**Solution**: Verify `X-Tenant-ID` matches the user's organization from JWT.

### 409 Conflict

**Cause**: Foreign key constraint violated

**Solution**: Ensure all referenced IDs (departmentId, vendorId, etc.) exist in the target tenant.

### 400 Bad Request

**Cause**: Validation error (invalid enum, required field missing, etc.)

**Solution**: Check error details in response. Validate data before calling Werkflow-ERP.

### Connection Refused

**Cause**: Werkflow-ERP not running or wrong URL

**Solution**:
```bash
# Check if running
curl http://localhost:8084/api/v1/actuator/health

# Or verify connector URL in Admin Portal
```

---

## Related Documents

- [API Usage Guide](./API-Usage-Guide.md) - Complete API examples for all domains (standalone usage)
- [README](../README.md) - Project overview
- ADR-300 Service Boundary Architecture — werkflow-platform `docs/adr/ERP-Sandbox/ADR-300-service-boundary-architecture.md` (in the `werkflow-platform` repository) - design rationale
