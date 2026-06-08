# 권한 모델

## 목적

소수 외부 사용자와 Agent를 대상으로 하는 최소 권한 모델을 정의한다.

초기에는 단순한 역할 기반 접근 제어를 사용하고, 추후 필요 시 역할을 추가한다.

## 역할

| 역할 | 의미 |
|---|---|
| `owner` | 프로젝트와 모든 설정을 관리하는 소유자 |
| `maintainer` | 지침과 Agent 운영을 담당하는 관리자 |
| `viewer` | 상태 조회와 알림 구독만 가능한 사용자 |
| `agent` | hook/MCP 보고만 가능한 Agent 주체 |

## 권한 표

| 기능 | owner | maintainer | viewer | agent |
|---|---:|---:|---:|---:|
| 프로젝트 조회 | 예 | 예 | 예 | 제한 |
| 프로젝트 설정 변경 | 예 | 아니오 | 아니오 | 아니오 |
| 사용자 초대/제거 | 예 | 아니오 | 아니오 | 아니오 |
| Agent 등록 | 예 | 예 | 아니오 | 아니오 |
| Agent 비활성화 | 예 | 예 | 아니오 | 아니오 |
| API Key 발급 | 예 | 예 | 아니오 | 아니오 |
| API Key 폐기 | 예 | 예 | 아니오 | 아니오 |
| 지침 생성 | 예 | 예 | 아니오 | 아니오 |
| 지침 수정 | 예 | 예 | 아니오 | 아니오 |
| 지침 취소/재시도 | 예 | 예 | 아니오 | 아니오 |
| 상태 조회 | 예 | 예 | 예 | 제한 |
| 알림 구독 | 예 | 예 | 예 | 아니오 |
| hook 보고 | 아니오 | 아니오 | 아니오 | 예 |
| MCP desired state 읽기 | 아니오 | 아니오 | 아니오 | 예 |
| MCP actual state 쓰기 | 아니오 | 아니오 | 아니오 | 예 |
| 감사 로그 조회 | 예 | 제한 | 아니오 | 아니오 |

## 역할별 설명

### `owner`

프로젝트의 최상위 관리자이다.

가능:

- 프로젝트 설정 변경
- 사용자 초대와 제거
- Agent 등록과 폐기
- API Key 발급과 폐기
- 모든 지침 관리
- 감사 로그 조회

### `maintainer`

프로젝트 운영 담당자이다.

가능:

- Agent 등록과 비활성화
- API Key 발급과 폐기
- 지침 생성, 수정, 취소, 재시도
- 상태 조회
- 알림 설정

불가:

- 프로젝트 소유권 변경
- owner 제거
- 프로젝트 삭제

### `viewer`

상태 확인 중심 사용자이다.

가능:

- 프로젝트 상태 조회
- Agent 상태 조회
- 지침 타임라인 조회
- 알림 구독

불가:

- 지침 생성 또는 변경
- Agent 설정 변경
- API Key 관리

### `agent`

시스템에 상태를 보고하는 실행 주체이다.

가능:

- hook 보고
- heartbeat 보고
- snapshot 보고
- lifecycle report 보고
- test result 보고
- MCP desired state 읽기
- MCP actual state 쓰기
- acknowledge 보고

불가:

- 사용자 화면 접근
- 프로젝트 설정 변경
- 다른 Agent 상태 변경
- 다른 프로젝트 접근

## Scope와 역할의 관계

사람 사용자는 역할 기반 권한을 사용한다.

Agent는 API Key scope 기반 권한을 사용한다.

예:

- `hook:write`
- `heartbeat:write`
- `snapshot:write`
- `lifecycle:write`
- `test_result:write`
- `mcp:read_desired_state`
- `mcp:write_actual_state`
- `mcp:ack_instruction`

## 기본 정책

- 사용자는 프로젝트별로 역할을 가진다.
- Agent는 하나 이상의 project scope에 묶인다.
- Agent API Key는 최소 scope만 가진다.
- BFF의 모든 Route Handler에서 권한을 다시 검사한다.
- UI에서 숨긴 기능도 서버 측 권한 검사를 생략하지 않는다.

