# Development Conventions Guide

This project follows standardized development conventions. Refer to:

1. **LLM.md** — Global development guidelines (all projects)
2. **[CLAUDE.md](../../../CLAUDE.md)** — Claude-specific extensions and overrides

## Quick Reference

### Documentation Standards

**File naming**: Title-Case for markdown files
```
[YES] Api-Reference.md
[YES] Architecture-Overview.md
[NO] api-reference.md
[NO] API_REFERENCE.md
```

**Exception**: `README.md` is always lowercase

### Git Conventions

**Branch naming**: kebab-case with prefix
```
feature/implement-multi-tenancy
fix/budget-validation-bug
docs/update-architecture
refactor/extract-validator-service
```

**Commit messages**: Conventional Commits format
```
feat(hr): add platform user linking endpoint
fix(procurement): validate budget category exists
docs(architecture): explain independence principle
```

**No auto-generated signatures** in commits:
```
[NO] Co-Authored-By: Claude Haiku <noreply@anthropic.com>
[NO] Generated with Claude Code
[YES] Just the commit message with no signature
```

### Change Tracking (IMPORTANT)

**This project is PRE-MVP**, so:
- [YES] Use **ROADMAP.md only** (no CHANGELOG.md yet)
- [YES] Track ALL changes: features, bugfixes, refactoring, docs
- [YES] Update ROADMAP.md after every completed task

**When switching to Post-MVP:**
- Switch to ROADMAP.md (features) + CHANGELOG.md (all changes)
- See `ROADMAP.md` instructions for transition process

### Project Hierarchy

```
LLM.md              # Base rules (global guidelines)
CLAUDE.md           # Claude-specific extensions (this project)
project/CLAUDE.md   # Project-specific overrides (if needed, create it)
```

**Precedence**: Project-specific > Claude-specific > Base LLM

---

## Key Conventions for werkflow-erp-sandbox

### Documentation File Names

Use Title-Case with hyphens (no spaces):

```
[YES] ADR-001-Service-Boundary-Architecture.md
[YES] Architecture-Overview.md
[YES] Independence-Checklist.md
[YES] Implementation-Summary.md
[YES] Conventions-Guide.md (this file)

[NO] adr-001-service-boundary-architecture.md
[NO] architecture overview.md
[NO] independence_checklist.md
```

### Roadmap Updates Workflow

**After completing each task in ROADMAP.md:**

1. Mark task as `[x]` (completed)
2. Add commit hash if applicable: `*(commit: abc123d)*`
3. Update `## Current Session State` with:
   - Task completed
   - Any notes about remaining work
   - Last commit hash
4. Mark next task as `[~]` (in progress) or `[ ]` (pending)
5. Commit both code and ROADMAP.md together

**Example**:
```markdown
## Current Session State

**Active Phase**: P0 — Critical Path to Production
**Current Task**: P0.1.2 — Update repository queries
**Last Commit**: abc123d (docs: add LLM.md and CLAUDE.md)
**Branch**: main

---

### P0.1 — Multi-Tenant Isolation

- [x] **P0.1.1** Add `tenantId` column to all entities
  *(commit: abc123d)*

- [~] **P0.1.2** Update all repository queries
  - Completed: EmployeeRepository, DepartmentRepository
  - Remaining: 8 more repositories in Finance, Procurement, Inventory
```

### Git Workflow

**Branching**:
1. Create feature branch from `main`
2. Make logical commits frequently
3. Rebase on latest main before pushing
4. Create PR with description template (see LLM.md)
5. Delete branch after merge

**Committing**:
```bash
# DO: Small, focused commits
git commit -m "feat(inventory): add multi-tenant scoping to queries"

# DO: Reference ROADMAP updates
git commit -m "feat(inventory): add multi-tenant scoping
- Update 5 repository methods
- Add TenantContext injection
- Verified all tests pass

ROADMAP: P0.1.2 completed"

# DON'T: Generic, vague commits
git commit -m "updates"
git commit -m "fix bugs"

# DON'T: Include generated signatures
git commit -m "feat: add feature

Co-Authored-By: Claude Haiku <noreply@anthropic.com>"
```

### Docker Conventions

**Service naming**: lowercase with dashes
```
[YES] business-service
[YES] postgres
[NO] BusinessService
[NO] BUSINESS_SERVICE
```

**Port allocation**: Documented in README.md and docker-compose.yml
```
werkflow-erp-sandbox: 8084
PostgreSQL: 5433
Keycloak: 8090
```

**Environment variables**: UPPER_SNAKE_CASE
```
[YES] POSTGRES_HOST
[YES] KEYCLOAK_URL
[YES] SERVER_PORT
[NO] postgresHost
[NO] keycloak-url
```

### API Naming Conventions

**URL paths**: kebab-case with version prefix
```
[YES] /api/v1/hr/employees
[YES] /api/v1/budget-categories
[YES] /api/v1/inventory/asset-instances
[NO] /api/v1/HR/employees
[NO] /api/v1/budgetCategories
```

