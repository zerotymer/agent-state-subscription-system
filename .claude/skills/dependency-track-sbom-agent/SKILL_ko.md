---
name: dependency-track-sbom-agent
version: 1.0.0
description: Dagu나 프로젝트 셸 스크립트 없이 에이전트 설정값만 사용해 Trivy CycloneDX SBOM을 생성하고 OWASP Dependency-Track에 업로드하는 Skill입니다.
---

# Dependency-Track SBOM Agent Skill

## 목표

에이전트가 프로젝트를 스캔해 `sbom.json`을 만들고 OWASP Dependency-Track에 업로드하여 취약점 분석을 시작할 때 사용합니다.

이 Skill은 agent-only 워크플로입니다. Dagu, cron YAML, `sbom-upload.sh`, 프로젝트 로컬 시크릿 파일을 만들거나 의존하지 않습니다. 기존 스케줄러/스크립트 예시는 마이그레이션 참고 자료로만 취급합니다.

## 필수 도구

- `trivy`
- `curl`
- `git`: 프로젝트 루트와 Git tag fallback 확인용으로 권장
- `node`, `python3`, `jq`: 메타데이터 추출과 응답 파싱용 선택 도구

## 에이전트 관리 설정값

재사용 설정값은 저장소가 아니라 에이전트 자체 설정 또는 시크릿 저장소에 보관합니다.

예시:

- Claude Code: `/config` 명령/설정 흐름 사용.
- Codex / Pi Agent / 기타 에이전트: 해당 에이전트의 settings, memory, secrets, runtime environment 기능 사용.

에이전트가 해석해야 하는 논리 키입니다.

| 논리 키 | 필수 | 설명 |
|---|---:|---|
| `DEPENDENCY_TRACK_URL` | 예 | Dependency-Track API base URL. 예: `http://depencytrack.com`. |
| `BOM_UPLOAD_API_KEY` | 예 | 업로드 요청에만 사용하는 API key. 로그에 출력하지 않습니다. |
| `PARENT_UUID` | 아니오 | 있으면 해당 parent 아래에 프로젝트를 생성/갱신합니다. 없으면 루트 레벨입니다. |

프로젝트 로컬 값은 보통 실행 시 추론합니다.

| 런타임 값 | 기본값 / 추론 |
|---|---|
| `TARGET_PATH` | Git root 또는 현재 프로젝트 루트. |
| `SBOM_FILE` | 프로젝트 루트의 `sbom.json`. |
| `PROJECT_NAME` | manifest name, module name, repository name, directory name. |
| `PROJECT_VERSION` | manifest version, 최신 Git tag, 또는 fallback `0.0.0-dev`. |
| `TRIVY_INCLUDE_VULN` | 취약점 검사 워크플로이므로 기본 `true`. |
| `TRIVY_SKIP_DIRS` | 생성물/대용량 디렉터리를 자동 감지. |

자동 추론이 틀린 경우가 아니라면 사용자가 `PROJECT_NAME`, `PROJECT_VERSION`, `SBOM_FILE`, `TARGET_PATH`, `TRIVY_INCLUDE_VULN`, `TRIVY_SKIP_DIRS`를 직접 관리하게 하지 않습니다.

## 메타데이터 추론

사용자에게 묻기 전에 먼저 추론합니다.

우선순위:

1. Node/TypeScript: `package.json`의 `name`, `version`.
2. Python: `pyproject.toml`의 `[project]` 또는 Poetry metadata.
3. Java Maven: root `pom.xml`의 `artifactId`, `version`.
4. Java Gradle: `settings.gradle*`의 `rootProject.name`, `build.gradle*`의 `version`.
5. Rust: `Cargo.toml`의 `[package]` `name`, `version`.
6. Go: `go.mod` module basename을 name으로 사용. version은 Git tag 또는 명시 설정 사용.
7. fallback name: Git repository 또는 directory basename.
8. fallback version: 최신 Git tag. 없으면 `0.0.0-dev`를 사용하고 fallback 사용 사실을 보고합니다.

lockfile 안의 dependency version으로 프로젝트 버전을 추론하지 않습니다.

## SBOM 파일 정책

별도 지정이 없으면 `<TARGET_PATH>/sbom.json`을 사용합니다. Git에 커밋되지 않게 관리합니다.

권장 `.gitignore` 항목:

```gitignore
# Generated SBOM
sbom.json
```

사용자가 SBOM artifact를 버전 관리하겠다고 명시하지 않는 한 `sbom.json`을 커밋하지 않습니다.

## Trivy 정책

기본 명령 형태:

