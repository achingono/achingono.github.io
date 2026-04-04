---
author: "Alfero Chingono"
title: "Designing Multi-Agent Systems: Lessons from Building an 8-Agent Engineering Orchestra"
date: 2025-08-28T09:00:00Z
draft: false
description: "What building CueMarshal taught me about real multi-agent design: role boundaries, permissions, identity, routing, and why orchestration matters more than agent count."
slug: designing-multi-agent-systems-lessons-from-building-an-8-agent-engineering-orchestra
tags: [
"CueMarshal",
"Multi-Agent Systems",
"AI Agents",
"MCP",
"Platform Engineering",
"DevOps"
]
categories: [
"Agentic AI",
"Platform Engineering",
"Build in Public"
]
image: ""
---

A lot of "multi-agent" demos are really one agent wearing different hats.

The names change. The prompts change. Sometimes the avatars change. But the authority model, the memory model, and the execution model are all still basically the same. That is fine for a demo. It is much less convincing when you are trying to build a system that can do real engineering work.

Building [CueMarshal](https://github.com/cuemarshal/cuemarshal) made that distinction impossible for me to ignore.

What I wanted was not eight personalities for marketing. I wanted a working system where planning, coding, review, testing, DevOps, documentation, and quality control could be separated cleanly enough to be trustworthy.

That is how the "engineering orchestra" idea emerged.

## The roles mattered because the boundaries mattered

CueMarshal's cast eventually became:

- **Marshal** for orchestration
- **Ava** for architecture
- **Dave** for implementation
- **Reese** for review
- **Tess** for testing
- **Devin** for DevOps
- **Dot** for documentation
- **Linton** for linting

What made that useful was not the naming. It was the fact that the roles had **different responsibilities, different tool access, and different default model tiers**.

That is the first lesson I would pass on to anyone designing a multi-agent system:

## 1. Different roles need different authority

If your reviewer can rewrite production code, your reviewer is not really a reviewer.

In CueMarshal, least privilege is deliberate. The reviewer is configured without write/edit permissions. The docs agent is restricted from shell access. The linter acts like a gate, not a developer with nicer manners.

That kind of restriction sounds limiting until you realize it is what gives each role meaning. Boundaries are not friction here. They are the mechanism that creates trust.

A good multi-agent system is not just a cluster of competencies. It is a set of constrained responsibilities.

## 2. Coordination needs durable state outside the model

One of the reasons I anchored CueMarshal in Git is that I did not want coordination to depend on hidden model memory.

Tasks become issues.
Work becomes branches.
Proposals become pull requests.
Reviews become durable comments and approvals.

The Conductor receives webhooks, uses Redis and BullMQ to manage asynchronous flow, and dispatches work through Gitea Actions. The runners themselves stay stateless; they rebuild context from the repository, the issue, and the tool layer every time.

That has been a much better trade than magical continuity.

Models forget.
Git does not.

## 3. Identity is part of the architecture

Another thing I underestimated early on was how important identity separation would be.

Each CueMarshal agent has its own account, token, and audit trail. That means the Git history shows who planned, who implemented, who reviewed, and who approved. Even when the "who" is an AI agent, the distinction still matters.

This has two benefits.

First, it improves explainability. The system becomes easier to inspect when actions are attributable.

Second, it changes how you think about safety. Once every agent has a clear identity and permission scope, you stop designing from a vague "assistant" mindset and start designing from explicit operational roles.

That shift is subtle, but it is foundational.

## 4. The tool layer is what makes the orchestra playable

This is where MCP became important for CueMarshal.

All of the agents connect to a structured tool layer instead of improvising raw integrations on the fly. The same Gitea, Conductor, and System capabilities can be used by the runner agents over stdio and by the orchestration layer over HTTP/SSE.

That matters because multi-agent systems are not only about reasoning. They are about coordination through reliable interfaces.

If the tools are vague, agents collide.
If the permissions are sloppy, trust collapses.
If the transports are inconsistent, reuse gets expensive.

The protocol is not the whole story, but it is the difference between a collection of prompts and a real system surface.

I wrote more about that in [MCP in Practice](/blog/2025/03/20/mcp-in-practice-what-anthropics-model-context-protocol-actually-means-for-developers/), because it deserves its own treatment.

## 5. Model routing is architecture, not optimization

Another lesson I came away with: not every role deserves the same model.

Architecture work is more expensive and more consequential than documentation cleanup. Review often needs stronger reasoning than linting. Mechanical work should not burn premium tokens if a cheaper tier can do it reliably.

CueMarshal's tiered routing reflects that reality:

- heavy reasoning for architecture
- balanced capability for implementation, review, testing, and DevOps
- lighter-weight models for docs and linting

That is not just a cost decision. It is part of how the system stays sustainable.

Too many agent systems treat model choice as an afterthought. I think it belongs in the design doc.

## 6. Closed loops beat hero agents

The more I build these systems, the less I believe in the "super-agent" story.

What works better is a closed loop:

1. detect work
2. route it clearly
3. execute with constrained roles
4. review it
5. merge it with human control
6. feed the next signal back into the system

CueMarshal's self-improvement workflow made this even clearer to me. Once SonarQube findings, scanners, issues, PRs, and agent roles all started participating in the same loop, the system became more useful than any single agent inside it.

That is why I think orchestration matters more than agent count.

## My current takeaway

If you are building a multi-agent system, start with these questions:

- What roles genuinely need to be different?
- What permissions should each role have?
- Where does coordination state live?
- How are actions attributed?
- What is the closed loop that turns outputs into the next inputs?

If you cannot answer those, adding more agents will mostly add more noise.

If you can answer them, the number of agents becomes much less important than the quality of the structure around them.

That has been the real lesson for me. The point of the orchestra is not to have more instruments. The point is to make the handoffs musical instead of chaotic.

If you want the adjacent pieces, [Why I Started Building My Own DevOps Platform](/blog/2025/02/15/why-i-started-building-my-own-devops-platform-and-what-i-learned/) covers the motivation, and [How I Run SonarQube in My Own CI Pipeline (And Let AI Fix What It Finds)](/blog/2026/03/05/how-i-run-sonarqube-in-my-own-ci-pipeline-and-let-ai-fix-what-it-finds/) shows what this architecture looks like when the feedback loop closes on itself.

References:

- [CueMarshal repository](https://github.com/cuemarshal/cuemarshal)
- [CueMarshal architecture overview](https://github.com/cuemarshal/cuemarshal/blob/main/docs/architecture/overview.md)
- [CueMarshal agent profiles](https://github.com/cuemarshal/cuemarshal/blob/main/docs/features/agents/overview.md)
- [CueMarshal conductor overview](https://github.com/cuemarshal/cuemarshal/blob/main/docs/features/conductor/overview.md)
