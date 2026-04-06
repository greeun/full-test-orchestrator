# Full Test Orchestrator

**한국어** | [English](README.md)

**20년차 QA 엔지니어** 수준으로 동작하는 Claude Code 스킬입니다 — 코드 분석, 기존 테스트 허위 양성 탐지, 10개 도메인 종합 테스트 생성, 실패 분류(구현 버그 vs 테스트 오류), 구현 코드 수정, 프로덕션 출시 준비도 검증까지 수행합니다.

## 주요 기능

현재 코드베이스를 기반으로:

1. 코드, 스택, 기존 테스트를 **분석**
2. 기존 테스트 중 **허위 양성 탐지** — 통과하지만 아무것도 검증하지 않는 테스트 식별
3. `tests/doc/`에 **테스트 문서** (시나리오 + 테스트 케이스) 생성
4. 프로젝트 프레임워크에 맞는 **실행 가능한 테스트 코드** 생성
5. 모든 테스트를 **실행**하고 실패를 **분류** (구현 버그 vs 테스트 오류 vs 환경 문제)
6. **올바른 코드를 수정** — 구현 버그는 구현 코드를, 테스트 오류는 테스트를 수정
7. **프로덕션 출시 준비도 검증** — 장애 복원력, 보안 실질성, 사용자 여정 완결성
8. 준비/실행/분류/허위 양성 감사 **리포트** 출력

모든 과정에서 **3-에이전트 병렬 실행**으로 대기 시간을 최소화합니다.

## 10개 테스트 도메인

| # | 도메인 | 검증 대상 |
|---|--------|-----------|
| 1 | **Unit** | 함수/모듈 단위 격리 검증 |
| 2 | **API** | 엔드포인트 요청/응답 계약 |
| 3 | **Integration** | 모듈 간 연동 |
| 4 | **E2E** | 전체 사용자 흐름 |
| 5 | **Security** | OWASP Top 10 취약점 |
| 6 | **Accessibility** | WCAG 2.1 AA 준수 |
| 7 | **Performance** | 사용자 경험 속도 (LCP, FID, TTFB) |
| 8 | **Load/Stress** | 시스템 용량 및 한계점 |
| 9 | **Smoke** | 배포 후 핵심 경로 빠른 검증 |
| 10 | **Chaos** | 장애 주입 및 복원력 |

## 워크플로우

```
Phase A:  코드 & 스펙 분석           (순차 — 공유 컨텍스트)
    ↓
Phase A+: 허위 양성 감사             (통과하지만 검증 안 하는 테스트 탐지)
    ↓
Phase B:  테스트 문서 생성            (3개 에이전트 병렬)
    ↓
Phase C:  테스트 코드 생성            (3개 에이전트 병렬)
    ↓
Phase D:  실행                      (3개 에이전트 병렬)
    ↓
Phase E:  실패 분류                  (IMPL_BUG / TEST_ERROR / ENV_ISSUE / FLAKY)
    ↓
Phase F:  수정                      (구현 또는 테스트, 근거 필수)
    ↓
Phase G:  재검증                    (전체 실행, 최대 5회 반복)
    ↓
Phase H:  리포트 + 프로덕션 준비도 감사
```

### 병렬 에이전트 그룹

| 에이전트 | 담당 도메인 |
|----------|-------------|
| Agent 1 | Unit + API + Smoke |
| Agent 2 | Integration + E2E |
| Agent 3 | Security + Accessibility + Performance + Load/Stress + Chaos |

## 핵심 기능 (v1.1)

### 허위 양성 탐지

통과하지만 아무것도 검증하지 않는 테스트를 탐지합니다 — **테스트가 없는 것보다 위험**합니다 (거짓 안전감).

| 패턴 | 예시 | 위험 |
|------|------|------|
| Null 통과 | `expect(result !== undefined).toBe(true)` | null도 통과 |
| 느슨한 상태 코드 | `expect([200, 201, 403, 429]).toContain(s)` | Rate Limit 비활성화돼도 통과 |
| 에러 삼킴 | `.catch(() => null)` 후 null 허용 | 에러 시 검증 건너뜀 |
| Graceful skip | `isVisible().catch(() => false)` → `console.log("[SKIP]")` | UI 회귀 미감지 |
| 빈 테스트 | `test("TC-001", async () => { /* TODO */ })` | 커버리지 착각 |

### 실패 분류 (Human-Level Triage)

모든 실패를 수정 전에 분류합니다:

| 분류 | 조치 |
|------|------|
| **IMPL_BUG** | 구현 코드 수정 (테스트 약화 금지) |
| **TEST_ERROR** | 테스트 수정 + 근거 문서화 |
| **ENV_ISSUE** | 환경 설정 필요 보��� |
| **FLAKY** | 결정성 수정 (mock clock, 고정 seed) |
| **SPEC_AMBIGUOUS** | 사용자에게 확인 요청 |

