# P0.2 Design: Idempotency for Safe Retries

**Date:** 2026-04-07
**Status:** Approved
**Phase:** P0 — Critical Path to Production

---

## Overview

Implement request idempotency so clients can safely retry POST/PUT operations without duplicating side effects. Cache the full HTTP response (body + status + headers) keyed by tenant ID + idempotency key.

---

## Architectural Decisions

All decisions recorded in `docs/ADR-001-Service-Boundary-Architecture.md` under P0.2 section.

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Validation** | Strict — payload must match original | Safety: prevent accidental duplicate processing if payload changes |
| **Storage** | PostgreSQL + Caffeine in-memory cache (write-through) | Durability + performance without premature optimization |
| **Cleanup** | Lazy (on read) + scheduled background job | Resilient: expired entries cleaned on access, background job prevents unbounded growth |
| **TTL** | 24 hours | Standard idempotency window; balance between safety and storage cost |
| **Position** | ServletFilter in SecurityFilterChain, after `BearerTokenAuthenticationFilter` | Consistency with `TenantContextFilter` pattern; early interception for all endpoints |
| **Cache Key** | `{tenantId}:{Idempotency-Key}` | Tenant isolation; prevents cross-tenant key collisions |
| **Response Cache** | Full (body + status + headers) | Industry standard (Stripe, AWS); headers may contain critical client information (e.g., Location for 201) |

---

## Components

### 1. IdempotencyRecord Entity

**Purpose:** Persistent store for idempotent responses. Survives application restart; used by both cache and cleanup job.

**Fields:**

```java
@Entity
@Table(name = "idempotency_record",
       uniqueConstraints = @UniqueConstraint(columnNames = {"tenant_id", "idempotency_key"}))
public class IdempotencyRecord extends BaseEntity {

    @Column(name = "tenant_id", nullable = false)
    private String tenantId;

    @Column(name = "idempotency_key", nullable = false, length = 255)
    private String idempotencyKey;

    @Column(name = "request_payload", columnDefinition = "TEXT")
    private String requestPayload;  // Original request body (for strict validation)

    @Column(name = "response_body", columnDefinition = "TEXT")
    private String responseBody;  // Full response body (JSON)

    @Column(name = "response_headers", columnDefinition = "TEXT")
    private String responseHeaders;  // Headers as JSON (Content-Type, Location, etc.)

    @Column(name = "status_code", nullable = false)
    private Integer statusCode;
}
```

**Indexes:**
- Unique composite: `(tenant_id, idempotency_key)`
- Single index: `(tenant_id, created_at)` for cleanup queries

**V22 Flyway Migration:** Create table with above schema.

---

### 2. IdempotencyRepository

**Purpose:** Data access layer for IdempotencyRecord.

```java
@Repository
public interface IdempotencyRepository extends JpaRepository<IdempotencyRecord, Long> {

    Optional<IdempotencyRecord> findByTenantIdAndIdempotencyKey(String tenantId, String key);

    void deleteByTenantIdAndCreatedAtBefore(String tenantId, LocalDateTime cutoff);
}
```

---

### 3. IdempotencyService

**Purpose:** Business logic for cache lookup, validation, and storage.

**Responsibilities:**
- Inject `IdempotencyRepository` and Caffeine cache manager
- Lookup cached response by tenant ID + key
- Validate request payload matches stored request (strict validation)
- Store new responses in cache + database (write-through)
- Handle TTL expiration

