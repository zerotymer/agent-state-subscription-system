# 알림 기능 정의

## 목적

Agent의 지침 진행, 완료, 테스트 완료, 실패, 차단, 오프라인 상태를 사용자가 놓치지 않도록 알림으로 표현한다.

알림은 상태 추적을 보조하는 기능이다. 알림 수신 여부는 브라우저, OS, 사용자 권한 설정에 영향을 받으므로 최종 책임은 사용자 환경에 있다. 시스템은 알림을 시도하고 이력을 남기되, 알림만을 유일한 상태 전달 수단으로 보지 않는다.

## 알림 원칙

- 모든 중요한 상태는 대시보드에서 확인 가능해야 한다.
- 알림은 대시보드 상태를 보조한다.
- 페이지 내부 알림은 기본 지원한다.
- 브라우저 푸시 알림은 사용자 권한이 있을 때만 지원한다.
- iOS는 홈 화면에 추가한 PWA 사용자만 푸시 알림 지원 대상으로 본다.
- 알림 실패가 지침 실패를 의미하지는 않는다.

## 알림 종류

### 페이지 내부 알림

웹 페이지가 열려 있을 때 표시하는 알림이다.

형태:

- 토스트
- 모달
- 상단 배너
- 상태 뱃지

대상 이벤트:

- 지침 완료
- 테스트 완료
- 실패
- 차단
- 사용자 확인 필요
- Agent offline
- Agent stale

특징:

- 모바일과 데스크톱에서 가장 안정적이다.
- 별도 OS 권한이 필요 없다.
- 사용자가 페이지를 닫으면 받을 수 없다.

### 브라우저 푸시 알림

브라우저의 Push API, Notifications API, Service Worker를 사용해 표시하는 OS 수준 알림이다.

대상 이벤트:

- `completed_and_tested`
- `failed`
- `blocked`
- `verification_required`

정책:

- 모든 lifecycle report를 푸시 알림으로 보내지 않는다.
- 완료, 테스트 완료, 실패, 차단처럼 주요 상태 변화만 보낸다.
- 사용자가 명시적으로 권한을 허용한 경우에만 보낸다.
- 알림 권한이 거부되면 페이지 내부 알림만 사용한다.

## 모바일 웹 지원

### Android

Android Chrome 계열 브라우저는 Web Push 지원이 비교적 안정적이다.

권장:

- PWA 설치를 권장한다.
- 설치하지 않아도 브라우저 권한이 허용되면 푸시 알림을 사용할 수 있는 경우가 많다.
- 그래도 안정성을 위해 PWA 설치 상태를 우선 지원 대상으로 본다.

### iOS

iOS는 홈 화면에 추가한 웹앱을 기준으로 지원한다.

지원 조건:

- iOS 16.4 이상
- Safari에서 사이트를 홈 화면에 추가
- 홈 화면 아이콘으로 웹앱 실행
- 웹앱 manifest 구성
- Service Worker 구성
- Push API와 Notifications API 구성
- 사용자의 알림 권한 허용

정책:

- 일반 Safari 탭에서의 iOS 푸시 알림은 지원 대상으로 보지 않는다.
- 인앱 브라우저에서의 iOS 푸시 알림도 지원 대상으로 보지 않는다.
- iOS 사용자는 "홈 화면에 추가" 후 알림을 켜야 한다.
- 알림 권한, 집중 모드, OS 알림 설정은 사용자 책임으로 둔다.

## 알림 상태

알림 이력에는 다음 상태를 기록한다.

- `pending`
- `sent`
- `displayed`
- `failed`
- `permission_denied`
- `unsupported`
- `expired_subscription`

## 알림 우선순위

### High

즉시 알림을 보낸다.

- 테스트 완료
- 최종 완료
- 실패
- 차단
- 사용자 확인 필요

### Medium

페이지 내부 알림 위주로 표현한다.

- Agent stale
- Agent offline
- 장시간 진행 없음

### Low

대시보드 기록만 한다.

- heartbeat
- lifecycle report
- 일반 진행 단계
- 체크박스 부분 완료

## 필요한 구현 요소

- Web App manifest
- Service Worker
- Push subscription 저장
- VAPID key 관리
- Notification permission UI
- 사용자별 알림 설정
- 기기별 subscription 관리
- 알림 전송 이력
- 만료된 subscription 정리

## MVP 범위

MVP에서는 다음을 구현한다.

- 페이지 내부 모달 알림
- 페이지 내부 토스트 알림
- 브라우저 알림 권한 요청 UI
- Android와 데스크톱 Web Push
- iOS PWA 조건 안내
- 중요 이벤트만 push 대상으로 지정

RabbitMQ 기반 재시도는 2차 확장으로 둔다.

## 확정 알림 규칙

| 이벤트 또는 상태 | 페이지 내부 알림 | 브라우저 푸시 | 비고 |
|---|---|---|---|
| `completed_and_tested` | 예 | 예 | 최종 성공 |
| `failed` | 예 | 예 | 즉시 확인 필요 |
| `blocked` | 예 | 예 | 사용자 조치 가능성 높음 |
| `waiting_input` | 예 | 선택 | MVP에서는 내부 알림 우선 |
| `stale` | 예 | 아니오 | 운영성 신호 |
| `offline` | 예 | 아니오 | 운영성 신호 |
| `completed` | 예 | 아니오 | 테스트 완료 전 |
| `tested` | 예 | 아니오 | completed와 조합 전 |
| `heartbeat` | 아니오 | 아니오 | 기록만 |
| `lifecycle_report` | 아니오 | 아니오 | 기록만 |
| `snapshot` | 아니오 | 아니오 | 화면 갱신만 |

브라우저 푸시는 주요 상태 변화에만 사용한다. Agent 연결성 신호인 `offline`, `stale`은 페이지 내부 알림과 대시보드 표시로 처리한다.
