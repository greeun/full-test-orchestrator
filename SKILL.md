---
name: full-test-orchestrator
description: "Full-spectrum test orchestrator that analyzes current implementation code and generates comprehensive test suites across 10 domains (unit, api, integration, e2e, security, accessibility, performance, load/stress, smoke, chaos). Produces test scenarios and cases in tests/doc/, then executable test code. Maximizes parallel execution using Agent tool. Use when implementation is complete, after coding, before PR, or when these keywords appear: EN: \"write tests\", \"generate tests\", \"create test suite\", \"test coverage\", \"run all tests\", \"coverage report\", \"QA\", \"test generation\", \"full test\", \"test code\" | KO: \"테스트 작성\", \"테스트 생성\", \"테스트 만들어\", \"테스트 코드\", \"커버리지\", \"검증\", \"모든 테스트\", \"테스트 실행\", \"테스트 시나리오\", \"테스트 케이스\", \"QA 검증\", \"품질 검증\""
---

# Full Test Orchestrator

Generate comprehensive test suites across 10 domains with parallel execution.

## Workflow

```
Phase A: Code Analysis (sequential)
    ↓
Phase B: Test Document Generation (parallel)
    ↓
Phase C: Test Code Generation (parallel)
    ↓
Phase D: Verification (parallel → aggregate)
    ↓
Phase E: Reports (Preparation + Execution)
```

## Phase A: Code Analysis

Analyze the current implementation to understand:

1. **Project stack** — language, framework, existing test setup
2. **Test framework detection** — find existing test config (jest.config, vitest.config, pytest.ini, playwright.config, etc.)
3. **Source files changed** — identify target code for testing
4. **Dependencies & integrations** — external APIs, databases, services
5. **Existing tests** — avoid duplication, identify gaps
6. **Dedup policy** — if `docs/testing/TEST_DEDUP_POLICY.md` exists, read and enforce it

If no test framework exists, recommend one appropriate for the project and confirm with the user before proceeding.

### Dedup Check (MANDATORY)

Before generating any test, perform these checks:

1. **Search existing tests**: `grep -r "endpoint_or_function" tests/ -l` to find files already covering the target
2. **Layer ownership**: Each verification item belongs to exactly ONE layer — do NOT duplicate across layers
   - Pure function logic → unit only (NOT also in api/integration)
   - API endpoint 200/400/401 → api only (NOT also in security)
   - Auth check (401/403) → in each feature's api file only (NOT in bulk security files)
   - Browser UI flow → e2e only (NOT api tests using request.get/post in e2e folder)
   - OWASP attacks → security only (NOT in api tests)
3. **Existing file check**: If a file already tests the same endpoint/service, add TC to that file instead of creating new file
4. **Prohibited patterns**:
   - `expect(true).toBeTruthy()` placeholder tests
   - Padding assertions: `expect(typeof x).toBe('string')` after already checking value
   - Re-export existence checks: `expect(typeof fn).toBe('function')`
   - Temp verify files: `verify-*.spec.ts`
   - Testing unimplemented features with `console.log("not implemented")` skip

Output: Share analysis summary (including dedup findings) with all agents in Phase B/C.

## Phase B: Test Document Generation (Parallel)

Generate test scenarios and test cases in `tests/doc/`.

Launch 3 agents in parallel:

### Agent 1: Unit + API + Smoke
- `tests/doc/scenarios/unit-scenarios.md`
- `tests/doc/scenarios/api-scenarios.md`
- `tests/doc/scenarios/smoke-scenarios.md`
- `tests/doc/testcases/unit-testcases.md`
- `tests/doc/testcases/api-testcases.md`
- `tests/doc/testcases/smoke-testcases.md`

### Agent 2: Integration + E2E
- `tests/doc/scenarios/integration-scenarios.md`
- `tests/doc/scenarios/e2e-scenarios.md`
- `tests/doc/testcases/integration-testcases.md`
- `tests/doc/testcases/e2e-testcases.md`

### Agent 3: Security + Accessibility + Performance + Load/Stress + Chaos
- `tests/doc/scenarios/{security,accessibility,performance,load-stress,chaos}-scenarios.md`
- `tests/doc/testcases/{security,accessibility,performance,load-stress,chaos}-testcases.md`

Document format: See [references/doc-templates.md](references/doc-templates.md)
Domain-specific guidance: See [references/test-domains.md](references/test-domains.md)

## Phase C: Test Code Generation (Parallel)

Generate executable test code using same 3-agent grouping.

Rules:
- Use the project's existing test framework and conventions
- Follow the project's file structure for test placement
- If no convention exists, propose structure and confirm with user
- Each test must be independently runnable
- Include setup/teardown where needed
- Write assertion messages that clarify expected vs actual

Output: **Test Preparation Report** — see [references/report-format.md](references/report-format.md)

## Phase D: Verification

Run all generated tests in parallel where possible:

```
├─ Agent 1: unit + api + smoke tests
├─ Agent 2: integration + e2e tests
└─ Agent 3: security + a11y + performance + load/stress + chaos tests
```

Aggregate results after all agents complete.

Output: **Test Execution Report** — see [references/report-format.md](references/report-format.md)

## Phase E: Reports

Display two reports sequentially:

1. **Test Preparation Report** (after Phase C) — what was generated
2. **Test Execution Report** (after Phase D) — what passed/failed + quality criteria

## Layer Ownership Rules (Dedup)

Each test item has exactly ONE owning layer. Never test the same item in multiple layers.

