---
name: full-test-orchestrator
description: "Human-level QA orchestrator that generates spec-based test suites across 10 domains, triages failures (implementation bug vs test error vs environment issue), fixes implementation code when tests correctly catch bugs, and iterates until the system is correct. Use when implementation is complete, after coding, before PR, or when these keywords appear: EN: \"write tests\", \"generate tests\", \"create test suite\", \"test coverage\", \"run all tests\", \"coverage report\", \"QA\", \"test generation\", \"full test\", \"test code\" | KO: \"테스트 작성\", \"테스트 생성\", \"테스트 만들어\", \"테스트 코드\", \"커버리지\", \"검증\", \"모든 테스트\", \"테스트 실행\", \"테스트 시나리오\", \"테스트 케이스\", \"QA 검증\", \"품질 검증\""
version: 1.0.0
---

# Full Test Orchestrator (Human-Level QA)

Generate spec-based test suites, triage failures like a human QA engineer, and fix the right code.

## Core Principle: Tests Are the Spec

> A human QA engineer tests against **requirements**, not against current implementation.
> When a test fails, the first question is: "Is the implementation wrong, or is the test wrong?"
> The default assumption is: **the implementation is wrong** until proven otherwise.

### Meaningful Test Rules (MANDATORY)

Every test must have a clear **purpose** (why it exists) and **goal** (what it proves). Before writing any test, answer these three questions:

1. **이 테스트가 없으면 어떤 버그가 프로덕션에 나갈 수 있는가?** — 답할 수 없으면 작성하지 않는다.
2. **이 검증 항목의 Layer Owner는 어디인가?** — 다른 계층에서 이미 검증하면 작성하지 않는다.
3. **이 테스트가 실패하면 어떤 행동을 취하는가?** — 행동이 불분명하면 테스트 설계가 잘못된 것이다.

**Prohibited test patterns** (produce no value — NEVER write these):

| Pattern | Example | Why it's meaningless |
|---------|---------|---------------------|
| Constant verification | `expect(API_PATH).toBe('/api/links')` | String constants don't have bugs; changes require updating both code and test |
| Existence check | `expect(typeof fn).toBe('function')` | TypeScript already guarantees this at compile time |
| Implementation copy | Duplicating implementation logic in test to compare | Tautology — tests the code against itself, not against spec |
| Placeholder | `expect(true).toBeTruthy()` | Padding that inflates count without catching anything |
| Pure delegation | Verifying A calls B (when B has its own tests) | Duplication; violates Layer Ownership |
| Static data structure | Testing menu arrays, route maps with no logic | Changes require updating test too; no bug prevention |
| Repetitive pattern | Same Rate Limit test × 14 endpoints | Test the middleware once + 2 representative endpoints |

### False Positive Detection (MANDATORY — 허위 양성 탐지)

기존 테스트가 "통과하지만 실제로 아무것도 검증하지 않는" 패턴을 적극 탐지합니다. **통과하는 테스트도 의심하라.** 통과하는 허위 양성은 **테스트가 없는 것보다 위험** — 거짓 안전감을 준다.

기존 테스트를 다룰 때(Phase A-4, Gap Iteration Step 0)는 **[references/false-positive-audit.md](references/false-positive-audit.md)를 Read**하여 허위 양성 패턴 표·탐지 grep·발견 시 조치를 적용한다.

### Anti-Bias Rules (MANDATORY)

1. **NEVER modify a test to match buggy implementation** — if the test's expected value aligns with the spec/requirement, the test is correct and the implementation must be fixed
2. **NEVER delete a failing test without root cause analysis** — every failure must be classified before any action
3. **NEVER add `skip`, `xtest`, `xit`, or `.todo` to bypass a failing test** — fix the root cause
4. **NEVER weaken assertions** (e.g., changing `toBe(200)` to `toBeTruthy()`) to make tests pass
5. **Test modification requires justification** — document WHY with one of: spec changed, test had wrong assumption, test was non-deterministic

## Workflow

