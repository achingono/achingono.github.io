---
author: "Alfero Chingono"
title: "Microsoft Agent Framework and LeanKernel (Part 2): Why a Modular Monolith Was the Right First Bet"
date: 2026-06-22T09:00:00Z
draft: false
description: "LeanKernel did not start with microservices. It started with strict module boundaries inside one deployable host to stabilize architecture before distribution."
slug: microsoft-agent-framework-and-leankernel-2-why-a-modular-monolith-was-the-right-first-bet
tags: [
"LeanKernel",
"Microsoft Agent Framework",
"Modular Monolith",
"Software Architecture",
"Platform Engineering"
]
categories: [
"Architecture",
"Agentic AI",
"Engineering"
]
image: ""
---

When people hear "agent platform," they often expect a microservices map on day one.

I made a different choice with LeanKernel: a modular monolith in .NET, with strict project boundaries and composition in one gateway host.

That was not a compromise. It was an architectural strategy.

## The decision criteria

Early-stage agent systems usually face three kinds of volatility:

- role definitions change as responsibilities become clearer
- orchestration flow changes as failure modes appear
- context and tool contracts change as real usage reveals ambiguity

If you distribute too early, every boundary change carries infrastructure tax: more deployment units, more network concerns, more version coordination, and more failure points while fundamentals are still moving.

I wanted architectural rigor without premature distributed complexity.

## How LeanKernel maps boundaries inside one solution

LeanKernel's structure is intentionally domain-oriented:

- `LeanKernel.Abstractions` and `LeanKernel.Core` define shared contracts and primitives
- `LeanKernel.Agents`, `LeanKernel.Thinker`, and `LeanKernel.Context` shape reasoning and orchestration behavior
- `LeanKernel.Tools` and `LeanKernel.Plugins` isolate capability surfaces
- `LeanKernel.Persistence` and `LeanKernel.Archivist` handle state and knowledge durability
- `LeanKernel.Channels`, `LeanKernel.Commander`, and `LeanKernel.Gateway` compose ingress, egress, and host concerns

The important detail is not the project count. It is the responsibility map.

Modules own behavior. The gateway composes them.

## Why this works well with Microsoft Agent Framework

Microsoft Agent Framework gives useful orchestration and protocol primitives, but it does not define your domain decomposition for you.

That decomposition is where maintainability lives.

Using a modular monolith means I can align framework capabilities to stable internal seams first:

- orchestration patterns map into `Thinker` and `Agents`
- external tool invocation maps into `Tools` and plugin surfaces
- context and memory shaping maps into `Context`, `Knowledge`, and persistence modules

Once those seams harden, distribution can be a deployment decision instead of a design gamble.

## Trade-offs I accepted intentionally

Every architecture is a trade.

With this approach, I accepted:

- less independent runtime scaling in the short term
- tighter process-level coupling while domains mature
- stronger discipline needed to prevent host-layer leakage

In return, I gained:

- faster feedback on architectural changes
- lower operational complexity during high-change phases
- clearer refactoring paths because module contracts stay explicit

For LeanKernel's stage, that was the better deal.

## What I would tell teams building similar systems

Do not ask "monolith or microservices" as a branding choice.

Ask this instead:

- are your boundaries stable enough to distribute safely?
- can each boundary tolerate independent deployment failures?
- do you already understand your cross-boundary contracts under load?

If those answers are still emerging, modular monolith is often the more honest architecture.

Ship the boundary map first. Split deployment units later when the split buys you reliability, not status.

---

Series navigation:

- Part 1: [The Problem Was Never Just Prompts](/blog/2026/06/21/microsoft-agent-framework-and-leankernel-1-the-problem-was-never-just-prompts/)
- Part 3: [The Runtime Contracts That Made Multi-Agent Handoffs Reliable](/blog/2026/06/23/microsoft-agent-framework-and-leankernel-3-the-runtime-contracts-that-made-multi-agent-handoffs-reliable/)
- Part 4: [Designing for Proactive Execution Without Losing Control](/blog/2026/06/24/microsoft-agent-framework-and-leankernel-4-designing-for-proactive-execution-without-losing-control/)
