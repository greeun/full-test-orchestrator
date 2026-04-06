# Full Test Orchestrator

[한국어](README.ko.md) | **English**

A Claude Code skill that acts as a **20-year veteran QA engineer** — analyzes code, detects false positives in existing tests, generates comprehensive test suites across 10 domains, triages failures, fixes the right code, and verifies production readiness.

## What It Does

Given your current codebase, this skill:

1. **Analyzes** your code, stack, and existing tests
2. **Audits existing tests for false positives** — catches tests that pass but verify nothing
3. **Generates test documents** (scenarios + test cases) in `tests/doc/`
4. **Generates executable test code** using your project's framework
5. **Runs all tests** and triages failures (implementation bug vs test error vs environment issue)
6. **Fixes the right code** — implementation bugs get implementation fixes, not test weakening
7. **Verifies production readiness** — resilience, security substance, user journey completeness
8. **Reports** preparation, execution, triage summary, and false positive audit

All with **3-agent parallel execution** to minimize wait time.

## 10 Test Domains

| # | Domain | What It Verifies |
|---|--------|------------------|
| 1 | **Unit** | Function/module isolation |
| 2 | **API** | Endpoint request/response contracts |
| 3 | **Integration** | Module-to-module interaction |
| 4 | **E2E** | Full user journey flows |
| 5 | **Security** | OWASP Top 10 vulnerabilities |
| 6 | **Accessibility** | WCAG 2.1 AA compliance |
| 7 | **Performance** | User experience speed (LCP, FID, TTFB) |
| 8 | **Load/Stress** | System capacity and breaking points |
| 9 | **Smoke** | Post-deploy critical path verification |
| 10 | **Chaos** | Fault injection and resilience |

## Workflow

```
Phase A:  Code & Spec Analysis        (sequential — shared context)
    ↓
Phase A+: False Positive Audit        (detect tests that pass but verify nothing)
    ↓
Phase B:  Test Document Gen           (3 agents in parallel)
    ↓
Phase C:  Test Code Gen               (3 agents in parallel)
    ↓
Phase D:  Execution                   (3 agents in parallel)
    ↓
Phase E:  Failure Triage              (IMPL_BUG / TEST_ERROR / ENV_ISSUE / FLAKY)
    ↓
Phase F:  Remediation                 (fix implementation or test, with justification)
    ↓
Phase G:  Re-verification             (run all tests, max 5 iterations)
    ↓
Phase H:  Reports + Production Readiness Audit
```

### Parallel Agent Groups

| Agent | Domains |
|-------|---------|
| Agent 1 | Unit + API + Smoke |
| Agent 2 | Integration + E2E |
| Agent 3 | Security + Accessibility + Performance + Load/Stress + Chaos |

## Key Features (v1.1)

### False Positive Detection

Detects tests that pass but verify nothing — **worse than no tests** because they create false confidence.

| Pattern | Example | Risk |
|---------|---------|------|
| Null passthrough | `expect(result !== undefined).toBe(true)` | null also passes |
| Loose status check | `expect([200, 201, 403, 429]).toContain(s)` | rate limit disabled still passes |
| Error swallowing | `.catch(() => null)` then allow null | skips verification on error |
| Graceful skip | `isVisible().catch(() => false)` → `console.log("[SKIP]")` | UI regression undetected |
| Empty test body | `test("TC-001", async () => { /* TODO */ })` | illusion of coverage |

### Human-Level Failure Triage

Every failing test is classified before any fix:

| Classification | Action |
|---|---|
| **IMPL_BUG** | Fix implementation code (NOT the test) |
| **TEST_ERROR** | Fix test with documented justification |
| **ENV_ISSUE** | Report environment setup needed |
| **FLAKY** | Fix determinism (mocked clocks, fixed seeds) |
| **SPEC_AMBIGUOUS** | Ask user to clarify |

### Production Readiness Audit

Goes beyond test counts — verifies your service can actually survive production:

