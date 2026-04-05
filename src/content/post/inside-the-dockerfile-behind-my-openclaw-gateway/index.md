---
author: "Alfero Chingono"
title: "Inside the Dockerfile Behind My OpenClaw Gateway"
date: 2026-03-15T17:00:00Z
draft: false
description: "A walkthrough of the Dockerfile I use for OpenClaw: why I extend the base image, what I add for Signal and Teams, and why the order of a few lines matters more than it looks."
slug: inside-the-dockerfile-behind-my-openclaw-gateway
tags: [
"OpenClaw",
"Docker",
"Dockerfile",
"Signal",
"Microsoft Teams"
]
categories: [
"Platform Engineering",
"Containers",
"Agentic AI"
]
image: "cover.png"
---

The first two posts in this series covered the Docker decision and the Signal/Teams channel integration. This one gets into the Dockerfile itself.

It's not very long, but it's doing more architectural work than its size suggests.

## I started from the OpenClaw base image on purpose

The first line matters:

```Dockerfile
FROM ghcr.io/openclaw/openclaw:latest
```

I didn't want to rebuild OpenClaw from scratch if I didn't have to.

The base image already gets me the core runtime. What I needed was an opinionated extension of that runtime for my own environment. So this Dockerfile is less "build a platform from zero" and more "declare the exact operational additions my setup needs."

That distinction keeps the file focused.

## The version arguments make the customization explicit

Very early in the file I keep a couple of version arguments:

```Dockerfile
ARG PNPM_VERSION=9.15.6
ARG SIGNALCLI_VERSION=0.14.1
```

I do this even when not every argument is fully threaded through every install step yet.

It makes the additions feel intentional instead of accidental, and it gives me a clean place to pin or update them over time without pretending the whole image is fully static.

## I keep extra Node runtime pieces out of the main install path

This block is one of the more important small decisions in the file:

```Dockerfile
ENV EXTRA_NODE_MODULES=/home/node/.openclaw-extra/node_modules \
    NODE_PATH=/home/node/.openclaw-extra/node_modules
```

That extra module path is how I keep the runtime additions separated from the base image's own install layout.

I like this because it keeps things layered:

- base image provides OpenClaw
- extra module path provides my environment-specific additions

It's cleaner than pretending I own the whole upstream install tree.

## The root phase is for system-level capability

After switching to `root`, the file installs the system packages the stack actually needs:

```Dockerfile
apt-get install -y --no-install-recommends \
    curl \
    jq \
    ca-certificates \
    gpg \
    openssh-client \
    libstdc++6 \
    zlib1g \
    libgcc-s1
```

This section isn't glamorous, but it matters.

These are the packages that make the runtime actually usable:

- `curl` and `jq` for scripting and diagnostics
- `gpg` and certificate tooling for package installs
- `openssh-client` because the agent works with repositories and tunnels
- the runtime libraries needed by installed binaries

I try hard not to let this layer become a junk drawer. If a package is there, I want to be able to explain why it exists.

## GitHub CLI belongs in the image because the agent actually uses it

I also install `gh` in the image.

It's one of those choices that looks unnecessary until you remember what OpenClaw is actually doing here. It's not just chatting. It's working with repositories, issues, and pull requests. I wanted the GitHub CLI available in the environment the agent actually runs in, not as some optional extra on the host.

That keeps the runtime consistent between the gateway and the CLI container, which matters.

## Signal CLI is not an afterthought

The Signal section is the clearest example of why this had to be a custom image:

```Dockerfile
RUN set -eux; \
    mkdir -p /etc/apt/keyrings; \
    curl -fsSL https://packaging.gitlab.io/signal-cli/gpg.key \
      | gpg --dearmor -o /etc/apt/keyrings/signal-cli.gpg; \
    echo "deb [signed-by=/etc/apt/keyrings/signal-cli.gpg] https://packaging.gitlab.io/signal-cli signalcli main" \
      > /etc/apt/sources.list.d/signal-cli.list; \
    apt-get update; \
    apt-get install -y --no-install-recommends signal-cli-native; \
    rm -rf /var/lib/apt/lists/*; \
    signal-cli --version
```

I wanted Signal to be a first-class runtime capability, not a hand-installed special case.

Once it's in the image, I know exactly where it comes from and which container owns it. Then I can mount the Signal state directory separately and let rebuilds stay rebuilds instead of becoming identity-loss events.

## Permissions matter more than people think

Before dropping privileges again, the Dockerfile prepares the runtime directories and fixes ownership for the non-root `node` user.

That kind of step is easy to skip when you're just trying to get it running. It's also the kind of step that bites you later when mounted state or runtime-generated files start colliding with user permissions.

I would rather be explicit here.

## The pnpm/corepack step has to happen before switching users

This is one of the details that looks small but is operationally important:

```Dockerfile
RUN corepack enable && corepack prepare pnpm@${PNPM_VERSION} --activate
```

I do that while still running as `root`.

That's deliberate. Those paths need write access, and I don't want to debug avoidable permission problems later. This is exactly the kind of detail that turns a Dockerfile from "technically valid" into "actually maintainable."

## I switch back to `node` for runtime behavior

Once the system-level setup is done, the file drops back to the non-root user and stays there for the runtime-facing steps.

That's the right default here. The container needs enough privilege to install what it needs at build time, but it doesn't need to run the application as root.

## The Teams-specific Node addition lives in the extra module path

The final runtime customization is this:

```Dockerfile
RUN pnpm add \
        --dir /home/node/.openclaw-extra \
        --save-exact \
        @microsoft/agents-hosting@1.3.1
```

That one line pretty much summarizes the whole Dockerfile philosophy.

I'm not forking OpenClaw. I'm not replacing the base image. I'm adding the exact extra runtime capability my environment needs, in a separate path, with a pinned version, after the base runtime is already in place.

That's the kind of extension model I trust more.

## The file is short because I wanted it to stay inspectable

I could have packed more into this image.

I could have added more helpers, more debugging tools, more convenience packages, more "while I'm here" installations.

I chose not to.

For this kind of system, I think a Dockerfile should be easy to read top to bottom and answer one question: **what does this runtime need that upstream doesn't already provide?**

In my case, the answer was:

- a few system tools
- GitHub CLI
- Signal CLI
- an extra Node path
- pnpm-managed Teams hosting support
- correct ownership and runtime defaults

That is enough.

Honestly, "enough" is one of the healthiest instincts you can have when building infrastructure for yourself.

Next in the series: [How I Split OpenClaw into Main and Personal Agents](/blog/2026/04/03/how-i-split-openclaw-into-main-and-personal-agents/).
