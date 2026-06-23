---
author: "Alfero Chingono"
title: "Microsoft Agent Framework and LeanKernel (Part 3): The Runtime Contracts That Made Multi-Agent Handoffs Reliable"
date: 2026-06-23T09:00:00Z
draft: true
description: "Three concrete problems from LeanKernel runtime history and the contract changes that fixed them."
slug: microsoft-agent-framework-and-leankernel-3-the-runtime-contracts-that-made-multi-agent-handoffs-reliable
tags: [
"LeanKernel",
"Microsoft Agent Framework",
"Multi-Agent"
]
categories: [
"Architecture",
"Agentic AI"
]
image: ""
---

This is a continuation of [Part 2](/blog/2026/06/22/microsoft-agent-framework-and-leankernel-2-why-a-modular-monolith-was-the-right-first-bet).

I ran into three specific contract failures while building multi-agent handoffs. Each one looked like a model problem but was actually a runtime contract problem.

## 1) Agent describing tools instead of using them

An agent in the workflow would explain what it intended to do rather than calling functions. The prompt was written well. The model was capable. But the runtime had not enabled function-calling mode:

```csharp
// AgentInvocationBuilder.cs — before
return new ChatOptions
{
    Tools = [.. context.Tools],
    // ToolMode = ChatToolMode.Auto  — was missing
};
```

When `ChatToolMode` defaults to `None`, the model can't emit structured function calls. The agent was literally not allowed to call functions. The fix was two lines:

```csharp
// AgentInvocationBuilder.cs — after
return new ChatOptions
{
    Tools = [.. context.Tools],
    ToolMode = ChatToolMode.Auto
};
```

This was the fix from commit `1745636`. It produced a bigger reliability improvement than switching models. The model was fine. The contract was wrong.

## 2) Health probe targeting the wrong path

An agent status check in the deployment health probe pointed to `/health` instead of `/health/liveliness`. The probe always returned 200 (the root health aggregator), but the liveliness endpoint was supposed to detect container stalls.

```yaml
# docker-stack.yml — before
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:5080/health"]

# docker-stack.yml — after
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:5080/health/liveliness"]
```

Two paths in the same service. Root health aggregated all dependencies and returned a 200 even when downstream services were unreachable. The liveliness endpoint only checked container-local process state. The wrong path meant Swarm never triggered a restart on stalled agents because the health check was checking the wrong level of the system.

## 3) Role-based context routing

The early handoff logic used a flat context bag shared across all agents. Every agent saw the full context regardless of role. That worked until agents with incompatible scope contradicted each other.

```csharp
// ContextAssemblyFilter.cs
public class ContextAssemblyFilter
{
    public AgentRole TargetRole { get; init; }
    public required string ContextGuidelines { get; init; }
    public bool IncludeExternalTools { get; init; }
}
```

The fix was filtering context assembly by the receiving agent's role. Each agent sees only the context it needs: the knowledge worker gets knowledge context, the tool executor gets tool schemas, the reviewer gets diff context. The cross-contamination stopped.

## What generalizes

None of these were model quality issues. They were delivery system issues: the runtime didn't enable function calling, the health probe didn't probe what mattered, and context was not scoped. Once those contracts were explicit, handoffs became reliable enough that I stopped wondering which turn they would fail.

---

Series navigation:

Part 1: [The Problem Was Never Just Prompts](/blog/2026/06/21/microsoft-agent-framework-and-leankernel-1-the-problem-was-never-just-prompts/).
Part 2: [Why a Modular Monolith Was the Right First Bet for LeanKernel](/blog/2026/06/22/microsoft-agent-framework-and-leankernel-2-why-a-modular-monolith-was-the-right-first-bet/).
Part 4: [Designing for Proactive Execution Without Losing Control](/blog/2026/06/24/microsoft-agent-framework-and-leankernel-4-designing-for-proactive-execution-without-losing-control/).
