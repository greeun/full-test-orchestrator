---
name: full-test-orchestrator
description: "Human-level QA orchestrator that generates spec-based test suites across 10 domains, triages failures (implementation bug vs test error vs environment issue), fixes implementation code when tests correctly catch bugs, and iterates until the system is correct. Use when implementation is complete, after coding, before PR, or when these keywords appear: EN: \"write tests\", \"generate tests\", \"create test suite\", \"test coverage\", \"run all tests\", \"coverage report\", \"QA\", \"test generation\", \"full test\", \"test code\" | KO: \"테스트 작성\", \"테스트 생성\", \"테스트 만들어\", \"테스트 코드\", \"커버리지\", \"검증\", \"모든 테스트\", \"테스트 실행\", \"테스트 시나리오\", \"테스트 케이스\", \"QA 검증\", \"품질 검증\""
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

기존 테스트가 "통과하지만 실제로 아무것도 검증하지 않는" 패턴을 적극 탐지합니다. **통과하는 테스트도 의심하라.**

| 허위 양성 패턴 | 예시 | 왜 위험한가 |
|---------------|------|-----------|
| Null 통과 검증 | `expect(result !== undefined).toBe(true)` | `null !== undefined`은 `true` — 폴백 실패해도 통과 |
| 느슨한 상태 코드 허용 | `expect([200, 201, 403, 429]).toContain(status)` | Rate Limit 비활성화돼도 통과 |
| .catch 후 null 허용 | `.catch(() => null)` → `if (result) { ... } else { expect(true).toBe(true) }` | 에러 발생 시 검증 자체를 건너뜀 |
| 조건부 검증 skip | `if (status === 429) { expect(...) } else { console.log("[SKIP]") }` | 핵심 동작 미작동해도 통과 |
| E2E graceful skip | `isVisible().catch(() => false)` → `console.log("[SKIP]")` | UI 회귀 발생해도 감지 불가 |
| 빈 테스트 본문 | `test("TC-001: 기능", async () => { /* TODO */ })` | 테스트 존재한다는 착각만 줌 |

**탐지 방법:**
1. `grep -r "expect(true)" tests/` — 무의미한 assertion
2. `grep -r "\[SKIP\]" tests/` — graceful skip 패턴
3. `grep -r "!== undefined" tests/` — null 통과 검증
4. `grep -r "expect(\[" tests/` — 느슨한 status 허용
5. `grep -rn "\.catch.*=> null" tests/` — 에러 삼킴 패턴

**발견 시 조치:**
- CRITICAL (핵심 기능 검증 skip) → `test.fail(true, "설명")` + `fixme` annotation으로 전환
- MEDIUM (부가 기능 skip) → `test.info().annotations.push({ type: "fixme" })` 추가
- 허위 양성은 **테스트가 없는 것보다 위험** — 거짓 안전감을 줌

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

중복을 **주장**하려면 아래 **3-Gate Verification**을 **반드시** 통과해야 한다. 3개 게이트 중 **1개라도 실패하면 중복이 아니다.**

#### Gate 1: Same Target Verification (동일 대상)
두 파일이 같은 함수/엔드포인트/플로우를 검증하는가?

**필수 증거** (grep 결과 직접 인용):
```bash
# 두 파일에서 실제로 import하는 함수가 같은가?
grep -n "^import" <fileA>
grep -n "^import" <fileB>

# 두 파일에서 호출하는 메서드가 같은가?
grep -n "\.methodName\|functionName(" <fileA>
grep -n "\.methodName\|functionName(" <fileB>
```

- 실패 예시: A는 `validateAliasBasic`, B는 `validateAliasData` → 다른 함수 → **Gate 1 실패, 중복 아님**

#### Gate 2: Same Intent Verification (동일 의미)
두 파일의 검증 **의도**가 같은가? (정상 비즈니스 vs 보안 공격 vs 성능은 모두 다른 의도)

**필수 증거**: 두 파일의 실제 test 본문을 읽고 인용한다. 이름만으로 판단 금지.

- 실패 예시: 둘 다 `/api/auth/login` 401을 검증하지만, A는 "잘못된 비밀번호"(비즈니스), B는 "SQL Injection 페이로드"(보안 공격) → **Gate 2 실패, 중복 아님**

#### Gate 3: Same Layer Verification (동일 계층)
Layer Ownership 기준으로 두 파일이 같은 owner인가?

**필수 증거**: 각 파일이 HTTP 요청을 사용하는지, 서비스 함수를 직접 호출하는지, DB를 mock하는지 real로 쓰는지 확인.

