# P0.1 Multi-Tenant Isolation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement tenant-scoped repository queries, middleware-based tenant context extraction, and cross-domain isolation enforcement.

**Architecture:** Three-layer approach:
1. **TenantContext** — Extracts and stores tenantId from JWT claims (thread-local, request-scoped)
2. **TenantContextFilter** — Middleware intercepts requests, extracts tenantId, stores in ThreadLocal, clears after response
3. **Repository Updates** — All queries accept tenantId parameter, all findBy* methods filter by tenant
4. **Service Updates** — Services inject TenantContext, extract tenantId, pass to repositories
5. **BudgetCheckService** — Special-case service for cross-domain budget validation, updated to filter by tenantId

**Tech Stack:** Spring Security (OAuth2/JWT), Spring Data JPA, ThreadLocal, Custom Filters

---

## File Structure

**New Files:**
- `services/business/src/main/java/com/werkflow/business/common/context/TenantContext.java` — Utility for tenant extraction
- `services/business/src/main/java/com/werkflow/business/common/filter/TenantContextFilter.java` — Request filter for ThreadLocal storage
- `services/business/src/test/java/com/werkflow/business/common/context/TenantContextTest.java` — Unit test
- `services/business/src/test/java/com/werkflow/business/common/filter/TenantContextFilterTest.java` — Filter test
- `services/business/src/test/java/com/werkflow/business/hr/service/EmployeeServiceTenantTest.java` — Tenant isolation test (HR)

**Modified Files:**
- `services/business/src/main/java/com/werkflow/business/config/SecurityConfig.java` — Register TenantContextFilter
- All *Repository.java files (18 total) — Add tenantId parameter to queries
- All *Service.java files (12 primary) — Inject TenantContext, pass tenantId to repos
- `services/business/src/main/java/com/werkflow/business/finance/service/BudgetCheckService.java` — Add tenantId filtering
- `services/business/src/main/java/com/werkflow/business/finance/controller/BudgetCheckController.java` — Extract tenantId from request

---

## Task 1: Create TenantContext Utility

**Files:**
- Create: `services/business/src/main/java/com/werkflow/business/common/context/TenantContext.java`
- Test: `services/business/src/test/java/com/werkflow/business/common/context/TenantContextTest.java`

### Step 1.1: Write failing test for TenantContext

Create test file with failing tests:

```java
package com.werkflow.business.common.context;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;

import java.time.Instant;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class TenantContextTest {

    private TenantContext tenantContext;

    @BeforeEach
    void setUp() {
        tenantContext = new TenantContext();
        tenantContext.clear(); // Clear any previous state
    }

    @Test
    void testSetAndGetTenantId() {
        tenantContext.setTenantId("tenant-123");
        assertEquals("tenant-123", tenantContext.getTenantId());
    }

    @Test
    void testGetTenantIdThrowsWhenNotSet() {
        assertThrows(IllegalStateException.class, () -> tenantContext.getTenantId());
    }

    @Test
    void testClear() {
        tenantContext.setTenantId("tenant-123");
        tenantContext.clear();
        assertThrows(IllegalStateException.class, () -> tenantContext.getTenantId());
    }

    @Test
    void testExtractFromJwtClaim() {
        Map<String, Object> claims = new HashMap<>();
        claims.put("organization_id", "acme-corp");
        claims.put("sub", "user-123");

        Jwt jwt = new Jwt("token", Instant.now(), Instant.now().plusSeconds(3600),
            Collections.singletonMap("alg", "HS256"), claims);

        String tenantId = tenantContext.extractTenantIdFromJwt(jwt);
        assertEquals("acme-corp", tenantId);
    }

    @Test
    void testExtractFromJwtThrowsWhenClaimMissing() {
        Map<String, Object> claims = new HashMap<>();
        claims.put("sub", "user-123");

        Jwt jwt = new Jwt("token", Instant.now(), Instant.now().plusSeconds(3600),
            Collections.singletonMap("alg", "HS256"), claims);

        assertThrows(IllegalArgumentException.class, () -> tenantContext.extractTenantIdFromJwt(jwt));
    }

    @Test
    void testExtractFromAuthentication() {
        Map<String, Object> claims = new HashMap<>();
        claims.put("organization_id", "acme-corp");
        Jwt jwt = new Jwt("token", Instant.now(), Instant.now().plusSeconds(3600),
            Collections.singletonMap("alg", "HS256"), claims);

        JwtAuthenticationToken auth = new JwtAuthenticationToken(jwt,
            Collections.emptyList());

        String tenantId = tenantContext.extractTenantIdFromAuthentication(auth);
        assertEquals("acme-corp", tenantId);
    }
}
```

Run: `mvn test -Dtest=TenantContextTest`
Expected: ALL FAIL with "class not found"

### Step 1.2: Create TenantContext utility class

