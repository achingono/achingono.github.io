---
author: "Alfero Chingono"
title: "Why I Started Building My Own DevOps Platform (And What I Learned)"
date: 2025-02-15T09:00:00Z
draft: false
description: "Why I started building CueMarshal: not to replace engineers, but to turn Git, reviews, and automation into a more practical coordination layer for software delivery."
slug: why-i-started-building-my-own-devops-platform-and-what-i-learned
tags: [
"CueMarshal",
"DevOps",
"Platform Engineering",
"Developer Experience",
"Build in Public",
"AI"
]
categories: [
"Build in Public",
"Platform Engineering",
"DevOps"
]
image: ""
---

For a while, I had the same reaction to most AI-for-software-delivery demos: impressive in a narrow way, but not something I would trust with real work. One tool could write code. Another could summarize a diff. Another could review a pull request. But the hard part of software delivery is rarely one isolated step. It is the handoff between steps.

That was the itch that eventually pushed me to start building [CueMarshal](https://github.com/cuemarshal/cuemarshal).

I did not start with the ambition to build "an AI company" or some abstract autonomous future. I started because I wanted a more coherent delivery system: one place where a task could move from idea to issue to branch to pull request to review without losing context every time responsibility changed hands.

## The problem I actually wanted to solve

CI/CD was never the whole problem. In many teams, the pipeline is the most deterministic part of the process. The mess usually lives around it:

- the design decision that only exists in a chat thread
- the issue that says too little
- the reviewer who has to reconstruct intent from commit history
- the documentation that is always "we'll do it after"
- the growing pile of tools that all know a little, but none of them own the workflow

What I wanted was not another dashboard. I wanted a delivery surface that respected how engineering work already happens.

That led me to a simple conviction: **Git should be the source of truth, not just the storage layer.**

If work already becomes legible through issues, branches, pull requests, labels, and reviews, then the orchestration layer should live there too. Not beside it. Not behind it. Inside it.

## Why I built it myself

There were three constraints that mattered to me from day one.

First, I wanted the system to be **self-hosted**. A lot of AI tooling assumes you are comfortable sending your code, your process, and your delivery metadata into someone else's black box. Many teams are not. I wanted an approach that made data sovereignty a feature, not an apology.

Second, I wanted the system to be **role-aware**. Real software delivery is not "one super-agent with a clever prompt." Design, implementation, review, testing, DevOps, and documentation are different jobs. Sometimes one person does multiple jobs, but the jobs are still different. That distinction matters.

Third, I wanted **human control to remain the final gate**. I am interested in automation, not surrender. If an AI system cannot work inside a reviewable pull-request workflow, I do not think it is mature enough for serious engineering work.

Those constraints eventually turned into the shape CueMarshal has now: a conductor service in TypeScript, specialized agents for architecture, development, review, testing, DevOps, docs, and linting, a Git-native workflow in Gitea, and a tool layer built around MCP so the same system can reason over structured interfaces instead of raw shell scripts and ad-hoc API calls.

## The architecture came later. The principles came first.

Long before the implementation solidified, the design principles were already obvious to me.

### 1. Git is a better coordination layer than most agent UIs

An issue is a task. A branch is a workstream. A pull request is a proposal. A review is a decision record. A merge is a controlled state change.

That sounds almost too obvious to say out loud, but it changed how I thought about the whole problem. Once I stopped treating Git as the place where code merely ends up, and started treating it as the place where engineering decisions become inspectable, the rest of the architecture got much simpler.

### 2. Specialization beats a "do everything" agent

In CueMarshal, the system is intentionally split into named roles: Marshal for orchestration, Ava for architecture, Dave for implementation, Reese for review, Tess for testing, Devin for DevOps, Dot for docs, and Linton for linting.

That is not branding for its own sake. It is an operational choice.

The moment one agent tries to be planner, coder, reviewer, tester, and documentarian all at once, you lose clarity. You also lose accountability. Specialization makes prompts sharper, tool permissions narrower, and outputs easier to judge.

### 3. Tool contracts matter more than prompt cleverness

One of the biggest lessons from building CueMarshal is that the quality of an agentic system is heavily constrained by the quality of its interfaces.

If an agent is forced to improvise around loosely structured APIs, fragile shell commands, or browser automation for tasks that should be typed and validated, the system becomes harder to trust. This is one reason MCP clicked for me so quickly later on: it gave a clean shape to something I already knew was essential.

Good tool contracts do not just help the model. They help the human operator understand what the system is even allowed to do.

### 4. Stateless workers are a feature, not a bug

CueMarshal's runners are intentionally stateless. They reconstruct context from the repository, the issue, the pull request, and the tool layer every time.

That may sound less magical than the "persistent AI teammate" narrative, but it is much easier to reason about. It scales better. It fails more cleanly. And it produces a better audit trail.

In practice, that has made me more skeptical of systems that depend on hidden memory to feel smart.

### 5. Human control is product design

The more I worked on this, the more convinced I became that "human in the loop" is not enough as a slogan. It has to be built into the workflow itself.

That is why I prefer issue-driven execution, reviewable pull requests, typed tools, explicit handoffs, and merge control. Those are not bureaucratic constraints. They are the difference between a system that can support real engineering and a system that is only good for demos.

## What I learned from building in public

The most useful part of this project has not been proving that agents can write code. We already knew that. The useful part has been learning where coordination breaks, where trust gets earned, and what kinds of structure make AI assistance actually usable.

It also made one thing clearer for me: the next layer of software delivery is not "more CI/CD." It is better orchestration around the work humans and machines are already doing together.

That is the reason I started building CueMarshal, and it is still the reason I keep working on it.

If you want the more technical follow-up, I wrote about [what MCP actually changed for developers](/blog/2025/03/20/mcp-in-practice-what-anthropics-model-context-protocol-actually-means-for-developers/) and the coordination lessons from [building an eight-agent engineering orchestra](/blog/2025/08/28/designing-multi-agent-systems-lessons-from-building-an-8-agent-engineering-orchestra/).

References:

- [CueMarshal repository](https://github.com/cuemarshal/cuemarshal)
- [CueMarshal architecture overview](https://github.com/cuemarshal/cuemarshal/blob/main/docs/architecture/overview.md)
- [CueMarshal agent profiles](https://github.com/cuemarshal/cuemarshal/blob/main/docs/features/agents/overview.md)
