# False Positive Audit (허위 양성 탐지)

SKILL.md의 "False Positive Detection" 규칙과 Phase A-4 "False Positive Audit"의 상세 절차를 담는다.
기존 테스트가 있을 때 Phase A-4에서, 그리고 Gap Iteration Mode Step 0에서 이 파일을 Read한다.

## False Positive Detection (MANDATORY — 허위 양성 탐지)

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

## A-4: False Positive Audit (MANDATORY — 기존 테스트가 있을 때)

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