```java
package com.werkflow.business.common.context;

import org.springframework.security.core.Authentication;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.stereotype.Component;

/**
 * Manages tenant context for the current request.
 * Stores tenantId in ThreadLocal for request-scoped access.
 */
@Component
public class TenantContext {

    private static final ThreadLocal<String> tenantIdHolder = new ThreadLocal<>();

    /**
     * Set the current tenant ID
     */
    public void setTenantId(String tenantId) {
        if (tenantId == null) {
            throw new IllegalArgumentException("tenantId cannot be null");
        }
        tenantIdHolder.set(tenantId);
    }

    /**
     * Get the current tenant ID
     * @throws IllegalStateException if tenantId not set for this request
     */
    public String getTenantId() {
        String tenantId = tenantIdHolder.get();
        if (tenantId == null) {
            throw new IllegalStateException("Tenant ID not set. " +
                "Ensure TenantContextFilter is registered in SecurityConfig");
        }
        return tenantId;
    }

    /**
     * Extract tenant ID from JWT token
     * Looks for "organization_id" claim in JWT
     */
    public String extractTenantIdFromJwt(Jwt jwt) {
        Object orgId = jwt.getClaim("organization_id");
        if (orgId == null) {
            throw new IllegalArgumentException("JWT claim 'organization_id' not found");
        }
        return (String) orgId;
    }

    /**
     * Extract tenant ID from current Authentication
     */
    public String extractTenantIdFromAuthentication(Authentication authentication) {
        if (authentication instanceof JwtAuthenticationToken) {
            JwtAuthenticationToken jwtAuth = (JwtAuthenticationToken) authentication;
            Jwt jwt = (Jwt) jwtAuth.getPrincipal();
            return extractTenantIdFromJwt(jwt);
        }
        throw new IllegalArgumentException("Authentication is not JWT-based");
    }

    /**
     * Clear the tenant ID (called on request completion)
     */
    public void clear() {
        tenantIdHolder.remove();
    }
}
```

### Step 1.3: Run tests to verify they pass

Run: `mvn test -Dtest=TenantContextTest`
Expected: ALL PASS

### Step 1.4: Commit

```bash
git add services/business/src/main/java/com/werkflow/business/common/context/TenantContext.java
git add services/business/src/test/java/com/werkflow/business/common/context/TenantContextTest.java
git commit -m "feat(P0.1.2): create TenantContext utility for tenant extraction

- ThreadLocal-backed storage for request-scoped tenantId
- Extract from JWT claim 'organization_id'
- Validates tenantId presence before use
- Includes comprehensive unit tests"
```

---

## Task 2: Create TenantContextFilter

**Files:**
- Create: `services/business/src/main/java/com/werkflow/business/common/filter/TenantContextFilter.java`
- Test: `services/business/src/test/java/com/werkflow/business/common/filter/TenantContextFilterTest.java`

### Step 2.1: Write failing test for filter

```java
package com.werkflow.business.common.filter;

import com.werkflow.business.common.context.TenantContext;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import org.springframework.http.HttpStatus;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContext;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;

import java.io.IOException;
import java.time.Instant;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class TenantContextFilterTest {

    private TenantContextFilter filter;
    private TenantContext tenantContext;

    @Mock
    private FilterChain filterChain;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        tenantContext = new TenantContext();
        filter = new TenantContextFilter(tenantContext);
    }

    @Test
    void testFilterSetsAndClearsTenantId() throws ServletException, IOException {
        // Setup JWT with organization_id claim
        Map<String, Object> claims = new HashMap<>();
        claims.put("organization_id", "acme-corp");
        Jwt jwt = new Jwt("token", Instant.now(), Instant.now().plusSeconds(3600),
            Collections.singletonMap("alg", "HS256"), claims);

        JwtAuthenticationToken auth = new JwtAuthenticationToken(jwt, Collections.emptyList());

        // Setup SecurityContext with JWT
        SecurityContext context = mock(SecurityContext.class);
        when(context.getAuthentication()).thenReturn(auth);
        SecurityContextHolder.setContext(context);

        MockHttpServletRequest request = new MockHttpServletRequest();
        MockHttpServletResponse response = new MockHttpServletResponse();

        // Doable test: verify filter calls doFilter
        filter.doFilter(request, response, filterChain);

        // Verify filterChain was called
        verify(filterChain).doFilter(request, response);
    }

    @Test
    void testFilterClearsContextAfterChain() throws ServletException, IOException {
        Map<String, Object> claims = new HashMap<>();
        claims.put("organization_id", "acme-corp");
        Jwt jwt = new Jwt("token", Instant.now(), Instant.now().plusSeconds(3600),
            Collections.singletonMap("alg", "HS256"), claims);

        JwtAuthenticationToken auth = new JwtAuthenticationToken(jwt, Collections.emptyList());

        SecurityContext context = mock(SecurityContext.class);
        when(context.getAuthentication()).thenReturn(auth);
        SecurityContextHolder.setContext(context);

        MockHttpServletRequest request = new MockHttpServletRequest();
        MockHttpServletResponse response = new MockHttpServletResponse();

        // After filter completes, TenantContext should be cleared
        filter.doFilter(request, response, filterChain);

        // Attempt to get tenantId after filter should throw
        assertThrows(IllegalStateException.class, () -> tenantContext.getTenantId());
    }
}
```

Run: `mvn test -Dtest=TenantContextFilterTest`
Expected: ALL FAIL

### Step 2.2: Create TenantContextFilter

```java
package com.werkflow.business.common.filter;

import com.werkflow.business.common.context.TenantContext;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

/**
 * Filter that extracts tenantId from JWT and stores in TenantContext (ThreadLocal)
 * Must be registered in SecurityFilterChain before authentication filters
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class TenantContextFilter extends OncePerRequestFilter {

    private final TenantContext tenantContext;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                  HttpServletResponse response,
                                  FilterChain filterChain) throws ServletException, IOException {
        try {
            // Extract authentication from SecurityContext (set by OAuth2 filters)
            Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

            if (authentication != null && authentication.isAuthenticated()) {
                try {
                    String tenantId = tenantContext.extractTenantIdFromAuthentication(authentication);
                    tenantContext.setTenantId(tenantId);
                    log.debug("Set tenant context to: {}", tenantId);
                } catch (Exception e) {
                    log.warn("Failed to extract tenant ID from authentication: {}", e.getMessage());
                    // If tenantId extraction fails, request is rejected by SecurityConfig anyway
                }
            } else {
                log.debug("No authentication found in SecurityContext");
            }

            // Continue filter chain
            filterChain.doFilter(request, response);

        } finally {
            // Always clear tenant context after request completes
            tenantContext.clear();
            log.debug("Cleared tenant context");
        }
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) throws ServletException {
        // Don't process for public endpoints (swagger, actuator, etc)
        String path = request.getRequestURI();
        return path.startsWith("/actuator/") ||
               path.startsWith("/api-docs") ||
               path.startsWith("/swagger-ui") ||
               path.startsWith("/v3/api-docs");
    }
}
```