- 실패 예시: A는 `request.post("/api/links")` (HTTP), B는 `createLink(...)` 직접 호출 + real DB (서비스 계층) → 다른 계층 → **Gate 3 실패, 중복 아님**

#### 3-Gate 보고 형식 (각 중복 주장마다 반드시 포함)

```
─── 중복 후보 ──────────────────────────────────
대상:           [함수명 또는 엔드포인트]
파일 A:         [경로:라인]
파일 B:         [경로:라인]

Gate 1 (Same Target):
  A import: [grep 결과 인용]
  B import: [grep 결과 인용]
  결과:     [PASS / FAIL]

Gate 2 (Same Intent):
  A describe/test: [실제 인용]
  B describe/test: [실제 인용]
  결과:             [PASS / FAIL]

Gate 3 (Same Layer):
  A uses: [HTTP / service call / mock / real DB]
  B uses: [HTTP / service call / mock / real DB]
  결과:   [PASS / FAIL]

판정:      [3-Gate 모두 PASS → 중복 / 1개라도 FAIL → 중복 아님]
권장 조치: [구체적 파일/라인]
─────────────────────────────────────────────────
```

#### 금지된 판정 근거

다음 근거로 중복을 주장하면 **자동 기각**:

- ❌ "파일명이 비슷하다"
- ❌ "같은 디렉토리에 있다"
- ❌ "둘 다 401/403을 검증한다" (의도 미검증)
- ❌ "describe 이름이 같다" (본문 미확인)
- ❌ "4개 파일에 분산되어 있다" (각 파일별 grep 증거 미첨부)
- ❌ "Layer Ownership 위반으로 보인다" (실제 코드 미확인)

#### Sample-Check Recommendation

중복 보고서가 10건 이상일 때, **HIGH 항목의 최소 30%를 사용자가 샘플 검증** 후에만 권장 조치를 수행한다. 자동 적용 금지.

### A-4: False Positive Audit (MANDATORY — 기존 테스트가 있을 때)

기존 테스트가 존재하면, 새 테스트 생성 전에 **통과하는 기존 테스트 중 허위 양성**을 먼저 탐지합니다.

**실행 순서:**
1. 아래 grep 명령을 순차 실행하여 의심 파일 수집
2. 각 파일의 해당 줄 컨텍스트를 읽고 실제 허위 양성인지 판별
3. 위험도 분류 (CRITICAL / HIGH / MEDIUM)
4. CRITICAL은 Phase C에서 수정 코드 포함, 나머지는 감사 보고서에 기록

```bash
# 1. 무의미한 assertion
grep -rn "expect(true)" tests/

# 2. graceful skip 패턴
grep -rn "\[SKIP\]" tests/

# 3. null 통과 검증
grep -rn "!== undefined" tests/

# 4. 느슨한 상태 코드 허용
grep -rn "expect(\[" tests/ | grep -i "contain"

# 5. 에러 삼킴 후 null 허용
grep -rn "\.catch.*=> null" tests/

# 6. 빈 테스트 본문
grep -rn "// \[.*REQUIRED\]\|// TODO\|// PLACEHOLDER" tests/
```

**판별 기준:**
- `expect(result !== undefined).toBe(true)` → **허위 양성** (null도 통과)
- `expect([200, 201, 403, 429]).toContain(status)` → **허위 양성** (핵심 검증 회피)
- `isVisible().catch(() => false)` → if/else → `console.log("[SKIP]")` → **허위 양성** (UI 회귀 미감지)
- `test("TC-001", async () => { /* [REQUIRED] */ })` → **빈 테스트** (검증 없음)

**결과물:** `tests/doc/FALSE_POSITIVE_AUDIT.md` — 파일별 허위 양성 목록 + 위험도 + 권장 수정

### A-5: Domain Coverage Gate (MANDATORY — 도메인 커버리지 게이트)

> **이 게이트를 통과하지 않으면 Phase B로 진행할 수 없다.**

Phase A 분석이 끝나면, **10개 도메인 전체**에 대해 아래 표를 **빠짐없이** 작성하여 사용자에게 제시한다.
"풀테스트" 요청 시 이 게이트는 건너뛸 수 없다.

#### 출력 형식 (10행 모두 필수)

