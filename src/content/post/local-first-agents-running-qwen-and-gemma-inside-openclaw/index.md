---
author: "Alfero Chingono"
title: "Local-First Agents: Running Qwen 3.6 and Gemma 4 Inside OpenClaw"
date: 2026-07-02T09:00:00Z
draft: true
description: "Wiring a local model into OpenClaw without breaking tool-calling contracts — and a frank look at where local still loses."
slug: local-first-agents-running-qwen-and-gemma-inside-openclaw
tags: [
"OpenClaw",
"Local AI",
"Ollama",
"MCP",
"AI Agents",
"docker"
]
categories: [
"AI Agents",
"OpenClaw"
]
image: ""
---

Running a local model is easy. Running a local model as the brain of an agent that has to call tools, honor schemas, and hand off to other agents is where it gets interesting. This is what I actually had to change inside OpenClaw to make Qwen 3.6 and Gemma 4 work as first-class agent backends — and where I still route to a frontier model on purpose.

## Outline

- Why local-first for agents: privacy, cost, rate-limit immunity, dev-loop speed
- The three things that usually break when you swap in a local model:
  - Tool-calling schema adherence
  - Long-context reasoning beyond the effective window
  - Silent degradation on multimodal inputs
- The OpenClaw gateway change: a model-router layer that picks local vs remote per task type
- Hands-on:
  - Ollama + the model choice and why
  - LiteLLM as the uniform call surface
  - MCP tool surface unchanged; only the backend swaps
  - A worked example of a non-trivial tool-calling task on Qwen 3.6
- Honest failure log: where local still loses today
- A routing policy that picks frontier vs local automatically based on task signature

## Artifacts to reference

- [Inside the Dockerfile Behind My OpenClaw Gateway](../inside-the-dockerfile-behind-my-openclaw-gateway/)
- [Why I Run OpenClaw in Docker on My Own Machine](../why-i-run-openclaw-in-docker-on-my-own-machine/)
- [MCP at Scale: How I Used Model Context Protocol to Connect AI Agents to Gitea](../mcp-at-scale-how-i-used-model-context-protocol-to-connect-ai-agents-to-gitea/)