### Step 2.3: Run tests to verify they pass

Run: `mvn test -Dtest=TenantContextFilterTest`
Expected: ALL PASS

### Step 2.4: Commit

```bash
git add services/business/src/main/java/com/werkflow/business/common/filter/TenantContextFilter.java
git add services/business/src/test/java/com/werkflow/business/common/filter/TenantContextFilterTest.java
git commit -m "feat(P0.1.3): create TenantContextFilter for request-scoped tenant extraction

- Extends OncePerRequestFilter to ensure single execution per request
- Extracts tenantId from JWT 'organization_id' claim via TenantContext
- Stores in ThreadLocal for request scope, clears after response
- Skips processing for public endpoints (actuator, swagger, etc)
- Logs at debug/warn level for troubleshooting"
```

---

## Task 3: Register TenantContextFilter in SecurityConfig

**Files:**
- Modify: `services/business/src/main/java/com/werkflow/business/config/SecurityConfig.java`

### Step 3.1: Update SecurityConfig to add TenantContextFilter

Read the current file and add the filter to the chain:

Current structure (from earlier read):
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            // ... other config ...
        return http.build();
    }
}
```

Update to:
```java
package com.werkflow.business.config;

import com.werkflow.business.common.filter.TenantContextFilter;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.convert.converter.Converter;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.jwt.JwtDecoder;
import org.springframework.security.oauth2.jwt.NimbusJwtDecoder;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    @Value("${spring.security.oauth2.resourceserver.jwt.jwk-set-uri}")
    private String jwkSetUri;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http, TenantContextFilter tenantContextFilter) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(
                    "/actuator/**",
                    "/api-docs/**",
                    "/v3/api-docs/**",
                    "/swagger-ui/**",
                    "/swagger-ui.html"
                ).permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            // Add TenantContextFilter AFTER OAuth2 authentication filters so JWT is already parsed
            .addFilterAfter(tenantContextFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    // ... rest of the class unchanged (jwtDecoder, jwtAuthenticationConverter, corsConfigurationSource, KeycloakRoleConverter)
}
```

### Step 3.2: Run build to verify no syntax errors

Run: `mvn clean compile`
Expected: BUILD SUCCESS

### Step 3.3: Commit

```bash
git add services/business/src/main/java/com/werkflow/business/config/SecurityConfig.java
git commit -m "feat(P0.1.3): register TenantContextFilter in SecurityFilterChain

- Inject TenantContextFilter bean into securityFilterChain
- Add filter AFTER OAuth2 authentication filters so JWT is parsed
- Filter runs before authorization checks, extracts tenantId for downstream use"
```

---

## Task 4: Update HR Repository and Service (Employee, Department, Leave, etc.)

**Files:**
- Modify:
  - `services/business/src/main/java/com/werkflow/business/hr/repository/EmployeeRepository.java`
  - `services/business/src/main/java/com/werkflow/business/hr/service/EmployeeService.java`
  - All other HR repositories and services
- Test: `services/business/src/test/java/com/werkflow/business/hr/service/EmployeeServiceTenantTest.java`

### Step 4.1: Update EmployeeRepository to add tenantId filtering

Current methods filter by organizationId. Add tenant-scoped variants:

```java
package com.werkflow.business.hr.repository;

import com.werkflow.business.hr.entity.Employee;
import com.werkflow.business.hr.entity.EmploymentStatus;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.time.LocalDate;
import java.util.List;
import java.util.Optional;

