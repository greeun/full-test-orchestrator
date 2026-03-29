# Report Formats

Three reports are generated: Preparation (after Phase C), Execution (after Phase G), and Triage Summary (after Phase H).

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

Spec Sources Used:
 - [list of docs, schemas, types used for expected values]

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

Display after Phase G (final re-verification) completes.

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

Remaining Failures: (if any)
 - [file:line] — [failure description] — [classification]

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
Triage Iterations: N / 5 max
═══════════════════════════════════════════════════
```

## Report 3: Triage Summary Report

Display after all triage-fix-verify iterations complete.

```
═══════════════════════════════════════════════════
 Triage Summary Report
═══════════════════════════════════════════════════
 Iterations Completed: N / 5 max
 Total Failures Triaged: N
 Final Pass Rate: --% (-/-)

 ─── By Classification ───────────────────────────
 IMPL_BUG:        N fixed  (implementation code changed)
 TEST_ERROR:      N fixed  (test code changed, with justification)
 SPEC_AMBIGUOUS:  N resolved (user clarified)
 ENV_ISSUE:       N reported (environment setup needed)
 FLAKY:           N fixed  (determinism improved)
 NOT_IMPLEMENTED: N reported (features missing)

 ─── Implementation Fixes ────────────────────────
 File                        Line  Change Summary
 src/lib/services/foo.ts      42   Fixed return type for null input
 src/app/api/bar/route.ts    105   Added missing 404 handling
 ...

 ─── Test Fixes (with justification) ─────────────
 File                        Line  Justification
 tests/unit/foo.test.ts       28   Spec says 201 not 200 (see API doc)
 tests/api/bar.test.ts        55   Mock used wrong schema version
 ...

 ─── Unresolved ──────────────────────────────────
 [List any failures remaining after max iterations]
 - [file:line] — [classification] — [why unresolved]

 ─── Regression Events ───────────────────────────
 [List any fixes that caused regressions and were reverted]
 - Iteration N: [fix description] → reverted (caused [regression])

 ─── Findings for Development Team ───────────────
 [Patterns or systemic issues discovered during triage]
 - e.g., "3 API endpoints missing 404 handling for invalid IDs"
 - e.g., "Zod schema for UserInput allows empty strings but DB rejects them"
═══════════════════════════════════════════════════
```

## Report Rules

1. Replace `-` with actual values
2. Use `[v]` for pass, `[!]` for warning, `[x]` for fail in quality criteria
3. List all remaining failures with file:line, description, and classification
4. Show actual coverage numbers from test runner output
5. Calculate pass rate as percentage
6. Show parallel execution time (wall clock, not sum)
7. **Triage Summary must distinguish implementation fixes from test fixes**
8. **Every test fix must have a documented justification**
9. **Findings section highlights systemic issues, not just individual fixes**
