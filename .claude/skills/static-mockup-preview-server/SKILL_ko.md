---
title: 정적 목업 확인용 서버
description: web-artifacts-builder 목업 결과물을 outout/{mockup} 경로에서 npx serve 또는 Python http.server로 잠깐 확인하는 방법.
published: true
date: 2026-05-25T00:00:00.000Z
tags: mockup, static-server, web-artifacts-builder, npx-serve, python-http-server
editor: markdown
dateCreated: 2026-05-25T00:00:00.000Z
---

# 정적 목업 확인용 서버

## 목적

`web-artifacts-builder`로 생성한 정적 목업 결과물을 잠깐 확인할 때 사용합니다.

기본 목업 경로는 다음을 기준으로 합니다.

```text
outout/{mockup}
```

`{mockup}`에는 실제 목업 디렉터리 이름을 넣습니다.

예시:

```text
outout/login-page
outout/dashboard-demo
```

> 참고: 프로젝트에서 `outout/{mockup}`이 아니라 `output/{mockup}` 경로를 사용한다면 그에 맞게 바꿔서 실행하세요.

---

## 추천 옵션: `npx serve`

Node.js와 npm이 설치되어 있을 때 사용합니다.

### 프로젝트 루트에서 실행

```bash
npx serve "outout/{mockup}" -l 4173
```

브라우저에서 엽니다.

```text
http://localhost:4173
```

### 목업 디렉터리로 이동 후 실행

```bash
cd "outout/{mockup}"
npx serve . -l 4173
```

### SPA fallback 모드

클라이언트 라우팅을 사용하는 목업에서 새로고침하거나 직접 경로 접근 시 404가 나면 사용합니다.

```bash
npx serve "outout/{mockup}" -s -l 4173
```

### 같은 LAN의 다른 기기에서 확인

같은 네트워크의 다른 기기에서 목업을 확인해야 할 때만 사용합니다.

```bash
npx serve "outout/{mockup}" -l tcp://0.0.0.0:4173
```

---

## 간단 옵션: Python `http.server`

Python 3만 설치되어 있고, 가장 단순하게 임시 서버를 띄우고 싶을 때 사용합니다.

### 프로젝트 루트에서 실행

```bash
python3 -m http.server 4173 --directory "outout/{mockup}"
```

브라우저에서 엽니다.

```text
http://localhost:4173
```

### 목업 디렉터리로 이동 후 실행

```bash
cd "outout/{mockup}"
python3 -m http.server 4173
```

### 같은 LAN의 다른 기기에서 확인

Python `http.server`는 보통 모든 인터페이스에 바인딩됩니다. 명시적으로 지정하려면 다음처럼 실행합니다.

```bash
python3 -m http.server 4173 --bind 0.0.0.0 --directory "outout/{mockup}"
```

---

## 선택 기준

`npx serve`를 쓰면 좋은 경우:

- Node.js/npm이 이미 설치되어 있음.
- 정적 호스팅 서버에 가까운 동작을 원함.
- SPA fallback을 `-s` 옵션으로 처리해야 함.

`python3 -m http.server`를 쓰면 좋은 경우:

- Python 3가 이미 설치되어 있음.
- 단순 정적 파일만 확인하면 됨.
- npm 패키지 다운로드 없이 바로 실행하고 싶음.

---

## 종료 방법

임시 서버는 다음 키로 종료합니다.

```bash
Ctrl + C
```

---

## 기본 추천 명령

대부분의 목업 확인에는 다음을 사용합니다.

```bash
npx serve "outout/{mockup}" -s -l 4173
```

Node.js/npm이 없을 때는 다음을 사용합니다.

```bash
python3 -m http.server 4173 --directory "outout/{mockup}"
```