```
═══════════════════════════════════════════════════
 Domain Coverage Gate — Phase A-5
═══════════════════════════════════════════════════
 #   Domain          Verdict          Reason / Scope
─────────────────────────────────────────────────── 
 1   Unit            PROCEED(N)       N개 함수/모듈 대상
 2   API             PROCEED(N)       N개 엔드포인트 대상
 3   Integration     PROCEED(N)       N개 서비스 연동 대상
 4   E2E             PROCEED(N)       N개 사용자 여정 대상
 5   Security        PROCEED(N)       N개 OWASP 항목 대상
 6   Accessibility   NOT_APPLICABLE   사유: [구체적 이유]
 7   Performance     PROCEED(N)       N개 함수/페이지 대상
 8   Load/Stress     DEFERRED         사유: [구체적 이유]
 9   Smoke           PROCEED(N)       N개 크리티컬 경로 대상
 10  Chaos           DEFERRED         사유: [구체적 이유]
═══════════════════════════════════════════════════
 PROCEED: N개 도메인 / DEFERRED: N개 / NOT_APPLICABLE: N개
═══════════════════════════════════════════════════
```

#### Verdict 규칙

| Verdict | 의미 | 조건 |
|---------|------|------|
| **PROCEED(N)** | 이 도메인에서 N개 TC를 생성한다 | 테스트 가능한 대상이 1개 이상 존재 |
| **NOT_APPLICABLE** | 이 프로젝트에 해당 도메인이 적용되지 않는다 | 아래 허용 사유 목록에 해당하는 경우만 |
| **DEFERRED** | 적용 가능하지만 현재 환경에서 실행 불가 | 인프라/의존성 부재 (DB, Redis, 브라우저 등) |

#### NOT_APPLICABLE 허용 사유 (이 목록에 없는 사유는 인정하지 않음)

| 도메인 | 유일하게 허용되는 NOT_APPLICABLE 사유 |
|--------|-------------------------------------|
| Unit | 프로젝트에 순수 함수/모듈이 단 하나도 없음 (극히 드묾) |
| API | 프로젝트에 API 엔드포인트가 없음 (프론트엔드 전용 등) |
| Integration | 서비스 간 연동이 없는 단일 모듈 프로젝트 |
| E2E | UI가 없는 순수 백엔드/라이브러리 프로젝트 |
| Security | API/입력 처리가 없는 순수 유틸리티 라이브러리 |
| Accessibility | 브라우저 UI가 없는 프로젝트 |
| Performance | 성능이 관심사가 아닌 내부 도구 (사용자 지정 시만) |
| Load/Stress | 동시성/부하가 관심사가 아닌 프로젝트 (사용자 지정 시만) |
| Smoke | 배포 파이프라인이 없는 로컬 전용 프로젝트 |
| Chaos | 외부 서비스 의존이 없는 프로젝트 |

#### DEFERRED 처리

DEFERRED 도메인은 **Phase H 보고서에 "미실행 도메인" 섹션으로 기록**하고, 실행에 필요한 환경 설정을 안내한다.

#### 금지 규칙 (Anti-Domain-Skip)

다음 근거로 도메인을 NOT_APPLICABLE 처리하면 **자동 기각**:

| 금지 사유 | 왜 금지인가 |
|----------|-----------|
| ❌ "이 도메인은 불필요하다" | 사용자가 명시적으로 제외하지 않은 도메인은 필수 |
| ❌ "Mock/인프라 설정이 복잡하다" | 복잡성은 DEFERRED 사유이지 NOT_APPLICABLE 사유가 아님 |
| ❌ "TC 수준의 meaningful test가 없다" | TC 품질 규칙은 개별 TC에 적용. 도메인 전체 스킵 근거 아님 |
| ❌ "기존 E2E가 이미 커버한다" | Layer Ownership 위반 — 각 도메인은 고유 검증 의도가 다름 |
| ❌ "사용자가 불필요한 테스트 금지라고 했다" | **불필요 = 무의미한 개별 TC. 도메인 스킵이 아님** |
| ❌ "프레임워크/인프라 코드는 테스트 불필요" | 인프라 위에 놓인 비즈니스 로직은 테스트 대상 |

> **핵심 구분**: "불필요한 테스트 작성 금지" = 도메인 내 무의미 TC 배제. ≠ 도메인 자체를 배제.
> 에이전트는 **도메인 범위 결정**과 **TC 품질 결정**을 반드시 분리해야 한다.

#### 사용자 확인

게이트 표를 제시한 후 사용자에게 확인을 구한다:
- "위 도메인 범위로 진행할까요? 제외하거나 추가할 도메인이 있으면 알려주세요."
- 사용자가 승인하면 Phase B 진행
- 사용자가 도메인을 제거하면 해당 도메인만 Selective Execution 적용

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

