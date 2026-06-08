# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 현재 상태

이 저장소는 **설계/문서 단계**다. 코드는 아직 없고 `docs/` 아래 20여 개의 한글 설계 문서만 있다. 빌드/린트/테스트 명령은 구현이 시작되면 추가된다. 새 기능을 구현할 때는 아래 "확정 스택"과 `docs/`의 설계 결정을 따른다. 임의로 다른 프레임워크나 구조를 택하지 않는다.

`docs/`가 단일 진실 공급원(source of truth)이다. 구현 전에 관련 문서를 먼저 읽고, 설계와 어긋나는 변경이 필요하면 코드보다 먼저 해당 문서를 갱신한다. README.md의 "문서 맵"에 각 문서 역할이 정리되어 있다.

## 무엇을 만드는가

여러 프로젝트에서 동작하는 다양한 에이전트의 상태/지시/진행/체크리스트/결과/오류를 **단방향으로 수집**해 정제·보관·표시·알림으로 변환하는 시스템. 여기서 "구독(subscription)"은 양방향 pub/sub가 아니라, 에이전트가 hook/skill 결과를 단방향으로 보내고 웹 계층이 이를 가공하는 구조를 뜻한다. 웹 계층은 에이전트를 **직접 제어하지 않는다** — 관찰과 표시가 1차 목표다.

## 확정 스택 (MVP)

`docs/framework-selection.md`에서 확정. 구현 시 이 선택을 따른다.

- **Web/UI**: Next.js (App Router) + React
- **BFF/API**: Next.js Route Handlers — 외부에 공개되는 유일한 계층
- **DB**: PostgreSQL (상태 원장 + 이력)
- **에이전트 통신**: HTTP Hook + MCP Endpoint/Adapter
- **실시간 화면**: Server-Sent Events (WebSocket 아님 — 단방향이라 SSE로 충분)
- **알림**: Web Push + 페이지 내부 모달
- **배포**: Docker Compose
- **Queue**: RabbitMQ (초기 필수 아님, 알림 재시도/worker 분리 시 도입)
- **MVP 제외**: Kubernetes, Kafka, MQTT, Reverse Proxy 필수화, 별도 backend service

## 핵심 아키텍처 불변식

설계의 중심은 **hook과 MCP의 책임 분리**, **desired/actual state 분리**, **instruction_version 기준 완료 판정**이다. 이 세 가지를 깨지 않는 것이 가장 중요하다.

### Hook vs MCP 책임 분리 (`docs/state-model.md`, `docs/connectivity.md`)

- **MCP = 높은 신뢰도(high reliability) 영역**: 최초 지시문, 전체 방향성, `instruction_version`, acknowledge, 명시적 actual_state 변경. `acknowledged` / `running` / `rejected` / `cancelled`는 MCP가 우선 경로다.
- **Hook = lifecycle 보고/진행 결과**: `waiting_input` / `blocked` / `completed` / `tested` / `failed` 보고, heartbeat, 진행 단계. 빈번한 진행 보고는 전부 hook으로 보낸다(모든 lifecycle을 MCP로 처리하지 않는다).
- **System/Timer**: 서버가 관찰로 판정. `created` / `queued` / `stale`(snapshot/heartbeat 지연) / `offline`(heartbeat 장기 미수신) / `completed_and_tested`.

### Desired / Actual State (`docs/mcp-state-model.md`)

웹은 에이전트를 강제 제어하지 않고, 에이전트가 읽을 `desired_state`를 저장한다. 에이전트는 자기 판단/실행 결과를 `actual_state`로 기록한다.

- 명령은 직접 실행 요청이 아니라 **에이전트가 읽는 상태**로 표현한다. 실제 수행 여부/시점은 에이전트가 결정한다.
- **desired_state와 actual_state는 같은 테이블에 섞지 않는다.**
- 모든 높은 신뢰도 상태에 `instruction_version`을 포함한다.
- MCP 세션 상태는 메모리에만 두지 않는다 — PostgreSQL 또는 Redis에 저장.
- 에이전트의 `instruction_version`이 현재 desired보다 낮으면 `superseded` 처리하고, 오래된 버전의 완료 보고는 최종 완료 판정에 쓰지 않는다.

### 완료 판정 (`docs/completion-policy.md`, `docs/state-model.md`)

`completed`만으로 최종 완료로 보지 않는다. **같은 `instruction_version` 안에서** 다음을 모두 만족할 때만 `completed_and_tested`로 판정한다: 해당 버전 `acknowledged` + `completed` 보고 + 성공한 `test_result` + 현재 지침이 `failed`/`cancelled`/`rejected`가 아님. 사용자 최종 승인은 MVP 완료 조건에 **포함하지 않는다**(필요 시 `waiting_input` 또는 확장 상태로).

### Instruction Version (`docs/instruction-version-policy.md`)