/**
 * Repository for Employee entity
 */
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    // Tenant-scoped methods (NEW)
    Optional<Employee> findByTenantIdAndEmail(@Param("tenantId") String tenantId,
                                              @Param("email") String email);

    Optional<Employee> findByTenantIdAndKeycloakUserId(@Param("tenantId") String tenantId,
                                                       @Param("keycloakUserId") String keycloakUserId);

    List<Employee> findByTenantIdAndOrganizationId(@Param("tenantId") String tenantId,
                                                   @Param("organizationId") Long orgId);

    List<Employee> findByTenantIdAndDepartmentCode(@Param("tenantId") String tenantId,
                                                   @Param("code") String code);

    List<Employee> findByTenantIdAndOrganizationIdAndDepartmentCode(@Param("tenantId") String tenantId,
                                                                    @Param("organizationId") Long orgId,
                                                                    @Param("code") String code);

    List<Employee> findByTenantIdAndDoaLevelGreaterThanEqual(@Param("tenantId") String tenantId,
                                                             @Param("level") Integer level);

    List<Employee> findByTenantIdAndEmploymentStatus(@Param("tenantId") String tenantId,
                                                     @Param("status") EmploymentStatus status);

    @Query("SELECT e FROM Employee e WHERE e.tenantId = :tenantId AND e.department.id = :departmentId")
    List<Employee> findByTenantIdAndDepartmentId(@Param("tenantId") String tenantId,
                                                 @Param("departmentId") Long departmentId);

    @Query("SELECT e FROM Employee e WHERE e.tenantId = :tenantId AND e.department.id = :departmentId " +
           "AND e.employmentStatus = :status")
    List<Employee> findByTenantIdAndDepartmentIdAndStatus(@Param("tenantId") String tenantId,
                                                          @Param("departmentId") Long departmentId,
                                                          @Param("status") EmploymentStatus status);

    @Query("SELECT e FROM Employee e WHERE e.tenantId = :tenantId AND " +
           "(LOWER(e.firstName) LIKE LOWER(CONCAT('%', :searchTerm, '%')) " +
           "OR LOWER(e.lastName) LIKE LOWER(CONCAT('%', :searchTerm, '%')) " +
           "OR LOWER(e.email) LIKE LOWER(CONCAT('%', :searchTerm, '%')))")
    List<Employee> searchEmployeesByTenant(@Param("tenantId") String tenantId,
                                          @Param("searchTerm") String searchTerm);

    @Query("SELECT e FROM Employee e WHERE e.tenantId = :tenantId " +
           "AND e.dateOfJoining BETWEEN :startDate AND :endDate")
    List<Employee> findByTenantIdAndJoinDateBetween(@Param("tenantId") String tenantId,
                                                    @Param("startDate") LocalDate startDate,
                                                    @Param("endDate") LocalDate endDate);

    boolean existsByTenantIdAndEmail(@Param("tenantId") String tenantId,
                                     @Param("email") String email);

    boolean existsByTenantIdAndKeycloakUserId(@Param("tenantId") String tenantId,
                                              @Param("keycloakUserId") String keycloakUserId);

    boolean existsByTenantIdAndDepartmentCodeAndDoaLevel(@Param("tenantId") String tenantId,
                                                         @Param("code") String code,
                                                         @Param("doaLevel") Integer doaLevel);

    boolean existsByTenantIdAndDepartmentCodeAndDoaLevelAndIdNot(@Param("tenantId") String tenantId,
                                                                  @Param("departmentCode") String departmentCode,
                                                                  @Param("doaLevel") Integer doaLevel,
                                                                  @Param("id") Long id);

    long countByTenantIdAndDepartmentCode(@Param("tenantId") String tenantId,
                                          @Param("code") String code);

    long countByTenantIdAndOrganizationId(@Param("tenantId") String tenantId,
                                          @Param("organizationId") Long orgId);

    long countByTenantIdAndEmploymentStatus(@Param("tenantId") String tenantId,
                                            @Param("status") EmploymentStatus status);

    // Legacy methods (kept for backward compatibility, but deprecated)
    @Deprecated(forRemoval = false, since = "1.0.0")
    Optional<Employee> findByEmail(String email);

    @Deprecated(forRemoval = false, since = "1.0.0")
    Optional<Employee> findByKeycloakUserId(String keycloakUserId);

    @Deprecated(forRemoval = false, since = "1.0.0")
    List<Employee> findByOrganizationId(Long orgId);

    @Deprecated(forRemoval = false, since = "1.0.0")
    List<Employee> findByDepartmentCode(String code);

    @Deprecated(forRemoval = false, since = "1.0.0")
    List<Employee> findByOrganizationIdAndDepartmentCode(Long orgId, String code);

    @Deprecated(forRemoval = false, since = "1.0.0")
    List<Employee> findByDoaLevelGreaterThanEqual(Integer level);

    @Deprecated(forRemoval = false, since = "1.0.0")
    List<Employee> findByEmploymentStatus(EmploymentStatus status);

    @Deprecated(forRemoval = false, since = "1.0.0")
    @Query("SELECT e FROM Employee e WHERE e.department.id = :departmentId")
    List<Employee> findByDepartmentId(@Param("departmentId") Long departmentId);

    @Deprecated(forRemoval = false, since = "1.0.0")
    @Query("SELECT e FROM Employee e WHERE e.department.id = :departmentId AND e.employmentStatus = :status")
    List<Employee> findByDepartmentIdAndStatus(@Param("departmentId") Long departmentId,
                                               @Param("status") EmploymentStatus status);

    @Deprecated(forRemoval = false, since = "1.0.0")
    @Query("SELECT e FROM Employee e WHERE LOWER(e.firstName) LIKE LOWER(CONCAT('%', :searchTerm, '%')) " +
           "OR LOWER(e.lastName) LIKE LOWER(CONCAT('%', :searchTerm, '%')) " +
           "OR LOWER(e.email) LIKE LOWER(CONCAT('%', :searchTerm, '%'))")
    List<Employee> searchEmployees(@Param("searchTerm") String searchTerm);

    @Deprecated(forRemoval = false, since = "1.0.0")
    @Query("SELECT e FROM Employee e WHERE e.dateOfJoining BETWEEN :startDate AND :endDate")
    List<Employee> findByJoinDateBetween(@Param("startDate") LocalDate startDate,
                                         @Param("endDate") LocalDate endDate);

    @Deprecated(forRemoval = false, since = "1.0.0")
    boolean existsByEmail(String email);

    @Deprecated(forRemoval = false, since = "1.0.0")
    boolean existsByKeycloakUserId(String keycloakUserId);

    @Deprecated(forRemoval = false, since = "1.0.0")
    boolean existsByDepartmentCodeAndDoaLevel(String code, Integer doaLevel);

    @Deprecated(forRemoval = false, since = "1.0.0")
    boolean existsByDepartmentCodeAndDoaLevelAndIdNot(String departmentCode, Integer doaLevel, Long id);

    @Deprecated(forRemoval = false, since = "1.0.0")
    long countByDepartmentCode(String code);

    @Deprecated(forRemoval = false, since = "1.0.0")
    long countByOrganizationId(Long orgId);

    @Deprecated(forRemoval = false, since = "1.0.0")
    long countByEmploymentStatus(EmploymentStatus status);
}
```

### Step 4.2: Update EmployeeService to use TenantContext and call tenant-scoped repo methods

```java
package com.werkflow.business.hr.service;

