# API Usage Guide — Werkflow-ERP

Complete step-by-step examples for all 4 business domains.

---

## Overview

This guide demonstrates real-world workflows across **HR**, **Finance**, **Procurement**, and **Inventory** domains using Werkflow-ERP APIs.

All examples assume:
- Service running at `http://localhost:8084/api/v1`
- Valid JWT in `Authorization: Bearer <token>` header
- Tenant ID in `X-Tenant-ID` header

---

## 1. HR Domain — Employee Onboarding

### Setup: Create Department

```bash
curl -X POST http://localhost:8084/api/v1/hr/departments \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Engineering",
    "code": "ENG",
    "deptHeadId": 1
  }'
```

Response:
```json
{
  "id": 101,
  "name": "Engineering",
  "code": "ENG",
  "deptHeadId": 1,
  "createdAt": "2026-04-10T10:00:00Z",
  "createdBy": "admin",
  "createdByDisplayName": "Admin User"
}
```

### Create Employee

```bash
curl -X POST http://localhost:8084/api/v1/hr/employees \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "X-Idempotency-Key: emp-john-2026" \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@acme.com",
    "departmentId": 101,
    "status": "ACTIVE",
    "joinDate": "2026-04-10"
  }'
```

Response (201 Created):
```json
{
  "id": 1001,
  "firstName": "John",
  "lastName": "Doe",
  "email": "john.doe@acme.com",
  "departmentId": 101,
  "status": "ACTIVE",
  "joinDate": "2026-04-10",
  "createdAt": "2026-04-10T10:00:00Z",
  "createdBy": "admin",
  "createdByDisplayName": "Admin User"
}
```

### Link to Keycloak User

After employee created, link their Keycloak account:

```bash
curl -X PATCH http://localhost:8084/api/v1/hr/employees/1001/keycloak-link \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "X-Idempotency-Key: keycloak-link-1001" \
  -H "Content-Type: application/json" \
  -d '{
    "kecycloakId": "6c4f8b2a-3f9e-4c12-b5d8-9e7c1a2b3d4e"
  }'
```

Response (200 OK):
```json
{
  "id": 1001,
  "keycloakId": "6c4f8b2a-3f9e-4c12-b5d8-9e7c1a2b3d4e",
  "linked": true,
  "message": "Employee linked to Keycloak user successfully"
}
```

### Request Leave

```bash
curl -X POST http://localhost:8084/api/v1/hr/leaves \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "X-Idempotency-Key: leave-john-2026-04" \
  -H "Content-Type: application/json" \
  -d '{
    "employeeId": 1001,
    "leaveType": "ANNUAL",
    "startDate": "2026-05-01",
    "endDate": "2026-05-05",
    "reason": "Personal vacation"
  }'
```

Response (201 Created):
```json
{
  "id": 2001,
  "employeeId": 1001,
  "leaveType": "ANNUAL",
  "startDate": "2026-05-01",
  "endDate": "2026-05-05",
  "reason": "Personal vacation",
  "status": "PENDING",
  "createdAt": "2026-04-10T10:00:00Z",
  "createdByDisplayName": "John Doe"
}
```

---

## 2. Finance Domain — Budget Planning & Expense Approval

### Create Budget Category

```bash
curl -X POST http://localhost:8084/api/v1/finance/budget-categories \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Cloud Services",
    "code": "CLOUD",
    "description": "AWS, GCP, Azure spending"
  }'
```

Response:
```json
{
  "id": 301,
  "name": "Cloud Services",
  "code": "CLOUD",
  "description": "AWS, GCP, Azure spending",
  "createdAt": "2026-04-10T10:00:00Z"
}
```

### Create Budget Plan

```bash
curl -X POST http://localhost:8084/api/v1/finance/budgets \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "X-Idempotency-Key: budget-eng-2026" \
  -H "Content-Type: application/json" \
  -d '{
    "departmentId": 101,
    "budgetCategoryId": 301,
    "allocatedAmount": 50000,
    "fiscalYear": 2026
  }'
```

Response (201 Created):
```json
{
  "id": 401,
  "departmentId": 101,
  "budgetCategoryId": 301,
  "allocatedAmount": 50000,
  "spentAmount": 0,
  "fiscalYear": 2026,
  "status": "ACTIVE",
  "createdAt": "2026-04-10T10:00:00Z"
}
```

### Check Budget Availability

Before creating an expense, check if budget is available:

```bash
curl -X POST http://localhost:8084/api/v1/finance/budget-check \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "Content-Type: application/json" \
  -d '{
    "departmentId": 101,
    "amount": 5000,
    "fiscalYear": 2026
  }'
```

Response:
```json
{
  "available": true,
  "availableAmount": 45000,
  "reason": "Budget available"
}
```

### Record Expense

```bash
curl -X POST http://localhost:8084/api/v1/finance/expenses \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "X-Idempotency-Key: expense-aws-2026-04" \
  -H "Content-Type: application/json" \
  -d '{
    "departmentId": 101,
    "budgetCategoryId": 301,
    "description": "AWS EC2 instances for April",
    "amount": 5000,
    "invoiceDate": "2026-04-10",
    "paymentDueDate": "2026-05-10"
  }'
```

