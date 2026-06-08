# 보존 정책

## 목적

상태 이벤트, heartbeat, lifecycle report, snapshot, 테스트 결과, 감사 로그의 기본 보존 기간을 정의한다.

기본 보존 기간은 문서 기준으로 고정하되, 실제 배포에서는 환경변수로 수정 가능하게 한다.

## 기본 보존 기간

| 데이터 | 기본 보존 기간 | 환경변수 | 비고 |
|---|---:|---|---|
| latest state | 장기 보존 | `RETENTION_LATEST_STATE_DAYS` | 기본값 없음. 삭제하지 않음 |
| state events | 90일 | `RETENTION_STATE_EVENTS_DAYS` | 상태 변화 이력 |
| heartbeat | 30일 | `RETENTION_HEARTBEAT_DAYS` | Agent 생존 신호 |
| lifecycle report | 30일 | `RETENTION_LIFECYCLE_REPORT_DAYS` | 최근 작업 보고 |
| snapshot | 90일 | `RETENTION_SNAPSHOT_DAYS` | 지침 상태 스냅샷 |
| test result | 장기 보존 | `RETENTION_TEST_RESULT_DAYS` | 기본값 없음. 삭제하지 않음 |
| audit logs | 장기 보존 | `RETENTION_AUDIT_LOG_DAYS` | 기본값 없음. 삭제하지 않음 |
| notification history | 90일 | `RETENTION_NOTIFICATION_HISTORY_DAYS` | 알림 전송 이력 |

## 환경변수 정책

환경변수 값은 일 단위 숫자로 설정한다.

예:

```env
RETENTION_STATE_EVENTS_DAYS=90
RETENTION_HEARTBEAT_DAYS=30
RETENTION_LIFECYCLE_REPORT_DAYS=30
RETENTION_SNAPSHOT_DAYS=90
RETENTION_NOTIFICATION_HISTORY_DAYS=90
```

값이 비어 있거나 설정되지 않은 경우:

- 장기 보존 항목은 삭제하지 않는다.
- 기간 보존 항목은 기본값을 사용한다.

값이 `0`인 경우:

- 자동 삭제하지 않는 것으로 해석한다.

## 보존 정책 적용 방식

보존 처리는 주기 작업으로 수행한다.

권장:

- 하루 1회 실행
- 삭제 전 dry-run 로그 남김
- 삭제 건수 기록
- 실패 시 감사 로그 또는 운영 로그 기록

## 삭제 대상

삭제는 원본 이력성 데이터에만 적용한다.

삭제 가능:

- 오래된 heartbeat
- 오래된 lifecycle report
- 오래된 state events
- 오래된 snapshot
- 오래된 notification history

삭제하지 않음:

- 최신 상태 스냅샷
- 지침 원문
- 지침 버전
- 테스트 결과
- 감사 로그
- API Key 발급/폐기 기록

## 요약 보존

추후 데이터가 많아지면 삭제 전에 요약 데이터를 남길 수 있다.

예:

- 일별 heartbeat 수
- Agent별 lifecycle report 수
- Agent별 실패 수
- 프로젝트별 완료 수

MVP에서는 요약 테이블을 필수로 두지 않는다.