import com.werkflow.business.common.context.TenantContext;
import com.werkflow.business.hr.dto.EmployeeRequest;
import com.werkflow.business.hr.dto.EmployeeResponse;
import com.werkflow.business.hr.entity.Department;
import com.werkflow.business.hr.entity.Employee;
import com.werkflow.business.hr.entity.EmploymentStatus;
import com.werkflow.business.hr.repository.DepartmentRepository;
import com.werkflow.business.hr.repository.EmployeeRepository;
import jakarta.persistence.EntityNotFoundException;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

/**
 * Service for Employee operations
 * All queries are tenant-scoped via TenantContext
 */
@Service
@RequiredArgsConstructor
@Slf4j
@Transactional(readOnly = true)
public class EmployeeService {

    private final EmployeeRepository employeeRepository;
    private final DepartmentRepository departmentRepository;
    private final RoleDisplayService roleDisplayService;
    private final TenantContext tenantContext;

    private String getTenantId() {
        return tenantContext.getTenantId();
    }

    public List<EmployeeResponse> getAllEmployees() {
        String tenantId = getTenantId();
        log.debug("Fetching all employees for tenant: {}", tenantId);
        return employeeRepository.findAll().stream()
            .filter(e -> e.getTenantId().equals(tenantId))
            .map(this::convertToResponse)
            .collect(Collectors.toList());
    }

    public EmployeeResponse getEmployeeById(Long id) {
        String tenantId = getTenantId();
        log.debug("Fetching employee by id: {} for tenant: {}", id, tenantId);
        Employee employee = employeeRepository.findById(id)
            .filter(e -> e.getTenantId().equals(tenantId))
            .orElseThrow(() -> new EntityNotFoundException("Employee not found with id: " + id));
        return convertToResponse(employee);
    }

    public EmployeeResponse getEmployeeByEmail(String email) {
        String tenantId = getTenantId();
        log.debug("Fetching employee by email: {} for tenant: {}", email, tenantId);
        Employee employee = employeeRepository.findByTenantIdAndEmail(tenantId, email)
            .orElseThrow(() -> new EntityNotFoundException("Employee not found with email: " + email));
        return convertToResponse(employee);
    }

    public EmployeeResponse getEmployeeByKeycloakUserId(String keycloakUserId) {
        String tenantId = getTenantId();
        log.debug("Fetching employee by keycloakUserId: {} for tenant: {}", keycloakUserId, tenantId);
        Employee employee = employeeRepository.findByTenantIdAndKeycloakUserId(tenantId, keycloakUserId)
            .orElseThrow(() -> new EntityNotFoundException(
                "Employee not found with keycloakUserId: " + keycloakUserId));
        return convertToResponse(employee);
    }

    public List<EmployeeResponse> getEmployeesByOrganization(Long orgId) {
        String tenantId = getTenantId();
        log.debug("Fetching employees for org: {} in tenant: {}", orgId, tenantId);
        return employeeRepository.findByTenantIdAndOrganizationId(tenantId, orgId).stream()
            .map(this::convertToResponse)
            .collect(Collectors.toList());
    }

    public List<EmployeeResponse> getEmployeesByDepartment(String deptCode) {
        String tenantId = getTenantId();
        log.debug("Fetching employees for dept: {} in tenant: {}", deptCode, tenantId);
        return employeeRepository.findByTenantIdAndDepartmentCode(tenantId, deptCode).stream()
            .map(this::convertToResponse)
            .collect(Collectors.toList());
    }

    public List<EmployeeResponse> searchEmployees(String searchTerm) {
        String tenantId = getTenantId();
        log.debug("Searching employees for term: {} in tenant: {}", searchTerm, tenantId);
        return employeeRepository.searchEmployeesByTenant(tenantId, searchTerm).stream()
            .map(this::convertToResponse)
            .collect(Collectors.toList());
    }

    public List<EmployeeResponse> getEmployeesByJoinDateRange(java.time.LocalDate startDate,
                                                               java.time.LocalDate endDate) {
        String tenantId = getTenantId();
        log.debug("Fetching employees joined between {} and {} for tenant: {}",
                  startDate, endDate, tenantId);
        return employeeRepository.findByTenantIdAndJoinDateBetween(tenantId, startDate, endDate)
            .stream()
            .map(this::convertToResponse)
            .collect(Collectors.toList());
    }

    @Transactional
    public EmployeeResponse createEmployee(EmployeeRequest request) {
        String tenantId = getTenantId();
        log.info("Creating employee in tenant: {}", tenantId);

        if (employeeRepository.existsByTenantIdAndEmail(tenantId, request.getEmail())) {
            throw new IllegalArgumentException("Employee with email already exists: " + request.getEmail());
        }

        Employee employee = Employee.builder()
            .tenantId(tenantId)
            .organizationId(request.getOrganizationId())
            .firstName(request.getFirstName())
            .lastName(request.getLastName())
            .email(request.getEmail())
            .phone(request.getPhone())
            .gender(request.getGender())
            .profilePhotoUrl(request.getProfilePhotoUrl())
            .departmentCode(request.getDepartmentCode())
            .position(request.getPosition())
            .dateOfJoining(request.getDateOfJoining())
            .employmentStatus(EmploymentStatus.ACTIVE)
            .salary(request.getSalary())
            .isActive(true)
            .build();

        if (request.getDepartmentId() != null) {
            Department dept = departmentRepository.findById(request.getDepartmentId())
                .filter(d -> d.getTenantId().equals(tenantId))
                .orElseThrow(() -> new EntityNotFoundException("Department not found"));
            employee.setDepartment(dept);
        }

        Employee saved = employeeRepository.save(employee);
        return convertToResponse(saved);
    }