- **Resilience**: Cache failure fallback, external service circuit breaker, memory leak prevention
- **Security substance**: Rate limit thresholds enforced, JWT actually verified, OAuth callbacks tested
- **User journey completeness**: Full lifecycle E2E, mobile-specific, i18n content verification
- **Consistency**: Cache-DB coherence, external service-DB orphan records, error response format

### Operational Readiness

Checks and generates operational infrastructure:

- Health check API with application metrics
- Circuit Breaker for external services
- Rollback playbook + DB migration rollback SQL
- Graceful Degradation policy + chaos tests

## Quality Criteria

All generated tests are measured against 10 standards:

| Criteria | Target |
|----------|--------|
| **False Positive Audit** | **0 false positives in existing tests** |
| Coverage | ≥ 98% line, ≥ 90% branch |
| Independence | No cross-test dependencies |
| Determinism | 0 flaky tests |
| Boundary values | null, empty, max, negative covered |
| Failure clarity | Assertions with expected/actual messages |
| Execution speed | unit < 1s, integration < 5s, e2e < 30s |
| Security | OWASP Top 10 covered |
| Accessibility | WCAG 2.1 AA pass |
| Performance | LCP < 2.5s, FID < 100ms |

## Output Structure

### Test Documents

```
tests/doc/
├── scenarios/
│   ├── unit-scenarios.md
│   ├── api-scenarios.md
│   ├── integration-scenarios.md
│   ├── e2e-scenarios.md
│   ├── security-scenarios.md
│   ├── accessibility-scenarios.md
│   ├── performance-scenarios.md
│   ├── load-stress-scenarios.md
│   ├── smoke-scenarios.md
│   └── chaos-scenarios.md
└── testcases/
    ├── unit-testcases.md
    ├── api-testcases.md
    └── ... (same pattern)
```

### Test Code

Placed according to your project's conventions. Default structure:

```
tests/
├── unit/
├── api/
├── integration/
├── e2e/
├── security/
├── a11y/
├── perf/
├── load/
├── smoke/
└── chaos/
```

## Reports

Four reports are generated:

### 1. False Positive Audit Report (after Phase A+)
Shows existing tests with false positives: file, line, pattern, risk level, recommended fix.

### 2. Test Preparation Report (after code generation)
Shows what was generated: scenarios, cases, test files, and lines per domain.

### 3. Test Execution Report (after running tests)
Shows results: pass/fail counts, coverage, duration, failures, and quality criteria status per domain.

### 4. Triage Summary Report (after remediation)
Shows what was fixed: IMPL_BUG count, TEST_ERROR count (with justification), unresolved items, regression events.

## Usage

### Trigger Keywords

The skill activates on these keywords in any Claude Code session:

**English**: `write tests`, `generate tests`, `create test suite`, `test coverage`, `run all tests`, `coverage report`, `QA`, `test generation`, `full test`, `test code`

**Korean**: `테스트 작성`, `테스트 생성`, `테스트 만들어`, `테스트 코드`, `커버리지`, `검증`, `모든 테스트`, `테스트 실행`, `테스트 시나리오`, `테스트 케이스`, `QA 검증`, `품질 검증`

### Selective Execution

Request specific domains only:

> "Only generate unit and security tests"
>
> "unit이랑 security 테스트만 만들어줘"

The skill will execute only the requested domains.

### Framework Auto-Detection

The skill detects your project's existing test framework (Jest, Vitest, Pytest, Playwright, etc.) and uses it. If none exists, it recommends one and confirms before proceeding.

## Installation

Copy this directory to your Claude Code skills path:

```bash
cp -r full-test-orchestrator ~/.claude/skills/
```

## Skill Structure

```
full-test-orchestrator/
├── SKILL.md                    # Main workflow instructions
├── README.md                   # English documentation
├── README.ko.md                # Korean documentation
├── references/
│   ├── test-domains.md         # 10 domain detailed guides
│   ├── quality-criteria.md     # 9 quality standards
│   ├── doc-templates.md        # Document structure rules
│   ├── parallel-strategy.md    # Agent parallel execution rules
│   └── report-format.md        # Report templates
└── assets/
    └── templates/
        ├── scenario-template.md
        └── testcase-template.md
```

## License

MIT