| Verification Item | Owner Layer | Forbidden In |
|-------------------|-------------|--------------|
| Pure function logic (input→output) | **unit** | api, integration |
| Zod schema validation | **unit** | api |
| Constants/config values | **unit** | any other |
| Component rendering | **unit** | e2e |
| API endpoint success (200/201) | **api** | integration |
| API input validation (400) | **api** | security |
| Auth check (401) | **api** (per-feature file) | bulk security file |
| Admin check (403) | **api** (per-feature file) | bulk security file |
| Service-to-service with real DB | **integration** | unit (with full mock) |
| Browser-based user flow | **e2e** | api (using request only) |
| OWASP attack vectors | **security** | api |
| WCAG accessibility | **accessibility** | e2e |
| Response time / load | **performance** | any other |

**File rules:**
- 1 endpoint (or tightly related group) = 1 file
- No aggregation files (e.g., `management.test.ts` bundling separate endpoints)
- No temp/verify files — add regression TC to the canonical file with bug reference
- E2E files MUST use browser (`page`) — `request`-only tests go to api layer

## Quality Criteria (98%+ coverage target)

All 9 criteria apply. See [references/quality-criteria.md](references/quality-criteria.md) for details.

| Criteria | Target |
|----------|--------|
| Coverage | ≥ 98% line, ≥ 90% branch |
| Independence | No cross-test dependencies |
| Determinism | 0 flaky tests |
| Boundary values | null, empty, max, negative covered |
| Failure clarity | Assertions with expected/actual messages |
| Execution speed | unit < 1s, integration < 5s, e2e < 30s |
| Security | OWASP Top 10 covered |
| Accessibility | WCAG 2.1 AA pass |
| Performance | LCP < 2.5s, FID < 100ms |

## Parallel Execution Rules

See [references/parallel-strategy.md](references/parallel-strategy.md) for Agent tool usage patterns.

Key rules:
- Launch independent agents in a **single message** with multiple Agent tool calls
- Each agent receives the full code analysis context from Phase A
- Agents write to non-overlapping file paths (no conflicts)
- Wait for all agents to complete before proceeding to next phase

## Gap Iteration Mode (Existing Tests)

When existing tests are detected in Phase A, switch to Gap Iteration Mode instead of generating from scratch.

### Gap Iteration Workflow

```
Step 1: Investigate gaps per domain
    ↓
Step 2: Document gap plan (scenarios + cases)
    ↓
Step 3: Generate test code for gaps only
    ↓
Step 4: Run all tests (existing + new)
    ↓
Step 5: Re-investigate gaps
    ↓
  ┌─ Gaps found → repeat from Step 2
  └─ No gaps → complete with reports
```

### Step 1: Gap Investigation

For each of the 10 domains, analyze:
- **Source coverage**: Which functions/endpoints/flows lack test coverage?
- **Domain coverage**: Which domains have zero tests? (e.g., no security tests at all)
- **Quality gaps**: Existing tests missing boundary values, error paths, or edge cases?
- **Dedup check**: Ensure investigation respects layer ownership rules

Output: Gap analysis summary per domain showing what is covered vs missing.

### Step 2: Gap Plan Documentation

Write gap-specific documents to `tests/doc/`:
- `tests/doc/scenarios/{domain}-gap-scenarios.md` — scenarios for missing coverage
- `tests/doc/testcases/{domain}-gap-testcases.md` — test cases for each gap scenario

Each document clearly references:
- What existing tests already cover (to avoid duplication)
- What new tests will be added and why

### Step 3: Gap Test Code Generation

Generate test code ONLY for identified gaps:
- Add to existing test files where the same endpoint/module is partially tested
- Create new files only for entirely untested areas
- Follow all Dedup and Layer Ownership rules

### Step 4: Run All Tests

Execute both existing and newly generated tests. Collect results.

### Step 5: Re-investigate and Repeat

After execution, re-analyze for remaining gaps:
- New tests may reveal additional untested paths
- Coverage report may show branches still uncovered
- Continue iteration until:
  - Coverage target (98%+ line, 90%+ branch) is met, OR
  - No actionable gaps remain across all 10 domains

**Maximum 5 iterations** to prevent infinite loops. Report progress per iteration.

### Gap Iteration Report

After each iteration, display:

```
═══════════════════════════════════════════════════
 Gap Iteration Report (#N)
═══════════════════════════════════════════════════
 Domain          Existing  New  Total  Gaps Remaining
───────────────────────────────────────────────────
 Unit                  -    -      -              -
 API                   -    -      -              -
 ...
───────────────────────────────────────────────────
 Coverage: --% → --%  (delta: +--%)
 Status: [Continue / Complete]
═══════════════════════════════════════════════════
```

## Selective Execution

If the user requests specific domains only (e.g., "unit이랑 security만"), execute only those domains. Skip unrequested domains and adjust agent grouping accordingly.

## 10 Test Domains Reference

Quick reference — for detailed guidance see [references/test-domains.md](references/test-domains.md).

| # | Domain | Focus |
|---|--------|-------|
| 1 | Unit | Function/module isolation |
| 2 | API | Endpoint request/response |
| 3 | Integration | Module interaction |
| 4 | E2E | Full user flow |
| 5 | Security | OWASP Top 10 vulnerabilities |
| 6 | Accessibility | WCAG 2.1 AA compliance |
| 7 | Performance | User experience speed (LCP, FID, TTFB) |
| 8 | Load/Stress | System limits (concurrency, TPS, error rate) |
| 9 | Smoke | Post-deploy critical path verification |
| 10 | Chaos | Fault injection & resilience |