    @Transactional
    public EmployeeResponse updateEmployee(Long id, EmployeeRequest request) {
        String tenantId = getTenantId();
        log.info("Updating employee {} in tenant: {}", id, tenantId);

        Employee employee = employeeRepository.findById(id)
            .filter(e -> e.getTenantId().equals(tenantId))
            .orElseThrow(() -> new EntityNotFoundException("Employee not found with id: " + id));

        // Update fields
        employee.setFirstName(request.getFirstName());
        employee.setLastName(request.getLastName());
        employee.setPhone(request.getPhone());
        employee.setGender(request.getGender());
        employee.setPosition(request.getPosition());
        employee.setDepartmentCode(request.getDepartmentCode());
        employee.setSalary(request.getSalary());

        if (request.getDepartmentId() != null) {
            Department dept = departmentRepository.findById(request.getDepartmentId())
                .filter(d -> d.getTenantId().equals(tenantId))
                .orElseThrow(() -> new EntityNotFoundException("Department not found"));
            employee.setDepartment(dept);
        }

        Employee updated = employeeRepository.save(employee);
        return convertToResponse(updated);
    }

    @Transactional
    public void deleteEmployee(Long id) {
        String tenantId = getTenantId();
        log.info("Deleting employee {} in tenant: {}", id, tenantId);

        Employee employee = employeeRepository.findById(id)
            .filter(e -> e.getTenantId().equals(tenantId))
            .orElseThrow(() -> new EntityNotFoundException("Employee not found with id: " + id));

        employeeRepository.delete(employee);
    }

    private EmployeeResponse convertToResponse(Employee employee) {
        // Conversion logic (unchanged)
        return EmployeeResponse.builder()
            .id(employee.getId())
            .firstName(employee.getFirstName())
            .lastName(employee.getLastName())
            .email(employee.getEmail())
            .phone(employee.getPhone())
            .gender(employee.getGender())
            .profilePhotoUrl(employee.getProfilePhotoUrl())
            .departmentId(employee.getDepartmentId())
            .departmentCode(employee.getDepartmentCode())
            .position(employee.getPosition())
            .dateOfJoining(employee.getDateOfJoining())
            .employmentStatus(employee.getEmploymentStatus())
            .salary(employee.getSalary())
            .isActive(employee.getIsActive())
            .build();
    }
}
```

### Step 4.3: Write tenant isolation test for EmployeeService

```java
package com.werkflow.business.hr.service;

import com.werkflow.business.common.context.TenantContext;
import com.werkflow.business.hr.dto.EmployeeRequest;
import com.werkflow.business.hr.dto.EmployeeResponse;
import com.werkflow.business.hr.entity.Employee;
import com.werkflow.business.hr.repository.DepartmentRepository;
import com.werkflow.business.hr.repository.EmployeeRepository;
import jakarta.persistence.EntityNotFoundException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.TestPropertySource;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDate;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
@Transactional
@TestPropertySource(properties = {
    "spring.datasource.url=jdbc:postgresql://localhost:5433/werkflow",
    "spring.datasource.username=werkflow_admin",
    "spring.datasource.password=secure_password_change_me"
})
class EmployeeServiceTenantTest {

    @Autowired
    private EmployeeService employeeService;

    @Autowired
    private EmployeeRepository employeeRepository;

    @Autowired
    private TenantContext tenantContext;

    @BeforeEach
    void setUp() {
        tenantContext.clear();
    }

    @Test
    void testEmployeeCreateIsScoped ToTenant() {
        // Set tenant to ACME
        tenantContext.setTenantId("acme-corp");

        EmployeeRequest req = EmployeeRequest.builder()
            .firstName("John")
            .lastName("Doe")
            .email("john@acme.com")
            .organizationId(1L)
            .dateOfJoining(LocalDate.now())
            .build();

        EmployeeResponse response = employeeService.createEmployee(req);
        assertNotNull(response.getId());

        // Verify employee was created with correct tenantId
        Employee employee = employeeRepository.findById(response.getId()).orElseThrow();
        assertEquals("acme-corp", employee.getTenantId());
    }

    @Test
    void testEmployeeQueryFilteredByTenant() {
        // Create employee in ACME tenant
        tenantContext.setTenantId("acme-corp");
        EmployeeRequest req1 = EmployeeRequest.builder()
            .firstName("John")
            .lastName("Doe")
            .email("john@acme.com")
            .organizationId(1L)
            .dateOfJoining(LocalDate.now())
            .build();
        EmployeeResponse emp1 = employeeService.createEmployee(req1);

        // Switch to BETA tenant
        tenantContext.clear();
        tenantContext.setTenantId("beta-corp");

        // Query for employee should return empty (different tenant)
        EmployeeRequest req2 = EmployeeRequest.builder()
            .firstName("Jane")
            .lastName("Smith")
            .email("jane@beta.com")
            .organizationId(2L)
            .dateOfJoining(LocalDate.now())
            .build();
        EmployeeResponse emp2 = employeeService.createEmployee(req2);

        // Verify BETA can only see its own employees
        assertEquals(1, employeeService.getAllEmployees().size());
        assertEquals(emp2.getId(), employeeService.getAllEmployees().get(0).getId());

        // Verify BETA cannot access ACME's employees
        tenantContext.clear();
        tenantContext.setTenantId("beta-corp");
        assertThrows(EntityNotFoundException.class, () -> employeeService.getEmployeeById(emp1.getId()));
    }