```
Phase A: Spec & Code Analysis (sequential)
    ↓
Phase A+: False Positive Audit (기존 테스트 허위 양성 탐지)
    ↓
Phase A-5: Domain Coverage Gate (10개 도메인 판정 → 사용자 확인)
    ↓
Phase B: Test Document Generation (parallel)
    ↓
Phase C: Test Code Generation (parallel)
    ↓
Phase D: Execution (parallel → aggregate)
    ↓
Phase E: Failure Triage (sequential, per failure)
    ↓
Phase F: Remediation (fix implementation or test, with justification)
    ↓
Phase G: Re-verification (run all tests again)
    ↓
  ┌─ Failures remain → repeat from Phase E (max 5 iterations)
  └─ All pass → Phase H: Reports
    ↓
Phase H: Final Reports (Preparation + Execution + Triage Summary + False Positive Audit)
```

## Phase A: Spec & Code Analysis

### A-1: Spec Extraction (NEW — Human QA Foundation)

Before writing any test, extract the **expected behavior** from these sources (in priority order):

1. **Explicit specs** — README, API docs, OpenAPI/Swagger, JSDoc, design docs in `docs/`
2. **Type contracts** — TypeScript interfaces, Zod schemas, Prisma models define the data contract
3. **Route definitions** — URL patterns, HTTP methods, middleware chains define the API contract
4. **Existing tests** — passing tests that cover the same area are validated specs
5. **Code intent** — function names, variable names, comments reveal developer intent
6. **Framework conventions** — Next.js App Router conventions, middleware patterns, etc.

Output: **Spec Summary** per module/endpoint:
```
[Module/Endpoint]: [path or function name]
  Expected Input:  [types, constraints, required fields]
  Expected Output: [return type, status codes, response shape]
  Side Effects:    [DB writes, external calls, cache updates]
  Error Cases:     [what should happen on invalid input, auth failure, etc.]
  Source:          [where this spec came from — doc, type, schema, etc.]
```

### A-2: Code Analysis (existing)

1. **Project stack** — language, framework, existing test setup
2. **Test framework detection** — find existing test config (jest.config, vitest.config, pytest.ini, playwright.config, etc.)
3. **Source files changed** — identify target code for testing
4. **Dependencies & integrations** — external APIs, databases, services
5. **Existing tests** — avoid duplication, identify gaps
6. **Dedup policy** — search for any `TEST_DEDUP_POLICY.md` in the project (common locations: `docs/testing/`, `tests/doc/`, `tests/doc/legacy/`). If found, read and enforce it. If not found in a project with existing tests, **propose creating one** before generating new tests.

If no test framework exists, recommend one appropriate for the project and confirm with the user before proceeding.

### A-3: Dedup Check (MANDATORY)

Before generating any test, perform these checks:

1. **Search existing tests**: `grep -r "endpoint_or_function" tests/ -l` to find files already covering the target
2. **Layer ownership**: Each verification item belongs to exactly ONE layer — do NOT duplicate across layers
   - Pure function logic → unit only (NOT also in api/integration)
   - API endpoint 200/400/401 → api only (NOT also in security)
   - Auth check (401/403) → in each feature's api file only (NOT in bulk security files)
   - Browser UI flow → e2e only (NOT api tests using request.get/post in e2e folder)
   - OWASP attacks → security only (NOT in api tests)
3. **Existing file check**: If a file already tests the same endpoint/service, add TC to that file instead of creating new file
4. **Prohibited patterns** (see also Meaningful Test Rules above):
   - `expect(true).toBeTruthy()` placeholder tests
   - Padding assertions: `expect(typeof x).toBe('string')` after already checking value
   - Re-export existence checks: `expect(typeof fn).toBe('function')`
   - Constant verification: `expect(CONST).toBe('value')` for static strings
   - Implementation copy: duplicating source logic in test assertions
   - Static data structure tests: testing arrays/maps/configs with no conditional logic
   - Repetitive middleware tests: same pattern × N endpoints (middleware unit test + 2 representatives)
   - Pure delegation tests: only verifying A calls B when B has its own tests
   - Temp verify files: `verify-*.spec.ts`
   - Testing unimplemented features with `console.log("not implemented")` skip

Output: Share analysis summary (Spec + Code + Dedup) with all agents in Phase B/C.

### A-3.5: Dedup Claim Verification Protocol (MANDATORY — 중복 감사 시)

> **중요**: 파일명 유사성, describe 블록 이름, 또는 주석만으로 중복을 판정하면 **60% 이상 false positive**가 발생한다. 실제 사례에서 검증됨.

