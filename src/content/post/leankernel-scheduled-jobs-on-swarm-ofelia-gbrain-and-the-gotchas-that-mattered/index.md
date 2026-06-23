---
author: "Alfero Chingono"
title: "LeanKernel Scheduled Jobs on Swarm"
date: 2026-06-26T09:00:00Z
draft: true
description: "Ofelia scheduled jobs on Docker Swarm: idle containers, gbrain consolidation, and the container label gotcha."
slug: leankernel-scheduled-jobs-on-swarm-ofelia-gbrain-and-the-gotchas-that-mattered
tags: [
"LeanKernel",
"Docker Swarm",
"Ofelia"
]
categories: [
"Operations",
"Build in Public"
]
image: ""
---

This is a continuation of the [deployment commit history post](/blog/2026/06/25/leankernel-swarm-deployment-commit-history-what-broke-and-how-i-fixed-it).

I needed two recurring workflows: a Microsoft To Do attention digest and a GBrain wiki sync with embedding refresh. Later I added deterministic identity self-heal ingest. All of them needed secrets, the gbrain CLI, the bun runtime, and wiki volume access.

Here's the first implementation I tried.

## Phase 1: Separate idle containers

I deployed an `ofelia` scheduler service and two idle containers. Each idle container ran `sleep 3153600000` (roughly 100 years) and carried Ofelia `job-exec` labels. When Ofelia's schedule fired, it would `docker exec` into the idle container to run the job script.

```yaml
  ms_todo_attention:
    image: ghcr.io/achingono/leankernel-gbrain:latest
    command: ["sleep", "3153600000"]
    deploy:
      labels:
        ofelia.job-exec.ms-todo-attention.schedule: "0 6 * * 1-5"
        ofelia.job-exec.ms-todo-attention.command: "/run/ms-todo-attention-job.sh"
```

It worked. But it duplicated runtime surface: two extra containers perpetually sleeping, with duplicated mounts and secret wiring. Every redeploy meant touching configuration in three places.

## Consolidation into gbrain

I stopped trying to mimic the gbrain runtime in separate containers and moved scheduled job execution into the existing `gbrain` container instead. The diff was straightforward:

```diff
   gbrain:
     image: ghcr.io/achingono/leankernel-gbrain:latest
+    labels:
+      ofelia.enabled: "true"
+      ofelia.job-exec.gbrain-sync-embed.schedule: "*/15 * * * *"
+      ofelia.job-exec.gbrain-sync-embed.command: "/run/gbrain-sync-embed-job.sh"
+      ofelia.job-exec.ms-todo-attention.schedule: "0 6 * * 1-5"
+      ofelia.job-exec.ms-todo-attention.command: "/run/ms-todo-attention-job.sh"
+    configs:
+      - source: gbrain_sync_embed_job
+        target: /run/gbrain-sync-embed-job.sh
+      - source: ms_todo_attention_job
+        target: /run/ms-todo-attention-job.sh
```

This cut the moving parts. The bigger win was parity: scheduled runs used the same runtime assumptions as interactive gbrain, so I stopped seeing "it works when I run it by hand" mysteries.

## Gotcha 1: Ofelia labels must be container-level, not service-level

During the migration I hit a silent failure. The scheduler was up, logs showed no errors, but jobs never ran.

Ofelia's `daemon --docker` mode discovers jobs by reading labels on **running containers** via the Docker API. In Docker Compose, labels under `deploy.labels` are service-level metadata and do not propagate to the container. They need to be at the top-level `labels:` block:

```yaml
# This will NOT work — labels are on the service, not the container
services:
  gbrain:
    deploy:
      labels:
        ofelia.job-exec.gbrain-sync-embed.schedule: "*/15 * * * *"

# This works — labels go on the container
services:
  gbrain:
    labels:
      ofelia.job-exec.gbrain-sync-embed.schedule: "*/15 * * * *"
```

That one detail explains a lot of "scheduler is up but jobs never run" debugging sessions.

## Gotcha 2: `gbrain init` idempotency and `set -eu`

`gbrain init` is designed to be safe to run repeatedly, but I still saw non-zero exits during redeploys. The problem was `set -eu` in the entrypoint — any non-zero exit from init would stop the service before it reached `gbrain serve`.

The fix was keeping job scripts explicit about secret loading and environment setup, and making initialization expectations visible in logs. After that I stopped treating "container started" as the end of the job. Deployment completion meant health checks plus scheduler registration, not just the process coming up.

## Why deterministic ingest mattered

The identity self-heal job writes two specific wiki pages: a routing rules page that tells agents which knowledge paths to follow, and a daily attention digest from Microsoft To Do. I learned this mattered when a routing rule page went missing during a redeploy and agents started answering "I don't have access to that information" for requests they normally handled.

The self-heal job made those artifacts idempotent: every run produces the same pages from the same state, so a missing page gets restored on the next cycle.

## Verification loop

After deployment, I trusted three checks:

- stack service list has expected LeanKernel services
- gbrain container exposes expected Ofelia labels
- Ofelia logs show job registration

Then I ran the job scripts manually inside the container with the real secrets and real wiki mounts to validate behavior before the next scheduled run.

---

Related reading:

- [LeanKernel operational notes (Swarm docs)](https://github.com/achingono/swarm/blob/main/docs/deployment/stacks/leankernel/operational-notes.md)
- [Ofelia scheduler docs](https://github.com/mcuadros/ofelia)
