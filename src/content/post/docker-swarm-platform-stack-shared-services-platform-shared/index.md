---
author: "Alfero Chingono"
title: "Docker Swarm Platform Stack: Shared Postgres, LiteLLM, Ollama, Redis, Registry, and Playwright"
date: 2026-06-30T09:00:00Z
draft: true
description: "The shared-services writeup: what lives in the `platform` stack, how service placement and networking are handled, and which rollout problems pushed me toward a stricter deployment model." 
slug: docker-swarm-platform-stack-shared-services-platform-shared
tags: [
  "docker",
  "swarm",
  "postgres",
  "litellm",
  "ollama",
  "redis",
  "traefik",
  "playwright"
]
categories: [
  "Infrastructure",
  "Platform Engineering"
]
image: "cover.png"
---

The `platform` stack is where I stopped letting each application pretend it was self-sufficient.

The stack file is `deploy/platform/docker-stack.yml`.

It provides:

- Postgres
- LiteLLM
- Ollama
- Redis
- private registry
- shared Playwright run-server

That service list is not abstract platform-language. Those were the things that kept showing up across app stacks and were easier to run once.

## The network is deliberately boring

Everything rides the same encrypted overlay network:

- `platform_shared`
- driver `overlay`
- `attachable: false`
- encryption enabled

The `attachable: false` choice is useful. Swarm services can join the network across stacks, but random standalone containers cannot.

That gives the platform layer a cleaner boundary.

## Why I became strict about node placement

The platform stack uses explicit placement constraints because the cluster hardware is not uniform and state is not free to wander.

From the stack file:

- Postgres: `node.labels.data == true`
- LiteLLM: worker nodes
- Ollama: `node.labels.ai == true`
- registry: manager node
- Redis: worker nodes

The point is not elegance. The point is removing ambiguity.

Earlier docs in the repo call out a move away from hostname-based placement toward role- and label-based placement. That was the right move. Hostnames are an incidental property. Roles and labels are operational intent.

## Postgres: centralize it, then design around it

The shared Postgres service uses `pgvector/pgvector:pg16` and a bootstrap SQL config mounted through Docker configs.

That buys a few things immediately:

- one place for relational state
- one place for vector support
- one place for backup and recovery policy

It also changes the app stacks. They no longer need their own database story. They need a database contract.

## LiteLLM: one gateway for model traffic

LiteLLM is configured with:

- shared Postgres
- shared Redis
- Docker secrets for provider keys and master key
- rendered config files from `deploy/platform/config/`

The important engineering benefit is centralization of routing and credentials.

Without that, every app stack gradually accumulates its own half-solution for provider selection, fallback, and secret handling.

## Ollama and the hardware reality

Ollama is one of the places where homelab hardware stops being a novelty and starts becoming an input to platform design.

Not every node should run model workloads. That is why the stack uses an AI label instead of assuming all workers are equally suitable.

On paper, that sounds obvious. On a cluster made from mixed machines, it becomes necessary.

## Redis and shared browser automation

Redis is unglamorous, which is a compliment.

It is here because shared caches, queues, and broker-like workflows should not be reinvented by app stacks.

The Playwright run-server exists for the same reason. Browser automation is expensive enough operationally that I would rather host it once and let apps consume it as a capability.

## The private registry taught the most expensive lesson

The private registry in this environment is simple and local. That simplicity comes with a sharp edge.

If you rely on `:latest`, worker nodes can keep running a cached image even after you believe you rolled out a new build.

This is why `deploy/scripts/build.sh` pushes uniquely tagged images and why the per-app deploy scripts learned to prefer immutable tags.

Representative usage:

```bash
IMAGE_TAG=<tag> ./deploy/scripts/build.sh firefly server client
```

That behavior is not a style preference. It is a response to debugging sessions where “the new version is deployed” and “all nodes are running the same image” were not the same statement.

## The platform contract for app stacks

Once this stack is healthy, app stacks can depend on a few stable service names and ports:

- `platform_postgres:5432`
- `platform_litellm:4000`
- `platform_playwright:6000` for browser automation consumers

That contract is the real product of the stack.

It narrows the amount of infrastructure each app has to describe.

## What I verify before blaming an app

When something looks broken upstream, these are the checks that matter first:

```bash
docker stack services platform
docker service ps platform_postgres --no-trunc
docker service ps platform_litellm --no-trunc
docker service logs platform_litellm --tail 100
```

If platform is unhealthy, application symptoms are usually downstream noise.

That is another reason the stack split helps. It gives failures an order.

First platform. Then apps.
