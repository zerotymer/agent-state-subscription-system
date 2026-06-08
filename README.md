# Agent State Subscription System

## 목적

여러 프로젝트에서 동작하는 다양한 종류의 에이전트의 현재 상태, 작업 지시, 진행 단계, 체크리스트, 결과, 오류를 한곳에서 받아 정제하고 추적하기 위한 시스템을 설계한다.

이 시스템에서 "구독"은 일반적인 양방향 pub/sub 제어 구조가 아니라, 에이전트가 hook 또는 skill 실행 결과를 단방향으로 전달하고 웹 계층이 이를 정제, 보관, 표시, 알림으로 변환하는 구조를 의미한다.

개별 에이전트를 직접 제어하는 것보다, 여러 에이전트의 실행 흐름을 관찰하고 소수의 외부 사용자에게 상태 변화와 완료 여부를 명확히 보여주는 것을 1차 목표로 한다.

웹 계층은 Next.js BFF를 중심으로 단일화한다. 사용자 화면, hook endpoint, MCP endpoint, 상태 조회, 알림 API를 같은 계층에서 다루고, worker나 queue는 필요 시 분리한다.

## 범위

### 포함

- 에이전트 상태 이벤트 수집
- 지시 또는 작업 단위의 진행 상태 추적
- 프로젝트별, 에이전트별, 지시별 hook 수신과 상태 정제
- 실시간 알림 또는 스트리밍 제공
- 상태 이력 저장과 조회
- 장애, 지연, 중단, 재시도 상태 표현
- 사용자용 대시보드 기반 조회
- 브라우저 알림과 페이지 내부 모달 알림

### 제외 또는 추후 검토

- 에이전트 자체의 실행 엔진 구현
- 에이전트별 내부 의사결정 로직 표준화
- 모든 프로젝트의 업무 도메인 모델 통합
- 대규모 외부 고객용 SaaS 패키징
- 에이전트에 대한 실시간 원격 제어

## 문서 맵
> docs/

- [requirements.md](requirements.md): 기능 요구사항, 비기능 요구사항, 상태 모델
- [subscription.md](subscription.md): 이 프로젝트에서 사용하는 구독 개념 정의
- [connectivity.md](connectivity.md): hook 기반과 MCP 기반 연결성 검토
- [infrastructure.md](infrastructure.md): 필요한 인프라 구성과 단계별 도입 계획
- [notifications.md](notifications.md): 페이지 내부 알림, 브라우저 푸시, 모바일 웹 알림 정책
- [framework-selection.md](framework-selection.md): MVP 기준 확정 프레임워크와 선정 이유
- [authentication.md](authentication.md): SSO, JWT cookie, Agent API Key, MCP/API 인증 방침
- [state-model.md](state-model.md): 상태 코드, 변경 경로, hook/MCP 책임 분리
- [completion-policy.md](completion-policy.md): Agent 기준 최종 완료와 테스트 완료 판정 정책
- [event-schema.md](event-schema.md): hook/MCP 이벤트 타입과 공통 payload 스키마
- [mcp-state-model.md](mcp-state-model.md): MCP desired/actual state 분리와 간접 지시 모델
- [instruction-version-policy.md](instruction-version-policy.md): 중요 변경 기준의 지침 버전 증가 정책
- [agent-lifecycle-policy.md](agent-lifecycle-policy.md): Agent 등록, 활성화, 비활성화, 폐기, 보관 정책
- [permission-model.md](permission-model.md): owner, maintainer, viewer, agent 역할과 권한
- [retention-policy.md](retention-policy.md): 데이터 보존 기간과 환경변수 기반 조정 정책
- [ui-scope.md](ui-scope.md): MVP 화면 목록과 화면별 책임
- [data-model-draft.md](data-model-draft.md): 구현 전 재검토를 전제로 한 DB 테이블 초안
- [architecture.md](architecture.md): 시스템 구성요소, 데이터 흐름, 기술 후보
- [process.md](process.md): 에이전트 등록, 상태 발행, 구독, 장애 처리 프로세스
- [roadmap.md](roadmap.md): 단계별 구현 계획
- [open-questions.md](open-questions.md): 불분명한 사항과 확인 질문

## 핵심 가정

- 에이전트는 hook, skill, 실행 로그, 산출물 등을 통해 상태를 공유한다.
- 에이전트 상태는 웹 계층에서 능동적으로 제어하기보다 단방향으로 수신한다.
- 모든 상태를 즉시 실시간으로 보장하기보다는, 핵심 상태 변화는 유실 없이 저장하고 구독자에게 전달하는 것을 우선한다.
- 지시의 진행 정도는 고정 퍼센트보다 가변 단계, 체크박스, 테스트 완료 여부, 산출물 링크로 표현하는 것이 더 중요하다.
