# Production Readiness Audit (프로덕션 출시 관점 감사)

Phase A에서 기존 테스트가 있을 때, Gap Investigation 전에 이 파일을 Read하여 체크리스트를 수행한다.

테스트 수치(TC 수, 파일 수)가 아니라 **"이 테스트를 모두 통과하면 프로덕션에서 장애 없이 운영될 수 있는가?"**를 기준으로 평가합니다.

## 필수 검증 체크리스트

Phase A에서 기존 테스트가 있을 때, Gap Investigation 전에 이 체크리스트를 먼저 수행합니다.

### 1. 장애 복원력 (Resilience)

| 검증 항목 | 질문 | 테스트 위치 |
|----------|------|-----------|
| 캐시 장애 폴백 | Redis 다운 시 DB 직접 조회로 핵심 기능이 동작하는가? | chaos |
| 외부 서비스 장애 | API 제공자 timeout/500 시 graceful degradation 하는가? | chaos |
| 메모리 누수 방지 | 재시도 큐, 버퍼 등에 크기 상한(cap)이 있는가? | unit |
| 동시성 안전 | 같은 리소스 동시 생성 시 unique constraint 처리가 되는가? | unit, integration |
| 트랜잭션 부분 실패 | 다단계 트랜잭션에서 중간 실패 시 데이터 일관성이 보장되는가? | integration |
| 서버 재시작 시 데이터 유실 | 인메모리 버퍼 flush 전 종료 시 허용 정책이 있는가? | 문서화 |

### 2. 보안 실질성 (Security Substance)

| 검증 항목 | 질문 | 테스트 위치 |
|----------|------|-----------|
| Rate Limit 임계값 | 실제로 limit을 초과하면 429가 반환되는가? (느슨한 검증 금지) | unit |
| JWT 실제 서명 | mock이 아닌 실제 서명/검증이 동작하는가? | integration |
| OAuth 콜백 처리 | 외부 제공자 mock 없이 빈 테스트로 남겨져 있지 않은가? | e2e |
| 구독 플랜 강제 | API 레벨에서 플랜 제한이 실제 적용되는가? (UI 안내만 검증은 불충분) | unit, api |

### 3. 사용자 여정 완결성 (User Journey Completeness)

| 검증 항목 | 질문 | 테스트 위치 |
|----------|------|-----------|
| 비회원 전체 여정 | 홈→입력→생성→복사→리다이렉트가 하나의 연속 시나리오로 검증되는가? | e2e |
| 회원 전체 여정 | 로그인→생성→분석확인→수정→삭제가 연속으로 검증되는가? | e2e |
| 모바일 전용 | 모바일 뷰포트에서 핵심 기능이 별도 spec으로 검증되는가? | e2e |
| i18n 실질성 | 언어 전환 후 실제 해당 언어 텍스트가 표시되는가? (번역 키 누락 감지) | e2e |
| 에러 상태 UX | 404, 만료, 서버 에러 시 사용자에게 적절한 안내가 표시되는가? | e2e |

### 4. 정합성 (Consistency)

| 검증 항목 | 질문 | 테스트 위치 |
|----------|------|-----------|
| 캐시-DB 정합성 | 삭제 후 캐시 무효화 전 stale 데이터로 리다이렉트되지 않는가? | unit |
| 외부 서비스-DB 정합성 | 외부 API 실패 시 DB에 고아 레코드가 남지 않는가? | unit |
| 에러 응답 일관성 | 모든 API 에러 응답이 동일한 형식인가? (내부 정보 미노출) | api |

## 운영 안정성 점검 (코드 외)

테스트 통과만으로는 프로덕션 안정성을 보장할 수 없습니다. 다음 항목도 확인합니다.

| 항목 | 확인 내용 | 산출물 |
|------|----------|--------|
| 헬스체크 API | DB/Redis/어플리케이션 메트릭이 단일 엔드포인트에서 확인 가능한가? | `/api/health` 강화 |
| Circuit Breaker | 외부 서비스 장애 시 자동 차단 + 복구 로직이 있는가? | `circuit-breaker.ts` |
| 롤백 플레이북 | 배포 후 장애 시 5분 이내 복구 절차가 문서화되어 있는가? | `ROLLBACK_PLAYBOOK.md` |
| DB 마이그레이션 롤백 | 모든 마이그레이션에 rollback.sql이 존재하는가? | `prisma/migrations/*/rollback.sql` |
| Graceful Degradation 정책 | 부가 서비스 장애 시 핵심 기능 계속 동작하는 정책이 명시되어 있는가? | `GRACEFUL_DEGRADATION_POLICY.md` |

## 성능 벤치마크 (Performance Baseline)

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
