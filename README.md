# Full Test Orchestrator

[한국어](README.ko.md) | **English**

A Claude Code skill that analyzes implementation code and generates comprehensive test suites across 10 domains — with parallel execution for speed.

## What It Does

Given your current codebase, this skill:

1. **Analyzes** your code, stack, and existing tests
2. **Generates test documents** (scenarios + test cases) in `tests/doc/`
3. **Generates executable test code** using your project's framework
4. **Runs all tests** and collects results
5. **Reports** preparation summary and execution results

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
Phase A: Code Analysis          (sequential — shared context)
    ↓
Phase B: Test Document Gen      (3 agents in parallel)
    ↓
Phase C: Test Code Gen          (3 agents in parallel)
    ↓  → Test Preparation Report
Phase D: Verification           (3 agents in parallel)
    ↓  → Test Execution Report
Phase E: Reports Display
```

### Parallel Agent Groups

| Agent | Domains |
|-------|---------|
| Agent 1 | Unit + API + Smoke |
| Agent 2 | Integration + E2E |
| Agent 3 | Security + Accessibility + Performance + Load/Stress + Chaos |

## Quality Criteria

All generated tests are measured against 9 standards:

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

Two separate reports are generated:

### Test Preparation Report (after code generation)

Shows what was generated: scenarios, cases, test files, and lines per domain.

### Test Execution Report (after running tests)

Shows results: pass/fail counts, coverage, duration, failures, and quality criteria status per domain.

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
