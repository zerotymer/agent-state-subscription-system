# 이벤트 스키마

## 목적

Agent가 hook 또는 MCP를 통해 보고하는 이벤트의 공통 형식과 타입별 payload를 정의한다.

이벤트 타입은 MVP에서 모두 사용하지 않더라도 향후 호환성을 위해 넉넉하게 정의한다. 단, 각 이벤트는 공통 envelope를 사용하여 저장, 필터링, 멱등 처리, 권한 검사를 일관되게 수행한다.

## 이벤트 타입

| 이벤트 타입 | 목적 | 기본 경로 | MVP 필수 |
|---|---|---|---|
| `heartbeat` | Agent 생존 확인 | `hook` 또는 `mcp` | 예 |
| `lifecycle_report` | 최근 작업, 직전 지시, 현재 상태 보고 | `hook` | 예 |
| `snapshot` | 현재 지침/단계/체크리스트 전체 스냅샷 | `hook` 또는 `mcp` | 예 |
| `stage_progress` | 특정 단계 시작/완료/진행 보고 | `hook` | 선택 |
| `checklist_update` | 지침 내 체크박스 상태 변경 보고 | `hook` | 선택 |
| `test_result` | 테스트 또는 검증 결과 보고 | `hook` 또는 `mcp` | 예 |
| `failure_report` | 실패 보고 | `hook` | 선택 |
| `blocked_report` | 차단 사유 보고 | `hook` | 선택 |
| `artifact_report` | 산출물 링크 또는 결과물 보고 | `hook` | 선택 |
| `ack_report` | 지침 버전 인지 보고 | `mcp` | 예 |
| `state_change` | actual state 명시 변경 | `mcp` | 예 |

## 공통 Envelope

모든 이벤트는 다음 공통 필드를 가진다.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `event_id` | string | 예 | 이벤트 고유 ID. 멱등 처리에 사용 |
| `event_type` | string | 예 | 이벤트 타입 |
| `schema_version` | string | 예 | 이벤트 스키마 버전 |
| `reliability_level` | string | 예 | `high`, `medium`, `low` |
| `source` | string | 예 | `hook`, `mcp`, `system`, `timer`, `user` |
| `project_id` | string | 예 | 프로젝트 ID |
| `agent_id` | string | 예 | Agent ID |
| `instruction_id` | string | 조건부 | 지침 관련 이벤트일 때 필수 |
| `instruction_version` | number | 조건부 | 지침 관련 이벤트일 때 권장, 높은 신뢰도 이벤트는 필수 |
| `sequence` | number | 권장 | Agent 기준 증가 번호 |
| `occurred_at` | string | 예 | Agent 또는 source에서 발생한 시각 |
| `sent_at` | string | 선택 | Agent가 전송한 시각 |
| `received_at` | string | 서버 | 서버가 수신한 시각 |
| `trace_id` | string | 선택 | 관련 이벤트 추적 ID |
| `correlation_id` | string | 선택 | 지침, 요청, 테스트 흐름 묶음 |
| `payload` | object | 예 | 이벤트 타입별 상세 데이터 |

## 공통 예시

```json
{
  "event_id": "evt_01HZX0001",
  "event_type": "lifecycle_report",
  "schema_version": "1.0",
  "reliability_level": "medium",
  "source": "hook",
  "project_id": "proj_001",
  "agent_id": "agent_001",
  "instruction_id": "inst_001",
  "instruction_version": 3,
  "sequence": 42,
  "occurred_at": "2026-06-07T10:00:00+09:00",
  "sent_at": "2026-06-07T10:00:01+09:00",
  "trace_id": "trace_abc",
  "payload": {}
}
```

## 타입별 Payload

### `heartbeat`

Agent 생존 여부를 확인한다.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `status` | string | 예 | `online`, `busy`, `idle` |
| `current_instruction_id` | string | 선택 | 현재 수행 중인 지침 |
| `current_instruction_version` | number | 선택 | 현재 수행 중인 지침 버전 |
| `agent_version` | string | 선택 | Agent 버전 |
| `capabilities_hash` | string | 선택 | capability 변경 감지용 hash |

### `lifecycle_report`

최근 작업과 현재 흐름을 요약 보고한다.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `actual_state` | string | 예 | 현재 Agent 기준 상태 |
| `current_stage` | string | 선택 | 현재 단계 |
| `last_action` | string | 선택 | 직전 수행 작업 |
| `last_instruction_summary` | string | 선택 | 직전 참조 지침 요약 |
| `message` | string | 선택 | 사용자에게 표시 가능한 요약 |
| `progress_note` | string | 선택 | 진행 설명 |

