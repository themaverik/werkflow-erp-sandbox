# Werkflow ERP — Roadmap

**Project**: Standalone ERP Data Service — HR, Finance, Procurement, Inventory
**Master Roadmap**: `~/Projects/werkflow-platform/docs/Roadmap.md` (authoritative for all future tasks)
**Last Updated**: 2026-07-23
**Target**: Internal Enterprise Demo — June 2026

> Future tasks in this file are synced from the master Roadmap (M1 + M7 ERP share).

---

## Current Session State

**Active Phase**: Maintenance / bugfix — all ERP milestones complete (M1 merged; P-5 shipped)
**Current Task**: none — awaiting MVP release cut
**Branch**: main
**Last fix (2026-06-05)**: Flyway ordering bug — `@DependsOn` added to `identityFlyway` bean so it runs after all domain Flywayeans (erp `0c9401c`). Prevents crash-loop on fresh deploys where `identity/V24` references `finance_service.budget_plans` before `financeFlyway` has run.
**Recent (2026-07-21 → 23)**: First-class asset insurance tracking on `asset_instances` — inventory migration `V23` (`insuranceProvider`, `insuranceExpiryDate`) + DTOs + `GET /asset-instances/expiring-insurance?daysFromNow=N` mirroring the warranty path (merged to main `f6babc7`; resolves asset-mgmt E2E Known-Gap #1). API-Usage-Guide documented the warranty/insurance expiry endpoints (`d592094`). Serves the platform asset-management MVP-gate E2E.
**Next ERP task**: MVP release cut (tag + CHANGELOG) or post-MVP ERP P3 work

---

## Known Issues (Pre-MVP)

- [ ] **`AccessDeniedException` returns 500 instead of 403** — the business service `GlobalExceptionHandler` (`services/business/src/main/java/com/werkflow/business/common/exception/GlobalExceptionHandler.java`) has a catch-all `@ExceptionHandler(Exception.class)` → 500 but no `AccessDeniedException` handler, so a `@PreAuthorize` denial (e.g. calling `POST /api-keys/generate` with an API key that only carries `ROLE_API_CLIENT`) falls through to the catch-all and is reported as `INTERNAL_SERVER_ERROR` / "Access Denied" instead of `403 Forbidden`. Non-blocking — authorization still correctly denies; only the status code + error envelope are wrong. Fix: add `@ExceptionHandler(AccessDeniedException.class)` → 403 (also covers Spring Security 6.x `AuthorizationDeniedException`, its subclass). Surfaced during enterprise 2026-07-02 ERP api-key manual testing.

---

## Phase Summary

| Phase | Status |
|-------|--------|
| P0 — Critical Path | ✅ COMPLETE |
| P1.1 — API Standardisation | ✅ COMPLETE |
| P1.2 — HR/Keycloak Linking | ✅ COMPLETE |
| P1.2.5 — User Identity (OIDC) | ✅ COMPLETE — 231 tests |
| P1.3 — User Name Enrichment | ⏸️ DEFERRED — superseded by P1.2.5 |
| P1.4 — Number Generation | ✅ COMPLETE |
| P1.5.1 — Contract Tests | ✅ COMPLETE — 255 total tests |
| P1.5.2 — Integration Tests | ⏳ PENDING — @WebMvcTest solution documented |
| P2.1 — Documentation Suite | ✅ COMPLETE — PR #8 merged |
| M1 — Enterprise Integration APIs | ✅ COMPLETE — 281 tests, gate passed |
| M7 — CI/CD (ERP share) | ⏳ PENDING |
| P2.2 — Load + Security Testing | 🔮 POST-DEMO (optional) |
| P3 — Future Enhancements | 🔮 POST-MVP |

---

## Active: M1 — ERP Enterprise Integration APIs

**Deps**: none
**Estimate**: 8–10 hours
**Required by**: werkflow-enterprise M3 (Groups 2–3 cannot wire ERP data without these)

- [ ] **P1.5.2** Integration tests — `@WebMvcTest + MockMvc` approach; spec in `docs/project/reports/P1.5.2-Integration-Tests-Spec.md` (4h)

- [x] **P1.6.1** Extend `users` table + profile endpoint
  - Add columns: `department_code`, `employee_id`, `cost_center`, `is_poc` to `users` table
  - Flyway V25 migration
  - New endpoint: `GET /api/v1/users/{keycloakId}/profile`
  - Required by ADR-003 (Keycloak semantic roles) + ADR-005 (department-scoped routing)
  - Estimate: 3h *(7 tests — 4 service, 3 controller — all green)*

- [x] **P1.6.2** CustodyMapping entity + API
  - Move `CustodyMapping` from werkflow-enterprise admin-service to werkflow-erp-sandbox (ADR-004)
  - Entity: `custody_owner (VARCHAR), candidate_groups (TEXT[]), tenant_id`
  - Endpoints: `GET/POST/PUT/DELETE /api/v1/custody-mappings`
  - Tenant-scoped, paginated, idempotent upsert
  - Required by ADR-004
  - Estimate: 3h *(17 tests — 8 service, 9 controller — all green)*

- [x] **P1.6.3** Department API verification + user resolution endpoint
  - Verified `GET /api/v1/departments` returns `deptCode` (alias on `code` field)
  - Added `GET /api/v1/departments/code/{deptCode}/members`
  - Required by ADR-005
  - Estimate: 2–3h *(2 service tests — all green)*

---

## M7 — CI/CD (ERP Share)

**Deps**: none hard; slot alongside enterprise M4–M6
**Estimate**: 2–3 hours

- [ ] CI (`ci.yml`): trigger on PR + push to main; job: Maven verify (all tests must pass)
- [ ] Release (`release.yml`): trigger on tag `v*`; build + push to `ghcr.io`

---

## M4.6 — ERP Webhook Integration

**Phase**: Pre-Internal-Demo (after M4.5 in werkflow-enterprise)
**Estimate**: 4–6 hours (ERP share of M4.6)
**Reference**: Master Roadmap M4.6

ERP publishes outbound webhook events that werkflow-enterprise M4.6 correlates to in-flight process instances.

- [x] Publish webhook on vendor status change (blacklist, approval, deactivation) — `POST {werkflow_webhook_url}/vendor-status-changed` *(commit: c3bce00)*
- [x] Publish webhook on PO state transitions (approved, dispatched, received, cancelled) — `POST {werkflow_webhook_url}/po-status-changed` *(commit: c3bce00)*
- [x] Seed `werkflow-erp-events` webhook connector definition (to be loaded into the connector catalog) *(commit: dee91d3 — enterprise)*
- [x] Update Asset Request and Procurement Approval BPMN sample files (in enterprise) to add Intermediate Message Catch Events for vendor-blacklist and PO-received transitions *(commit: dee91d3 — enterprise)*

---

## Deferred — P2.2 Load + Security Testing

**Status**: Optional — deferred until after June demo

- [ ] **P2.2.1** Load test: 1000 concurrent requests
- [ ] **P2.2.2** Security audit: SQL injection, JWT, rate limiting

---

## Deferred — P3 Future Enhancements

Not tracked for MVP.

- [ ] **P3.1** Audit logging: all mutations logged with user + timestamp
- [ ] **P3.2** CapEx workflow implementation (currently stubbed)
- [ ] **P3.3** Advanced filtering: complex queries
- [ ] **P3.4** Webhook support: notify werkflow on critical data changes
- [ ] **P3.5** Bulk operations: `POST /api/v1/bulk/asset-instances`
- [ ] **P3.6** Custom fields: tenant-specific metadata on core entities

---

## Historical Summary — Completed

| Phase | Highlights | Tests |
|-------|-----------|-------|
| P0 | Multi-tenant isolation (23 entities), idempotency, processInstanceId, FK validation, API versioning, pagination | 40 |
| P1.1 | Error responses (GlobalExceptionHandler), enum metadata (15 enums), DTO examples | 118 |
| P1.2 | Keycloak link endpoint (PATCH idempotent, tenant-scoped, conflict detection) | 131 |
| P1.2.5 | OIDC user identity — UserInfoResolver (Caffeine cache), UserContext/Filter, OidcRoleConverter, 13 audit DTOs with display names | 231 |
| P1.4 | PR/PO/GRN number sequences (DB-level) | 231 |
| P1.5.1 | 24 contract tests across HR, Finance, Procurement, Inventory | 255 |
| P2.1 | Documentation suite — README, API-Usage-Guide, Werkflow-Integration-Guide, Architecture-Overview, ADRs (PRs #6–#8) | 255 |

---

## Related Documents

- ADR-300 Service Boundary Architecture — werkflow-platform `docs/adr/ERP-Sandbox/ADR-300-service-boundary-architecture.md`
- ADR-302 User Identity and JWT Claims — werkflow-platform `docs/adr/ERP-Sandbox/ADR-302-user-identity-and-jwt-claims.md`
- `docs/project/reports/P1.5.2-Integration-Tests-Spec.md`
- `docs/project/reports/P1.5.2-Integration-Tests-Implementation-Notes.md`
