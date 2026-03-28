# Report Formats

Two reports are generated: Preparation (after Phase C) and Execution (after Phase D).

## Report 1: Test Preparation Report

Display after Phase C (test code generation) completes.

```
═══════════════════════════════════════════════════
 Test Preparation Report
═══════════════════════════════════════════════════

Target: [analysis target summary]
Date: [YYYY-MM-DD]
Stack: [language / framework / test tool]

───────────────────────────────────────────────────
 Domain          Scenarios  Cases  Test Files  Lines
───────────────────────────────────────────────────
 Unit                   -      -           -      -
 API                    -      -           -      -
 Integration            -      -           -      -
 E2E                    -      -           -      -
 Security               -      -           -      -
 Accessibility          -      -           -      -
 Performance            -      -           -      -
 Load/Stress            -      -           -      -
 Smoke                  -      -           -      -
 Chaos                  -      -           -      -
───────────────────────────────────────────────────
 Total                  -      -           -      -
───────────────────────────────────────────────────

Documents:
 - tests/doc/scenarios/*.md (N files)
 - tests/doc/testcases/*.md (N files)

Test Code:
 - tests/unit/**/*.test.* (N files)
 - tests/api/**/*.test.* (N files)
 - tests/integration/**/*.test.* (N files)
 - tests/e2e/**/*.spec.* (N files)
 - tests/security/**/*.test.* (N files)
 - tests/a11y/**/*.test.* (N files)
 - tests/perf/**/*.test.* (N files)
 - tests/load/**/*.test.* (N files)
 - tests/smoke/**/*.test.* (N files)
 - tests/chaos/**/*.test.* (N files)

Preparation Time: Xm Xs (parallel)
═══════════════════════════════════════════════════
```

## Report 2: Test Execution Report

Display after Phase D (verification) completes.

```
═══════════════════════════════════════════════════
 Test Execution Report
═══════════════════════════════════════════════════

Target: [analysis target summary]
Date: [YYYY-MM-DD]

───────────────────────────────────────────────────
 Domain          Tests  Pass  Fail  Skip  Duration
───────────────────────────────────────────────────
 Unit                -     -     -     -        -
 API                 -     -     -     -        -
 Integration         -     -     -     -        -
 E2E                 -     -     -     -        -
 Security            -     -     -     -        -
 Accessibility       -     -     -     -        -
 Performance         -     -     -     -        -
 Load/Stress         -     -     -     -        -
 Smoke               -     -     -     -        -
 Chaos               -     -     -     -        -
───────────────────────────────────────────────────
 Total               -     -     -     -        -
───────────────────────────────────────────────────

Coverage: --% line / --% branch
Pass Rate: --% (-/-)

Failures: (if any)
 - [file:line] — [failure description]

Quality Criteria:
 [v] Coverage >= 98%
 [v] Independence — no cross-test deps
 [v] Determinism — 0 flaky tests
 [v] Boundary values — edge cases covered
 [v] Failure clarity — assertions with messages
 [v] Execution speed — within limits
 [v] Security — OWASP Top 10 covered
 [v] Accessibility — WCAG 2.1 AA pass
 [v] Performance — within thresholds

Execution Time: Xm Xs (parallel)
═══════════════════════════════════════════════════
```

## Report Rules

1. Replace `-` with actual values
2. Use `[v]` for pass, `[!]` for warning, `[x]` for fail in quality criteria
3. List all failures with file:line and description
4. Show actual coverage numbers from test runner output
5. Calculate pass rate as percentage
6. Show parallel execution time (wall clock, not sum)