```bash
trivy fs --format cyclonedx --scanners vuln --output sbom.json <TARGET_PATH>
```

`TRIVY_INCLUDE_VULN=true`이면 `--scanners vuln`을 사용합니다. 사용자가 순수 SBOM만 원하고 Dependency-Track만 취약점 분석 주체로 두고 싶을 때만 false로 둡니다.

`TRIVY_SKIP_DIRS`는 `TARGET_PATH` 아래 실제 존재하는 디렉터리 기준으로 자동 생성합니다. 일반 후보:

```text
.git node_modules .next .nuxt dist build coverage .turbo out .cache
target .gradle .mvn/wrapper __pycache__ .pytest_cache .mypy_cache
.venv venv .tox vendor/bundle
```

규칙:

- 생성물/대용량 디렉터리만 제외합니다.
- `package.json`, lockfile, `pom.xml`, `build.gradle`, `pyproject.toml`, `requirements*.txt`, `Cargo.toml`, `Cargo.lock`, `go.mod`, `go.sum` 같은 manifest/lockfile은 제외하지 않습니다.
- 각 skip 항목은 `--skip-dirs <path>` 인자로 개별 전달합니다.

## 업로드 정책

Dependency-Track BOM API에 multipart form으로 업로드합니다.

```bash
curl -fsS -X POST "${DEPENDENCY_TRACK_URL%/}/api/v1/bom"   -H "X-Api-Key: ${BOM_UPLOAD_API_KEY}"   -F "autoCreate=true"   -F "projectName=${PROJECT_NAME}"   -F "projectVersion=${PROJECT_VERSION}"   -F "bom=@${SBOM_FILE};type=application/json"
```

`PARENT_UUID`가 있으면 다음 항목을 추가합니다.

```bash
-F "parentUUID=${PARENT_UUID}"
```

동작:

- `autoCreate=true`는 프로젝트/버전이 없으면 생성합니다.
- `PARENT_UUID` 있음: 해당 parent 아래에 생성/갱신합니다.
- `PARENT_UUID` 없음: Dependency-Track 루트 레벨에 생성/갱신합니다.
- 업로드 성공은 BOM 접수 성공을 의미합니다. Dependency-Track 분석은 이후 비동기로 계속될 수 있습니다.

## 에이전트 절차

1. `TARGET_PATH`를 프로젝트 루트로 결정합니다.
2. `DEPENDENCY_TRACK_URL`, `BOM_UPLOAD_API_KEY`, 선택값 `PARENT_UUID`를 에이전트 설정/시크릿에서 읽습니다.
3. 프로젝트에서 `PROJECT_NAME`, `PROJECT_VERSION`을 추론합니다.
4. 별도 지정이 없으면 `SBOM_FILE=sbom.json`으로 둡니다.
5. 저장소 수정이 허용된 경우 `sbom.json`이 Git ignore 되도록 보장합니다.
6. 자동 skip directory를 포함해 Trivy 옵션을 구성합니다.
7. Trivy를 실행하고 `sbom.json` 생성 여부를 확인합니다.
8. multipart form upload로 Dependency-Track에 SBOM을 업로드합니다.
9. 프로젝트명/버전, 대상 경로, parent 사용 여부, scanner mode, skip dirs, HTTP 결과, 반환 token이 있으면 token을 보고합니다.

## 실패 처리

| 실패 | 에이전트 동작 |
|---|---|
| Dependency-Track URL/API key 없음 | Claude Code `/config` 또는 동등한 에이전트 설정/시크릿 저장소에 저장하도록 요청합니다. |
| 프로젝트 이름 없음 | manifest/module/repository/directory fallback을 사용하고 보고합니다. |
| 프로젝트 버전 없음 | manifest/Git tag를 사용합니다. 없으면 `0.0.0-dev`를 사용하고 보고합니다. |
| Trivy 미설치 | 중단하고 에이전트 런타임에 Trivy 설치를 요청합니다. |
| HTTP 401/403 | API key가 잘못됐거나 BOM upload / project auto-create 권한이 부족합니다. |
| HTTP 404 | Dependency-Track API base URL과 port를 확인합니다. |
| HTTP 413 | SBOM payload가 큽니다. scan scope를 줄이거나 서버/proxy 제한을 조정합니다. |
| HTTP 5xx | Dependency-Track 서버 장애로 보고하고 non-secret 응답 요약만 남깁니다. |

## 하지 말 것

- Git에 시크릿을 저장하지 않습니다.
- `BOM_UPLOAD_API_KEY`를 출력하지 않습니다.
- 업로드 접수 직후 취약점 분석이 완료됐다고 말하지 않습니다.