```
═══════════════════════════════════════════════════
 Triage Summary Report
═══════════════════════════════════════════════════
 Iterations: N / 5 max
 Total Failures Triaged: N

 ─── By Classification ───────────────────────────
 IMPL_BUG:        N fixed  (implementation code changed)
 TEST_ERROR:      N fixed  (test code changed, justified)
 SPEC_AMBIGUOUS:  N resolved (user clarified)
 ENV_ISSUE:       N reported (environment setup needed)
 FLAKY:           N fixed  (determinism improved)
 NOT_IMPLEMENTED: N reported (features missing)

 ─── Implementation Fixes ────────────────────────
 File                        Change Summary
 src/lib/services/foo.ts     Fixed return type for edge case
 src/app/api/bar/route.ts    Added missing 404 handling
 ...

 ─── Test Fixes (with justification) ─────────────
 File                        Justification
 tests/unit/foo.test.ts      Spec says 201, test had 200
 tests/api/bar.test.ts       Mock setup used wrong schema
 ...

 ─── Unresolved ──────────────────────────────────
 [List any failures that remain after max iterations]

 ─── Regression Events ───────────────────────────
 [List any fixes that were reverted due to regression]
═══════════════════════════════════════════════════
```

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

When existing tests are detected in Phase A, switch to Gap Iteration Mode instead of generating from scratch.

### Gap Iteration Workflow

```
Step 0: False Positive Audit (기존 통과 테스트 중 허위 양성 탐지)
    ↓
Step 0.5: Domain Coverage Gate (Phase A-5 — 10개 도메인 판정 → 사용자 확인)
    ↓
Step 1: Investigate gaps per PROCEED 도메인
    ↓
Step 2: Document gap plan (scenarios + cases)
    ↓
Step 3: Generate test code for gaps only + 허위 양성 수정
    ↓
Step 4: Run all tests (existing + new)
    ↓
Step 5: Triage any failures (Phase E rules apply!)
    ↓
Step 6: Remediate (Phase F rules apply!)
    ↓
Step 7: Re-verify and re-investigate gaps
    ↓
  ┌─ Gaps found → repeat from Step 2
  └─ No gaps + all pass → complete with reports
```

### Step 0: False Positive Audit

**기존 테스트가 존재할 때 가장 먼저 실행합니다.** Gap을 찾기 전에, 통과하는 테스트가 실제로 검증을 하고 있는지 확인합니다.

Phase A-4의 grep 명령을 실행하여 허위 양성을 수집하고, Step 3에서 gap 코드와 함께 수정합니다.

허위 양성을 먼저 수정하지 않으면:
- Gap 분석 시 "이미 테스트됨"으로 잘못 판별 → 실제 gap을 놓침
- 허위 양성 TC가 커버리지에 포함 → 실제 커버리지보다 높게 측정

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

### Step 5-6: Triage and Remediate

**Apply Phase E (Triage) and Phase F (Remediation) rules to ALL failures.**

This is the critical difference from the old workflow:
- Old: failures → adjust tests → re-run
- New: failures → classify → fix the RIGHT code → re-run

### Step 7: Re-investigate and Repeat

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

 Triage This Iteration:
   IMPL_BUG: N fixed | TEST_ERROR: N fixed | ENV_ISSUE: N

 Status: [Continue / Complete]
