---
author: "Alfero Chingono"
title: "Why My OpenClaw Reminders Weren't Reaching Signal or Teams"
date: 2026-04-04T21:23:46Z
draft: false
description: "A small routing detail broke OpenClaw reminder delivery in exactly the wrong place. Here's how I traced it, fixed same-channel delivery, and stopped the agent from claiming success when scheduling had actually failed."
slug: why-my-openclaw-reminders-werent-reaching-signal-or-teams
tags: [
"OpenClaw",
"AI Agents",
"Signal",
"Microsoft Teams",
"Automation",
"Debugging"
]
categories: [
"Agentic AI",
"Automation",
"Platform Engineering"
]
image: "cover.png"
---

This is the fifth and final post in my short OpenClaw series. If you want the background first, read [why I run OpenClaw in Docker](/blog/2026/03/05/why-i-run-openclaw-in-docker-on-my-own-machine/), [how I wired Signal and Teams into a custom image](/blog/2026/03/15/how-i-wired-signal-and-microsoft-teams-into-a-custom-openclaw-image/), [the Dockerfile walkthrough](/blog/2026/03/15/inside-the-dockerfile-behind-my-openclaw-gateway/), and [the main/personal agent split](/blog/2026/04/03/how-i-split-openclaw-into-main-and-personal-agents/).

I ran into an OpenClaw bug this week that annoyed me more than a normal delivery failure would have.

A reminder was supposed to come back to me later telling me to submit a GO Transit delay claim. The agent said it had scheduled the reminder. Then later... nothing showed up in Signal. Nothing showed up in Teams either.

That kind of bug is worse than a simple send failure because it creates false confidence. If a reminder system quietly drops the reminder, that's bad. If it tells you the reminder is set when it actually isn't, that's much worse.

The fix turned out to be pretty specific: reminder jobs needed to preserve the **originating session route**, not just "some channel" to send to later.

## The symptom wasn't just "Signal is broken"

The first thing I did was inspect the stored cron jobs and their run logs.

I found two failed reminder jobs in OpenClaw's cron state. Both had ended with the same kind of error:

```text
Error: Signal RPC -1: Failed to send message
```

At first glance, that looks like a Signal transport problem. But that theory fell apart pretty quickly.

I sent a proactive Signal message directly through OpenClaw's normal outbound path, outside cron, and it worked:

```bash
openclaw session send --agent personal --message "test delivery"
```

That mattered a lot.

It meant Signal itself was healthy. The account was connected. The RPC path was fine. Proactive outbound delivery was possible. So the bug wasn't "OpenClaw can't send Signal messages." The bug was narrower: **cron-delivered reminders were failing in their announce/delivery path**.

## The failure was hiding behind a second bug

While tracing the original conversation, I found something even more frustrating.

The first time the agent tried to create the reminder, it used invalid CLI flags. Then it tried again with another invalid form. Then it hit a gateway error. After that, instead of telling the user scheduling had failed, it wrote notes into `HEARTBEAT.md` and `MEMORY.md` and still acted as if the reminder had been set.

That is not a reminder. It is a private note pretending to be one.

Later in the same session, the agent finally did create a real cron job. So there were actually two different problems:

1. the agent could falsely claim success after `cron add` failed
2. even when a real reminder job existed, delivery could still fail later

I wanted both fixed.

## The real clue was in the job metadata

The failed Signal reminder job looked roughly like this:

```json
{
  "delivery": {
    "mode": "announce",
    "channel": "signal",
    "to": "uuid:..."
  },
  "agentId": null,
  "sessionKey": null
}
```

That looked suspicious immediately.

OpenClaw reminder jobs run in an isolated cron session. That isolation is useful, but it also means the job needs enough routing context to find its way back to the human who asked for it.

The important detail here is that "send a reminder later" is not just content generation. It's also a routing problem.

The reminder has to know:

- which agent owns the conversation
- which exact session started the request
- which channel to send back to
- which exact recipient or thread target to use

Without that information, the agent can successfully generate the reminder summary and still fail at the final delivery step.

That's exactly what was happening.

The cron run succeeded at the AI part. It produced a perfectly good reminder summary. Then the last hop failed.

## Same-channel delivery was the right rule

There was one behavioral clarification that mattered a lot here: reminders should go back to the **channel that initiated the request**, not to every connected channel.

That sounds obvious once you say it out loud, but it changes the design.

If I asked for a reminder in Signal, I want it back in that Signal conversation.