Response (201 Created):
```json
{
  "id": 501,
  "departmentId": 101,
  "budgetCategoryId": 301,
  "description": "AWS EC2 instances for April",
  "amount": 5000,
  "status": "SUBMITTED",
  "invoiceDate": "2026-04-10",
  "createdAt": "2026-04-10T10:00:00Z",
  "createdByDisplayName": "John Doe"
}
```

---

## 3. Procurement Domain — Purchase Order Workflow

### Create Vendor

```bash
curl -X POST http://localhost:8084/api/v1/procurement/vendors \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Tech Supplies Inc",
    "code": "TECH-SUP",
    "email": "sales@techsupplies.com",
    "status": "ACTIVE"
  }'
```

Response:
```json
{
  "id": 601,
  "name": "Tech Supplies Inc",
  "code": "TECH-SUP",
  "email": "sales@techsupplies.com",
  "status": "ACTIVE",
  "createdAt": "2026-04-10T10:00:00Z"
}
```

### Create Purchase Request

```bash
curl -X POST http://localhost:8084/api/v1/procurement/purchase-requests \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "X-Idempotency-Key: pr-keyboards-2026-04" \
  -H "Content-Type: application/json" \
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
    "justification": "Replacement for aging equipment"
  }'
```

Response (201 Created):
```json
{
  "id": 701,
  "requestNumber": "PR-acme-2026-00001",
  "requestingDeptId": 101,
  "priority": "HIGH",
  "status": "DRAFT",
  "lineItems": [
    {
      "description": "Mechanical Keyboards",
      "quantity": 10,
      "estimatedUnitCost": 150,
      "lineTotal": 1500
    }
  ],
  "totalAmount": 1500,
  "createdAt": "2026-04-10T10:00:00Z",
  "createdByDisplayName": "John Doe"
}
```

### Create Purchase Order (After Approval)

Once werkflow approves the purchase request, create a PO:

```bash
curl -X POST http://localhost:8084/api/v1/procurement/purchase-orders \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "X-Idempotency-Key: po-keyboards-2026-04" \
  -H "Content-Type: application/json" \
  -d '{
    "purchaseRequestId": 701,
    "vendorId": 601,
    "expectedDeliveryDate": "2026-04-20",
    "lineItems": [
      {
        "description": "Mechanical Keyboards",
        "quantity": 10,
        "unitCost": 150
      }
    ]
  }'
```

Response (201 Created):
```json
{
  "id": 801,
  "poNumber": "PO-acme-2026-00001",
  "purchaseRequestId": 701,
  "vendorId": 601,
  "status": "ISSUED",
  "expectedDeliveryDate": "2026-04-20",
  "lineItems": [
    {
      "description": "Mechanical Keyboards",
      "quantity": 10,
      "unitCost": 150,
      "lineTotal": 1500
    }
  ],
  "totalAmount": 1500,
  "createdAt": "2026-04-10T10:00:00Z"
}
```

### Record Receipt (GRN)

When goods arrive:

```bash
curl -X POST http://localhost:8084/api/v1/procurement/receipts \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "X-Idempotency-Key: grn-keyboards-2026-04-20" \
  -H "Content-Type: application/json" \
  -d '{
    "purchaseOrderId": 801,
    "receiptDate": "2026-04-20",
    "lineItems": [
      {
        "quantity": 10,
        "receivedCondition": "GOOD"
      }
    ]
  }'
```

Response (201 Created):
```json
{
  "id": 901,
  "grnNumber": "GRN-acme-2026-00001",
  "purchaseOrderId": 801,
  "receiptDate": "2026-04-20",
  "status": "RECEIVED",
  "lineItems": [
    {
      "quantity": 10,
      "receivedCondition": "GOOD"
    }
  ],
  "createdAt": "2026-04-20T10:00:00Z",
  "createdByDisplayName": "Warehouse Manager"
}
```

---

## 4. Inventory Domain — Asset Management

### Create Asset Category

```bash
curl -X GET http://localhost:8084/api/v1/inventory/asset-categories \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp"
```

List of existing categories (pre-seeded in database).

### Create Asset Definition

```bash
curl -X POST http://localhost:8084/api/v1/inventory/asset-definitions \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "Content-Type: application/json" \
  -d '{
    "categoryId": 1,
    "name": "Dell Laptop",
    "sku": "DELL-XPS-13",
    "expectedLifeYears": 5
  }'
```

### Request Asset

```bash
curl -X POST http://localhost:8084/api/v1/inventory/asset-requests \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "X-Idempotency-Key: asset-req-john-2026-04" \
  -H "Content-Type: application/json" \
  -d '{
    "requesterUserId": "john.doe@acme.com",
    "assetDefinitionId": 1,
    "quantity": 1,
    "justification": "New laptop for hire"
  }'
```