**에이전트 행동에 영향을 주는 변경만** 새 버전을 만든다. 목표/범위/완료조건/테스트요구/requested_actions/desired_state/체크리스트 항목 변경, 취소·재시도·할당 변경 → 버전 증가. 오타/설명 보강/표시용 제목/태그 → 버전 유지. 취소·재시도도 버전을 증가시킨다.

### 이벤트 모델 (`docs/event-schema.md`)

- 상태를 덮어쓰지 않고 **모든 변화를 이벤트로 남긴다**(event-sourcing). 최신 상태는 별도 스냅샷으로 유지.
- 모든 이벤트는 공통 envelope를 쓴다: `event_id`, `event_type`, `schema_version`, `reliability_level`(`high`/`medium`/`low`), `source`, `project_id`, `agent_id`, `instruction_id`, `instruction_version`, `sequence`, `occurred_at`, `payload` 등.
- **멱등 처리**: `event_id` 중복 → 무시. `agent_id+instruction_id+sequence` 중복 가능. sequence가 현재 snapshot보다 오래되면 최신 상태를 덮어쓰지 않는다.
- MVP 필수 이벤트: `heartbeat`, `lifecycle_report`, `snapshot`, `test_result`, `ack_report`, `state_change`. 나머지 타입은 스키마만 열어두고 선택 수신.
- 상태 코드와 이벤트 타입은 MVP에서 다 쓰지 않더라도 향후 호환성을 위해 넉넉히 정의한다.
- 긴 실행 로그는 상태 이벤트에 넣지 않는다 — 요약과 참조(`artifact_refs`)만 넣는다.

### 인증/공개 경계 (`docs/authentication.md`)

- 외부에 공개되는 계층은 **Next.js BFF 하나뿐**. PostgreSQL/Redis/RabbitMQ/Worker/분리된 MCP server는 내부망 전용.
- **사람 사용자**: SSO(Google/GitHub OAuth 우선) → HttpOnly Secure Cookie 세션. JWT를 브라우저 JS/localStorage에 노출하지 않는다(JWT cookie wrapping). 모든 Route Handler에서 서버 측 권한 검사.
- **Agent/MCP**: API Key(`ass_live_<key_id>_<secret>` 형식, `Authorization: Bearer`). secret 원문은 저장하지 않고 해시만. scope로 권한 분리(`hook:write`, `mcp:read_desired_state`, `mcp:write_actual_state` 등).
- 권한 검사는 UI 표시 여부와 무관하게 BFF endpoint에서 반드시 수행.
- 역할: `owner` / `maintainer` / `viewer` / `agent`.

## 구현 시 구조 규칙

`docs/framework-selection.md`, `docs/architecture.md`에서 명시한 분리 원칙:

- Route Handler는 **HTTP 요청 처리 계층**으로만 둔다. 상태 검증, 지침 버전 관리, 이벤트 저장 로직은 별도 **service 모듈**로 분리한다.
- 긴 작업, 알림 재시도, 대량 후처리는 요청 안에서 처리하지 않고 worker로 분리한다.
- scope check는 service module에서 공통화한다.
- 데이터 모델은 `docs/data-model-draft.md`에 초안이 있으나 "구현 전 재검토 전제"다 — 실제 ORM/PostgreSQL 설계 기준으로 다시 검토한다.

## 작업 컨벤션

- 커뮤니케이션/설계 문서는 한글, 코드 주석은 영어 (전역 CLAUDE.md 규칙).
- 테스트 없는 기능 구현 PR은 만들지 않는다.
- `npm`보다 `pnpm`, `pip`보다 `uv pip` 우선.
- main 브랜치 직접 push 금지 — 브랜치를 만들고 작업한다.
- secret/API 키 하드코딩 금지. `.env` / credentials 커밋 금지.

## 레퍼런스 레지스트리 (엔티티 식별자 부여)

> 스킬: `.claude/skills/llm-reference-registry` 참조 (UUID 5b33f658-5caa-4ac0-b711-d1fd6cfb57a9)

이 프로젝트의 **기능 추가·신규 이슈·아키텍처 변경**이 발생하면, 코드/문서 작업과 함께 **반드시 `llm-reference-registry`를 사용해 해당 단위에 안정적인 UUID(엔티티 식별자)를 부여**한다. 사용자가 별도로 요청하지 않아도, 식별 가능한 단위(기능/화면/API/인프라/코드 심볼/이슈)가 생기거나 의미 있게 바뀌면 등록·갱신한다.

### 적용 시점 (적극 사용)

- 새로운 기능/컴포넌트를 설계하거나 명세에 추가할 때 → `FEATURE` / `UI_AREA` / `CODE_SYMBOL` 등록.
- 새 노드 명령, 프로토콜 메시지 타입, API 엔드포인트, Warehouse 테이블 추가 시 → `CODE_SYMBOL` / `API` / `INFRA_UNIT` 등록.
- 버그·작업·추적 항목이 생기면 → `ISSUE` 등록.
- 기존 단위가 바뀌면 같은 UUID로 **갱신**(PATCH/재-ingest). UUID는 불변이며 절대 새로 만들지 않는다.
- 작업 시작 전, 이미 등록된 단위는 먼저 resolve/search로 **조회 후 재사용**한다(중복 등록 금지).

