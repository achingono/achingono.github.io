---
author: "Alfero Chingono"
title: "LeanKernel Scheduled Jobs on Swarm: Ofelia, GBrain, and the Gotchas That Mattered"
date: 2026-06-26T09:00:00Z
draft: false
description: "How LeanKernel scheduled jobs evolved on Docker Swarm: from idle containers to gbrain consolidation, plus the specific runtime gotchas uncovered along the way."
slug: leankernel-scheduled-jobs-on-swarm-ofelia-gbrain-and-the-gotchas-that-mattered
tags: [
"LeanKernel",
"Docker Swarm",
"Ofelia",
"Scheduled Jobs",
"GBrain"
]
categories: [
"Operations",
"Agentic AI",
"Build in Public"
]
image: ""
---

Scheduled jobs looked straightforward in LeanKernel until we tried to run them reliably in Swarm.

The commit trail around `97c3e1c` and `00d06c0` shows the actual story: design, failure modes, and consolidation decisions.

## What we were trying to automate

Two core recurring workflows:

- Microsoft To Do attention digest generation
- GBrain wiki sync + embedding refresh

Later, a third workflow was added:

- deterministic identity self-heal ingest (`00d06c0`)

All of these jobs needed the same things: secrets, gbrain CLI access, bun runtime, and wiki volume access.

## Phase 1 worked, but cost too much complexity

The first implementation (captured in `97c3e1c`) used:

- an `ofelia` scheduler service
- two separate idle job containers (`sleep 3153600000`)
- `job-exec` labels targeting those idle containers

It functioned, but it duplicated runtime surface area:

- extra always-on containers
- duplicated mounts and secret wiring
- more places for drift between job runtime and actual gbrain runtime

The design was correct enough to prove the behavior, but too expensive to operate long-term.

## Consolidation into gbrain was the right move

The key architectural shift was to execute scheduled jobs directly inside the existing `gbrain` container.

That reduced moving parts and aligned the execution context with where the dependencies already lived.

In practice, it gave us:

- fewer services to deploy and monitor
- fewer duplicated mounts/secrets
- tighter parity between interactive gbrain behavior and scheduled-job behavior

This is one of those cases where "fewer containers" was not about cost-cutting. It was about reducing failure surface.

## The gotcha that can silently break scheduling

One important lesson from this migration:

Ofelia in Docker mode discovers jobs from **container labels**, not Swarm service metadata.

If labels are put under `deploy.labels`, job discovery fails quietly because those are service labels. The fix was placing labels at top-level `labels:` for the target service.

That one detail explains a lot of "scheduler is up but jobs never run" behavior.

## The second gotcha: init idempotency + strict shell mode

`gbrain init` is meant to be safe to run repeatedly, but operational notes call out real cases where non-zero exits can happen during re-deploy.

With `set -eu` entrypoints, a non-zero init can prevent the service from reaching `gbrain serve`.

The mitigation pattern became:

- keep job scripts explicit about secret loading and environment setup
- make initialization expectations visible in logs
- verify post-deploy health and scheduler registration, not just container start

## Why deterministic ingest mattered for scheduled jobs

`00d06c0` added deterministic identity self-heal ingest.

This extended the scheduled-job model beyond "sync whatever changed" into "guarantee specific identity artifacts exist and are embedded."

That is an important evolution for agent systems. Some periodic jobs are maintenance. Others are control-plane correctness guarantees.

Treating them the same is how drift sneaks in.

## Practical verification loop we kept

After deployment, the reliable checks were:

- stack service list has expected LeanKernel services
- gbrain container exposes expected Ofelia labels
- Ofelia logs show job registration
- forced manual execution of job scripts succeeds in-container

The rule became: do not trust scheduler "up" status alone. Trust registration + execution.

## Final takeaway

The scheduled-job architecture became better when we optimized for execution context clarity, not abstract modularity.

If jobs require gbrain runtime, run them in gbrain.
If scheduler discovery depends on container labels, wire labels exactly there.
If periodic workflows maintain core behavior, validate them like production code paths.

That is what made this setup durable.

---

Related commits reviewed:

- `97c3e1c` Ofelia scheduled jobs + consolidation into gbrain
- `00d06c0` deterministic identity self-heal ingest

Related reading:

- [LeanKernel operational notes (Swarm docs)](https://github.com/achingono/swarm/blob/main/docs/deployment/stacks/leankernel/operational-notes.md)
- [LeanKernel deployment deviations (Swarm docs)](https://github.com/achingono/swarm/blob/main/docs/deployment/stacks/leankernel/deviations-from-plan.md)