### 프로덕션 출시 준비도 감사

테스트 수치가 아니라 **실제 프로덕션 장애를 방지할 수 있는가**를 검증합니다:

- **장애 복원력**: 캐시 장애 폴백, 외부 서비스 Circuit Breaker, 메모리 누수 방지
- **보안 실질성**: Rate Limit 임계값 필수 검증, JWT 실제 서명, OAuth 콜백 검증
- **사용자 여정 완결성**: 전체 lifecycle E2E, 모바일 전용, i18n 실제 텍스트 검증
- **정합성**: 캐시-DB 일관성, 외부 서비스-DB 고아 레코드, 에러 응답 형식

### 운영 안정성 점검

코드 외 운영 체계도 확인하고 생성합니다:

- 헬스체크 API + 어플리케이션 메트릭
- 외부 서비스 Circuit Breaker
- 배포 롤백 플레이북 + DB 마이그레이션 rollback SQL
- Graceful Degradation 정책 + chaos 테스트

## 품질 기준

생성된 모든 테스트는 10가지 기준으��� 측정됩니다:

| 기준 | 목표 |
|------|------|
| **허위 양성 감사** | **기존 테스트 허위 양성 0건** |
| 커버리지 | ≥ 98% line, ≥ 90% branch |
| 독립성 | 테스트 간 의존성 없음 |
| 결정성 | flaky 테스트 0건 |
| 경계��� | null, empty, max, negative 포함 |
| 실패 명확성 | 기대값/실제값 포함 assertion 메시지 |
| ���행 속��� | unit < 1s, integration < 5s, e2e < 30s |
| 보안 | OWASP Top 10 커버 |
| 접근성 | WCAG 2.1 AA 통과 |
| 성능 | LCP < 2.5s, FID < 100ms |

## 산출물 구조

### 테스트 문서

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
    └── ... (동일 패턴)
```

### 테스트 코드

프로젝트 규칙에 따라 배치됩니다. 기본 구조:

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

## 리포트

4개의 리포트가 생성됩니다:

### 1. 허위 양성 감사 리포트 (Phase A+ 후)
기존 테스트의 허위 양성 목록: 파일, 줄 번호, 패턴, 위험도, 권장 수정.

### 2. 테스트 준비 리포트 (코드 생성 후)
생성된 내용을 표시합니다: 도메인별 시나리오, 케이스, 테스트 파일, 라인 수.

### 3. 테스트 실행 리포트 (테스트 실행 후)
실행 결과를 표시합니다: 도메인별 pass/fail 수, 커버리지, 소요 시간, 실패 항목, 품질 기준 상태.

### 4. 분류 요약 리포트 (수정 후)
수정 내역: IMPL_BUG 수(구현 수정), TEST_ERROR 수(근거 포함), 미해결 항목, 회귀 이벤트.

## 사용법

### 트리거 키워드

다음 키워드로 모든 Claude Code 세션에서 활성화됩니다:

**한국어**: `테스트 작성`, `테스트 생성`, `테스트 만들어`, `테스트 코드`, `커버리지`, `검증`, `모든 테스트`, `테스트 실행`, `테스트 시나리오`, `테스트 케이스`, `QA 검증`, `품질 검증`

**English**: `write tests`, `generate tests`, `create test suite`, `test coverage`, `run all tests`, `coverage report`, `QA`, `test generation`, `full test`, `test code`

### 선택적 실행

특정 도메인만 요청할 수 있습니다:

> "unit이랑 security 테스트만 만들어줘"
>
> "Only generate unit and security tests"

요청한 도메인만 실행됩니다.

### 프레임워크 자동 감지

프로젝트의 기존 테스트 프레임워크(Jest, Vitest, Pytest, Playwright 등)를 감지하여 사용합니다. 없는 경우 적합한 것을 추천하고 확인 후 진행합니다.

## 설치

Claude Code 스킬 경로에 복사합니다:

```bash
cp -r full-test-orchestrator ~/.claude/skills/
```

## 스킬 구조

```
full-test-orchestrator/
├── SKILL.md                    # 메인 워크플로우 지침
├── README.md                   # 영문 문서
├── README.ko.md                # 한국어 문서
├── references/
│   ├── test-domains.md         # 10개 도메인 상세 가이드
│   ├── quality-criteria.md     # 9가지 품질 기준
│   ├── doc-templates.md        # 문서 구조 규칙
│   ├── parallel-strategy.md    # 에이전트 병렬 실행 규칙
│   └── report-format.md        # 리포트 템플릿
└── assets/
    └── templates/
        ├── scenario-template.md
        └── testcase-template.md
```

## 라이선스

MIT
