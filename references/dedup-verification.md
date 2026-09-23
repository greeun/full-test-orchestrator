# Dedup Claim Verification Protocol (중복 감사 3-Gate)

Phase A-3.5의 상세 절차. 중복을 **주장**할 때 이 파일을 Read하여 3-Gate Verification을 적용한다.

## A-3.5: Dedup Claim Verification Protocol (MANDATORY — 중복 감사 시)

> **중요**: 파일명 유사성, describe 블록 이름, 또는 주석만으로 중복을 판정하면 **60% 이상 false positive**가 발생한다. 실제 사례에서 검증됨.

중복을 **주장**하려면 아래 **3-Gate Verification**을 **반드시** 통과해야 한다. 3개 게이트 중 **1개라도 실패하면 중복이 아니다.**

### Gate 1: Same Target Verification (동일 대상)
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

### Gate 2: Same Intent Verification (동일 의미)
두 파일의 검증 **의도**가 같은가? (정상 비즈니스 vs 보안 공격 vs 성능은 모두 다른 의도)

**필수 증거**: 두 파일의 실제 test 본문을 읽고 인용한다. 이름만으로 판단 금지.

- 실패 예시: 둘 다 `/api/auth/login` 401을 검증하지만, A는 "잘못된 비밀번호"(비즈니스), B는 "SQL Injection 페이로드"(보안 공격) → **Gate 2 실패, 중복 아님**

### Gate 3: Same Layer Verification (동일 계층)
Layer Ownership 기준으로 두 파일이 같은 owner인가?

**필수 증거**: 각 파일이 HTTP 요청을 사용하는지, 서비스 함수를 직접 호출하는지, DB를 mock하는지 real로 쓰는지 확인.

- 실패 예시: A는 `request.post("/api/links")` (HTTP), B는 `createLink(...)` 직접 호출 + real DB (서비스 계층) → 다른 계층 → **Gate 3 실패, 중복 아님**

### 3-Gate 보고 형식 (각 중복 주장마다 반드시 포함)

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

### 금지된 판정 근거

다음 근거로 중복을 주장하면 **자동 기각**:

- ❌ "파일명이 비슷하다"
- ❌ "같은 디렉토리에 있다"
- ❌ "둘 다 401/403을 검증한다" (의도 미검증)
- ❌ "describe 이름이 같다" (본문 미확인)
- ❌ "4개 파일에 분산되어 있다" (각 파일별 grep 증거 미첨부)
- ❌ "Layer Ownership 위반으로 보인다" (실제 코드 미확인)

### Sample-Check Recommendation

중복 보고서가 10건 이상일 때, **HIGH 항목의 최소 30%를 사용자가 샘플 검증** 후에만 권장 조치를 수행한다. 자동 적용 금지.
