# Test Domains Guide

## 1. Unit

**Focus**: Single function/module in isolation
**Mock**: External dependencies (DB, API, filesystem)
**Patterns**:
- Arrange-Act-Assert
- One assertion per logical concept
- Cover: happy path, error path, boundary values, null/undefined

**Scenarios to consider**:
- Input validation (empty, null, negative, overflow)
- Return value correctness
- Error/exception throwing
- State mutations
- Edge cases (empty array, single element, max length)

## 2. API

**Focus**: HTTP endpoint request/response contract
**Verify**:
- Status codes (200, 201, 400, 401, 403, 404, 500)
- Response body schema
- Headers (Content-Type, CORS, auth)
- Query params / path params / body parsing
- Error response format

**Scenarios to consider**:
- CRUD operations
- Authentication required vs public
- Invalid payload / missing fields
- Rate limiting behavior
- Pagination

## 3. Integration

**Focus**: Module-to-module interaction
**Verify**:
- Database read/write roundtrip
- External service communication
- Message queue produce/consume
- Cache hit/miss behavior
- Transaction rollback

**Scenarios to consider**:
- Service A calls Service B with valid data
- Service B is unavailable — fallback behavior
- Data consistency across modules
- Concurrent access patterns

## 4. E2E

**Focus**: Full user journey through the application
**Tools**: Playwright, Cypress, Selenium
**Verify**:
- Critical user flows (login → action → result)
- Navigation and routing
- Form submission and validation
- Data persistence across pages

**Scenarios to consider**:
- Happy path user journey
- Error recovery (network failure, invalid input)
- Multi-step workflows
- Cross-browser behavior (if applicable)

### E2E 필수 시나리오 (프로덕션 출시 전)

#### 전체 사용자 여정 (Lifecycle Journey)
각 기능을 개별로 테스트하는 것만으로는 기능 간 인터페이스 버그를 잡을 수 없습니다. **하나의 연속 시나리오**로 검증해야 합니다.

| 여정 | 필수 단계 |
|------|----------|
| 비회원 | 홈 접속 → 입력 → 생성 → 복사 버튼 확인 → 리다이렉트 302 확인 |
| 회원 | 로그인 → 생성 → 클릭 발생 → 분석 확인(clickCount ≥ 1) → 수정 → 삭제 → 삭제 확인(404) |
| 관리자 | 로그인 → 대시보드 → 사용자 관리 → 설정 변경 |

#### 모바일 전용 테스트
`e2e-mobile` 프로젝트가 데스크톱 spec을 재실행하는 것은 모바일 검증이 아닙니다. **별도 spec 파일**이 필요합니다.

| 검증 항목 | 방법 |
|----------|------|
| 뷰포트 내 표시 | `scrollWidth <= innerWidth` (수평 스크롤 없음) |
| 터치 영역 크기 | 주요 버튼 ≥ 44x44px (WCAG 2.5.5) |
| 햄버거 메뉴 | 열기/닫기 + 내부 링크 동작 |
| 핵심 기능 | 모바일에서 링크 생성 + 결과 표시 |

```typescript
// 모바일 뷰포트 설정
import { devices } from "@playwright/test";
test.use({ ...devices["iPhone 13"] });
```

#### i18n 컨텐츠 실질 검증
언어 전환 메커니즘만 검증하면 **번역 키 누락 시 `undefined` 표시를 감지하지 못합니다.**

| 검증 항목 | 방법 |
|----------|------|
| 해당 언어 문자 존재 | ko: `/[\uAC00-\uD7AF]/`, ja: `/[\u3040-\u30FF\u4E00-\u9FFF]/` |
| 번역 키 누락 감지 | body에 `undefined`, `missing_key`, `common.`, `errors.` 부재 확인 |
| 언어 전환 효과 | `/ko` → `/en` 전환 후 main 영역에서 한글 비율 감소 확인 |

#### E2E [SKIP] 패턴 금지
```typescript
// ❌ 금지: 핵심 기능 검증을 skip하면 false positive
if (await element.isVisible().catch(() => false)) {
  // 검증
} else {
  console.log("[SKIP] 요소 미표시");
}

// ✅ 올바른 패턴 1: 명시적 대기 + 실패
await expect(element).toBeVisible({ timeout: 15000 });

// ✅ 올바른 패턴 2: 환경 의존적이면 fixme annotation
if (!(await element.isVisible().catch(() => false))) {
  test.info().annotations.push({ type: "fixme", description: "요소 미표시 — 환경 확인" });
  test.fail(true, "핵심 UI 요소 미표시");
  return;
}
```

## 5. Security

**Focus**: OWASP Top 10 vulnerability coverage
**Verify**:
- XSS (reflected, stored, DOM-based)
- SQL/NoSQL injection
- CSRF protection
- Authentication bypass
- Authorization escalation
- Sensitive data exposure
- Security headers
- Input sanitization

**Scenarios to consider**:
- Malicious input in all user-facing fields
- JWT/token manipulation
- Path traversal
- Rate limiting / brute force protection
- CORS misconfiguration

## 6. Accessibility

