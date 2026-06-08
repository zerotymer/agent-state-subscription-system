# instructions/ — 작업 지침서 운영 규칙
> 이 문서는 이 프로젝트의 **모든 후속 작업이 반드시 참고**하는 운영 지침입니다.
> 지시서를 실행하는 에이전트/작업자는 작업 시작 전에 이 문서를 먼저 읽습니다.

## 1. 목적

`instructions/`는 ECN을 단계적으로 구현하기 위한 **지시서(작업 지시)**를 보관·추적하는 공간입니다.
각 지시서는 한 단위의 실행 가능한 작업을 정의하며, 완료되면 보관 디렉토리로 이동하고 로그에 기록합니다.

## 2. 디렉토리 구조

```
instructions/
├── README.md          # (이 문서) 작업 운영 지침 — 변경 시 모두에게 영향
├── completed.log      # 완료 지시서 누적 로그: UUID | 내용 | 일시
├── NNNN_name.md       # 진행 대기/진행 중 지시서 (순번 4자리 + 내용)
└── .completed/        # 완료된 지시서 보관 디렉토리
    └── NNNN_name.md
```

## 3. 지시서 파일 규칙

- 파일명: `NNNN_<짧은-내용>.md` — `NNNN`은 4자리 순번(`0001`부터), `<짧은-내용>`은 지시 내용을 나타내는 kebab/스네이크 식별자.
- 한 파일 = 하나의 완결된 작업 단위. 순번은 권장 실행 순서이며, 의존성은 각 지시서의 `선행 조건`에 명시.
- 모든 산문/커뮤니케이션은 한글, 코드·주석·식별자는 영어(전역 컨벤션).

### 지시서 표준 구조 (템플릿)

```markdown
# NNNN — <제목>

> 엔티티 UUID: <uuid>   (llm-reference-registry 등록 식별자)
> 상태: TODO | IN_PROGRESS | DONE
> 브랜치: feature/<name>-<uuid8>   (작업 시작 시 기입)

## 목표
<무엇을, 왜>

## 선행 조건
<완료되어 있어야 할 지시서 번호 / 외부 의존>

## 범위
- 포함:
- 제외:

## 작업 항목
1. ...

## 산출물
- <파일/문서/엔드포인트 등 + 각 산출물 UUID>

## 완료 기준 (DoD)
- [ ] 테스트 포함 (테스트 없는 기능 PR 금지)
- [ ] 가드레일 준수 확인 (§7)
- [ ] 관련 UUID 레지스트리 등록/갱신
- [ ] 브랜치에서 작업 후 PR (main 직접 push 금지)

## 가드레일 체크
<이 작업이 §7 범위 제외 사항을 건드리지 않는지>
```

## 4. 지시서 생명주기

1. **생성**: `instructions/NNNN_name.md`로 작성. 상태 `TODO`. 새 단위이므로 UUID 부여(§5).
2. **착수**: 상태 `IN_PROGRESS`로 변경, 브랜치 생성(§6) 후 브랜치명을 지시서에 기입. 세션 기록 갱신(§8).
3. **완료**: 완료 기준 충족 → 상태 `DONE` → 파일을 `instructions/.completed/`로 **이동** → `completed.log`에 1줄 추가.
4. **변경/재개**: 완료된 지시서를 다시 손대야 하면 새 순번의 후속 지시서를 만든다(완료 기록은 보존). UUID는 동일 단위면 재사용·갱신.

### completed.log 형식

탭/파이프 구분, 한 줄에 하나. 최신이 아래로 append.

```
UUID | 내용 | 일시(YYYY-MM-DD HH:MM:SS +0900)
```

## 5. UUID 식별자 부여 (필수)

> 스킬: `.claude/skills/llm-reference-registry` (UUID `5b33f658-5caa-4ac0-b711-d1fd6cfb57a9`)

- **모든 식별 가능한 단위**(지시서, 기능, API, 노드 명령, 프로토콜 메시지, Warehouse 테이블, 코드 심볼, 이슈)에 안정적인 UUID를 부여한다.
- 절차: 작업 전 `GET /resolve` 또는 `/search`로 **기존 등록 조회 후 재사용**. 신규는 `python3 -c "import uuid;print(uuid.uuid4())"`로 미리 생성해 `id` 고정 후 `POST /ingest/batch` 등록.
- 연결 정보: 게이트웨이 `https://llm-reg.zerotymer.net/api/v1/*`, 헤더 `X-API-Key: $LLM_REFERENCE_REGISTRY_API_KEY`, 프로젝트 `embbedded`.
- `ingest/batch`는 **각 엔티티 객체와 `source` 모두에 `project_id: "embbedded"`** 를 넣어야 한다. 신규 엔티티는 `candidate` 상태로 시작, 사람 검토 후 `active` 승격. 삭제 금지(폐기는 `deprecated`/`archived`).
- UUID는 **불변**. 같은 단위가 바뀌면 동일 UUID로 PATCH/재-ingest. 새로 만들지 않는다.
- 부여한 UUID는 관련 `docs/`·코드·지시서에 주석/머리말로 역기입한다.

## 6. 브랜치 전략 (기능 추가/수정 필수)

> 스킬: `.claude/skills/git-branch-guideline` (UUID `e03f48fb-3e00-41d7-b99d-c32854567d67`)

- `main`·`staging`에는 직접 commit/push **금지**. 기능 작업은 항상 브랜치에서.
- 기능: `feature/<name>-<uuid8>` (main 기준 분기). 버그: `fix/<YYYY-MM-DD>/<name>-<uuid8>`. 임시: `temp/<uuidv7>`.
- `<uuid8>`은 지시서 UUID 앞 8자리를 사용해 지시서↔브랜치를 연결한다.
- 작업 완료는 PR로 머지. 테스트 없는 기능 PR 금지(전역 규칙).

## 7. 범위 가드레일 (절대 준수)

`README.md` 및 `docs/05_security_and_compliance.md` 기준. 모든 지시서/코드/제안에 적용.

- **금지**: IP 은닉·익명화, 접근 제한 우회, 인증·결제 장벽 우회, 탐지·차단 회피, 회피 목적 트래픽 분산, 제3자 장비 무단 수집, 약관·법 위반 자동화.
- **필수**: robots.txt·약관 존중, 도메인별 rate limit, SSRF 방지(localhost·사설/link-local IP·클라우드 metadata 차단, 리다이렉트 후 IP 재검증, 포트 allowlist), 모든 노드 통신 TLS + 상호 신원 검증, 명령 **allowlist만 허용**(노드 임의 코드 실행 금지), 관리자 작업 감사 로그.
- 지시 내용이 가드레일을 약화/위반하면 **구현하지 말고 먼저 보고**한다.

## 8. 세션 기록 (새 세션 시작 시 필수)

> 스킬: `.claude/skills/project-session` (UUID `b3230cd0-6081-4570-9bda-183543f585e9`)

- 새 에이전트 세션 시작 시 `.claude/session`(현재 세션 상세) 및 `.claude/session.log`(`<session-id> - <topic>` 한 줄)를 생성/갱신한다.
- 이 파일들은 **로컬 런타임 상태**이며 Git에 커밋하지 않는다(`.gitignore` 처리됨).

## 9. 명세서

구현 시 권위 있는 명세는 `docs/`이다.
