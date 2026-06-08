---
title: Static Mockup Preview Server
description: Temporary static server guide for previewing web-artifacts-builder mockups from outout/{mockup} using npx serve or Python http.server.
published: true
date: 2026-05-25T00:00:00.000Z
tags: mockup, static-server, web-artifacts-builder, npx-serve, python-http-server
editor: markdown
dateCreated: 2026-05-25T00:00:00.000Z
---


# Static Mockup Preview Server

## Purpose

Use this skill when you need to briefly preview static mockup output from `web-artifacts-builder`.

The expected mockup path is:

```text
outout/{mockup}
```

Replace `{mockup}` with the actual mockup directory name.

Example:

```text
outout/login-page
outout/dashboard-demo
```

> Note: If your project uses `output/{mockup}` instead of `outout/{mockup}`, adjust the path accordingly.

---

## Recommended Option: `npx serve`

Use this when Node.js and npm are available.

### Serve from the project root

```bash
npx serve "outout/{mockup}" -l 4173
```

Open:

```text
http://localhost:4173
```

### Serve after changing into the mockup directory

```bash
cd "outout/{mockup}"
npx serve . -l 4173
```

### SPA fallback mode

Use this if the mockup uses client-side routing and direct page refreshes return 404.

```bash
npx serve "outout/{mockup}" -s -l 4173
```

### LAN access

Use this only when another device on the same network needs to view the mockup.

```bash
npx serve "outout/{mockup}" -l tcp://0.0.0.0:4173
```

---

## Simple Option: Python `http.server`

Use this when Python 3 is available and you want the simplest possible temporary server.

### Serve from the project root

```bash
python3 -m http.server 4173 --directory "outout/{mockup}"
```

Open:

```text
http://localhost:4173
```

### Serve after changing into the mockup directory

```bash
cd "outout/{mockup}"
python3 -m http.server 4173
```

### LAN access

Python's `http.server` usually binds to all interfaces by default. To be explicit:

```bash
python3 -m http.server 4173 --bind 0.0.0.0 --directory "outout/{mockup}"
```

---

## Quick Decision Guide

Use `npx serve` when:

- Node.js/npm is already installed.
- You want behavior closer to a static hosting server.
- You need SPA fallback with `-s`.

Use `python3 -m http.server` when:

- Python 3 is already installed.
- You only need to preview plain static files.
- You want zero npm dependency or package download.

---

## Cleanup

Stop the temporary server with:

```bash
Ctrl + C
```

---

## Recommended Default

For most mockup checks:

```bash
npx serve "outout/{mockup}" -s -l 4173
```

Fallback when Node.js/npm is not available:

```bash
python3 -m http.server 4173 --directory "outout/{mockup}"
```

