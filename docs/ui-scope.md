# MVP 화면 범위

## 목적

MVP에서 필요한 화면 목록과 각 화면의 책임을 정의한다.

상세 UI 디자인은 별도 디자이너 또는 후속 작업에서 진행한다. 이 문서는 정보구조와 기능 범위만 정의한다.

## 화면 목록

| 화면 | 목적 |
|---|---|
| 로그인 화면 | SSO 로그인 진입 |
| 프로젝트 목록 | 사용자가 접근 가능한 프로젝트 목록 표시 |
| 프로젝트 상세 대시보드 | 프로젝트의 Agent, 지침, 상태 요약 표시 |
| Agent 목록/상세 | Agent 상태, heartbeat, 사용량, API Key 연결 상태 표시 |
| 지침 목록/상세 타임라인 | 지침 상태, 버전, 단계, 체크리스트, 이벤트 타임라인 표시 |
| 알림 설정 | 페이지 내부 알림, 브라우저 푸시, 기기별 구독 관리 |
| API Key 관리 | Agent API Key 발급, 폐기, scope 확인 |
| 간단한 감사 로그 | 주요 변경과 보안 이벤트 확인 |

## 화면별 주요 기능

### 로그인 화면

- SSO 로그인
- 허용되지 않은 사용자 안내
- 세션 만료 안내

### 프로젝트 목록

- 프로젝트 이름
- 최근 상태
- active Agent 수
- 진행 중 지침 수
- 실패 또는 차단 지침 수

### 프로젝트 상세 대시보드

- Agent 상태 요약
- 지침 상태 요약
- 최근 주요 이벤트
- `completed_and_tested` 항목
- `failed`, `blocked`, `waiting_input` 항목
- `stale`, `offline` Agent 표시

### Agent 목록/상세

- Agent lifecycle 상태
- 마지막 heartbeat
- 마지막 snapshot
- 현재 지침
- API Key 사용량
- hook 호출 수
- MCP read/write 호출 수
- 최근 오류

### 지침 목록/상세 타임라인

- 지침 제목
- instruction_version
- desired state
- actual state
- ack 상태
- 현재 단계
- 체크리스트
- 테스트 결과
- 산출물
- 이벤트 타임라인

### 알림 설정

- 브라우저 알림 권한 상태
- PWA 설치 안내
- 기기별 subscription 목록
- 알림 대상 이벤트 설정
- 내부 알림 표시 여부

### API Key 관리

- Agent별 API Key 발급
- API Key 폐기
- scope 확인
- 마지막 사용 시각
- 사용량 요약

### 감사 로그

- 사용자 로그인
- Agent 등록/비활성화/폐기
- API Key 발급/폐기
- 지침 버전 변경
- 권한 변경
- MCP acknowledge

## MVP 제외

- 상세 디자인 시스템
- 복잡한 차트
- 드래그 앤 드롭 워크플로우
- 모바일 전용 native UX
- 조직 단위 관리 화면
- 결제 또는 플랜 관리