═══════════════════════════════════════════════════
```

## Production Readiness Audit (프로덕션 출시 관점 감사)

테스트 수치(TC 수, 파일 수)가 아니라 **"이 테스트를 모두 통과하면 프로덕션에서 장애 없이 운영될 수 있는가?"**를 기준으로 평가합니다.

### 필수 검증 체크리스트

Phase A에서 기존 테스트가 있을 때, Gap Investigation 전에 이 체크리스트를 먼저 수행합니다.

#### 1. 장애 복원력 (Resilience)

| 검증 항목 | 질문 | 테스트 위치 |
|----------|------|-----------|
| 캐시 장애 폴백 | Redis 다운 시 DB 직접 조회로 핵심 기능이 동작하는가? | chaos |
| 외부 서비스 장애 | API 제공자 timeout/500 시 graceful degradation 하는가? | chaos |
| 메모리 누수 방지 | 재시도 큐, 버퍼 등에 크기 상한(cap)이 있는가? | unit |
| 동시성 안전 | 같은 리소스 동시 생성 시 unique constraint 처리가 되는가? | unit, integration |
| 트랜잭션 부분 실패 | 다단계 트랜잭션에서 중간 실패 시 데이터 일관성이 보장되는가? | integration |
| 서버 재시작 시 데이터 유실 | 인메모리 버퍼 flush 전 종료 시 허용 정책이 있는가? | 문서화 |

#### 2. 보안 실질성 (Security Substance)

| 검증 항목 | 질문 | 테스트 위치 |
|----------|------|-----------|
| Rate Limit 임계값 | 실제로 limit을 초과하면 429가 반환되는가? (느슨한 검증 금지) | unit |
| JWT 실제 서명 | mock이 아닌 실제 서명/검증이 동작하는가? | integration |
| OAuth 콜백 처리 | 외부 제공자 mock 없이 빈 테스트로 남겨져 있지 않은가? | e2e |
| 구독 플랜 강제 | API 레벨에서 플랜 제한이 실제 적용되는가? (UI 안내만 검증은 불충분) | unit, api |

#### 3. 사용자 여정 완결성 (User Journey Completeness)

| 검증 항목 | 질문 | 테스트 위치 |
|----------|------|-----------|
| 비회원 전체 여정 | 홈→입력→생성→복사→리다이렉트가 하나의 연속 시나리오로 검증되는가? | e2e |
| 회원 전체 여정 | 로그인→생성→분석확인→수정→삭제가 연속으로 검증되는가? | e2e |
| 모바일 전용 | 모바일 뷰포트에서 핵심 기능이 별도 spec으로 검증되는가? | e2e |
| i18n 실질성 | 언어 전환 후 실제 해당 언어 텍스트가 표시되는가? (번역 키 누락 감지) | e2e |
| 에러 상태 UX | 404, 만료, 서버 에러 시 사용자에게 적절한 안내가 표시되는가? | e2e |

#### 4. 정합성 (Consistency)

| 검증 항목 | 질문 | 테스트 위치 |
|----------|------|-----------|
| 캐시-DB 정합성 | 삭제 후 캐시 무효화 전 stale 데이터로 리다이렉트되지 않는가? | unit |
| 외부 서비스-DB 정합성 | 외부 API 실패 시 DB에 고아 레코드가 남지 않는가? | unit |
| 에러 응답 일관성 | 모든 API 에러 응답이 동일한 형식인가? (내부 정보 미노출) | api |

### 운영 안정성 점검 (코드 외)

테스트 통과만으로는 프로덕션 안정성을 보장할 수 없습니다. 다음 항목도 확인합니다.

| 항목 | 확인 내용 | 산출물 |
|------|----------|--------|
| 헬스체크 API | DB/Redis/어플리케이션 메트릭이 단일 엔드포인트에서 확인 가능한가? | `/api/health` 강화 |
| Circuit Breaker | 외부 서비스 장애 시 자동 차단 + 복구 로직이 있는가? | `circuit-breaker.ts` |
| 롤백 플레이북 | 배포 후 장애 시 5분 이내 복구 절차가 문서화되어 있는가? | `ROLLBACK_PLAYBOOK.md` |
| DB 마이그레이션 롤백 | 모든 마이그레이션에 rollback.sql이 존재하는가? | `prisma/migrations/*/rollback.sql` |
| Graceful Degradation 정책 | 부가 서비스 장애 시 핵심 기능 계속 동작하는 정책이 명시되어 있는가? | `GRACEFUL_DEGRADATION_POLICY.md` |

### 성능 벤치마크 (Performance Baseline)

07-performance 도메인에 반드시 포함해야 할 벤치마크:

```typescript
// 핵심 함수 p50/p95/p99 측정 패턴
function percentile(values: number[], p: number): number {
  const sorted = [...values].sort((a, b) => a - b);
  return sorted[Math.ceil(sorted.length * (p / 100)) - 1];
}

// 100회 호출 → p50, p95, p99 계산 → 임계값 검증
const times = await Promise.all(Array(100).fill(0).map(() => measureTime(fn)));
expect(percentile(times, 95)).toBeLessThan(threshold);
```

| 대상 | p95 임계값 | 비고 |
|------|----------|------|
| 핵심 비즈니스 함수 (mock 환경) | < 50ms | resolveLink, createLink 등 |
| 순수 함수 | < 1ms | validateLinkStatus, checkPasswordProtection |
| 동시 100건 호출 | 전체 < 500ms | Promise.all 시뮬레이션 |
| 캐시 히트 vs 미스 | 히트 p95 < 미스 p95 | 캐시 효과 검증 |

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