    @Test
    void testGetEmployeeByEmailIsScoped ToTenant() {
        // Create same email in different tenants
        tenantContext.setTenantId("acme-corp");
        EmployeeRequest req1 = EmployeeRequest.builder()
            .firstName("John")
            .lastName("Doe")
            .email("shared@test.com")
            .organizationId(1L)
            .dateOfJoining(LocalDate.now())
            .build();
        EmployeeResponse emp1 = employeeService.createEmployee(req1);

        tenantContext.clear();
        tenantContext.setTenantId("beta-corp");
        EmployeeRequest req2 = EmployeeRequest.builder()
            .firstName("Jane")
            .lastName("Smith")
            .email("shared@test.com")
            .organizationId(2L)
            .dateOfJoining(LocalDate.now())
            .build();
        EmployeeResponse emp2 = employeeService.createEmployee(req2);

        // Verify each tenant sees only its own employee
        EmployeeResponse acmeEmp = employeeService.getEmployeeByEmail("shared@test.com");
        assertEquals(emp1.getId(), acmeEmp.getId());

        tenantContext.clear();
        tenantContext.setTenantId("beta-corp");
        EmployeeResponse betaEmp = employeeService.getEmployeeByEmail("shared@test.com");
        assertEquals(emp2.getId(), betaEmp.getId());
    }
}
```

### Step 4.4: Run tests to verify isolation works

Run: `mvn test -Dtest=EmployeeServiceTenantTest`
Expected: ALL PASS

### Step 4.5: Commit

```bash
git add services/business/src/main/java/com/werkflow/business/hr/repository/EmployeeRepository.java
git add services/business/src/main/java/com/werkflow/business/hr/service/EmployeeService.java
git add services/business/src/test/java/com/werkflow/business/hr/service/EmployeeServiceTenantTest.java
git commit -m "feat(P0.1.2): tenant-scoped queries for HR (Employee Service)

