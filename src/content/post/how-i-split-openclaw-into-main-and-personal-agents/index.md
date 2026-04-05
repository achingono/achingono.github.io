---
author: "Alfero Chingono"
title: "How I Split OpenClaw into Main and Personal Agents"
date: 2026-04-03T09:00:00Z
draft: false
description: "How I split OpenClaw into main and personal agents, gave them separate workspaces, and routed Signal and Teams differently so the system felt more natural to live with."
slug: how-i-split-openclaw-into-main-and-personal-agents
tags: [
"OpenClaw",
"AI Agents",
"Multi-Agent Systems",
"Signal",
"Microsoft Teams"
]
categories: [
"Agentic AI",
"Platform Engineering",
"Build in Public"
]
image: "cover.png"
---

By this point in the series, the Docker stack was solid, the custom image was working, and both channel integrations were live. The next question wasn't purely technical — it was architectural.

Should one agent answer everything?

I didn't think so.

## I wanted separation without turning the system into theater

I'm generally skeptical of fake multi-agent setups where the same assistant just changes costumes.

What I wanted here was simpler and more practical:

- one agent for the main, more operational surface
- another agent for the more personal, direct-chat surface

That's how I ended up with:

- **Marshal** as the `main` agent
- **Ava** as the `personal` agent

This was not about branding. It was about boundaries.

## The split shows up directly in config

The OpenClaw config defines both the shared defaults and the named agent list:

```json
"agents": {
  "defaults": {
    "workspace": "/home/node/.openclaw/workspaces/main"
  },
  "list": [
    {
      "id": "main",
      "name": "Marshal",
      "workspace": "/home/node/.openclaw/workspaces/main"
    },
    {
      "id": "personal",
      "name": "Ava",
      "workspace": "/home/node/.openclaw/workspaces/personal"
    }
  ]
}
```

What I like is that it makes the split durable.

This isn't "try to remember which tone to use."

It is separate agent identity, separate workspace, separate routing rules, and separate session histories.

That's a much stronger foundation.

## Separate workspaces were part of the point

Each agent gets its own workspace directory.

That matters because workspaces are not just folders. They're where the agent's local memory, notes, conventions, and operating context live.

In my setup:

- `main` uses `/home/node/.openclaw/workspaces/main`
- `personal` uses `/home/node/.openclaw/workspaces/personal`

Both workspaces can still see the mounted project repositories, but they do not have to behave like the same conversational surface.

I like that a lot.

It lets the system stay coherent without forcing every interaction through the same shared context.

## The channel bindings made the split real

The cleanest part of the setup is the routing section:

```json
"bindings": [
  {
    "type": "route",
    "agentId": "personal",
    "match": {
      "channel": "signal"
    }
  },
  {
    "type": "route",
    "agentId": "main",
    "match": {
      "channel": "msteams"
    }
  }
]
```

That means:

- Signal routes to `personal`
- Teams routes to `main`

I like this because it reflects how I actually use those channels.

Signal is more personal and direct. Teams is more operational, thread-oriented, work-shaped. The routing isn't arbitrary — it maps the social surface to the agent surface.

## The rest of the config reinforces the separation

A few other config details matter more than they might look.

One is:

```json
"session": {
  "dmScope": "per-channel-peer"
}
```

That keeps DM scope tied to the peer within each channel, which makes the session model predictable. A Signal DM and a Teams DM aren't secretly the same conversation just because they involve the same human.

Another is the Teams reply behavior:

```json
"replyStyle": "thread"
```

It means routing isn't just "channel = Teams." The conversation can carry thread structure too, which becomes very relevant once background tasks and reminders need to find their way back to the right place.

## The model defaults stay shared, but the identity does not

Both agents inherit the same broad model strategy from `agents.defaults`: cloud-first, with fallbacks, and Ollama available as the local last resort.

I think that's the right trade.

I didn't need two completely different intelligence stacks. I needed two differently situated agents.

That's a useful distinction.

Too many AI systems try to create variety through persona language alone. I trust structural differences more than stylistic ones.

## This setup made later routing bugs easier to diagnose

One side effect of the split is that it made the reminder-delivery bug much easier to reason about later.

Because the system was already explicit about:

- which agent owned which channel
- which workspace belonged to which agent
- which session route belonged to which conversation

it became much easier to spot when a background reminder job had lost the routing context it needed.

In other words, the multi-agent setup wasn't the source of the bug.

It was part of what made the bug diagnosable.

That's a nice example of why explicit structure pays off.

## My takeaway

I don't think every AI setup needs multiple agents.

But if you're going to split them, I think the split should show up in real places:

- workspaces
- routing rules
- session boundaries
- permissions
- operational expectations

Otherwise you're mostly decorating one assistant instead of structuring a system.

For this setup, the `main`/`personal` split was just enough structure to feel useful without becoming ceremonial.

And it set up the final problem in this series perfectly: what happens when a reminder job is created in one conversation but later forgets how to get back there.

Next in the series: [Why My OpenClaw Reminders Weren't Reaching Signal or Teams](/blog/2026/04/04/why-my-openclaw-reminders-werent-reaching-signal-or-teams/).
