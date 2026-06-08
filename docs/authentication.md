# 인증과 권한 방침

## 기본 원칙

외부에 공개되는 계층은 Next.js BFF 하나로 제한한다.

공개 endpoint:

- Web Dashboard
- BFF API
- Hook endpoint
- MCP endpoint
- Web Push subscription endpoint

비공개 또는 내부 전용:

- PostgreSQL
- Redis
- RabbitMQ
- Notification Worker
- 별도 Node MCP server를 분리하는 경우에도 BFF와 같은 내부 배포망에 둔다.

API와 MCP 사용자는 BFF 계층만 통과하도록 유도한다. Agent, 외부 사용자, 브라우저, MCP client는 모두 BFF에서 인증과 권한 검사를 받는다.

## 인증 주체

### Human User

대시보드와 설정 화면을 사용하는 사람이다.

권장 인증:

- SSO 우선
- OAuth/OIDC provider 연동
- 세션은 HttpOnly Secure Cookie로 관리
- Next.js BFF에서 세션 검증

후보 SSO provider:

- Google
- GitHub
- Microsoft Entra ID
- Keycloak
- Authentik

초기 개인용/소수 사용자 대상이라면 Google 또는 GitHub OAuth가 가장 단순하다. 자체 계정과 비밀번호는 후순위로 둔다.

### Agent

작업 상태를 보고하고 MCP endpoint와 통신하는 실행 주체이다.

권장 인증:

- Agent API Key
- Agent별 scope
- project별 권한
- key rotation
- key revocation

Agent는 사람 계정의 아이디/비밀번호를 사용하지 않는다.

### MCP Client 또는 Adapter

Agent 옆에서 MCP 통신을 담당하는 client이다.

권장 인증:

- MCP 전용 API Key
- Agent API Key와 동일한 키를 사용할 수 있으나 scope는 분리한다.
- `mcp:read_desired_state`
- `mcp:write_actual_state`
- `mcp:ack_instruction`
- `mcp:write_snapshot`

MCP/API 사용 시 아이디/비밀번호 로그인을 사용하지 않는다.

## 사용자 세션 정책

### JWT Cookie Wrapping

Next.js BFF에서는 JWT를 브라우저 JavaScript에 직접 노출하지 않고 쿠키로 감싼다.

권장:

- HttpOnly
- Secure
- SameSite=Lax 또는 Strict
- 짧은 access session lifetime
- 필요 시 refresh session 별도 관리
- 서버 측에서 매 요청 검증

정책:

- 브라우저 localStorage에 access token을 저장하지 않는다.
- 클라이언트 컴포넌트에서 JWT를 직접 읽지 않는다.
- Route Handler, Server Action, Server Component에서 서버 측 세션을 확인한다.
- 권한 검사는 UI 표시 여부와 별개로 BFF endpoint에서 반드시 수행한다.

## API Key 정책

### 키 발급

사용자는 대시보드에서 Agent API Key를 발급받는다.

발급 시 저장:

- `key_id`
- `hashed_key`
- `owner_user_id`
- `project_id`
- `agent_id`
- `scopes`
- `created_at`
- `expires_at`
- `last_used_at`
- `revoked_at`

원문 API Key는 최초 발급 시 한 번만 보여준다.

### 키 형식

예시:

```text
ass_live_<key_id>_<secret>
```

구성:

- prefix: 키 종류와 환경 구분
- key_id: 조회용 공개 식별자
- secret: 충분히 긴 랜덤 문자열

서버는 secret 원문을 저장하지 않고 해시만 저장한다.

### 키 검증

검증 순서:

1. Authorization header 확인
2. key prefix 확인
3. key_id로 후보 조회
4. secret hash 비교
5. revoked/expired 확인
6. scope 확인
7. project/agent 권한 확인
8. last_used_at 갱신

권장 header:

```http
Authorization: Bearer ass_live_xxx
```

기본은 `Authorization: Bearer`를 사용한다.

## Scope 모델

### 사용자 권한

- `project:read`
- `project:admin`
- `agent:read`
- `agent:manage`
- `instruction:read`
- `instruction:create`
- `instruction:update`
- `notification:manage`
- `api_key:manage`

### Agent 권한

- `hook:write`
- `heartbeat:write`
- `snapshot:write`
- `lifecycle:write`
- `test_result:write`
- `mcp:read_desired_state`
- `mcp:write_actual_state`
- `mcp:ack_instruction`

## SSO 정책

SSO는 사람 사용자 인증에 사용한다.

초기 추천:

- Google OAuth 또는 GitHub OAuth
- 소수 사용자 allowlist
- 승인된 이메일 또는 organization 기준 접근 허용

확장:

- OIDC provider 교체 가능 구조
- Keycloak 또는 Authentik 같은 자체 IdP
- 조직별 SSO

사용자 최초 로그인 시:

1. OAuth/OIDC 인증
2. 이메일 검증
3. allowlist 또는 초대 상태 확인
4. 내부 user 생성 또는 매핑
5. session cookie 발급

## Agent 연동성과 사용량 조회

Agent API Key를 기준으로 사용량을 기록한다.

기록 대상:

- hook 호출 수
- MCP read/write 호출 수
- heartbeat 수
- snapshot 수
- lifecycle report 수
- test result 수
- 마지막 호출 시각
- 실패한 인증 수
- rate limit 초과 수

대시보드에서 제공:

- Agent별 최근 호출
- Agent별 이벤트 수
- Agent별 오류율
- Agent별 마지막 heartbeat
- API Key별 사용량
- 프로젝트별 총 사용량

## Rate Limit

Rate limit은 사용자와 Agent를 분리한다.

예:

- 사용자 대시보드 API: user_id 기준
- hook endpoint: agent_id 또는 key_id 기준
- MCP endpoint: agent_id 또는 key_id 기준
- 알림 subscription: user_id + device_id 기준

초기에는 단순한 분당 제한으로 시작하고, 필요 시 Redis 기반 sliding window로 확장한다.

## Endpoint별 인증

| Endpoint | 인증 방식 |
|---|---|
| Web Dashboard | SSO session cookie |
| Dashboard API | SSO session cookie |
| Hook API | Agent API Key |
| Heartbeat API | Agent API Key |
| Snapshot API | Agent API Key |
| MCP Endpoint | MCP/Agent API Key |
| Web Push Subscription | SSO session cookie |
| Admin API | SSO session cookie + admin role |

## 권장 구현

### Next.js BFF

- 모든 Route Handler에서 인증과 권한을 검증한다.
- 사용자 세션은 HttpOnly cookie로 관리한다.
- Agent/MCP 요청은 API Key로 검증한다.
- service module에서 scope check를 공통화한다.

### 데이터베이스

필수 테이블:

- users
- user_identities
- sessions
- projects
- project_members
- agents
- agent_api_keys
- api_key_usage
- audit_logs

## 감사 로그

기록 대상:

- 사용자 로그인
- API Key 발급
- API Key 폐기
- Agent 등록
- 지침 생성
- 지침 버전 변경
- MCP acknowledge
- 권한 변경
- 알림 설정 변경

감사 로그는 일반 lifecycle report보다 오래 보존한다.

## 최종 정책

- 사람 사용자는 SSO로 로그인한다.
- 브라우저 세션은 JWT cookie wrapping으로 관리한다.
- Agent와 MCP/API 사용자는 API Key로 인증한다.
- 아이디/비밀번호 기반 Agent 인증은 사용하지 않는다.
- API와 MCP는 BFF 계층만 외부에 열어둔다.
- 내부 저장소와 queue는 외부에 직접 노출하지 않는다.