중복을 **주장**하려면 **3-Gate Verification**(Same Target / Same Intent / Same Layer)을 **반드시** 통과해야 한다. 3개 게이트 중 **1개라도 실패하면 중복이 아니다.** 중복을 주장하기 전에 **[references/dedup-verification.md](references/dedup-verification.md)를 Read**하여 각 Gate의 필수 증거·보고 형식·금지된 판정 근거·Sample-Check 규칙을 적용한다.

### A-4: False Positive Audit (MANDATORY — 기존 테스트가 있을 때)

기존 테스트가 존재하면, 새 테스트 생성 전에 **통과하는 기존 테스트 중 허위 양성**을 먼저 탐지합니다. 이 단계에서 **[references/false-positive-audit.md](references/false-positive-audit.md)를 Read**하여 실행 순서(grep 수집 → 컨텍스트 확인 → CRITICAL/HIGH/MEDIUM 분류)와 판별 기준을 적용한다. 결과물은 `tests/doc/FALSE_POSITIVE_AUDIT.md` — 파일별 허위 양성 목록 + 위험도 + 권장 수정.

### A-5: Domain Coverage Gate (MANDATORY — 도메인 커버리지 게이트)

> **이 게이트를 통과하지 않으면 Phase B로 진행할 수 없다.**

Phase A 분석이 끝나면, **10개 도메인 전체**에 대해 Verdict 표(PROCEED/NOT_APPLICABLE/DEFERRED)를 **빠짐없이** 작성하여 사용자에게 제시하고 확인을 받는다. "풀테스트" 요청 시 이 게이트는 건너뛸 수 없다. **에이전트가 자의적으로 도메인을 축소하는 것은 금지** — 축소는 사용자만 할 수 있다.

이 게이트 실행 시 **[references/domain-coverage-gate.md](references/domain-coverage-gate.md)를 Read**하여 10행 출력 형식·Verdict 규칙·NOT_APPLICABLE 허용 사유·Anti-Domain-Skip 금지 규칙·사용자 확인 절차를 그대로 적용한다.

## Phase B: Test Document Generation (Parallel)

Generate test scenarios and test cases in `tests/doc/`.

**IMPORTANT**: Test expected values come from Phase A-1 Spec Summary, NOT from reading the current implementation's return values.

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
- **Expected values come from spec, not from running the implementation first**

Output: **Test Preparation Report** — see [references/report-format.md](references/report-format.md)

## Phase D: Execution

Run all generated tests in parallel where possible:

```
├─ Agent 1: unit + api + smoke tests
├─ Agent 2: integration + e2e tests
└─ Agent 3: security + a11y + performance + load/stress + chaos tests
```

Aggregate results after all agents complete.

Output: Raw execution results (pass/fail per test with error output).

## Phase E: Failure Triage (Human QA Core)

**This is the critical phase that distinguishes human-level QA from naive test generation.**

For EVERY failing test, perform root cause analysis in this exact order:

### E-1: Classify Each Failure

```
For each failing test:
  1. Read the test code — what does it expect?
  2. Read the implementation code — what does it actually do?
  3. Read the spec (from Phase A-1) — what SHOULD it do?
  4. Classify:
```

| Classification | Criteria | Action |
|---|---|---|
| **IMPL_BUG** | Test expectation matches spec, but implementation deviates | Fix implementation in Phase F |
| **TEST_ERROR** | Test expectation does NOT match spec (wrong expected value, wrong setup, flawed assertion logic) | Fix test in Phase F with documented justification |
| **SPEC_AMBIGUOUS** | Spec is unclear or contradictory — cannot determine correct behavior | Ask user to clarify before proceeding |
| **ENV_ISSUE** | Test itself is correct but fails due to: missing env var, DB not running, port conflict, dependency not installed | Report environment fix needed, do not modify code |
| **FLAKY** | Test passes sometimes, fails sometimes (timing, race condition, external dependency) | Fix test to be deterministic (use mocks, fixed seeds, retries with backoff) |
| **NOT_IMPLEMENTED** | Feature described in spec but code doesn't exist yet | Report as missing implementation, do not create stub tests |

### E-2: Triage Decision Tree