**Request/Response headers**: Kebab-Case
```
[YES] X-Idempotency-Key
[YES] X-Tenant-ID
[YES] Authorization
[NO] x-idempotency-key
[NO] Xidempotencykey
```

---

## Architecture Decisions to Follow

### 1. Independence Principle

werkflow-erp-sandbox **must not import werkflow code**:
```java
[NO] import com.werkflow.engine.*;
[NO] import org.keycloak.admin.*;
[YES] import org.springframework.security.oauth2.*;
```

**Every PR review checks**: `grep -r "import com.werkflow" services/business/`

### 2. Platform User Linking (Without Coupling)

Use optional `platformUserId` field:
```java
@Nullable
private String platformUserId;  // External system links this
```

Never validate against external systems. Trust the caller.

### 3. Data Validation

**Cross-domain FK validation**: Use internal repositories, not REST calls
```java
// [YES] Correct
if (!budgetCategoryRepository.existsById(id)) {
    throw new EntityNotFoundException(...);
}

// [NO] Wrong
adminServiceClient.validateBudgetCategory(id);
```

### 4. Status Updates (Not Workflows)

Provide pure state machine endpoints:
```
PATCH /api/v1/resource/{id}/status
Body: { status: "NEW_STATUS" }
```

No conditional logic, no business rules. Caller decides what to do next.

---

## Documentation Index

| Document | Purpose | Owner |
|----------|---------|-------|
| [README.md](../../../README.md) | Quick start, API overview | Team |
| [Roadmap.md](../../../Roadmap.md) | Implementation priorities, task tracking | Team |
| LLM.md | Global conventions (commit, naming, git, docker) | Global |
| [CLAUDE.md](../../../CLAUDE.md) | Claude-specific extensions (change tracking, task execution) | Global |
| werkflow-platform `docs/adr/ERP-Sandbox/ADR-300-service-boundary-architecture.md` (platform repo) | Service boundaries, independence, user linking decisions | Architecture |
| [docs/explanation/Architecture-Overview.md](../../explanation/Architecture-Overview.md) | Visual explanation of three deployment scenarios and flow diagrams | Architecture |
| [docs/project/qa/Independence-Checklist.md](../qa/Independence-Checklist.md) | PR review checklist, anti-patterns, forbidden imports | Quality |
| [docs/project/reports/Implementation-Summary.md](../reports/Implementation-Summary.md) | Executive summary of key decisions | Summary |
| [docs/project/design-notes/Conventions-Guide.md](./Conventions-Guide.md) | This file — quick reference for conventions | Reference |

---

## Common Tasks

### Starting a New Session

1. Read `ROADMAP.md`  (calls) find first `[~]` or `[ ]` task
2. Check git log: `git log --oneline -5`
3. Confirm branch: `git status`
4. Ask: "Continue on current branch or create new feature branch?"
5. Resume work

### Completing a Task

1. Make commits with clear messages (see Git Workflow above)
2. Update `ROADMAP.md`:
   - Mark task `[x]`
   - Add commit hash
   - Update Current Session State
   - Mark next task `[~]`
3. Commit both code and ROADMAP.md: `git commit -m "..."`
4. Don't push yet (R1 rule: confirm before pushing)

### Creating a Pull Request

1. Follow template in LLM.md
2. Title: `<type>(<scope>): <subject>` (e.g., `feat(inventory): add multi-tenancy`)
3. Description: Summary, Changes, Testing, Checklist
4. Request review
5. Address feedback before merge

### Updating Documentation

1. Use Title-Case file names (e.g., `New-Feature.md`)
2. Include in ROADMAP.md if major change
3. Link from relevant index (README.md or docs/)
4. Don't add emojis (except [YES], 🚧, etc. for status)

---

## When in Doubt

1. Check existing files for patterns (they follow these conventions)
2. Refer to LLM.md and CLAUDE.md
3. Check this file (Conventions-Guide.md)
4. Ask for clarification

---

## Tools & Configuration

**Java formatting**: Google Java Style (IntelliJ built-in)
- Settings  (calls) Code Style  (calls) Java  (calls) Scheme: GoogleStyle

**Git hooks** (optional): Add to `.git/hooks/pre-commit`
```bash
#!/bin/bash
# Reject commits with "import com.werkflow" or "Co-Authored-By"
if grep -r "import com.werkflow" services/business/src; then
  echo "ERROR: werkflow imports forbidden"
  exit 1
fi
```

---

## Summary

- [YES] Use **Title-Case** for markdown filenames
- [YES] Follow **Conventional Commits** for git messages
- [YES] Update **ROADMAP.md** after completing tasks
- [YES] No **werkflow imports** in werkflow-erp-sandbox
- [YES] Use **opaque platform user IDs** (don't validate externally)
- [YES] Provide **pure status updates** (no workflows)
- [YES] **Don't push** without confirming first (unless explicitly authorized)