### 검증된 연결 정보 (2026-06-06 동작 확인)

- **인증 키 환경변수: `LLM_REFERENCE_REGISTRY_API_KEY`** (스킬 문서의 예시 `REGISTRY_API_KEY`가 아님). 키는 코드/문서에 하드코딩 금지, 환경변수에서만 읽는다.
- **프로젝트 ID: `embbedded`** (게이트웨이 `/projects`로 확인). 이 키는 이 프로젝트에만 스코프됨.
- **게이트웨이(우선): `https://llm-reg.zerotymer.net/api/v1/*`**, 헤더 `X-API-Key: $LLM_REFERENCE_REGISTRY_API_KEY`. 직접 백엔드 폴백: `https://llm-api.zerotymer.net/*`.
- **`POST /ingest/batch` 주의:** `project_id`를 **각 엔티티 객체(및 `source`)에 명시**해야 한다. top-level에만 넣거나 생략하면 `FORBIDDEN(API key restricted to a different project)`로 거부된다. 단건 `POST /entities`는 본문에 `project_id`를 넣으면 된다.
- 신규 엔티티 기본 상태는 `candidate`. 사람 검토 후 `active`로 승격. 삭제 불가 → 폐기는 `deprecated`(+`replacement_entity_id`) 또는 `archived`(PATCH).

### 권장 절차 (Resolve → Bundle → Act / 신규는 ingest)

1. 기존 참조는 `GET /resolve?alias=...&locale=ko` 또는 `GET /search?q=...`로 먼저 조회.
2. 신규 단위는 UUID를 미리 생성(`python3 -c "import uuid;print(uuid.uuid4())"`)해 `id`에 고정하고, `aliases`(ko+en)·`contexts`(최소 `summary`)·`relations`(`CONTAINS`/`DEPENDS_ON` 등)와 함께 `POST /ingest/batch`로 등록.
3. 등록한 UUID는 가능하면 관련 `docs/` 문서나 코드에 주석으로 역기입해 후속 에이전트가 alias 추측 없이 참조하도록 한다.

> 이미 등록된 시드(candidate): `Embedded Layer` = `7abfc36a-272b-4a51-90af-898d035430be`(FEATURE), `FETCH_HTTP command` = `181cf4d7-e78c-4605-9bd5-f8cf265f5d46`(CODE_SYMBOL), `CONTAINS` 관계로 연결됨.

## SKILL
### SKILL IMPORT
> .claude/skills/wiki-skill-import 참조

`wiki-skill-import`로 스킬을 가져오면(import 모드로 `skills/<skill-name>/`에 파일을 작성한 경우) **반드시 아래 표에 해당 스킬을 자동으로 한 행 추가**한다. 사용자가 별도로 요청하지 않아도 import가 완료된 시점에 식별자(UUID)·이름·설명을 채워 기록한다. 이미 존재하는 UUID면 새 행을 만들지 말고 기존 행을 갱신한다.

| 식별자(UUID) | 이름 | 설명 |
| --- | --- | --- |
| 14cea96f-f1e2-461e-b9a7-96b5a6f63b67 | wiki-skill-import | 사용자가 UUID를 제공하면 wikijs-agent-gateway를 통해 Wiki.js 사이트에서 프로젝트 로컬 스킬을 가져온다. 영어와 한국어 페이지를 가져와 SKILL.md와 SKILL_ko.md를 작성한다 |
| e03f48fb-3e00-41d7-b99d-c32854567d67 | git-branch-guideline | 프로젝트 Git 브랜치 이름 및 사용 규칙 |
| 55723065-ac44-4a21-828e-aca40d0011c5 | dependency-track-sbom-agent | Dagu나 프로젝트 셸 스크립트 없이 에이전트 설정값만 사용해 Trivy CycloneDX SBOM을 생성하고 OWASP Dependency-Track에 업로드한다 |
| 5b33f658-5caa-4ac0-b711-d1fd6cfb57a9 | llm-reference-registry | UI 영역·기능·인프라 단위·API·코드 심볼·이슈의 UUID 기반 영구 참조 저장소. 코딩 에이전트가 MCP 도구와 HTTP API(Next BFF 게이트웨이 또는 백엔드 직접)로 엔티티를 조회·resolve·상호참조하는 방법을 안내한다 |
| b3230cd0-6081-4570-9bda-183543f585e9 | project-session | 코딩 에이전트 세션 시작·추적 시 프로젝트 로컬 `.codex/session`·`.codex/session.log`(또는 `.claude/session`·`.claude/session.log`) 파일을 생성·갱신한다. Git에 커밋하지 않는 로컬 실행 상태 |
| e6274b24-2c08-4367-8859-b5a92bd98d59 | static-mockup-preview-server | web-artifacts-builder 목업 결과물을 `outout/{mockup}` 경로에서 npx serve 또는 Python http.server로 잠깐 확인하는 정적 서버 가이드 |