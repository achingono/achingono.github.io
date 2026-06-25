---
author: "Alfero Chingono"
title: "Microsoft Agent Framework and LeanKernel (Part 2): Why a Modular Monolith Was the Right First Bet"
date: 2026-07-09T09:00:00Z
draft: true
description: "Why LeanKernel started with strict module boundaries inside one deployable host instead of microservices."
slug: microsoft-agent-framework-and-leankernel-2-why-a-modular-monolith-was-the-right-first-bet
tags: [
"LeanKernel",
"Microsoft Agent Framework",
"Modular Monolith"
]
categories: [
"Architecture",
"Agentic AI"
]
image: ""
---

This is a continuation of [Part 1](/blog/2026/06/21/microsoft-agent-framework-and-leankernel-1-the-problem-was-never-just-prompts).

When people hear "agent platform," they expect a microservices map on day one. I made a different choice with LeanKernel: a modular monolith in .NET, with strict project boundaries and composition in one gateway host.

Here is the solution structure that came out of that choice:

```text
LeanKernel.sln
├── LeanKernel.Abstractions   # shared contracts
├── LeanKernel.Core           # primitives
├── LeanKernel.Agents         # agent orchestration
├── LeanKernel.Thinker        # reasoning pipeline
├── LeanKernel.Context        # prompt assembly
├── LeanKernel.Tools          # tool definitions
├── LeanKernel.Plugins        # dynamic skill loading
├── LeanKernel.Persistence    # state durability
├── LeanKernel.Archivist      # knowledge management
├── LeanKernel.Channels       # ingress/communication
├── LeanKernel.Commander      # egress/execution
└── LeanKernel.Gateway        # host + composition
```

Modules own behavior. The gateway composes them. The project count matters less than the responsibility map.

## Why not microservices

I learned the cost of wrong boundaries the hard way with the browser automation service. The original implementation was a single monolithic service: one `BrowserServiceConfig`, one client interface, one health probe, one Docker image. It looked simpler on paper but mixed transport concerns with app-specific guardrails.

```csharp
// Before — monolithic browser service
services.AddSingleton<IBrowserServiceClient, BrowserServiceClient>();
services.AddHealthChecks().AddCheck<BrowserServiceHealthProbe>("browser");
```

When I needed to add queue behavior and per-agent policies, the boundary wasn't right. I had to refactor it into a split architecture: a shared Playwright runtime in the platform stack and a `Webwright` sidecar scoped to LeanKernel. The diff tells the story:

```diff
-config/browser-service/
+config/webwright/
  Dockerfile
  app/main.py
```

That refactor taught me more about boundary design than any upfront planning. If I had started with microservices, I would have been managing that infrastructure tax while still figuring out which seams were stable.

## How this works with Microsoft Agent Framework

Framework orchestration patterns map into `Thinker` and `Agents`. External tool invocation maps into `Tools` and plugin surfaces. Context shaping maps into `Context` and persistence modules. Once those seams harden, distribution becomes a deployment decision instead of a design gamble.

## What I accepted

I accepted less independent runtime scaling in the short term and tighter process-level coupling while domains matured. I also needed more discipline to prevent host-layer leakage. In return, I got faster feedback on architectural changes and clearer refactoring paths because module contracts stayed explicit.

## What I would tell teams

Don't ask "monolith or microservices" as a branding choice. Ask whether your boundaries are stable enough to distribute safely, and whether you understand the cross-boundary contracts under load — including what happens when one side deploys and the other does not. If those answers are still emerging, keep the boundary map inside one deployable unit until it stops changing week to week.

---

Series navigation:

Part 1: [The Problem Was Never Just Prompts](/blog/2026/06/21/microsoft-agent-framework-and-leankernel-1-the-problem-was-never-just-prompts/).
Part 3: [The Runtime Contracts That Made Multi-Agent Handoffs Reliable](/blog/2026/06/23/microsoft-agent-framework-and-leankernel-3-the-runtime-contracts-that-made-multi-agent-handoffs-reliable/).
Part 4: [Designing for Proactive Execution Without Losing Control](/blog/2026/06/24/microsoft-agent-framework-and-leankernel-4-designing-for-proactive-execution-without-losing-control/).
