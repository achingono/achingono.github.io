---
author: Alfero Chingono
title: How I Wired Signal and Microsoft Teams into a Custom OpenClaw Image
date: 2026-03-15T13:00:00.000Z
description: How I got OpenClaw talking to Signal and Microsoft Teams from one custom Docker image, and why the base image needed a few extra runtime pieces to make that practical.
slug: how-i-wired-signal-and-microsoft-teams-into-a-custom-openclaw-image
tags:
  - OpenClaw
  - Docker
  - Signal
  - Microsoft Teams
  - Self-Hosted AI
categories:
  - Agentic AI
  - Platform Engineering
  - Automation
image: cover.png
---

I didn't start with “one neat image” here. I started with a mess of runtime needs that were pulling in different directions.

Signal wanted a durable CLI runtime and a place to keep identity state. Teams wanted the right Node hosting pieces, a webhook port, and credentials that lived in config instead of inside the image. If I had split those into separate ad hoc setups, I would have ended up debugging my own infrastructure instead of using it.

So I made the image do the runtime work and the compose file do the state work.

## Signal needed a stable runtime, not just a container that could send a message

For Signal, the container needed `signal-cli-native`. That part is straightforward. The less obvious part is persistence. If the account state disappears every time the container is rebuilt, then the setup is fragile no matter how clean the Dockerfile looks.

That is why the compose file mounts the Signal data directory into:

```text
/home/node/.local/share/signal-cli
```

That split feels small, but it changes the whole system:

- the image owns the executable
- the volume owns the identity

That is the kind of boundary I look for now. If state is replaceable and the binary is reproducible, the setup is easier to trust.

## Teams needed the image to behave like a real hosting environment

Teams pushed on a different seam.

The container needed the Microsoft hosting layer support, and the gateway had to expose port `3978` so the remote edge could forward `/api/messages` traffic into the local runtime. The Teams credentials stayed in OpenClaw config, which is where I wanted them. I did not want credentials baked into the image, because the image should describe capability, not environment-specific identity.

That distinction ended up being the difference between “this kind of works on my machine” and “this is a system I can rebuild without rereading my own notes.”

## The custom image was really about reducing drift

The base OpenClaw image got me close, but not all the way there. I needed:

- `signal-cli-native`
- GitHub CLI and SSH tooling
- extra Node module support
- pnpm-prepared extras
- one runtime shape that both `gateway` and `cli` could share

Once I put those pieces into `openclaw-local:teams`, the operational story got simpler. The gateway and CLI stopped being two slightly different environments with slightly different failure modes.

That matters more than it sounds. A lot of container problems are really drift problems.

## The compose file is where the system became legible

The image alone was not the whole answer. Compose is what made the setup readable:

- mounted OpenClaw config
- mounted workspaces
- mounted Signal data
- shared access to source repos
- local Ollama dependency
- editor access
- network sharing between gateway and CLI

That is the part of Docker Compose I still like. It does not just run things; it shows the operating assumptions.

If someone asks me how this setup works, I can point to the compose file and say: this is the runtime, this is the state, this is the boundary between them.

## The review question I kept asking

The real question was never “can I make Signal and Teams work?” It was “can I make the trust and state boundaries obvious enough that I still understand this setup after a week away from it?”

That is the standard I keep coming back to.

The image should be boringly capable. The config should be explicit. The volumes should hold the durable parts. And the routing story should stay simple enough that I can explain it without inventing a diagram on the spot.

The next problem in the series is routing: once multiple channels are live, which agent answers where, what state belongs to which workspace, and how does a background task get back to the right chat?