Response (201 Created):
```json
{
  "id": 1101,
  "requesterUserId": "john.doe@acme.com",
  "assetDefinitionId": 1,
  "quantity": 1,
  "status": "PENDING",
  "justification": "New laptop for hire",
  "createdAt": "2026-04-10T10:00:00Z"
}
```

### Approve Asset Request (via werkflow Callback)

After werkflow approves, update status:

```bash
curl -X POST http://localhost:8084/api/v1/inventory/asset-requests/callback/approve \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "Content-Type: application/json" \
  -d '{
    "assetRequestId": 1101,
    "approverUserId": "manager@acme.com"
  }'
```

Response:
```json
{
  "id": 1101,
  "status": "APPROVED",
  "message": "Asset request approved"
}
```

### Create Asset Instance (After Procurement)

Once the asset is received and available:

```bash
curl -X POST http://localhost:8084/api/v1/inventory/asset-instances \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "Content-Type: application/json" \
  -d '{
    "assetDefinitionId": 1,
    "serialNumber": "DELL-XPS-ABC123",
    "status": "IN_USE",
    "assignedToUserId": "john.doe@acme.com",
    "assignedAt": "2026-04-10"
  }'
```

Response (201 Created):
```json
{
  "id": 1201,
  "assetDefinitionId": 1,
  "serialNumber": "DELL-XPS-ABC123",
  "status": "IN_USE",
  "assignedToUserId": "john.doe@acme.com",
  "assignedAt": "2026-04-10",
  "createdAt": "2026-04-10T10:00:00Z"
}
```

### Create Custody Record

Document who owns the asset:

```bash
curl -X POST http://localhost:8084/api/v1/inventory/custody-records \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "Content-Type: application/json" \
  -d '{
    "assetInstanceId": 1201,
    "custodianUserId": "john.doe@acme.com",
    "custodyType": "EMPLOYEE",
    "startDate": "2026-04-10"
  }'
```

Response:
```json
{
  "id": 1301,
  "assetInstanceId": 1201,
  "custodianUserId": "john.doe@acme.com",
  "custodyType": "EMPLOYEE",
  "startDate": "2026-04-10",
  "createdAt": "2026-04-10T10:00:00Z"
}
```

### Track Warranty & Insurance Expiry

Asset instances carry optional warranty and insurance fields, set on create or update:

```bash
curl -X PUT http://localhost:8084/api/v1/inventory/asset-instances/1201 \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp" \
  -H "Content-Type: application/json" \
  -d '{
    "assetDefinitionId": 1,
    "serialNumber": "DELL-XPS-ABC123",
    "status": "IN_USE",
    "warrantyExpiryDate": "2027-04-10",
    "insuranceProvider": "Acme General Insurance",
    "insuranceExpiryDate": "2026-10-10"
  }'
```

Query instances whose warranty or insurance expires within a window. `daysFromNow`
defaults to `30`:

```bash
# Warranty expiring within the next 60 days
curl -X GET "http://localhost:8084/api/v1/inventory/asset-instances/expiring-warranty?daysFromNow=60" \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp"

# Insurance expiring within the next 60 days
curl -X GET "http://localhost:8084/api/v1/inventory/asset-instances/expiring-insurance?daysFromNow=60" \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp"
```

Each returns the list of matching asset instances (same shape as `GET /asset-instances/{id}`),
including `warrantyExpiryDate`, `insuranceProvider`, and `insuranceExpiryDate`. A scheduled
werkflow process can poll these endpoints to drive renewal reminders.

---

## Common Patterns

### Idempotency Keys

Always use `X-Idempotency-Key` for POST requests. If the request fails and is retried, you'll get the same response:

```bash
# First call  (calls) 201 Created
curl -X POST ... -H "X-Idempotency-Key: unique-key-123"

# Retry with same key  (calls) 200 OK (cached response)
curl -X POST ... -H "X-Idempotency-Key: unique-key-123"
```

### Pagination

List endpoints support pagination:

```bash
curl "http://localhost:8084/api/v1/hr/employees?page=0&size=20&sort=createdAt,desc" \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: acme-corp"
```

### Error Responses

All errors follow this format:

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Invalid department",
  "timestamp": "2026-04-10T10:00:00Z",
  "details": {
    "fieldName": "departmentId",
    "reason": "Department not found: 999"
  }
}
```

---

## Troubleshooting

### "401 Unauthorized"
- JWT token missing or expired
- Use `/auth` endpoint to refresh token

### "403 Forbidden"
- Tenant ID in header doesn't match JWT claim
- Ensure `X-Tenant-ID` matches your user's tenant

### "409 Conflict"
- FK constraint violated (e.g., department doesn't exist)
- Check that all referenced IDs are valid

### "400 Bad Request"
- Missing required fields or invalid enum values
- Check the `details` field in error response

---

## Related Documents

- **[README](../../README.md)** — Project overview and quick start
- **[Integration Guide](./Werkflow-Integration-Guide.md)** — Connector setup and BPMN examples
- **[API Overview](../../README.md#api-overview)** — Complete endpoint reference