```java
@Service
public class IdempotencyService {

    private static final String CACHE_NAME = "idempotency";
    private static final long TTL_MINUTES = 24 * 60;  // 24 hours

    private final IdempotencyRepository repository;
    private final CacheManager cacheManager;

    public Optional<CachedResponse> getIfPresent(String tenantId, String key, String currentPayload) {
        // 1. Check Caffeine cache
        Cache cache = cacheManager.getCache(CACHE_NAME);
        String cacheKey = buildCacheKey(tenantId, key);
        IdempotencyRecord cached = cache.get(cacheKey, IdempotencyRecord.class);

        if (cached != null) {
            // Lazy cleanup: check if expired
            if (isExpired(cached)) {
                repository.delete(cached);
                return Optional.empty();
            }

            // Strict validation: payload must match
            validatePayload(cached.getRequestPayload(), currentPayload);
            return Optional.of(toCachedResponse(cached));
        }

        // 2. Fall back to database
        Optional<IdempotencyRecord> dbRecord = repository.findByTenantIdAndIdempotencyKey(tenantId, key);
        if (dbRecord.isPresent()) {
            IdempotencyRecord record = dbRecord.get();

            // Lazy cleanup
            if (isExpired(record)) {
                repository.delete(record);
                return Optional.empty();
            }

            // Populate cache and validate
            validatePayload(record.getRequestPayload(), currentPayload);
            cache.put(cacheKey, record);
            return Optional.of(toCachedResponse(record));
        }

        return Optional.empty();
    }

    public void store(String tenantId, String key, String requestPayload, HttpResponse response) {
        IdempotencyRecord record = new IdempotencyRecord();
        record.setTenantId(tenantId);
        record.setIdempotencyKey(key);
        record.setRequestPayload(requestPayload);
        record.setResponseBody(response.getBody());
        record.setResponseHeaders(serializeHeaders(response.getHeaders()));
        record.setStatusCode(response.getStatusCode());

        // Write-through: save to DB, then cache
        IdempotencyRecord saved = repository.save(record);
        Cache cache = cacheManager.getCache(CACHE_NAME);
        String cacheKey = buildCacheKey(tenantId, key);
        cache.put(cacheKey, saved);
    }

    private void validatePayload(String storedPayload, String currentPayload) {
        if (!storedPayload.equals(currentPayload)) {
            throw new IdempotencyException(
                "Idempotency-Key reused with different payload. " +
                "Use a new key for different requests."
            );
        }
    }

    private boolean isExpired(IdempotencyRecord record) {
        LocalDateTime expiresAt = record.getCreatedAt().plusMinutes(TTL_MINUTES);
        return LocalDateTime.now(ZoneOffset.UTC).isAfter(expiresAt);
    }

    private String buildCacheKey(String tenantId, String key) {
        return tenantId + ":" + key;
    }
}
```

**Error Handling:**
- `IdempotencyException` (409 Conflict) — payload mismatch
- Log warning — database failures during store (cache miss on retry is acceptable)

---

### 4. IdempotencyFilter (ServletFilter)

**Purpose:** Intercept POST/PUT requests, deduplicate based on Idempotency-Key header.

**Position:** SecurityFilterChain, after `BearerTokenAuthenticationFilter`, alongside `TenantContextFilter`.

