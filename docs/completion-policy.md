# 완료 판정 정책

## 원칙

완료 판정은 Agent 기준으로 한다.

사용자 최종 승인은 MVP의 완료 판정 조건에 포함하지 않는다. 사용자가 확인해야 하는 경우에는 `waiting_input` 또는 별도 확장 상태로 표현한다.

## 완료 상태 구분

| 상태 | 의미 |
|---|---|
| `completed` | Agent가 지침 수행 또는 구현을 완료함 |
| `tested` | Agent가 테스트 또는 검증을 완료함 |
| `completed_and_tested` | Agent 기준으로 수행 완료와 테스트 완료가 모두 충족됨 |

## `completed_and_tested` 조건

다음 조건을 모두 만족하면 `completed_and_tested`로 판정한다.

- 현재 `instruction_version`에 대해 `completed`가 보고됨
- 같은 `instruction_version`에 대해 `test_result`가 보고됨
- `test_result.status`가 `success` 또는 허용 가능한 성공 상태임
- Agent가 해당 `instruction_version`을 `acknowledged` 했음
- Agent가 다른 지침 버전으로 작업한 것이 아님
- 현재 지침이 `failed`, `cancelled`, `rejected` 상태가 아님

## 사용자 확인

사용자 확인은 최종 완료 조건에 포함하지 않는다.

필요한 경우:

- `waiting_input`: Agent가 사용자 응답을 기다리는 상태
- `verification_required`: 추후 사용자 검수가 필요한 확장 상태
- `user_verified`: 추후 사용자가 결과를 승인한 확장 상태

MVP에서는 `verification_required`와 `user_verified`를 필수 상태로 두지 않는다.

## 알림

`completed_and_tested`는 높은 우선순위 알림 대상이다.

알림 방식:

- 페이지 내부 모달 알림
- 브라우저 푸시 알림
- 대시보드 상태 갱신

