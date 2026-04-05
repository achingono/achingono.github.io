---
author: "Alfero Chingono"
title: "MCP at Scale: How I Used Model Context Protocol to Connect AI Agents to Gitea"
date: 2025-11-27T09:00:00Z
draft: true
description: "Why standard API calls aren't enough for agentic workflows, and how I built an MCP server to give CueMarshal's agents deep repo awareness in Gitea."
slug: mcp-at-scale-how-i-used-model-context-protocol-to-connect-ai-agents-to-gitea
tags: [
"MCP",
"Gitea",
"CueMarshal",
"AI Agents",
"Protocol",
"TypeScript",
"Build in Public"
]
categories: [
"Agentic AI",
"Build in Public"
]
image: "cover.png"
---

When I first wrote about [MCP in practice](/blog/2025/03/20/mcp-in-practice-what-anthropics-model-context-protocol-actually-means-for-developers/), I saw it as a powerful way to connect desktop tools to LLMs. But the real potential of the Model Context Protocol (MCP) becomes clear when you move it to the server side and use it as the "glue" for an entire [agentic engineering orchestra](/blog/2025/08/28/designing-multi-agent-systems-lessons-from-building-an-8-agent-engineering-orchestra/).

In [CueMarshal](https://www.cuemarshal.com), the agents need more than just the ability to "read a file." They need to understand the *state* of the repository: Which issues are open? What are the pending reviews? Are there conflicting changes in a PR?

Instead of writing custom API wrappers for every agent, I built a centralized **MCP Server for Gitea**.

## The Problem with Traditional API Wrappers

In the early days of CueMarshal, each agent had its own set of "tools" that were essentially thin wrappers around the Gitea REST API. This led to several issues:

1.  **Prompt Bloat:** Every time an agent started, I had to describe every single API endpoint in its system prompt.
2.  **Inconsistent Schema:** One agent might get the issue body as raw text, while another might get it as HTML, leading to parsing errors.
3.  **Fragility:** If the API changed, I had to update the tools across all eight agents.

## Why MCP is a Better Way

MCP changes the relationship between the model and the data. Instead of the model "knowing" how to call an API, the MCP server "advertises" its capabilities. The model just says "I want to list the issues in this repo," and the protocol handles the rest.

### The CueMarshal Gitea MCP Server

I built the server in TypeScript using the `@modelcontextprotocol/sdk`. Here is what it exposes to the agents:

*   **Resources:** Agents can "subscribe" to a specific file or a pull request. If the file changes, the MCP server can (in theory) notify the agent.
*   **Tools:**
    *   `list_repo_issues`: Returns a structured list of issues with metadata.
    *   `get_pr_diff`: Fetches the unified diff of a pull request.
    *   `create_issue_comment`: Allows agents to "talk" to humans on the Git platform.
    *   `apply_suggestion`: A specialized tool for applying code changes directly to a branch.

## Scaling to Eight Agents

What makes this setup work is that all eight CueMarshal agents, from [Marshal (Orchestrator) to Linton (Linter)](/blog/2025/08/28/designing-multi-agent-systems-lessons-from-building-an-8-agent-engineering-orchestra/), connect to the same MCP server.

This gives every agent a "shared reality." If the Reviewer agent (Reese) sees a diff, the Developer agent (Dave) sees the exact same diff through the same tool. That consistency matters for complex, multi-step tasks.

## Security and the "Read-Only" Gate

One of the most important aspects of using MCP at scale is security. My Gitea MCP server implements a strict **Read-Only mode** by default. An agent has to explicitly request "write" permissions for specific tools (like `apply_suggestion`), and even then, every write action is logged and requires [human approval in the final PR](/blog/2025/02/15/why-i-started-building-my-own-devops-platform-and-what-i-learned/).

## The Future: Multi-Platform MCP

The ultimate goal for CueMarshal is to make the agentic layer platform-agnostic. By using MCP, I can eventually swap out the Gitea MCP server for a GitHub or GitLab MCP server without changing a single line of agent code.

The protocol *is* the abstraction. And it's what makes [Agentic Delivery](/blog/2025/06/18/beyond-ci-cd-why-ai-agents-are-the-next-layer-of-software-delivery/) truly possible at scale.

---
*Related reading:*
- [MCP in Practice: What it Actually Means for Developers](/blog/2025/03/20/mcp-in-practice-what-anthropics-model-context-protocol-actually-means-for-developers/)
- [Beyond CI/CD: Why AI Agents Are the Next Layer of Software Delivery](/blog/2025/06/18/beyond-ci-cd-why-ai-agents-are-the-next-layer-of-software-delivery/)
