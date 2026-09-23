# Gap Iteration Mode (Existing Tests)

When existing tests are detected in Phase A, switch to Gap Iteration Mode instead of generating from scratch. 이 모드 진입 시 이 파일을 Read한다.

## Gap Iteration Workflow

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

## Step 0: False Positive Audit

**기존 테스트가 존재할 때 가장 먼저 실행합니다.** Gap을 찾기 전에, 통과하는 테스트가 실제로 검증을 하고 있는지 확인합니다.

Phase A-4의 grep 명령을 실행하여 허위 양성을 수집하고, Step 3에서 gap 코드와 함께 수정합니다. (grep 명령과 판별 기준: `references/false-positive-audit.md`)

허위 양성을 먼저 수정하지 않으면:
- Gap 분석 시 "이미 테스트됨"으로 잘못 판별 → 실제 gap을 놓침
- 허위 양성 TC가 커버리지에 포함 → 실제 커버리지보다 높게 측정

## Step 1: Gap Investigation

For each of the 10 domains, analyze:
- **Source coverage**: Which functions/endpoints/flows lack test coverage?
- **Domain coverage**: Which domains have zero tests? (e.g., no security tests at all)
- **Quality gaps**: Existing tests missing boundary values, error paths, or edge cases?
- **Dedup check**: Ensure investigation respects layer ownership rules

Output: Gap analysis summary per domain showing what is covered vs missing.

## Step 2: Gap Plan Documentation

Write gap-specific documents to `tests/doc/`:
- `tests/doc/scenarios/{domain}-gap-scenarios.md` — scenarios for missing coverage
- `tests/doc/testcases/{domain}-gap-testcases.md` — test cases for each gap scenario

Each document clearly references:
- What existing tests already cover (to avoid duplication)
- What new tests will be added and why

## Step 3: Gap Test Code Generation

Generate test code ONLY for identified gaps:
- Add to existing test files where the same endpoint/module is partially tested
- Create new files only for entirely untested areas
- Follow all Dedup and Layer Ownership rules

## Step 4: Run All Tests

Execute both existing and newly generated tests. Collect results.

## Step 5-6: Triage and Remediate

**Apply Phase E (Triage) and Phase F (Remediation) rules to ALL failures.**

This is the critical difference from the old workflow:
- Old: failures → adjust tests → re-run
- New: failures → classify → fix the RIGHT code → re-run

## Step 7: Re-investigate and Repeat

After execution, re-analyze for remaining gaps:
- New tests may reveal additional untested paths
- Coverage report may show branches still uncovered
- Continue iteration until:
  - Coverage target (98%+ line, 90%+ branch) is met, OR
  - No actionable gaps remain across all 10 domains

**Maximum 5 iterations** to prevent infinite loops. Report progress per iteration.

## Gap Iteration Report

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
