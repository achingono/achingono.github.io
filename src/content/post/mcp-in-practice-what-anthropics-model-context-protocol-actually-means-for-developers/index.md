---
author: "Alfero Chingono"
title: "MCP in Practice: What Anthropic's Model Context Protocol Actually Means for Developers"
date: 2025-03-20T09:00:00Z
draft: false
description: "Why MCP matters in practice: not as another AI buzzword, but as a clean protocol for connecting models to real tools, systems, and delivery workflows."
slug: mcp-in-practice-what-anthropics-model-context-protocol-actually-means-for-developers
tags: [
"MCP",
"Model Context Protocol",
"Anthropic",
"CueMarshal",
"AI",
"Developer Tools"
]
categories: [
"Agentic AI",
"Platform Engineering"
]
image: "cover.png"
---

When Anthropic announced the [Model Context Protocol](https://www.anthropic.com/news/model-context-protocol), the most interesting part to me was not "LLMs can call tools." We already knew that. The interesting part was that someone was finally trying to standardize the connection.

That may sound like a small distinction, but it is the difference between a clever demo and an architecture you can actually build on.

For developers, MCP matters because it turns tool access into something more portable, more inspectable, and less bespoke. Instead of wiring every model to every internal system in a slightly different way, you get a shared protocol for secure, two-way connections between AI clients and the systems where work actually lives.

In other words: fewer one-off connectors, fewer weird wrappers, and less glue code pretending to be strategy.

## The real problem MCP solves

Without a protocol, most AI integrations end up with the same shape:

- custom JSON formats
- hand-rolled function schemas
- transport logic mixed into business logic
- a different adapter for every new client

You can absolutely ship systems that way. Many people already have. But you pay for it later in duplication, debugging, and lock-in.

Anthropic's framing resonated with me because it describes a problem I had already been running into while building [CueMarshal](https://www.cuemarshal.com). I did not need agents that could merely "use tools." I needed a stable way for different parts of the system to use the **same tools** in different contexts.

That is where MCP becomes practical.

## What it changed in my own thinking

In CueMarshal, I ended up with three MCP servers:

- a **Gitea MCP server** for issues, pull requests, repositories, workflows, and search
- a **Conductor MCP server** for task coordination and agent state
- a **System MCP server** for costs, runners, and health

That split was not arbitrary. It reflected a design choice: organize tool access around bounded responsibilities instead of dumping everything into one giant catch-all toolbox.

Even more important, the same MCP server code supports **two transports**:

- **stdio** for agent runners
- **HTTP/SSE** for the long-running chat/orchestration layer

This is the part I think many developers will underestimate. The value is not just that the model can invoke a tool. The value is that your tool layer stops being trapped inside one execution model.

The CueMarshal runners can spawn MCP servers directly as child processes. The Conductor can hold long-lived connections to those same tool surfaces over the network. Same capability, different runtime, no duplicated tool logic.

That is not just elegant. It is operationally useful.

## MCP is really about interface discipline

One thing building AI systems teaches very quickly is that "prompting" gets too much credit for problems that are really interface problems.

If the tool schema is vague, the model will behave vaguely.

If the permissions are broad, the behavior will feel risky.

If the transport is brittle, the whole system looks flaky even when the reasoning is fine.

What I like about MCP is that it nudges teams toward better engineering habits:

1. **Typed tools instead of implied behavior**
2. **Separation between protocol and implementation**
3. **Reusable tool layers across multiple clients**
4. **Clearer permission boundaries**

That discipline matters even if you never use Anthropic's stack directly.

## What developers should actually do with it

My advice is to treat MCP less like a product feature and more like a systems design decision.

If you are building AI-assisted software delivery, internal automation, or even just richer developer tools, start by asking:

- What are the real systems my assistant needs to access?
- Which of those interactions deserve typed, validated interfaces?
- Which capabilities should be shared across chat, automation, and background agents?
- Where do I want auditability and permission scoping to live?

That line of thinking will produce a better architecture whether you adopt MCP tomorrow or not.

In my own work, it pushed me away from raw `curl`-driven integration and toward a universal tool layer. Once I made that shift, a lot of downstream problems became easier: orchestration, reuse, security boundaries, and even explanation. It is easier to trust a system when you can say, very plainly, "here are the tools it has, here is what they do, and here is how they are invoked."

## What MCP does **not** solve

MCP does not magically make an agent reliable.

It does not fix poor workflow design.

It does not remove the need for human review.

And it definitely does not turn vague prompts into good engineering.

What it does is give you a cleaner control plane for connecting models to real systems. That is already a meaningful improvement.

For me, that is why MCP feels important. Not because it adds more AI theater, but because it reduces architectural friction in a place where friction compounds very fast.

If you are curious how that idea plays out in a larger system, I wrote more about the broader coordination problem in [Why I Started Building My Own DevOps Platform](/blog/2025/02/15/why-i-started-building-my-own-devops-platform-and-what-i-learned/) and the orchestration lessons in [Designing Multi-Agent Systems: Lessons from Building an 8-Agent Engineering Orchestra](/blog/2025/08/28/designing-multi-agent-systems-lessons-from-building-an-8-agent-engineering-orchestra/).

References:

- [Introducing the Model Context Protocol](https://www.anthropic.com/news/model-context-protocol)
- [MCP quickstart and specification](https://modelcontextprotocol.io/quickstart)
- [CueMarshal MCP server overview](https://github.com/cuemarshal/cuemarshal/blob/main/docs/features/mcp-servers/overview.md)
- [CueMarshal architecture overview](https://github.com/cuemarshal/cuemarshal/blob/main/docs/architecture/overview.md)
