# Quality Criteria (10 Standards)

All generated tests must meet these criteria.

## 0. Meaningful Tests (Gate — checked before all other criteria)

Every test must pass the **3-Question Gate** before it is written:

1. **What production bug does this prevent?** — If you can't name a concrete failure scenario, don't write it.
2. **Which layer owns this verification?** — If another layer already covers it, don't duplicate.
3. **What action does a failure trigger?** — If the response to failure is unclear, the test design is wrong.

Tests that fail this gate are worse than no tests — they add maintenance cost, slow CI, and create false confidence. Remove them.

**Prohibited patterns** (auto-reject during review):
- Constant verification: `expect(PATH).toBe('/api/...')`
- Existence check: `expect(typeof fn).toBe('function')`
- Implementation copy: duplicating source logic in assertions
- Placeholder: `expect(true).toBeTruthy()`
- Pure delegation: only checking that A calls B
- Static data: testing arrays/maps with no conditional logic
- Repetitive middleware: same pattern × N endpoints (test middleware once + 2 representatives)

## 0.5. False Positive Audit (허위 양성 탐지 — Gate 2)

통과하는 테스트 중 **실제로 아무것도 검증하지 않는 테스트**를 탐지합니다. 이는 테스트가 없는 것보다 위험합니다.

**탐지 대상 패턴:**

| 패턴 | grep 명령 | 위험도 |
|------|----------|--------|
| `expect(x !== undefined).toBe(true)` | `grep -r "!== undefined" tests/` | CRITICAL — null도 통과 |
| `expect([200, 201, 403, 429]).toContain(s)` | `grep -rn "expect(\[" tests/` | CRITICAL — 핵심 동작 미검증 |
| `.catch(() => null)` + null 허용 | `grep -rn "\.catch.*=> null" tests/` | HIGH — 에러 삼킴 |
| `console.log("[SKIP]")` | `grep -r "\[SKIP\]" tests/` | HIGH — false positive |
| `expect(true).toBe(true)` | `grep -r "expect(true)" tests/` | CRITICAL — 무조건 통과 |
| 빈 테스트 본문 | `// TODO`, `// [REQUIRED]` | HIGH — 검증 없음 |

**발견 시 조치:**
- CRITICAL → 즉시 수정 (정확한 값 검증으로 대체)
- HIGH → `test.fail(true, "설명")` 또는 `test.info().annotations.push({ type: "fixme" })` 전환
- 수량 보고: "허위 양성 N건 탐지, M건 수정"

## 1. Coverage (≥ 98% line, ≥ 90% branch)

- Every public function/method has at least one test
- All conditional branches (if/else, switch, ternary) tested
- Error handling paths covered
- Measure with: istanbul/nyc (JS), coverage.py (Python), go test -cover (Go)

## 2. Independence

- Each test runs in isolation — no shared mutable state
- Test order does not affect results
- Setup/teardown cleans up properly
- No test depends on another test's output

## 3. Determinism

- Same input → same result, every time
- No reliance on: current time, random values, network, file system state
- Use: fixed seeds, time mocking, deterministic fixtures
- Target: 0 flaky tests

## 4. Boundary Value Coverage

Every input parameter tests:
- **Null/undefined/empty**: null, undefined, "", [], {}
- **Minimum**: 0, 1, -1, empty string
- **Maximum**: MAX_INT, max length, large payload
- **Just outside bounds**: n+1, n-1 of valid range
- **Type coercion**: "123" vs 123, true vs "true"

## 5. Failure Clarity

- Every assertion includes a descriptive message
- Failed test output shows: expected value, actual value, context
- Test names describe the behavior being verified
- Pattern: `should [expected behavior] when [condition]`

```
// Good
expect(result).toBe(201, "User creation should return 201 for valid input")

// Bad
expect(result).toBe(201)
```

## 6. Execution Speed

| Domain | Target | Hard Limit |
|--------|--------|------------|
| Unit | < 100ms per test | < 1s |
| API | < 500ms per test | < 2s |
| Integration | < 2s per test | < 5s |
| E2E | < 10s per test | < 30s |
| Security | < 1s per test | < 5s |
| Accessibility | < 1s per test | < 3s |
| Performance | < 5s per test | < 15s |
| Load/Stress | varies | defined per scenario |
| Smoke | < 30s total | < 2min |
| Chaos | varies | defined per scenario |

## 7. Security Coverage (OWASP Top 10)

Must verify protection against:
1. Injection (SQL, NoSQL, OS, LDAP)
2. Broken Authentication
3. Sensitive Data Exposure
4. XML External Entities (XXE)
5. Broken Access Control
6. Security Misconfiguration
7. Cross-Site Scripting (XSS)
8. Insecure Deserialization
9. Using Components with Known Vulnerabilities
10. Insufficient Logging & Monitoring

## 8. Accessibility (WCAG 2.1 AA)

- Color contrast ≥ 4.5:1 (normal text), ≥ 3:1 (large text)
- All interactive elements keyboard accessible
- ARIA labels on non-semantic elements
- Focus visible on all interactive elements
- Alt text on all informational images
- Form inputs have associated labels
- Error messages programmatically associated

## 9. Performance Thresholds

| Metric | Good | Needs Work | Poor |
|--------|------|------------|------|
| LCP | < 2.5s | 2.5-4s | > 4s |
| FID | < 100ms | 100-300ms | > 300ms |
| CLS | < 0.1 | 0.1-0.25 | > 0.25 |
| TTFB | < 200ms | 200-600ms | > 600ms |
| API Response | < 200ms | 200-1000ms | > 1000ms |