```java
@Component
@Order(6)  // After BearerTokenAuthenticationFilter (order 5) and TenantContextFilter (order 5.5)
public class IdempotencyFilter extends OncePerRequestFilter {

    private final IdempotencyService service;
    private final TenantContext tenantContext;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        // Only process POST and PUT
        String method = request.getMethod();
        if (!("POST".equalsIgnoreCase(method) || "PUT".equalsIgnoreCase(method))) {
            chain.doFilter(request, response);
            return;
        }

        // Check for Idempotency-Key header (optional)
        String idempotencyKey = request.getHeader("Idempotency-Key");
        if (idempotencyKey == null || idempotencyKey.isBlank()) {
            chain.doFilter(request, response);
            return;
        }

        // Get tenant ID from TenantContext
        String tenantId = tenantContext.getTenantId();
        if (tenantId == null) {
            chain.doFilter(request, response);
            return;
        }

        // Capture request body for payload validation
        String requestPayload = captureRequestBody(request);

        // Check cache
        Optional<CachedResponse> cached = service.getIfPresent(tenantId, idempotencyKey, requestPayload);
        if (cached.isPresent()) {
            CachedResponse cachedResp = cached.get();

            // Return cached response immediately (skip controller)
            response.setStatus(cachedResp.getStatusCode());
            cachedResp.getHeaders().forEach(response::setHeader);
            response.getWriter().write(cachedResp.getBody());
            return;
        }

        // Cache miss: proceed to controller, capture response
        ContentCachingResponseWrapper wrappedResponse = new ContentCachingResponseWrapper(response);

        try {
            chain.doFilter(request, wrappedResponse);
        } finally {
            // After controller execution, store response if successful (2xx)
            if (wrappedResponse.getStatus() >= 200 && wrappedResponse.getStatus() < 300) {
                try {
                    service.store(tenantId, idempotencyKey, requestPayload,
                                  buildHttpResponse(wrappedResponse));
                } catch (Exception e) {
                    // Log but don't fail the request
                    logger.warn("Failed to store idempotency record", e);
                }
            }

            wrappedResponse.copyBodyToResponse();
        }
    }

    private String captureRequestBody(HttpServletRequest request) throws IOException {
        ContentCachingRequestWrapper wrapper = new ContentCachingRequestWrapper(request);
        // Ensure wrapper buffers the body
        wrapper.getInputStream();
        return new String(wrapper.getContentAsByteArray(), StandardCharsets.UTF_8);
    }
}
```

**Behavior:**
1. Only intercept POST/PUT requests
2. Skip if no Idempotency-Key header
3. On cache hit: validate payload, return cached response, skip controller
4. On cache miss: proceed to controller, capture response, store in cache + database
5. Only cache successful (2xx) responses

---

### 5. Cleanup Scheduled Job

**Purpose:** Daily background task to remove expired idempotency records from database.

```java
@Component
public class IdempotencyCleanupJob {

    private final IdempotencyRepository repository;
    private final TenantRepository tenantRepository;  // Iterate per tenant

    @Scheduled(cron = "0 2 * * *")  // 2 AM UTC daily
    public void cleanupExpiredRecords() {
        LocalDateTime cutoff = LocalDateTime.now(ZoneOffset.UTC).minusHours(24);

        // Clean all tenants
        List<String> allTenants = tenantRepository.findAllTenantIds();
        for (String tenantId : allTenants) {
            repository.deleteByTenantIdAndCreatedAtBefore(tenantId, cutoff);
        }
    }
}
```

---

## Data Flow: Happy Path (Idempotent Request)

```
[Client]
  POST /api/v1/procurement/purchase-requests
  Idempotency-Key: abc-123-def
  X-Tenant-ID: ACME
  Body: { "vendorId": "v1", "amount": 1000 }

   (next step)
[SecurityFilterChain]
  ├─ BearerTokenAuthenticationFilter: Validate JWT
  ├─ TenantContextFilter: Extract tenantId = "ACME"
  ├─ IdempotencyFilter:
  │   ├─ Extract key = "abc-123-def"
  │   ├─ Capture body = '{"vendorId":"v1","amount":1000}'
  │   ├─ Query cache for "ACME:abc-123-def"
  │   ├─ MISS  (calls) proceed
  │   │
  │   [DispatcherServlet]
  │     [PurchaseRequestController.create(dto)]
  │       [PurchaseRequestService.create(dto)]
  │         [PurchaseRequestRepository.save(entity)]
  │          (calls) CREATE PR-ACME-2026-00042 in DB
  │      (next step) return 201 Created
  │
  │   └─ Response capture:
  │       ├─ Status: 201
  │       ├─ Headers: Content-Type: application/json; Location: /api/v1/.../42
  │       ├─ Body: {"id":42,"number":"PR-ACME-2026-00042",...}
  │       ├─ Store to IdempotencyService.store(...)
  │       ├─ Write cache: "ACME:abc-123-def"  (calls) IdempotencyRecord
  │       └─ Write database: INSERT into idempotency_record

[Client]
   (next step) (network hiccup, client retries)

[Client]
  POST /api/v1/procurement/purchase-requests
  Idempotency-Key: abc-123-def
  X-Tenant-ID: ACME
  Body: { "vendorId": "v1", "amount": 1000 }  ← identical

   (next step)
[IdempotencyFilter]
  ├─ Extract key = "abc-123-def"
  ├─ Capture body = '{"vendorId":"v1","amount":1000}'
  ├─ Query cache for "ACME:abc-123-def"
  ├─ HIT! Retrieved IdempotencyRecord
  ├─ Validate payload matches: [DONE]
  ├─ Return cached response immediately:
  │   ├─ Status: 201
  │   ├─ Headers: Content-Type: ...; Location: /api/v1/.../42
  │   └─ Body: {"id":42,"number":"PR-ACME-2026-00042",...}
  │
  └─ Skip controller entirely
     (No new PR created in DB)

[Client] receives 201 with same PR#42
```

