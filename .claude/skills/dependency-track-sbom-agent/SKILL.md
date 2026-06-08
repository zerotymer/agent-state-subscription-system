---
name: dependency-track-sbom-agent
version: 1.0.0
description: Generate a Trivy CycloneDX SBOM and upload it to OWASP Dependency-Track using only agent-managed settings, without Dagu or project shell scripts.
---

# Dependency-Track SBOM Agent Skill

## Goal

Use this skill when an agent must scan a project, create `sbom.json`, and upload it to OWASP Dependency-Track for vulnerability analysis.

This is an agent-only workflow. Do not create or depend on Dagu, cron YAML, `sbom-upload.sh`, or project-local secret files. Existing scheduler/script examples are migration context only.

## Required tools

- `trivy`
- `curl`
- `git` recommended for project root and tag fallback
- `node`, `python3`, `jq` optional for better metadata/response parsing

## Agent-managed settings

Store reusable values in the agent's own configuration or secret store, not in the repository.

Examples:

- Claude Code: use its `/config` command/settings flow.
- Codex / Pi Agent / other agents: use the agent's settings, memory, secrets, or runtime environment mechanism.

Logical keys the agent must resolve:

| Logical key | Required | Notes |
|---|---:|---|
| `DEPENDENCY_TRACK_URL` | yes | Dependency-Track API base URL, for example `http://depencytrack.com`. |
| `BOM_UPLOAD_API_KEY` | yes | API key used only for the upload request. Do not print it in logs. |
| `PARENT_UUID` | no | If present, upload/create the project below this parent. If absent, the project is root-level. |

Project-local values are normally inferred at runtime:

| Runtime value | Default / inference |
|---|---|
| `TARGET_PATH` | Git root or current project root. |
| `SBOM_FILE` | `sbom.json` in the project root. |
| `PROJECT_NAME` | Manifest name, module name, repository name, or directory name. |
| `PROJECT_VERSION` | Manifest version, latest Git tag, or fallback `0.0.0-dev`. |
| `TRIVY_INCLUDE_VULN` | `true` for this vulnerability-check workflow. |
| `TRIVY_SKIP_DIRS` | Auto-detected generated/heavy directories. |

Do not require the user to maintain `PROJECT_NAME`, `PROJECT_VERSION`, `SBOM_FILE`, `TARGET_PATH`, `TRIVY_INCLUDE_VULN`, or `TRIVY_SKIP_DIRS` unless automatic inference is wrong.

## Metadata inference

Infer before asking the user.

Preferred order:

1. Node/TypeScript: `package.json` `name`, `version`.
2. Python: `pyproject.toml` `[project]` or Poetry metadata.
3. Java Maven: root `pom.xml` `artifactId`, `version`.
4. Java Gradle: `settings.gradle*` `rootProject.name`, `build.gradle*` `version`.
5. Rust: `Cargo.toml` `[package]` `name`, `version`.
6. Go: `go.mod` module basename for name; Git tag or explicit setting for version.
7. Fallback name: Git repository or directory basename.
8. Fallback version: latest Git tag; otherwise `0.0.0-dev` and report the fallback.

Never infer the project version from dependency versions in lockfiles.

## SBOM file policy

Use `<TARGET_PATH>/sbom.json` unless overridden. Ensure it is not committed.

Recommended `.gitignore` entry:

```gitignore
# Generated SBOM
sbom.json
```

Do not commit `sbom.json` unless the user explicitly wants SBOM artifacts versioned.

## Trivy policy

Default command shape:

```bash
trivy fs --format cyclonedx --scanners vuln --output sbom.json <TARGET_PATH>
```

Use `--scanners vuln` when `TRIVY_INCLUDE_VULN` is true. Set it false only when the user wants a pure SBOM and Dependency-Track as the only vulnerability analyzer.

Auto-generate `TRIVY_SKIP_DIRS` from directories that actually exist under `TARGET_PATH`. Common candidates:

```text
.git node_modules .next .nuxt dist build coverage .turbo out .cache
target .gradle .mvn/wrapper __pycache__ .pytest_cache .mypy_cache
.venv venv .tox vendor/bundle
```

Rules:

- Skip generated/heavy directories only.
- Never skip manifests or lockfiles such as `package.json`, lockfiles, `pom.xml`, `build.gradle`, `pyproject.toml`, `requirements*.txt`, `Cargo.toml`, `Cargo.lock`, `go.mod`, or `go.sum`.
- Pass each skip item as a separate `--skip-dirs <path>` argument.

## Upload policy

Upload with Dependency-Track BOM API:

```bash
curl -fsS -X POST "${DEPENDENCY_TRACK_URL%/}/api/v1/bom"   -H "X-Api-Key: ${BOM_UPLOAD_API_KEY}"   -F "autoCreate=true"   -F "projectName=${PROJECT_NAME}"   -F "projectVersion=${PROJECT_VERSION}"   -F "bom=@${SBOM_FILE};type=application/json"
```

When `PARENT_UUID` is set, add:

```bash
-F "parentUUID=${PARENT_UUID}"
```

Behavior:

- `autoCreate=true` creates the project/version if missing.
- `PARENT_UUID` present: project is created/updated below that parent.
- `PARENT_UUID` absent: project is created/updated at Dependency-Track root level.
- A successful upload only means the BOM was accepted. Dependency-Track analysis may continue asynchronously.

## Agent procedure

1. Resolve `TARGET_PATH` as the project root.
2. Read `DEPENDENCY_TRACK_URL`, `BOM_UPLOAD_API_KEY`, and optional `PARENT_UUID` from agent-managed settings/secrets.
3. Infer `PROJECT_NAME` and `PROJECT_VERSION` from the project.
4. Set `SBOM_FILE=sbom.json` unless overridden.
5. Ensure `sbom.json` is ignored by Git when modifying the repository is allowed.
6. Build Trivy options, including automatic skip directories.
7. Run Trivy and verify that `sbom.json` exists.
8. Upload the SBOM to Dependency-Track using multipart form upload.
9. Report project name/version, target path, parent usage, scanner mode, skip dirs, HTTP result, and processing token if returned.

## Failure handling

| Failure | Agent behavior |
|---|---|
| Missing Dependency-Track URL/API key | Ask the user to save it in the agent's config/secrets, such as Claude Code `/config` or the equivalent settings store. |
| Missing project name | Use manifest/module/repository/directory fallback and report it. |
| Missing project version | Use manifest/Git tag; otherwise `0.0.0-dev` and report it. |
| Trivy not installed | Stop and ask for Trivy in the agent runtime. |
| HTTP 401/403 | API key is invalid or lacks BOM upload / project auto-create permission. |
| HTTP 404 | Check Dependency-Track API base URL and port. |
| HTTP 413 | SBOM payload is too large; reduce scan scope or adjust server/proxy limits. |
| HTTP 5xx | Treat as Dependency-Track server failure and report the non-secret response summary. |

## Do not do

- Do not store secrets in Git.
- Do not print `BOM_UPLOAD_API_KEY`.
- Do not claim vulnerability analysis is complete immediately after upload acceptance.

