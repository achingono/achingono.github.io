---
author: "Alfero Chingono"
title: "LeanKernel on Swarm: What Broke in Deployment, and How We Fixed It"
date: 2026-06-25T09:00:00Z
draft: false
description: "A commit-history walkthrough of LeanKernel deployment on Docker Swarm: the real failure modes, and the architecture/script changes that resolved them."
slug: leankernel-swarm-deployment-commit-history-what-broke-and-how-we-fixed-it
tags: [
"LeanKernel",
"Docker Swarm",
"Deployment",
"Platform Engineering",
"Postmortem"
]
categories: [
"Operations",
"Architecture",
"Build in Public"
]
image: ""
---

I went back through the `swarm` repo commit history to answer a practical question:

What deployment problems did we actually hit while rolling out LeanKernel, and what decisions fixed them?

This is not a polished architecture diagram version. It is the real sequence.

## 1) First deployment worked, but operational assumptions were too manual

The initial LeanKernel stack landed in `051748d` with the right core shape: engine, oauth2-proxy, gbrain, and deployment/verification scripts.

What it did not yet solve was operational drift. The scripts and runtime assumptions were still too dependent on careful human steps.

That is normal for a first cut. But that drift showed up quickly as the stack evolved.

## 2) Secret handling and deploy script drift became a scaling tax

A cluster of commits tells this story:

- `d139d34`: removed brittle remote file-upload/dir bootstrap behavior from deploy scripts
- `a7a1248`: stopped pre-creating secrets manually and let `docker stack deploy` own secret creation; added empty-secret warnings
- `960b155`: refactored per-stack deploy scripts onto shared option parsing + common build flow

The challenge was not one dramatic outage. It was repeatability.

When each stack script handles arguments, secret prep, and deployment logic slightly differently, incidents become hard to debug because behavior depends on which script path you took.

The resolution was deliberate standardization: one common deployment interface, earlier validation, and fewer bespoke code paths.

## 3) Browser automation architecture changed because the original plan was too coupled

`867e8c7` introduced Webwright into the LeanKernel stack and updated entrypoints/config.

This aligned with a key deviation documented in deployment docs: split browser capability into two layers instead of a single monolithic browser service.

- shared Playwright run-server in platform
- stack-scoped `webwright` sidecar in LeanKernel

The challenge here was control boundaries.

A single shared browser API looked simpler on paper, but it mixed transport/runtime concerns with app-specific guardrails and queue behavior.

The resolution was cleaner responsibility boundaries. Shared capability stayed shared; policy stayed app-local.

## 4) Skills packaging and path conventions caused avoidable friction

`de088c2` moved CLI binaries from `tools/bin` to `skills/bin`, then updated deploy sync logic to copy the full `skills/` tree.

Why this mattered:

- dynamic skills definitions and binaries drift when you sync them differently
- partial copy logic creates mismatches between what SKILL.md expects and what PATH resolves

The resolution was a single deploy contract for skills: definitions + binaries move together, then the runtime PATH points to one predictable directory.

## 5) Tool count limits surfaced as a deployment/runtime config issue

`fe8abab` fixed a concrete production failure: tool registration exceeded Azure's 128-tool limit.

The stack-side piece of the fix was explicit cap propagation (`LEANKERNEL__LITELLM__MAXTOOLS`) through deploy/env configuration.

The challenge was subtle: the user-facing error looked like model-provider reachability, but the root cause was tool payload size.

The resolution combined platform and app thinking: tighten deployment config, then adjust tool strategy in runtime behavior.

## What this commit trail changed in practice

The strongest pattern in this history is simple.

Most deployment failures were not caused by Docker Swarm itself. They came from ambiguous contracts:

- who owns secret lifecycle
- where config defaults are enforced
- how skills are packaged and resolved
- which layer owns browser policy

Once those contracts became explicit in scripts and stack config, LeanKernel deployments became much more predictable.

That is the part worth reusing: operational clarity beats clever shell glue every time.

---

Related commits reviewed:

- `051748d` initial LeanKernel stack deployment scaffolding
- `d139d34`, `a7a1248`, `960b155` deployment script hardening and standardization
- `867e8c7` Webwright integration + architecture shift
- `de088c2` tools-to-skills packaging consolidation
- `fe8abab` max-tools config propagation fix

Related reading:

- [LeanKernel Stack docs in swarm repo](https://github.com/achingono/swarm/tree/main/docs/deployment/stacks/leankernel)
- [Microsoft Agent Framework and LeanKernel (Part 4): Designing for Proactive Execution Without Losing Control](/blog/2026/06/24/microsoft-agent-framework-and-leankernel-4-designing-for-proactive-execution-without-losing-control/)