If I asked for it in a Teams chat or thread, I want it back there.

I don't want a reminder system that gets "helpful" and starts spraying notifications across every channel it knows about. That's not smarter. That's just noisier.

This also means the exact route matters:

- Signal direct chats may use UUID-form targets
- Teams direct chats and Teams channel threads have different route shapes
- a thread target is not interchangeable with a generic user target

So "channel = signal" or "channel = msteams" is not enough by itself. The job has to preserve the full conversation identity.

## Healthy jobs were already telling me what the fix should be

One of the most useful clues came from looking at working reminder jobs that were already stored in the system.

Existing Teams reminder jobs already had both of these fields populated:

```json
"agentId": "main",
"sessionKey": "agent:main:msteams:channel:..."
```

That was the pattern the broken Signal jobs were missing.

Once I saw that, the problem became much clearer:

- healthy jobs preserved the originating session route
- broken jobs only had partial delivery info
- cron isolation meant partial delivery info was not reliable enough

So the fix wasn't "retry Signal harder."

The fix was "preserve the route that the reminder belongs to."

## What I changed

I made two concrete changes.

### 1. I updated the agent workspace instructions

I added explicit reminder-delivery rules to both the `main` and `personal` agent workspaces so future reminder jobs must:

- send only to the chat that asked for the reminder
- include `--agent <current-agent-id>`
- include `--session-key <current-session-key>`
- reuse the exact `--channel` and `--to` from the current session
- tell the user plainly if `cron add` fails

The practical shape now looks like this:

```bash
openclaw cron add \
  --name "..." \
  --at "..." \
  --announce \
  --agent main \
  --session-key agent:main:signal:direct:uuid:<recipient> \
  --channel signal \
  --to uuid:<recipient> \
  --message "..."
```

If the reminder started in Teams, the same rule applies with the Teams session key and exact Teams target.

### 2. I repaired the already-failed jobs

I patched the stored failed Signal reminder jobs so they now carry the missing `agentId`, `sessionKey`, and explicit destination metadata.

First I listed the jobs to identify the broken ones:

```bash
openclaw cron list
```

Then I edited each failed job to restore the routing context:

```bash
openclaw cron edit <job-id> \
  --announce \
  --agent personal \
  --session-key agent:personal:signal:direct:uuid:<recipient> \
  --channel signal \
  --to uuid:<recipient>
```

That gave me a clean way to test the actual failing path instead of just assuming the theory was right.

## The result

After repairing the routing metadata, I reran the missed transit-claim reminder job:

```bash
openclaw cron run <job-id>
```

Previously its run log ended like this:

```text
status: error
deliveryStatus: unknown
Error: Signal RPC -1: Failed to send message
```

After the fix, the rerun finished like this:

```text
status: ok
deliveryStatus: delivered
delivered: true
```

That one successful rerun told me a lot.

It confirmed that:

- the reminder content generation path was fine
- Signal itself was fine
- the broken piece was the missing session routing metadata
- preserving the original route fixed the actual production failure

Because it was a one-shot reminder, the job then removed itself after succeeding, which is exactly what I wanted.

## The bigger lesson

I've been spending a lot of time thinking about multi-agent systems, orchestration, and tool contracts lately, and this bug fit that theme perfectly.

In agent systems, routing context is not incidental metadata. It's part of the work.

If a background task is supposed to come back to a human later, then "who asked for this?" and "where should it return?" are first-class data, not optional fields you can fill in later if you feel like it.

The other lesson is even simpler: **don't claim automation succeeded unless it actually did**.

That sounds almost embarrassingly obvious, but it's the kind of failure mode that destroys trust very quickly. A failed reminder should be reported as failed. A note in memory isn't a substitute for delivery. And a scheduler shouldn't quietly lose the route back to the person who asked for the work.

Small bug. Important fix.

And honestly, those are often the bugs worth writing about.

If you're interested in the broader design side of this kind of system, I wrote more about [why I started building my own DevOps platform](/blog/2025/02/15/why-i-started-building-my-own-devops-platform-and-what-i-learned/), [what MCP changed for me in practice](/blog/2025/03/20/mcp-in-practice-what-anthropics-model-context-protocol-actually-means-for-developers/), and [what building a real multi-agent system taught me about roles, boundaries, and orchestration](/blog/2025/08/28/designing-multi-agent-systems-lessons-from-building-an-8-agent-engineering-orchestra/).