### `snapshot`

현재 지침 상태 전체를 보고한다.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `actual_state` | string | 예 | 현재 상태 |
| `current_stage` | string | 선택 | 현재 단계 |
| `stages` | array | 선택 | 단계 목록 |
| `checklist` | array | 선택 | 체크박스 목록 |
| `artifacts` | array | 선택 | 산출물 목록 |
| `test_status` | string | 선택 | `not_run`, `running`, `success`, `failed`, `skipped` |
| `summary` | string | 선택 | 현재 상태 요약 |

### `stage_progress`

특정 단계의 진행을 보고한다.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `stage_id` | string | 선택 | 단계 ID |
| `stage_name` | string | 예 | 단계명 |
| `stage_status` | string | 예 | `pending`, `running`, `completed`, `failed`, `skipped` |
| `stage_index` | number | 선택 | 현재 단계 번호 |
| `stage_total` | number | 선택 | 전체 단계 수 |
| `message` | string | 선택 | 단계 설명 |

### `checklist_update`

지침 내 체크박스 상태를 보고한다.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `checklist_item_id` | string | 선택 | 체크 항목 ID |
| `label` | string | 예 | 체크박스 문구 |
| `checked` | boolean | 예 | 완료 여부 |
| `source_line` | number | 선택 | 지침 원문 내 줄 번호 |
| `evidence` | string | 선택 | 완료 근거 |

### `test_result`

테스트 또는 검증 결과를 보고한다.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `test_status` | string | 예 | `success`, `failed`, `skipped`, `not_applicable` |
| `test_command` | string | 선택 | 실행한 테스트 명령 |
| `passed` | number | 선택 | 통과 수 |
| `failed` | number | 선택 | 실패 수 |
| `duration_ms` | number | 선택 | 테스트 시간 |
| `summary` | string | 선택 | 테스트 요약 |
| `report_ref` | string | 선택 | 테스트 리포트 참조 |

### `failure_report`

실패를 보고한다.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `error_code` | string | 선택 | 오류 코드 |
| `error_message` | string | 예 | 오류 메시지 |
| `retryable` | boolean | 선택 | 재시도 가능 여부 |
| `failed_stage` | string | 선택 | 실패 단계 |
| `details_ref` | string | 선택 | 상세 로그 참조 |

### `blocked_report`

차단 상태를 보고한다.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `reason` | string | 예 | 차단 사유 |
| `blocking_dependency` | string | 선택 | 막고 있는 외부 의존성 |
| `required_action` | string | 선택 | 필요한 조치 |
| `retryable` | boolean | 선택 | 재시도 가능 여부 |

### `artifact_report`

산출물을 보고한다.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `artifact_id` | string | 선택 | 산출물 ID |
| `artifact_type` | string | 예 | `file`, `url`, `log`, `report`, `diff` |
| `title` | string | 선택 | 산출물 제목 |
| `ref` | string | 예 | 파일 경로, URL, object key 등 |
| `summary` | string | 선택 | 산출물 요약 |

### `ack_report`

Agent가 지침 버전을 인지했음을 보고한다.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `ack_status` | string | 예 | `acknowledged`, `rejected` |
| `acknowledged_instruction_version` | number | 예 | 인지한 지침 버전 |
| `capability_match` | boolean | 선택 | 수행 capability 충족 여부 |
| `message` | string | 선택 | Agent 메시지 |

### `state_change`

Agent가 actual state를 명시적으로 변경한다.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `previous_state` | string | 선택 | 이전 상태 |
| `actual_state` | string | 예 | 변경 후 상태 |
| `reason` | string | 선택 | 변경 사유 |
| `effective_at` | string | 선택 | 상태 적용 시각 |

## 멱등 처리

서버는 다음 순서로 중복을 방지한다.

1. `event_id`가 이미 있으면 중복으로 처리한다.
2. `agent_id + instruction_id + sequence`가 이미 있으면 중복으로 처리할 수 있다.
3. sequence가 현재 snapshot보다 오래된 경우 최신 상태를 덮어쓰지 않는다.

## 신뢰도 처리

| 신뢰도 | 사용처 |
|---|---|
| `high` | MCP ack, instruction_version, desired/actual state |
| `medium` | lifecycle report, snapshot, test_result, failure_report |
| `low` | 단순 로그성 artifact, 상세 진행 메시지 |

## MVP 필수 이벤트

MVP에서 반드시 처리할 이벤트:

- `heartbeat`
- `lifecycle_report`
- `snapshot`
- `test_result`
- `ack_report`
- `state_change`

나머지는 스키마를 열어두고 선택적으로 수신한다.