- Add tenantId-filtered methods to EmployeeRepository
- Inject TenantContext into EmployeeService
- Update all queries to use tenantId-scoped repo methods
- All service methods now extract tenantId via TenantContext.getTenantId()
- Comprehensive integration tests verify cross-tenant isolation
- Mark legacy (non-tenant-scoped) methods as @Deprecated"
```

---

## Task 5: Update Finance Repository and Service (BudgetPlan, BudgetCategory, Expense, etc.)

Similar to Task 4, but for Finance domain:

**Files:**
- Modify:
  - `services/business/src/main/java/com/werkflow/business/finance/repository/*.java`
  - `services/business/src/main/java/com/werkflow/business/finance/service/*.java`

**Repositories to update:**
- BudgetPlanRepository
- BudgetCategoryRepository
- BudgetLineItemRepository
- ExpenseRepository
- ApprovalThresholdRepository

**Services to update:**
- BudgetPlanService
- BudgetCategoryService
- BudgetLineItemService
- ExpenseService
- ApprovalThresholdService

### Step 5.1: Follow same pattern as Task 4.1-4.5

For each repository:
1. Add tenantId-scoped query methods
2. Mark legacy methods @Deprecated

For each service:
1. Inject TenantContext
2. Call getTenantId() at start of each method
3. Pass tenantId to repository queries
4. Filter results to verify tenant scope

### Step 5.2: Commit after all Finance updates

```bash
git add services/business/src/main/java/com/werkflow/business/finance/repository/*.java
git add services/business/src/main/java/com/werkflow/business/finance/service/*.java
git commit -m "feat(P0.1.2): tenant-scoped queries for Finance domain

- Add tenantId-filtered methods to all Finance repositories
- Inject TenantContext into Finance services
- Update all queries to use tenantId-scoped repo methods
- All Finance services now extract tenantId via TenantContext
- 5 services updated: BudgetPlan, BudgetCategory, BudgetLineItem, Expense, ApprovalThreshold
- Mark legacy methods as @Deprecated"
```

---

## Task 6: Update Procurement Repository and Service

**Files:**
- Modify:
  - `services/business/src/main/java/com/werkflow/business/procurement/repository/*.java`
  - `services/business/src/main/java/com/werkflow/business/procurement/service/*.java`

**Repositories to update:**
- VendorRepository
- PurchaseRequestRepository
- PurchaseOrderRepository
- PrLineItemRepository
- PoLineItemRepository
- ReceiptRepository
- ReceiptLineItemRepository

### Step 6.1: Follow same pattern as Task 4

Update all 7 Procurement repositories with tenantId-scoped methods.

### Step 6.2: Commit after Procurement updates

```bash
git add services/business/src/main/java/com/werkflow/business/procurement/repository/*.java
git add services/business/src/main/java/com/werkflow/business/procurement/service/*.java
git commit -m "feat(P0.1.2): tenant-scoped queries for Procurement domain

- Add tenantId-filtered methods to all Procurement repositories
- Inject TenantContext into Procurement services
- Update all queries to use tenantId-scoped repo methods
- 7 repositories updated with tenant isolation
- Mark legacy methods as @Deprecated"
```

---

## Task 7: Update Inventory Repository and Service

**Files:**
- Modify:
  - `services/business/src/main/java/com/werkflow/business/inventory/repository/*.java`
  - `services/business/src/main/java/com/werkflow/business/inventory/service/*.java`

**Repositories to update:**
- AssetCategoryRepository
- AssetDefinitionRepository
- AssetInstanceRepository
- CustodyRecordRepository
- TransferRequestRepository
- MaintenanceRecordRepository

### Step 7.1: Follow same pattern as Task 4

### Step 7.2: Commit after Inventory updates

```bash
git add services/business/src/main/java/com/werkflow/business/inventory/repository/*.java
git add services/business/src/main/java/com/werkflow/business/inventory/service/*.java
git commit -m "feat(P0.1.2): tenant-scoped queries for Inventory domain

- Add tenantId-filtered methods to all Inventory repositories
- Inject TenantContext into Inventory services
- Update all queries to use tenantId-scoped repo methods
- 6 repositories updated with tenant isolation
- Mark legacy methods as @Deprecated"
```

---

## Task 8: Update BudgetCheckService for Cross-Domain Tenant Validation

**Files:**
- Modify:
  - `services/business/src/main/java/com/werkflow/business/finance/service/BudgetCheckService.java`
  - `services/business/src/main/java/com/werkflow/business/finance/controller/BudgetCheckController.java`
- Test: `services/business/src/test/java/com/werkflow/business/finance/service/BudgetCheckServiceTenantTest.java`

### Step 8.1: Read current BudgetCheckService to understand signature

Read the current service to understand the method signature and how it queries budgets.

### Step 8.2: Update BudgetCheckService.checkBudgetAvailability() signature

Add tenantId parameter:

```java
public BudgetCheckResponse checkBudgetAvailability(String tenantId, Long departmentId,
                                                   BigDecimal requestedAmount) {
    // Inject TenantContext is already available
    // Add tenantId scoping to all budget queries

    List<BudgetPlan> budgets = budgetPlanRepository
        .findByTenantIdAndDepartmentId(tenantId, departmentId);
    // ... rest of logic
}
```

### Step 8.3: Update BudgetCheckController to extract tenantId

```java
@RestController
@RequestMapping("/api/budget-check")
@RequiredArgsConstructor
public class BudgetCheckController {

    private final BudgetCheckService budgetCheckService;
    private final TenantContext tenantContext;

    @PostMapping("/check-availability")
    public ResponseEntity<BudgetCheckResponse> checkBudgetAvailability(
            @RequestBody BudgetCheckRequest request) {
        String tenantId = tenantContext.getTenantId();
        BudgetCheckResponse response = budgetCheckService.checkBudgetAvailability(
            tenantId,
            request.getDepartmentId(),
            request.getRequestedAmount());
        return ResponseEntity.ok(response);
    }
}
```

### Step 8.4: Write cross-tenant isolation test

```java
@Test
void testBudgetCheckPreventsCrossTenantAccess() {
    // Setup ACME budget
    tenantContext.setTenantId("acme-corp");
    BudgetPlan acmeBudget = createBudgetForTenant("acme-corp", 1L, new BigDecimal("10000"));

    // Attempt to check BETA budget against ACME's budget (should fail)
    tenantContext.clear();
    tenantContext.setTenantId("beta-corp");

    BudgetCheckResponse response = budgetCheckService.checkBudgetAvailability("beta-corp", 1L, new BigDecimal("5000"));

    // Should indicate no budget available (because beta has no budget, not accessing acme's)
    assertFalse(response.isAvailable());
}
```

### Step 8.5: Run tests to verify cross-tenant isolation

Run: `mvn test -Dtest=BudgetCheckServiceTenantTest`
Expected: ALL PASS

### Step 8.6: Commit

```bash
git add services/business/src/main/java/com/werkflow/business/finance/service/BudgetCheckService.java
git add services/business/src/main/java/com/werkflow/business/finance/controller/BudgetCheckController.java
git add services/business/src/test/java/com/werkflow/business/finance/service/BudgetCheckServiceTenantTest.java
git commit -m "feat(P0.1.4): tenant-scope BudgetCheckService for cross-domain isolation

- Update BudgetCheckService.checkBudgetAvailability() to accept tenantId parameter
- Update BudgetCheckController to extract tenantId from TenantContext
- All budget queries now filter by tenantId + departmentId
- Comprehensive tests verify cross-tenant budget data is never accessible
- Prevents tenant from accessing another tenant's budget information"
```

---

## Task 9: Build and Run Full Test Suite

### Step 9.1: Clean build with all tests

Run: `mvn clean test`
Expected: BUILD SUCCESS, ALL TESTS PASS

### Step 9.2: Commit ROADMAP progress

Update Roadmap.md to mark P0.1.2, P0.1.3, P0.1.4 as complete.

---

## Verification Checklist

After all tasks complete:

- [ ] All 4 V21 Flyway migrations applied successfully (HR, Finance, Procurement, Inventory)
- [ ] TenantContext utility tests pass
- [ ] TenantContextFilter tests pass
- [ ] EmployeeService tenant isolation tests pass
- [ ] Finance services tenant isolation tests pass
- [ ] Procurement services tenant isolation tests pass
- [ ] Inventory services tenant isolation tests pass
- [ ] BudgetCheckService cross-tenant isolation tests pass
- [ ] Full mvn test suite passes (150+ tests)
- [ ] No compiler warnings or errors
- [ ] All commits follow conventional commit format
- [ ] ROADMAP.md updated to reflect completion
