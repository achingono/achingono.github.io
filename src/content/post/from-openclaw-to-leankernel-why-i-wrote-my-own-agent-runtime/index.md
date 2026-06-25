---
author: "Alfero Chingono"
title: "From OpenClaw to LeanKernel: Why I Wrote My Own Agent Runtime"
date: 2026-06-25T09:00:00Z
draft: false
description: "What OpenClaw upgrades taught me about building my own agent harness."
slug: from-openclaw-to-leankernel-why-i-wrote-my-own-agent-runtime
tags: [
"OpenClaw",
"LeanKernel",
"Agent Infrastructure"
]
categories: [
"Agentic AI",
"Architecture"
]
image: ""
---

On May 6th, 2026, I posted this on X/Twitter:

> I'm kinda tired of upgrades breaking my rig.

That was the public signal. The private signal had been building for a month.

I had been running [OpenClaw](https://openclaw.ai) as my personal agent gateway since early March. Multiple agents (Marshal, Dot, Ava), wired into Signal and Teams, with shell tools, cron jobs, and a custom Docker image. It worked. And then upgrades would come in and the gateway would fail to start, or the agent would start behaving differently, or the config I had carefully shaped would be gone.

Here is what actually happened, traced through the backup directory I kept.

## The clobbered files

OpenClaw's gateway writes its own config on startup. When an upgrade changed the schema or the wizard ran, it would overwrite my `config/openclaw.json` with defaults. The original config was saved as `openclaw.json.clobbered.{timestamp}`.

The first time I noticed this was April 5th. Three clobbered files in one morning:

```
openclaw.json.clobbered.2026-04-05T04-33-21-164Z
openclaw.json.clobbered.2026-04-05T04-46-41-904Z
openclaw.json.clobbered.2026-04-05T04-53-32-743Z
```

Each one had stripped my custom provider definitions. The diff on the first one:

```diff
-      "azure-pro": {
-        "provider": "azure",
-        "mode": "api_key"
-      }
```

Gone. Also gone: the `localjson` secrets provider, the LiteLLM embedding config. Replaced with the default Ollama-only setup from version `2026.4.1`.

I restored the config, re-added `azure-pro`, and moved on.

April 27th was worse. The Dockerfile base image update triggered the clobbering loop. **Eight** clobbered files between 21:14 and 22:17 UTC:

```text
openclaw.json.clobbered.2026-04-27T21-14-15-255Z
openclaw.json.clobbered.2026-04-27T21-25-26-375Z
openclaw.json.clobbered.2026-04-27T21-39-10-056Z
openclaw.json.clobbered.2026-04-27T21-46-17-186Z
openclaw.json.clobbered.2026-04-27T21-57-58-106Z
openclaw.json.clobbered.2026-04-27T22-07-32-543Z
openclaw.json.clobbered.2026-04-27T22-11-54-377Z
openclaw.json.clobbered.2026-04-27T22-17-02-742Z
```

Every time the gateway restarted, it wiped the `azure-pro` and `azure-prem` providers. Every time I restored them, the next restart wiped them again. The container was in a crash loop and each recycle clobbered the config fresh.

The naming convention in `config/` tells the story of how I fought this:

```text
openclaw.json                    # current (fragile)
openclaw.json.bak                # manual save
openclaw.json.bak.1              # manual save
openclaw.json.bak.2              # manual save
openclaw.json.bak.3              # manual save
openclaw.json.bak.4              # manual save
openclaw.json.last-good          # known working version
openclaw.json.lastgood-*.json    # earlier known working
openclaw.json.pre-hierarchy.bak  # before Council change
openclaw.json.clobbered.*        # gateway's backup of what it overwrote
```

Six layers of backup for a JSON config file. That is not a sustainable relationship with your agent gateway.

Between April 5th and May 3rd, I counted 22 clobbered files.

## The LiteLLM pivot

By early May, I had integrated LiteLLM as a proxy layer to route across multiple model providers. The commit history shows the arc:

```
2026-04-23  feat: update AI models to GPT-5.4 series
2026-04-24  Add remote ollama instance as vision fallback
2026-04-24  Add remote models to general fallback chain
2026-04-27  Update Dockerfile base image and enhance openclaw configuration with new Azure and NVIDIA models
2026-05-03  feat: integrate LiteLLM proxy for multi-provider model routing
```

The LiteLLM integration was my attempt to work around the provider lock-in. If I could route through a proxy, I could swap models without touching the gateway config. The `docker-compose.yml` grew a LiteLLM container:

```yaml
litellm:
  image: ghcr.io/achingono/litellm:latest
  environment:
    - LITELLM_MASTER_KEY=${LITELLM_API_KEY}
```

And `config/openclaw.json` pointed embeddings at it:

```json
{
  "baseUrl": "http://litellm:4000",
  "api": "openai-completions",
  "apiKey": "${LITELLM_API_KEY}"
}
```

The proxy worked. But the clobbering kept happening. The gateway did not know about the LiteLLM provider. Every clobber removed it.

## The wiki that the agent could not remember

The deeper problem was the memory system.

I had six agent workspaces (main, business, career, financial, personal, spiritual), each backed by a SQLite database. The agent was supposed to remember what it learned across sessions. It did not hold.

I tried building a wiki system inside OpenClaw. Structured markdown pages, organized by subject, with recovery state committed to git:

```
2026-04-20  Commit wiki recovery state
2026-04-21  Commit wiki state
2026-04-23  Commit wiki recovery state
2026-05-03  Commit wiki state
```

The agent would read the wiki, acknowledge it, and then on the next turn behave as if none of it existed. The retrieval pipeline did not surface the content reliably. I was adding knowledge to a system that was not designed to keep it.

The commit logs show the tension. A `pre-wiki-ingest` backup of the cron jobs was taken on April 20, and a `pre-wiki-replan` backup on the same day, suggesting I replanned the strategy within hours of rolling it out.

## The last commit

May 3rd was the last commit in the OpenClaw backup. Two fixes for the LiteLLM embedding provider configuration, and the final wiki state. Then nothing.

Three days later came the tweet.

I did not stop using OpenClaw that day. The workspace directories show activity through May 23rd. The cron jobs ran through May 29th. The WAL files on the SQLite databases kept updating. But I had already started thinking about what a different architecture would look like.

## What LeanKernel does differently

The [LeanKernel README at commit 3783854](https://github.com/achingono/leankernel/blob/3783854eafa67409c60ce58b8e0e5af818439213/README.md) describes the positioning:

> LeanKernel is the personal AI agent for builders who want reliable output, lower token spend, and full control of context. Instead of bloated chat history and unpredictable behavior, LeanKernel gives you a lean, observable agent runtime.

The key architectural differences that address the OpenClaw pain points:

**The config is not owned by the runtime.** OpenClaw wrote to `openclaw.json` on startup and on every schema migration. LeanKernel uses `appsettings.json` and environment variables. The runtime reads config, it does not write it. There is no code path that can clobber your provider definitions.

**Memory is a deterministic subsystem, not a vector search.** OpenClaw's memory was a SQLite-backed vector store with inconsistent retrieval. LeanKernel uses a [5W1H wiki](https://github.com/achingono/leankernel/blob/main/docs/architecture/architecture.md#wiki-system) persisted as markdown with YAML frontmatter. Each fact carries a confidence score and source citation. Retrieval is deny-by-default: context is explicitly admitted, not vaguely searched.

**Provider routing is built into the runtime, not jury-rigged through a proxy.** The LiteLLM integration I hacked into OpenClaw became a first-class configuration surface in LeanKernel. The `LiteLLMConfiguration` class controls model selection, fallback chains, and tool count limits from one place.

**Upgrades do not change behavior.** There is no wizard that rewrites config. There is no schema migration that changes model defaults. The Docker image is rebuilt from a Dockerfile, the Compose stack is version-controlled in the [swarm repo](https://github.com/achingono/swarm), and the runtime contract is explicit.

---

Earlier posts in this series:

- [Part 1: The Problem Was Never Just Prompts](/blog/2026/06/21/microsoft-agent-framework-and-leankernel-1-the-problem-was-never-just-prompts/)
- [Part 2: Why a Modular Monolith Was the Right First Bet for LeanKernel](/blog/2026/06/22/microsoft-agent-framework-and-leankernel-2-why-a-modular-monolith-was-the-right-first-bet/)
- [Part 3: The Runtime Contracts That Made Multi-Agent Handoffs Reliable](/blog/2026/06/23/microsoft-agent-framework-and-leankernel-3-the-runtime-contracts-that-made-multi-agent-handoffs-reliable/)
- [Part 4: Designing for Proactive Execution Without Losing Control](/blog/2026/06/24/microsoft-agent-framework-and-leankernel-4-designing-for-proactive-execution-without-losing-control/)
- [LeanKernel Scheduled Jobs on Swarm](/blog/2026/06/26/leankernel-scheduled-jobs-on-swarm-ofelia-gbrain-and-the-gotchas-that-mattered/)
- [LeanKernel Deployment History](/blog/2026/06/25/leankernel-swarm-deployment-commit-history-what-broke-and-how-i-fixed-it/)

Related reading:

- [LeanKernel architecture docs](https://github.com/achingono/leankernel/blob/main/docs/architecture/architecture.md)
- [OpenClaw](https://openclaw.ai)
