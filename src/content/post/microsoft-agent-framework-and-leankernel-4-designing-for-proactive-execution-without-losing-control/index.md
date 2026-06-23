---
author: "Alfero Chingono"
title: "Microsoft Agent Framework and LeanKernel (Part 4): Designing for Proactive Execution Without Losing Control"
date: 2026-06-24T09:00:00Z
draft: true
description: "How LeanKernel approaches proactive execution: scheduled jobs, authentication gates, tool caps, and the single-request default."
slug: microsoft-agent-framework-and-leankernel-4-designing-for-proactive-execution-without-losing-control
tags: [
"LeanKernel",
"Microsoft Agent Framework",
"Proactive Execution"
]
categories: [
"Architecture",
"Agentic AI"
]
image: ""
---

This is a continuation of [Part 3](/blog/2026/06/23/microsoft-agent-framework-and-leankernel-3-the-runtime-contracts-that-made-multi-agent-handoffs-reliable).

The most instructive failure in proactive execution was not about scheduling or workflows. It was about tool registration.

## The 128-tool limit

LeanKernel registers agent capabilities as tools — function definitions the model can invoke. At deployment, it registered too many. The error surfaced as a 500 from the model provider:

```text
Error: Invalid 'tools': array too long. Expected an array with maximum length 128
```

The stack trace led to `POST /chat/completions` with a `tools` payload of 129 entries. Azure's API has a hard limit of 128. I added `MaxTools` to the config to cap registration and a fallback strategy for the overflow:

```csharp
// LeanKernel.Core/Configuration/LiteLLMConfiguration.cs
public class LiteLLMConfiguration
{
    public int MaxTools { get; set; } = 128;
}
```

```yaml
# docker-stack.yml
- LEANKERNEL__LITELLM__MAXTOOLS=128
```

The fallback selects the most relevant tools when the count exceeds the limit. If there is no clear selection, the agent has a fallback economy model route where it tells the caller which capabilities are configured. The failure mode changed from silent 500 to explicit capability negotiation.

## Scheduled execution with Ofelia

Scheduled jobs run through Ofelia labels on the gbrain container. The pattern is documented in the [scheduled jobs post](/blog/2026/06/26/leankernel-scheduled-jobs-on-swarm-ofelia-gbrain-and-the-gotchas-that-mattered/), but the key design point: jobs run in the same container as the runtime, with the same secrets and volume mounts. That cut the "works in dev but not on schedule" class of bugs.

## The human gate

The auth integration with oauth2-proxy ensures that external endpoints require valid tokens. The `.docker-stack.yml` exposes port 5080 through oauth2-proxy and the `LEANKERNEL__AUTH__HEADER` maps the user identity into downstream headers. This means no external tool invocation happens without a validated identity, even through the auth guard.

```yaml
services:
  oauth2-proxy:
    image: quay.io/oauth2-proxy/oauth2-proxy:latest
    ports:
      - "443:443"
    environment:
      - OAUTH2_PROXY_UPSTREAMS=http://engine:5080
      - OAUTH2_PROXY_PASS_AUTHORIZATION_HEADER=false
```

## The default behavior

The simplest proactive execution policy in LeanKernel: agents are given context about the current request and do not trigger side effects unless explicitly configured. No agent can autonomously produce outbound effects without clearance through `Commander`. One request, one completion, one response. Everything beyond that requires an explicit schedule, a configured tool, or an authenticated session.

---

Related reading:

- [LeanKernel scheduled jobs on Swarm](/blog/2026/06/26/leankernel-scheduled-jobs-on-swarm-ofelia-gbrain-and-the-gotchas-that-mattered/)
- [LeanKernel deployment history](/blog/2026/06/25/leankernel-swarm-deployment-commit-history-what-broke-and-how-i-fixed-it/)

Series navigation:

Part 1: [The Problem Was Never Just Prompts](/blog/2026/06/21/microsoft-agent-framework-and-leankernel-1-the-problem-was-never-just-prompts/).
Part 2: [Why a Modular Monolith Was the Right First Bet for LeanKernel](/blog/2026/06/22/microsoft-agent-framework-and-leankernel-2-why-a-modular-monolith-was-the-right-first-bet/).
Part 3: [The Runtime Contracts That Made Multi-Agent Handoffs Reliable](/blog/2026/06/23/microsoft-agent-framework-and-leankernel-3-the-runtime-contracts-that-made-multi-agent-handoffs-reliable/).