---

## Error Cases

### Case 1: Payload Mismatch

```
Request 1:
  Idempotency-Key: abc-123-def
  Body: { "vendorId": "v1", "amount": 1000 }
  ← Stored in cache

Request 2:
  Idempotency-Key: abc-123-def  ← same key
  Body: { "vendorId": "v2", "amount": 2000 }  ← different payload
  ← Service detects mismatch

Response:
  409 Conflict
  {
    "error": "Idempotency-Key reused with different payload. Use a new key."
  }
```

### Case 2: Expired Cache Entry

```
Request 1 (24h ago):
  Idempotency-Key: abc-123-def
  ← Stored, now expired

Request 2 (today):
  Idempotency-Key: abc-123-def
  ← IdempotencyFilter.getIfPresent() detects TTL expired
  ← Lazy cleanup deletes record from DB
  ← Treat as cache miss, process as new request
```

### Case 3: Database Failure During Store

```
Response capture successful, but repository.save() fails
  ← Log warning
  ← Return original response to client (request succeeds)
  ← Next retry with same key: cache miss, process as new request
  (Acceptable tradeoff: idempotency not guaranteed, but request still succeeds)
```

---

## Caffeine Cache Configuration

In `application.yml`:

```yaml
spring:
  cache:
    type: caffeine
    caffeine:
      spec: "maximumSize=10000,expireAfterWrite=24h"
    cache-names:
      - idempotency
```

Or in `@Configuration`:

```java
@Bean
public CacheManager cacheManager() {
    CaffeineCacheManager cm = new CaffeineCacheManager("idempotency");
    cm.setCaffeine(
        Caffeine.newBuilder()
            .maximumSize(10000)
            .expireAfterWrite(24, TimeUnit.HOURS)
            .build()
    );
    return cm;
}
```

---

## Testing Strategy

### Unit Tests (IdempotencyService)

1. **Cache hit, payload match** — returns cached response
2. **Cache hit, payload mismatch** — throws IdempotencyException (409)
3. **Cache miss, database has record** — populates cache, returns response
4. **Expired record, lazy cleanup** — deletes from DB, returns empty
5. **Store operation** — writes to cache and database (write-through)

### Integration Tests (IdempotencyFilter)

1. **POST with Idempotency-Key, cache miss** — processes request, caches response
2. **POST retry with same key** — returns cached response immediately (verify controller not invoked)
3. **POST with different key** — processes as new request, creates new record
4. **POST without Idempotency-Key** — processes normally, no caching
5. **Multi-tenant isolation** — tenant A's key doesn't interfere with tenant B

### Contract Tests

1. **Tenant isolation** — verify composite key `(tenantId, idempotencyKey)` prevents collisions
2. **Response replay** — verify body + status + headers all returned from cache

---

## Related Documents

- `docs/ADR-001-Service-Boundary-Architecture.md` — P0.2 architectural decisions
- `ROADMAP.md` — P0 execution timeline
- `docs/superpowers/specs/` — Design specs for all P0 phases