```
Test fails
  │
  ├─ Does the expected value match the spec?
  │   ├─ YES → Implementation is wrong (IMPL_BUG)
  │   │         → Go to Phase F: fix implementation
  │   │
  │   └─ NO → Is the spec clear?
  │       ├─ YES → Test is wrong (TEST_ERROR)
  │       │         → Go to Phase F: fix test with justification
  │       │
  │       └─ NO → (SPEC_AMBIGUOUS)
  │                → Ask user for clarification
  │
  ├─ Is it an environment/infra error? (connection refused, ENOENT, timeout on setup)
  │   └─ YES → (ENV_ISSUE)
  │             → Report, skip this test for now
  │
  ├─ Does it pass on retry but fail intermittently?
  │   └─ YES → (FLAKY)
  │             → Fix determinism issue
  │
  └─ Does the function/endpoint not exist at all?
      └─ YES → (NOT_IMPLEMENTED)
                → Report as missing feature
```

### E-3: Triage Output

For each failure, document:

```
─── Failure Triage ───────────────────────────────
Test:           [test file:line — test name]
Classification: [IMPL_BUG | TEST_ERROR | SPEC_AMBIGUOUS | ENV_ISSUE | FLAKY | NOT_IMPLEMENTED]
Expected:       [what the test expects]
Actual:         [what the implementation returned]
Spec Says:      [what the spec defines as correct]
Root Cause:     [1-2 sentence explanation]
Action:         [specific fix to apply]
──────────────────────────────────────────────────
```

## Phase F: Remediation

Apply fixes based on Phase E classification. **Each fix type has different rules.**

### F-1: IMPL_BUG — Fix Implementation Code

- Read the failing test to understand the expected behavior
- Read the spec to confirm the expected behavior
- Modify the **implementation source code** (NOT the test)
- Keep the fix minimal — only change what's needed to satisfy the spec
- Do NOT refactor surrounding code
- If the fix is non-trivial (> 20 lines changed), ask user to confirm approach before applying

### F-2: TEST_ERROR — Fix Test Code (with justification)

- **MUST document justification** in a comment above the fixed test:
  ```
  // Fixed: [reason] — spec says [X], test incorrectly expected [Y]
  ```
