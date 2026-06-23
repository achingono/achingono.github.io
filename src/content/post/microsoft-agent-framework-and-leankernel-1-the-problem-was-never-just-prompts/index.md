---
author: "Alfero Chingono"
title: "Microsoft Agent Framework and LeanKernel (Part 1): The Problem Was Never Just Prompts"
date: 2026-06-21T09:00:00Z
draft: true
description: "Why the delivery system around the model matters more than the model itself, starting from a specific two-line fix."
slug: microsoft-agent-framework-and-leankernel-1-the-problem-was-never-just-prompts
tags: [
"Microsoft Agent Framework",
"LeanKernel",
"AI Agents"
]
categories: [
"Agentic AI",
"Architecture"
]
image: ""
---

I was working on an agent workflow where one agent needed to call tools to fulfill user requests. Instead of calling functions, the agent kept responding in natural language: "I would use the search tool to find relevant documents." It described intent instead of executing it.

Here is what the code looked like when I traced the issue:

```csharp
// AgentInvocationBuilder.cs — before
return new ChatOptions
{
    Tools = [.. context.Tools]
    // ToolMode defaults to None — model doesn't know it can call functions
};
```

The prompt had no instruction to invoke tools, and `ChatToolMode` defaulted to `None`. The model was working correctly within its constraints — it just wasn't told it could emit structured function calls. The fix was two lines:

```csharp
// AgentInvocationBuilder.cs — after
return new ChatOptions
{
    Tools = [.. context.Tools],
    ToolMode = ChatToolMode.Auto  // tells the model function calling is available
};
```

And one line in the system prompt:

```csharp
// PromptAssembler.cs
parts.Add("You have access to the functions listed above. When a user asks you to do something that requires a tool, use the function call mechanism rather than describing what you would do.");
```

That fix (commit `1745636`) changed more about agent reliability than any model swap. The model wasn't the bottleneck. The delivery system was.

## What I was actually trying to build

I wanted a system that could take a request from idea to merged change with reliability, not just fluency. That meant:

- role-specific execution instead of one "do everything" assistant
- tool contracts instead of ad-hoc integration scripts
- deterministic context assembly and durable state transitions
- traceability across planning, implementation, review, and operations

When Microsoft Agent Framework became generally available, what stood out was the platform intent: stable SDK surface, protocol-first collaboration, and clearer orchestration primitives. Those aligned with the problems I was hitting.

## Why LeanKernel started as a modular monolith

A lot of people asked why I didn't start with distributed microservices. Early on, the hardest question was stabilizing contracts, not scaling traffic. I needed fast iteration on agent boundaries and context assembly rules. A modular monolith gave me one deployable unit with explicit internal boundaries:

```text
LeanKernel.sln
├── LeanKernel.Abstractions   # shared contracts
├── LeanKernel.Core           # primitives
├── LeanKernel.Agents         # agent behavior
├── LeanKernel.Thinker        # reasoning/orchestration
├── LeanKernel.Context        # prompt/runtime assembly
├── LeanKernel.Tools          # tool definitions
├── LeanKernel.Plugins        # dynamic skill loading
├── LeanKernel.Persistence    # state durability
├── LeanKernel.Archivist      # knowledge management
├── LeanKernel.Channels       # ingress
├── LeanKernel.Commander      # egress
└── LeanKernel.Gateway        # host composition
```

The project count matters less than what each owns. Modules own behavior. The gateway composes them.

## What early iterations taught me

The first versions were over-optimistic about autonomy. You assume the model can fill more gaps than it should. You discover that implicit assumptions become failures under load. Handoffs are where quality silently degrades.

The design principle that emerged: make state explicit, keep responsibilities explicit, and treat orchestration as a runtime contract instead of a conversation.

---

Series navigation:

Part 2: [Why a Modular Monolith Was the Right First Bet for LeanKernel](/blog/2026/06/22/microsoft-agent-framework-and-leankernel-2-why-a-modular-monolith-was-the-right-first-bet/).
Part 3: [The Runtime Contracts That Made Multi-Agent Handoffs Reliable](/blog/2026/06/23/microsoft-agent-framework-and-leankernel-3-the-runtime-contracts-that-made-multi-agent-handoffs-reliable/).
Part 4: [Designing for Proactive Execution Without Losing Control](/blog/2026/06/24/microsoft-agent-framework-and-leankernel-4-designing-for-proactive-execution-without-losing-control/).
