# Full Test Orchestrator

**한국어** | [English](README.md)

구현 코드를 분석하여 10개 테스트 도메인에 대한 종합 테스트 스위트를 생성하는 Claude Code 스킬입니다. 병렬 실행으로 속도를 극대화합니다.

## 주요 기능

현재 코드베이스를 기반으로:

1. 코드, 스택, 기존 테스트를 **분석**
2. `tests/doc/`에 **테스트 문서** (시나리오 + 테스트 케이스) 생성
3. 프로젝트 프레임워크에 맞는 **실행 가능한 테스트 코드** 생성
4. 모든 테스트를 **실행**하고 결과 수집
5. 준비 요약 및 실행 결과 **리포트** 출력

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
Phase A: 코드 분석               (순차 — 공유 컨텍스트)
    ↓
Phase B: 테스트 문서 생성          (3개 에이전트 병렬)
    ↓
Phase C: 테스트 코드 생성          (3개 에이전트 병렬)
    ↓  → 테스트 준비 리포트
Phase D: 검증                    (3개 에이전트 병렬)
    ↓  → 테스트 실행 리포트
Phase E: 리포트 표시
```

### 병렬 에이전트 그룹

| 에이전트 | 담당 도메인 |
|----------|-------------|
| Agent 1 | Unit + API + Smoke |
| Agent 2 | Integration + E2E |
| Agent 3 | Security + Accessibility + Performance + Load/Stress + Chaos |

## 품질 기준

생성된 모든 테스트는 9가지 기준으로 측정됩니다:

| 기준 | 목표 |
|------|------|
| 커버리지 | ≥ 98% line, ≥ 90% branch |
| 독립성 | 테스트 간 의존성 없음 |
| 결정성 | flaky 테스트 0건 |
| 경계값 | null, empty, max, negative 포함 |
| 실패 명확성 | 기대값/실제값 포함 assertion 메시지 |
| 실행 속도 | unit < 1s, integration < 5s, e2e < 30s |
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

두 개의 리포트가 분리되어 생성됩니다:

### 테스트 준비 리포트 (코드 생성 후)

생성된 내용을 표시합니다: 도메인별 시나리오, 케이스, 테스트 파일, 라인 수.

### 테스트 실행 리포트 (테스트 실행 후)

실행 결과를 표시합니다: 도메인별 pass/fail 수, 커버리지, 소요 시간, 실패 항목, 품질 기준 상태.

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