**Focus**: WCAG 2.1 AA compliance
**Tools**: axe-core, pa11y, Lighthouse
**Verify**:
- Color contrast ratios (≥ 4.5:1 normal, ≥ 3:1 large text)
- Keyboard navigation (Tab, Enter, Escape, Arrow keys)
- Screen reader compatibility (ARIA labels, roles, landmarks)
- Focus management
- Alt text for images

**Scenarios to consider**:
- Full keyboard-only navigation
- Screen reader announcement order
- Dynamic content updates (live regions)
- Form error announcement
- Modal/dialog focus trap

## 7. Performance

**Focus**: User experience speed metrics
**Metrics**:
- LCP (Largest Contentful Paint) < 2.5s
- FID (First Input Delay) < 100ms
- CLS (Cumulative Layout Shift) < 0.1
- TTFB (Time to First Byte) < 200ms
- API response time < 200ms

**Scenarios to consider**:
- Initial page load
- Subsequent navigation
- Large data rendering
- Image/asset loading
- Bundle size impact

## 8. Load/Stress

**Focus**: System capacity and breaking point
**Tools**: k6, JMeter, Artillery, Locust
**Metrics**:
- Max concurrent users
- Requests per second (TPS)
- Error rate under load
- Response time degradation curve
- Recovery time after overload

**Scenarios to consider**:
- Ramp-up: gradual increase to target load
- Spike: sudden traffic burst
- Soak: sustained load over time
- Stress: beyond expected capacity
- Resource exhaustion (memory, CPU, connections)

## 9. Smoke

**Focus**: Post-deploy critical path verification
**Characteristics**:
- Fast (< 2 minutes total)
- Covers only critical paths
- Binary pass/fail
- Run after every deployment

**Scenarios to consider**:
- Application starts and responds to health check
- Login flow works
- Core CRUD operation succeeds
- Database connectivity confirmed
- External service connectivity confirmed

## 10. Chaos

**Focus**: Fault injection and system resilience
**Tools**: Chaos Monkey, Litmus, Gremlin, tc (traffic control), Vitest mock injection
**Inject**:
- Network latency / packet loss
- Service crash / restart
- Database connection failure
- Disk full / memory pressure
- DNS failure

**Scenarios to consider**:
- Downstream service timeout — circuit breaker activates
- Database failover — app reconnects
- Partial network partition — graceful degradation
- Memory pressure — no OOM crash
- Recovery after fault removal

### Chaos 테스트 필수 시나리오 (프로덕션 출시 전)

아래 시나리오는 서비스 유형에 따라 적용합니다. **모든 Chaos 테스트에서 핵심 기능(리다이렉트 등)이 정상 동작하는지를 최종 검증해야 합니다.**

#### Graceful Degradation 검증 패턴
```
부가 서비스 장애 (캐시/GeoIP/분석 등)
  → 핵심 기능은 정상 동작하는가?
  → 부가 기능만 저하되는가?
```

| 장애 주입 | 기대 동작 | 검증 포인트 |
|----------|----------|-----------|
| Redis 캐시 throw | DB fallback으로 정상 조회 | `expect(result).toEqual(expectedData)` — null 허용 금지 |
| 외부 API (Cloudflare 등) null 반환 | 기능 저하되지만 서비스 계속 | DB에 저장 성공, Cloudflare 필드만 null |
| GeoIP 서비스 throw | fire-and-forget 무시, 핵심 동작 정상 | 리다이렉트 성공 |
| 배치 큐 enqueue throw | fire-and-forget 무시, 핵심 동작 정상 | 리다이렉트 성공 |
| Redis + GeoIP + 배치 큐 동시 장애 | DB만으로 핵심 동작 | 리다이렉트 성공 |
| Redis 장애 → 복구 | 자동 재연결, 수동 재시작 불필요 | 첫 요청 DB fallback, 두 번째 요청 캐시 반환 |

#### Circuit Breaker 검증 패턴
```
CLOSED → 연속 N회 실패 → OPEN → cooldown 경과 → HALF_OPEN → 성공 → CLOSED
```

**필수 TC:**
1. failureThreshold 도달 시 OPEN 전환
2. OPEN에서 fn 호출 없이 즉시 실패/fallback 반환
3. cooldown 후 HALF_OPEN → 성공 시 CLOSED 복구
4. HALF_OPEN → 실패 시 다시 OPEN

#### 메모리 누수 방지 검증
```
재시도 큐, 버퍼 등 인메모리 자료구조에 상한(cap)이 존재하는가?
```

**필수 TC:**
1. 재시도 큐가 maxRetries 초과 시 항목 drop
2. 전체 큐 크기가 maxSize 초과 시 오래된 항목 drop
3. drop된 항목 수가 통계에 반영

#### 캐시-DB 정합성 Race Condition
```
삭제 → 캐시 무효화 사이 시간 창에서 stale 데이터 반환 가능성
```

**필수 TC:**
1. stale 데이터 반환 시간 창 존재 확인 (race condition 문서화)
2. 2차 방어선(상태 검증 함수)이 stale 데이터를 차단하는지 확인
3. TTL 만료 후 DB 최신 상태 반환 확인