- Common valid reasons:
  - Wrong HTTP status code expected (spec says 201, test had 200)
  - Incorrect mock setup (mocked return value doesn't match actual schema)
  - Wrong field name in assertion (typo or outdated field)
  - Test was asserting implementation detail, not behavior
- **INVALID reasons** (do NOT use these to justify test changes):
  - "Implementation returns X instead of Y" (that's an IMPL_BUG)
  - "Easier to change the test" (laziness is not justification)
  - "The test is too strict" (strictness catches bugs — keep it)

### F-3: SPEC_AMBIGUOUS — Escalate to User

- Present the ambiguity clearly:
  ```
  Spec says: [quote or paraphrase]
  Test expects: [value]
  Implementation does: [value]
  Question: Which is the intended behavior?
  ```
- Wait for user response before proceeding
- After user clarifies, classify as IMPL_BUG or TEST_ERROR and fix accordingly

### F-4: ENV_ISSUE — Report and Skip

- List required environment setup
- Do NOT modify code
- Move to next failure

### F-5: FLAKY — Fix Determinism

- Replace time-dependent logic with mocked clocks
- Replace random values with fixed seeds
- Replace real network calls with mocks (in unit/api tests)
- Add proper `waitFor` / polling in E2E tests instead of fixed `sleep`

### F-6: NOT_IMPLEMENTED — Report as Gap

- Add to a "Missing Implementation" section in the report
- Do NOT create placeholder implementations
- Do NOT create tests that expect undefined behavior

## Phase G: Re-verification

After Phase F fixes are applied:

1. **Run ALL tests** (not just the fixed ones) — fixes may cause regressions
2. **Compare results** against Phase D execution
3. **If new failures appear**: return to Phase E for triage (these are likely regressions from Phase F fixes)
4. **Iteration limit**: Maximum 5 triage-fix-verify cycles

### Re-verification Loop

```
Iteration 1: Phase D → E → F → G (run all)
  ├─ All pass → proceed to Phase H
  └─ New failures → Iteration 2
      Iteration 2: Phase E → F → G (run all)
        ├─ All pass → proceed to Phase H
        └─ New failures → Iteration 3 (max 5)
```

### Regression Guard

If a previously passing test now fails after a Phase F fix:
- The Phase F fix likely introduced a regression
- **Revert the fix** and find a non-breaking approach
- NEVER fix the regression by weakening the now-failing test

## Phase H: Final Reports

Display three reports sequentially:

1. **Test Preparation Report** (from Phase C) — what was generated
2. **Test Execution Report** (from Phase G final run) — what passed/failed
3. **Triage Summary Report** (NEW) — what was fixed and how

### Triage Summary Report Format

세 보고서의 상세 템플릿(Preparation / Execution / Triage Summary)은 [references/report-format.md](references/report-format.md)에 있다. Phase H 진입 시 이 파일을 Read하여 Triage Summary Report(및 축약 변형)를 렌더링한다. Triage Summary는 반드시 **구현 수정과 테스트 수정을 구분**하고, **모든 테스트 수정에 근거(justification)를 기록**한다.

## Layer Ownership Rules (Dedup)

각 테스트 항목은 정확히 하나의 소유 계층(owning layer)을 가진다. 같은 항목을 여러 계층에서 테스트하지 않는다. 계층별 소유권 표(unit/api/integration/e2e/security/accessibility/performance)와 파일 규칙(1 엔드포인트=1 파일, 집계 파일 금지, temp/verify 파일 금지, E2E는 반드시 브라우저 `page` 사용)은 Phase A-3 Dedup Check와 Phase B/C 배치 시 **[references/layer-ownership.md](references/layer-ownership.md)를 Read**하여 적용한다.

## Quality Criteria (98%+ coverage target)

All 9 criteria apply. See [references/quality-criteria.md](references/quality-criteria.md) for details.

| Criteria | Target |
|----------|--------|
| Coverage | >= 98% line, >= 90% branch |
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

Phase A에서 기존 테스트가 감지되면, 처음부터 생성하는 대신 **Gap Iteration Mode**로 전환한다. 워크플로우는 Step 0 False Positive Audit → Step 0.5 Domain Coverage Gate → Step 1 Gap Investigation → Step 2 Gap Plan → Step 3 Gap Code(+허위 양성 수정) → Step 4 Run all → Step 5-6 Triage/Remediate(Phase E·F 규칙 적용) → Step 7 Re-verify로, gap이 없고 전부 통과할 때까지 **최대 5회** 반복한다.

이 모드 진입 시 **[references/gap-iteration.md](references/gap-iteration.md)를 Read**하여 각 Step 상세와 Gap Iteration Report 형식을 적용한다.

## Production Readiness Audit (프로덕션 출시 관점 감사)

테스트 수치(TC 수, 파일 수)가 아니라 **"이 테스트를 모두 통과하면 프로덕션에서 장애 없이 운영될 수 있는가?"**를 기준으로 평가한다. Phase A에서 기존 테스트가 있을 때, Gap Investigation 전에 이 감사를 먼저 수행한다.

이 단계에서 **[references/production-readiness.md](references/production-readiness.md)를 Read**하여 필수 검증 체크리스트(장애 복원력·보안 실질성·사용자 여정 완결성·정합성), 운영 안정성 점검(코드 외), 성능 벤치마크(p50/p95/p99) 기준을 적용한다.

## Selective Execution

### 명시적 도메인 지정 시
사용자가 특정 도메인만 요청하면 (e.g., "unit이랑 security만") 해당 도메인만 실행한다. 나머지는 스킵.

### 풀테스트 / 전체 테스트 요청 시 (기본값)
"풀테스트", "전체 테스트", "모든 테스트", "full test", "generate tests" 등 **도메인을 한정하지 않는 요청**은 **10개 도메인 전체 평가가 기본값**이다.

- Phase A-5 Domain Coverage Gate를 **반드시** 실행
- NOT_APPLICABLE / DEFERRED 판정은 A-5 규칙에 따름
- **에이전트가 자의적으로 도메인을 축소하는 것은 금지** — 축소는 사용자만 할 수 있음

### 도메인 축소 권한
| 주체 | 축소 가능 여부 |
|------|-------------|
| 사용자가 명시적으로 제외 | ✅ 허용 |
| 사용자 확인 후 에이전트가 DEFERRED 처리 | ✅ 허용 |
| 에이전트가 단독으로 도메인 스킵 | ❌ **금지** |

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
