---
author: "Alfero Chingono"
title: "How I Configured Five AI Agents on One Microsoft Teams Bot"
date: 2026-04-05T09:00:00Z
draft: true
description: "How I expanded my OpenClaw gateway from two agents to five, routed each to its own Teams channel, and discovered why RSC permissions matter more than config files."
slug: how-i-configured-five-ai-agents-on-one-teams-bot
tags: [
"OpenClaw",
"AI Agents",
"Multi-Agent Systems",
"Microsoft Teams",
"Bot Framework"
]
categories: [
"Agentic AI",
"Platform Engineering",
"Build in Public"
]
image: "cover.png"
---

In the [previous post](/blog/2026/04/03/how-i-split-openclaw-into-main-and-personal-agents/) I described splitting OpenClaw into two agents: Marshal for operations and Ava for personal conversations. That setup worked well enough that I wanted to push it further.

I had three more personas sitting in the [CueMarshal](https://www.cuemarshal.com) project that had never been deployed as live agents: Dave the Beaver (business advisory), Dot the Parrot (financial advisory), and Reese the Eagle (spiritual / reflective). Instead of spinning up separate bot registrations, I wanted all five agents to share a single Teams bot and route messages based on which channel the conversation happened in.

This is how I did it, and the one thing that almost derailed the whole setup.

## The architecture: one bot, five agents, channel-based routing

OpenClaw supports multiple agents in a single gateway instance. Each agent gets its own workspace directory (with its own identity, personality, and memory), its own model configuration, and its own API credential rotation. But they all share the same Teams bot registration, the same webhook endpoint, and the same Docker container.

The routing is handled by **bindings** in `openclaw.json`. Each binding maps an inbound message pattern to a specific agent:

```json
[
  {
    "type": "route",
    "agentId": "main",
    "match": { "channel": "msteams" }
  },
  {
    "type": "route",
    "agentId": "business",
    "match": {
      "channel": "msteams",
      "peer": { "kind": "channel", "id": "19:524d24…@thread.tacv2" }
    }
  }
]
```

The first matching binding wins. Specific channel bindings go before the catch-all so that messages in dedicated channels reach the right agent, while everything else falls through to Marshal.

## What each agent needed

Setting up a new agent isn't just adding a name to a list. Each one requires a complete directory tree:

**Workspace** (`data/workspaces/<agentId>/`):
- `IDENTITY.md`: the persona, including name, creature emoji, avatar path, and behavioral directives
- `SOUL.md`: domain expertise, personality boundaries, and how the agent relates to the others
- Shared files copied from existing agents: `USER.md`, `TOOLS.md`, `AGENTS.md`, `HEARTBEAT.md`
- `.openclaw/workspace-state.json`: must include `setupCompletedAt` or the agent tries to re-run the bootstrap flow
- `avatars/`: the agent's avatar image
- `memory/`: daily memory files that the agent creates automatically

**Config** (`data/config/agents/<agentId>/`):
- `agent/auth-profiles.json`: all API credentials with `lastGood` tracking
- `agent/models.json`: model provider configuration
- `sessions/sessions.json`: initialized with `{ "sessions": {} }`

The `auth-profiles.json` files deserve special attention. Each agent maintains its own record of which API credential was last successful. I spread the `lastGood` values across agents so they naturally distribute load across Google API keys rather than all hammering the same one.

## Per-agent model overrides

The existing agents (Marshal and Ava) inherit the global default model. For the three new agents, I wanted to try GitHub Copilot's `gpt-5.4` as the primary model instead:

```json
{
  "id": "business",
  "name": "Dave",
  "workspace": "business",
  "model": {
    "primary": "github-copilot/gpt-5.4",
    "fallbacks": [
      "google/gemini-3-flash-preview",
      "github-copilot/claude-sonnet-4.6",
      "ollama/llama3.2:3b"
    ]
  }
}
```

The primary model also needs to appear in the global `agents.defaults.models` allowlist. The fallback chain gives each agent three escape hatches before the request fails entirely.

## The binding schema gotcha

My first attempt at bindings used a flat string for the peer ID:

```json
"peer": "19:524d24…@thread.tacv2"
```

The gateway rejected this with a cryptic `bindings.2: Invalid input` error repeated for each new binding. Digging into the Zod validation schema inside the gateway container revealed that `peer` must be an object:

```json
"peer": { "kind": "channel", "id": "19:524d24…@thread.tacv2" }
```

Valid `kind` values are `direct`, `group`, `channel`, and `dm`. Teams channel conversations use `channel`. This isn't documented anywhere obvious. I found it by reading the bundled validation code at `/app/dist/io-DhtVmzAJ.js` inside the running container.

## The real problem: Teams wasn't delivering thread replies

After fixing the binding schema and restarting the gateway, all five agents responded to @mentions in their respective channels. For a moment, I thought the problem was solved.

Then I tried replying in a thread without @mentioning the agent. Nothing. No response from any agent, including Marshal who had been working fine before.

I spent a while investigating the OpenClaw side. The gateway config had `requireMention: false` set globally for Teams. The mention-gating code clearly showed that when `requireMention` is false, no messages are skipped. The gateway logs confirmed that every message it received was dispatched and processed successfully.

The problem was that the messages never arrived.

Looking at the session history for the spiritual agent, I could see two messages, both sent with @mentions, and responses to both. But the thread reply I sent without @mention was simply absent. Teams never forwarded it to the bot's webhook.

## RSC permissions: the missing piece

The Teams app manifest already had the right RSC permission:

```json
{
  "authorization": {
    "permissions": {
      "resourceSpecific": [
        { "name": "ChannelMessage.Read.Group", "type": "Application" }
      ]
    }
  }
}
```

`ChannelMessage.Read.Group` tells Teams to deliver *all* channel messages to the bot. Without it, Teams only delivers @mentions. But there's a catch that's easy to miss:

**RSC permissions are evaluated only at installation time.**

If the bot was installed in a team before this permission was added to the manifest, Teams doesn't retroactively grant it. The bot continues operating under the old permission set. The fix is to uninstall the bot from the team and reinstall it using the updated app package. The team owner gets a new consent prompt that includes the RSC permissions, and after that, all channel messages start flowing.

This is a Teams platform behavior, not an OpenClaw issue. But it's the kind of thing that can waste hours if you're only looking at config files and application logs.

## Two layers of "require mention"

There are actually two independent systems controlling whether thread replies trigger a response:

1. **Teams platform:** controls whether the message is *delivered* to the bot webhook. Without `ChannelMessage.Read.Group` RSC, only @mentions are delivered. With it, all channel messages are delivered.

2. **OpenClaw gateway:** controls whether a *delivered* message is *processed*. The `channels.msteams.requireMention` setting (default: `true`) gates this. Even if Teams delivers all messages, OpenClaw will silently drop non-@mention messages unless you set `requireMention: false`.

Both layers need to be configured correctly. Setting `requireMention: false` in OpenClaw without RSC permissions means the gateway is ready to process messages that never arrive. Having RSC permissions without `requireMention: false` means the messages arrive but get dropped.

The `requireMention` setting resolves through a chain: per-channel config → per-team config → global msteams config → `true`. If you have per-team entries in the `teams` section of your msteams config, those take precedence over the global setting.

## What I'd do differently

If I were starting over, I'd configure the Teams manifest with all RSC permissions from day one, before the first installation. Retrofitting permissions onto an already-installed bot is not hard; you just reinstall. But it's invisible enough that you can spend a long time debugging the wrong layer.

I'd also document the binding schema earlier. The `peer` object format is strict and the error message doesn't hint at the structure it expects. Having a working example in your notes saves real time.

## The current state

Five agents, one bot registration, one Docker container. Each agent has its own personality, its own model preferences, and its own Teams channel. Marshal handles the catch-all; the others respond in their dedicated channels. The gateway processes every message it receives without requiring @mentions.

The next step is reinstalling the Teams app in each team with the updated manifest so that thread replies start flowing without @mention. After that, I want to experiment with cross-agent awareness, letting one agent hand off a conversation to another when the domain shifts.

---

*This is part of an ongoing series about running OpenClaw as a self-hosted AI agent stack. Previous posts:*
- *[Why I Run OpenClaw in Docker on My Own Machine](/blog/2026/03/05/why-i-run-openclaw-in-docker-on-my-own-machine/)*
- *[Inside the Dockerfile Behind My OpenClaw Gateway](/blog/2026/03/09/inside-the-dockerfile-behind-my-openclaw-gateway/)*
- *[How I Wired Signal and Microsoft Teams into a Custom OpenClaw Image](/blog/2026/03/15/how-i-wired-signal-and-microsoft-teams-into-a-custom-openclaw-image/)*
- *[How I Split OpenClaw into Main and Personal Agents](/blog/2026/04/03/how-i-split-openclaw-into-main-and-personal-agents/)*
