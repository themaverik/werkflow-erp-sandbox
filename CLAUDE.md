# werkflow-erp-sandbox Development Configuration

**Status:** pre-MVP / graphify-enabled  
**Tech stack:** ERP domain modeling + code graph (graphify)  
**Session continuity:** ROADMAP.md tracks task state  
**Rules enforcement:** All rules auto-enforced via `.claude/rules/`

---

## Execution Workflow (as of 2026-04-15)

**Primary:** ECC + Caveman + Graphify + Claude-mem for token efficiency and code quality  
**Architectural decisions:** Superpowers (manual invoke only)

Follow strict lifecycle:
1. **Discover:** Map dependencies using `graphify`
2. **Recall:** Query `claude-mem` for existing context
3. **Plan:** Brief, targeted execution (no long multi-phase plans)
4. **Execute:** Use `caveman` for boilerplate; `ecc` skills for complex logic
5. **Record:** Save decisions/patterns to `claude-mem` immediately
6. **Purge:** Run `/compact` to clear context buffer

## Skill Triggers

- "structure", "scope", "dependencies" → `graphify` (dependency mapping)
- "quick change", "boilerplate" → `caveman` (fast file ops)
- language + "test", "build", "review" → `ecc:language-skill`
- "remember", "past pattern" → `knowledge-query` (search claude-mem)
- "update memory", "record decision", "save to memory" → `/claude-mem:do` (write to claude-mem)
- "save decision", "log pattern" → `knowledge-ingest` + `adr-manager`
- "resume", "continue" → `project-resume`
- Docker / containers → `docker-dev`

## Graphify

This project has a graphify knowledge graph at graphify-out/.

Rules:
- Before answering architecture or codebase questions, read graphify-out/GRAPH_REPORT.md for god nodes and community structure
- If graphify-out/wiki/index.md exists, navigate it instead of reading raw files
- After modifying code files in this session, run `python3 -c "from graphify.watch import _rebuild_code; from pathlib import Path; _rebuild_code(Path('.'))"` to keep the graph current

---

## Milestone Verification Protocol

Run verification **once at milestone end** (not per commit). During dev, run only affected tests.

### M1 Gate (ERP Enterprise APIs)

| Step | Command |
|------|---------|
| Primary gate | `mvn clean verify` — 255+ tests must pass; new P1.6 tests must be included |
| Review | `ecc:java-review` on new/changed controllers + services only |
| Security | `/superpowers:code-reviewer` for custody and auth endpoints only |
| E2E | Not required for M1 (pure backend) |

### Key Efficiency Rules

1. Run `graphify` first — scope exactly which files changed; never scan the whole module
2. Pass **only changed files** to `ecc` reviewer — not the whole module
3. Full test suite (`mvn clean verify`) runs once at PR/milestone end, not per commit
4. During dev: `mvn test -Dtest=<NewTestClass>` — targeted only
5. Superpowers reviewer reserved for auth/custody endpoints — one targeted review per security-sensitive PR
6. New P1.6.x endpoints must have: controller test + service test + one integration test minimum

### Dev Loop (Java — M1)

```
Write code → mvn test -Dtest=<NewTest> → ecc:java-review (changed files only) → fix → mvn clean verify at PR
```

---

## Code Quality Standards

- All new endpoints: tenant-scoped, paginated list, idempotent upsert pattern (matches P0 baseline)
- Flyway migrations: sequential versioning; never modify existing migrations
- No business logic in controllers — delegate to service layer
- All public methods have docstrings; no over-commenting obvious code
- Tests: unit (service layer) + controller test + at least one integration path per endpoint

---

## Coding Behavior (Karpathy Principles)

> Full guidelines in `karpathy-guidelines` skill. These bias toward caution over speed.

* **Think before coding:** State assumptions explicitly. Surface ambiguity. Ask before guessing.
* **Simplicity first:** Minimum code that solves the problem. No speculative features or abstractions.
* **Surgical changes:** Touch only what the task requires. Don't improve adjacent code. Match existing style.
* **Goal-driven execution:** Define verifiable success criteria before starting. Loop until verified.
