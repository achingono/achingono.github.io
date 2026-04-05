---
author: "Alfero Chingono"
title: "Why I Run OpenClaw in Docker on My Own Machine"
date: 2026-03-05T09:00:00Z
draft: false
description: "Why I chose to run OpenClaw in Docker instead of directly on the host: cleaner dependencies, repeatable rebuilds, durable state, and a setup that plays nicely with tunnels and self-hosted services."
slug: why-i-run-openclaw-in-docker-on-my-own-machine
tags: [
"OpenClaw",
"Docker",
"Self-Hosted AI",
"Platform Engineering",
"Automation"
]
categories: [
"Agentic AI",
"Platform Engineering",
"Build in Public"
]
image: "cover.png"
---

This is the first post in a short series on how I run OpenClaw. It sits earlier than the rest because it is really about the initial Docker Compose setup, before Signal and Teams entered the picture.

I didn't choose Docker because it's fashionable. I chose it because I wanted the boring things to stay boring.

## I wanted repeatability more than cleverness

OpenClaw in my setup is not just "one process on one machine."

It's a local agent gateway, a browser-based editor, an Ollama runtime for local models, a CLI container, persistent state, channel integrations, and a reverse tunnel that exposes selected services through a remote nginx entry point.

That's already enough moving parts that I didn't want to manage them as a pile of host-level installs.

I have done that sort of thing before. It works right up until:

- one package wants a different runtime version
- one upgrade changes behavior in a way you didn't expect
- one machine setup detail becomes tribal knowledge
- one rebuild turns into a half-day archaeology project

Docker was the easiest way to say: this stack should come up the same way every time.

## I wanted the odd dependencies contained

The biggest reason I didn't want a host-first install was that OpenClaw was going to talk to real channels.

That meant the environment needed more than just "run the app."

It needed things like:

- `signal-cli-native` for Signal
- extra Node modules for the Teams side
- GitHub CLI and SSH tooling because the agent does real repo work
- a stable place for OpenClaw config, workspace state, and channel data

None of that is impossible to manage directly on the host. I just didn't want those concerns smeared across the machine.

I wanted the container image to be the integration boundary. That gave me a much cleaner mental model:

- the host provides Docker, storage mounts, and network access
- the image defines the runtime and channel dependencies
- compose defines how the services fit together

That boundary is much easier to reason about when something breaks.

## I still wanted persistent state

One thing I don't like about naive container setups is when everything becomes disposable except the part you accidentally needed.

I didn't want that here.

So the setup is intentionally a mix of **ephemeral runtime** and **durable mounted state**.

The important state lives outside the container:

- OpenClaw config and session state
- agent workspaces
- Signal data
- Ollama model data

I can rebuild or replace the image when I need to, but I don't lose the identity of the system every time I do it. The agents still have their workspaces. The channel state is still there. The local models are still there.

That's the behavior I wanted from the start.

## Compose matched the shape of the system

The Docker Compose layout also ended up matching how I think about OpenClaw operationally.

I'm not really running "one container." I'm running a small local AI gateway stack:

- **gateway** for the main OpenClaw runtime and channel/webhook entry points
- **cli** for interactive management using the same image and runtime assumptions
- **editor** for browser-based VS Code access
- **ollama** for local inference
- **ollama-init** to pull the local models once and get out of the way

That separation is useful.

The gateway owns the channel-facing behavior. The CLI shares the network namespace so it can talk to the gateway naturally. The editor stays focused on workspace access. Ollama remains local and isolated. Nothing here feels overloaded.

## Running it locally did not mean exposing it sloppily

One reason I like this architecture is that it keeps the heavy lifting on my own machine while still letting me expose selected services in a controlled way.

The public entry point is not "open every container port to the internet."

Instead, the local stack stays behind Docker, and a reverse SSH tunnel forwards only the ports I explicitly want exposed to a remote VM. nginx on that VM handles TLS termination, routing, and upstream auth.

That design worked well with Docker because it kept the local machine focused on running the real services while the remote VM handled the public edge concerns.

If I'd built this out of ad hoc host installs, I'd probably have ended up with a messier boundary between "local runtime" and "public ingress."

## The trade was absolutely worth it

Docker does add a little ceremony.

You have to think about volumes. You have to think about ports. You have to think about image contents and rebuilds. And once you start adding channel integrations, that image becomes part of the product, not just part of the packaging.

But for this kind of system, I think that is the right trade.

I would much rather spend time shaping one explicit container image and one compose stack than wonder which random host package made the environment drift.

For me, Docker made OpenClaw feel less like a clever experiment and more like an actual service I could live with.

Next in the series: [How I Wired Signal and Microsoft Teams into a Custom OpenClaw Image](/blog/2026/03/15/how-i-wired-signal-and-microsoft-teams-into-a-custom-openclaw-image/).
