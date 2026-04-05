---
author: "Alfero Chingono"
title: "How I Wired Signal and Microsoft Teams into a Custom OpenClaw Image"
date: 2026-03-15T13:00:00Z
draft: false
description: "How I got OpenClaw talking to Signal and Microsoft Teams from one custom Docker image, and why the base image needed a few extra runtime pieces to make that practical."
slug: how-i-wired-signal-and-microsoft-teams-into-a-custom-openclaw-image
tags: [
"OpenClaw",
"Docker",
"Signal",
"Microsoft Teams",
"Self-Hosted AI"
]
categories: [
"Agentic AI",
"Platform Engineering",
"Automation"
]
image: "cover.png"
---

In the [first post](/blog/2026/03/05/why-i-run-openclaw-in-docker-on-my-own-machine/) I explained why I wanted Docker as the foundation. This one is the next practical problem: how to get OpenClaw talking to **Signal** and **Microsoft Teams** without turning the host machine into a dependency junk drawer.

The short version is that I ended up building a custom image because the base runtime got me close, but not all the way there.

## Signal and Teams were different kinds of problems

Signal and Teams pushed on different parts of the system.

Signal is much more of a runtime-tooling problem.

You need a working Signal CLI runtime in the container and a persistent place to keep Signal state. If the container can send Signal messages but the identity disappears on rebuild, you have not actually solved the problem.

Teams is more of a Node/runtime integration problem.

It needs the right hosting support in the image, the right webhook port exposed, and the right application credentials sitting in OpenClaw config so the bot framework side can talk to Microsoft properly.

I didn't want to solve those two things in two completely different operational styles.

So I chose one image and one compose layout that could support both.

## Why I did not stop at the base OpenClaw image

The base OpenClaw image was a good starting point, but I needed more in the environment:

- `signal-cli-native` installed in the container
- GitHub CLI and SSH tooling for the agent's repo workflows
- an extra Node modules path for additional runtime packages
- a clean way for both the gateway container and the CLI container to share the same capabilities

That led to a custom image tagged locally as `openclaw-local:teams`.

The name reflects where I started operationally, but the important part is not the tag. The important part is that the image became the place where I declared, very explicitly, "this is the OpenClaw runtime I actually depend on."

## How I handled Signal

For Signal, the key decision was to keep the runtime inside the image and the account state outside it.

The image installs `signal-cli-native`, which gives the container the actual tool it needs to send and receive Signal messages.

Then the compose file mounts the Signal data directory into:

```text
/home/node/.local/share/signal-cli
```

That was the right split for me:

- the image owns the executable
- the volume owns the durable Signal identity

This matters more than it sounds.

If the image knows how to run Signal but the identity is trapped inside a replaceable container layer, every rebuild becomes risky. If the state is mounted and durable, rebuilds are much less dramatic.

On the OpenClaw side, the Signal channel is configured with pairing and allowlists so the bot is not just open to the world. That let me keep Signal useful without letting it become an uncontrolled ingress point.

## How I handled Teams

Teams had a different shape.

The container needed the extra Node package support for the Microsoft hosting layer, and the gateway needed to expose the Teams webhook port. In my setup that means port `3978` is published by the gateway container so the remote edge can forward `/api/messages` traffic back to the local OpenClaw runtime.

The actual Teams app credentials live in OpenClaw config, not in the image. That's exactly where I want them.

The image should describe runtime capability.

The config should describe environment-specific identity.

That boundary kept the setup much easier to move, rebuild, and reason about.

## One image, two operational benefits

Using the same custom image for both `gateway` and `cli` gave me two benefits I really wanted.

First, it removed "works in one container but not the other" drift.

If the gateway can use the runtime, the CLI can too. If the CLI can inspect or patch something, it is doing so in the same environment the gateway actually uses. I've learned to value that kind of consistency a lot.

Second, it let me keep the OpenClaw-specific runtime tweaks in one place:

- the extra Node modules path
- the installed system packages
- Signal CLI
- GitHub CLI
- pnpm-prepared extras

It's a much nicer maintenance story than trying to remember which bits live on the host, which belong to the container, and which only exist in some forgotten shell session.

## The compose file completed the picture

The image by itself was not enough. The compose file is what turned it into a working system.

That is where I defined the things that make the setup feel real:

- mounted OpenClaw config
- mounted workspaces
- mounted Signal data
- shared access to source repositories
- local Ollama dependency
- editor access
- network sharing between the gateway and CLI

This is one reason I still like Docker Compose for personal infrastructure. It doesn't just run containers. It describes the operating assumptions of the stack in one place.

## The setup was already hinting at the routing story

Even at this stage, there was a lesson hiding in plain sight: getting the channels working is the easy part. Once one runtime can talk to Signal and Teams, the questions come quickly: which agent answers where, what state belongs to which workspace, and how does a background task find its way back to the right chat? That's where the series goes next.

Next in the series: [Inside the Dockerfile Behind My OpenClaw Gateway](/blog/2026/03/15/inside-the-dockerfile-behind-my-openclaw-gateway/).
