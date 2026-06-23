---
author: "Alfero Chingono"
title: "LeanKernel on Swarm: What Broke in Deployment"
date: 2026-06-25T09:00:00Z
draft: true
description: "A walk through the git history of LeanKernel deployment on Docker Swarm: what broke, and what I changed to fix it."
slug: leankernel-swarm-deployment-commit-history-what-broke-and-how-i-fixed-it
tags: [
"LeanKernel",
"Docker Swarm",
"Deployment"
]
categories: [
"Operations",
"Build in Public"
]
image: ""
---

I went back through the `swarm` repo commit history to find the answer to a practical question: what deployment problems did I actually hit while rolling out LeanKernel, and what did I change to fix them?

Here is the sequence as it played out.

## 1) First deployment worked, but the scripts were too manual

The initial stack landed in `deploy/leankernel/` with the right core shape — engine, oauth2-proxy, gbrain, deployment scripts, and a verify script. What it did not solve was operational drift. The scripts depended on careful human steps: creating secrets by hand, uploading files to remote hosts, running commands in the right order.

```yaml
# docker-stack.yml — initial version
services:
  engine:
    image: ghcr.io/achingono/leankernel-engine:latest
    ports:
      - "5080:5080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Swarm
      - LEANKERNEL__GBRAIN__URL=http://gbrain:8789
```

That drift showed up quickly as the stack evolved.

## 2) Secret handling and deploy script drift

A cluster of commits tells this story. I touched three failure points:

- removed brittle remote file-upload and directory bootstrap behavior (`d139d34`)
- stopped pre-creating secrets manually and let `docker stack deploy` own secret creation (`a7a1248`), adding empty-secret warnings since a missing secret file would fail silently
- refactored per-stack deploy scripts onto shared option parsing and a common build flow (`960b155`)

The real problem was repeatability, not any single outage. Before the refactor, each stack script handled arguments, secret prep, and deployment logic differently. Debugging meant tracing through which script path was taken. Standardizing to one common interface in `deploy/scripts/build.sh` fixed that:

```bash
# deploy/scripts/build.sh — shared build entrypoint
Usage: build.sh <stack-name> [options]
  -b, --build     Build images before deploying
  -d, --deploy    Deploy the stack after building
```

## 3) Browser automation: from monolith to sidecar

The original plan was a single monolithic browser service. Commit `867e8c7` replaced that with two layers: a shared Playwright run-server in the platform stack and a `webwright` sidecar scoped to LeanKernel.

```yaml
# platform stack — shared Playwright runtime
  playwright:
    image: ghcr.io/achingono/playwright:latest
    deploy:
      placement:
        constraints: [node.labels.data == true]

# leankernel stack — scoped webwright sidecar
  webwright:
    image: ghcr.io/achingono/webwright:latest
    environment:
      - PLAYWRIGHT_URL=http://playwright:3000
```

The monolith looked simpler on paper but mixed transport concerns with app-specific guardrails. Splitting them meant the shared capability stayed shared, and the policy stayed app-local.

## 4) Skills packaging and path conventions

CLI binaries were originally in `tools/bin`, but `de088c2` moved them to `skills/bin` and updated the deploy sync to copy the full `skills/` directory. Before that, dynamic skill definitions and binaries drifted when synced differently — partial copy logic created mismatches between what `SKILL.md` expected and what `PATH` resolved.

```diff
-# build-tools.sh — old output path
-OUTPUT_DIR="tools/bin"
+# build-tools.sh — new output path
+OUTPUT_DIR="skills/bin"
```

The fix was a single deploy contract: definitions and binaries move together, and the runtime `PATH` points to one predictable directory.

## 5) Tool count limits as a deployment issue

This one looked like a model provider failure but was a deployment config gap. Tool registration exceeded Azure's 128-tool limit and surfaced as "cannot reach the configured model provider" — a 500 with no indication of the real cause.

```text
Error: Invalid 'tools': array too long. Expected an array with maximum length 128
```

The stack-side fix added `LEANKERNEL__LITELLM__MAXTOOLS` to the deploy config:

```yaml
services:
  engine:
    environment:
      - LEANKERNEL__LITELLM__MAXTOOLS=128
```

With an economy-model fallback to select relevant tools when the count exceeds the limit.

---

The pattern across all of these: deployment failures came from ambiguous contracts around who owns secret lifecycle, where config defaults are enforced, how skills are packaged, and which layer owns browser policy. Once those contracts became explicit in scripts and stack config, deployments became predictable.

Related reading:

- [LeanKernel Stack docs in swarm repo](https://github.com/achingono/swarm/tree/main/docs/deployment/stacks/leankernel)
