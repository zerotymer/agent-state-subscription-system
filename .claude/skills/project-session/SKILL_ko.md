---
name: project-session
description: Use when starting or tracking a coding agent session in a project; creates or updates project-local .codex/session and .codex/session.log (or .claude/session and .claude/session.log) files.
---

# Project Session Tracking Skill
프로젝트 세션 추적 규칙

## 목적

모든 코딩 에이전트 세션은 프로젝트 로컬 세션 기록을 남겨야 한다.

세션 기록은 각 에이전트가 현재 프로젝트에서 어떤 작업을 했는지 확인하기 위한 용도이다. 이 파일들은 로컬 실행 상태이므로 Git에 커밋하면 안 된다.

이 규칙은 다음에 적용된다.

- Codex / OpenAI 코딩 에이전트
- Claude Code
- 이 지침 파일을 따르는 기타 호환 코딩 에이전트

## 프로젝트 로컬 규칙

모든 세션 파일은 현재 프로젝트 루트를 기준으로 생성해야 한다.

사용자의 홈 디렉터리, 전역 설정 디렉터리, 다른 공유 위치에 작성하면 안 된다.

각 프로젝트는 독립적인 세션 기록을 가져야 한다.

## 세션 파일

| 에이전트 | 현재 세션 파일 | 세션 로그 파일 |
| --- | --- | --- |
| Codex | `.codex/session` | `.codex/session.log` |
| Claude Code | `.claude/session` | `.claude/session.log` |

## 세션 시작 시 필수 동작

새 에이전트 세션이 시작되면 에이전트는 다음을 수행해야 한다.

1. 프로젝트 루트를 감지한다.
2. 실행 중인 에이전트를 식별한다.
3. 필요한 에이전트 디렉터리가 없으면 생성한다.
4. 현재 세션 파일을 생성하거나 업데이트한다.
5. 현재 세션에 세션 ID가 없으면 새로 생성한다.
6. 사용자의 최초 요청에서 짧은 세션 주제를 도출한다.
7. 해당 에이전트의 세션 로그에 다음 형식으로 한 줄을 추가하거나 업데이트한다.

```text
<session-id> - <session-topic>
```
