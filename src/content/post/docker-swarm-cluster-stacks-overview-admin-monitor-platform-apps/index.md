---
author: "Alfero Chingono"
title: "Docker Swarm in Stacks: Admin, Monitor, Platform, and Apps"
date: 2026-08-20T09:00:00Z
draft: true
description: "A practical writeup of the Docker Swarm layout I ended up with after turning two broken-screen Surface Pros into worker nodes and learning the hard parts of networking, routing, and rollout discipline." 
slug: docker-swarm-cluster-stacks-overview-admin-monitor-platform-apps
tags: [
  "docker",
  "swarm",
  "traefik",
  "wireguard",
  "homelab",
  "networking"
]
categories: [
  "Infrastructure",
  "Platform Engineering"
]
image: "cover.png"
---

This cluster exists because I did not want two broken-screen Surface Pro laptops to become e-waste.

Both machines were still fine as computers. They were just bad laptops. Once Debian was on them and the pointless parts of “being a laptop” were stripped away, they became useful worker nodes.

That sounds cleaner than it was.

The install work was annoyingly physical: external keyboard, external display, wired networking, boot media, firmware quirks, and a lot of trial-and-error around turning consumer hardware into something that can sit on a shelf and behave like infrastructure.

The part that made it worth doing was the moment those machines stopped feeling like salvaged hardware and started acting like real cluster members.

## The layout that survived contact with reality

The final shape is a three-node Swarm plus one separate monitoring host:

- `walnut`
  - manager
  - shared ingress edge inside Swarm
  - Portainer
  - shared auth services
  - private registry
  - shared platform services that need tighter placement
- `cherry`
  - worker
  - repurposed Surface Pro
- `teak`
  - worker
  - repurposed Surface Pro
- `pecan`
  - monitoring host
  - Beszel hub outside the Swarm

There is also a small public-facing edge VM that terminates TLS and forwards traffic over a private tunnel into the Swarm.

That split did not come from theory. It came from trying to keep one concern from destabilizing another.

## Why I split the cluster into four stack layers

The repo now has four layers of concern:

1. `admin`
2. `monitor`
3. `platform`
4. application stacks

In repo terms, `admin` is the `management` stack. The naming matters less than the boundary.

I needed one place to solve shared routing and identity. I needed one place for monitoring. I needed one place for databases, queues, model routing, and browser automation. Then I needed app stacks to stay mostly boring.

Without those boundaries, every deployment turns into a small integration project.

## The problem that forced the first serious redesign

The most important early failure was overlay networking.

The worker nodes had multiple network identities at different points during setup. Swarm joined them successfully, but the overlay network was not actually healthy. The symptom was straightforward and brutal: containers scheduled on different nodes could not talk to each other over the overlay, even though the machines themselves were reachable on the LAN.

The repo has a full troubleshooting trail for this. The failure chain was:

- the workers had stale or competing addresses
- Swarm had learned one address
- the kernel was sourcing packets from another
- cross-node overlay traffic was dropped

The fix was not a Docker command by itself. It was host hygiene.

I standardized on wired static addresses with NetworkManager, rejoined the nodes, and updated the route-preference logic used during Swarm bootstrap so the intended address was also the address the kernel preferred for that path.

The relevant docs and scripts are all in the repo:

- `docs/deployment/network-configuration.md`
- `docs/troubleshooting/overlay-network-connectivity-walkthrough.md`
- `scripts/remote/run-manager-setup.sh`
- `scripts/remote/join-worker.sh`

Representative checks looked like this:

```bash
docker node ls
docker info | grep 'Node Address'
ip addr show
ip route show
```

That was the point where the project stopped being “install Docker on three boxes” and became “treat the network as part of the system.”

## The public request path

The public path is intentionally simple:

1. browser hits `https://<host>`
2. edge VM terminates TLS
3. request crosses the tunnel to the manager
4. in-Swarm Traefik routes by `Host()`
5. request lands on either an auth gateway or an app service

Only one service in the management stack publishes a port inside Swarm: the Traefik edge on port `80`, published in host mode.

That detail is not cosmetic. It was part of fixing forwarded-header behavior so downstream auth services could correctly treat the original request as HTTPS.

## The stack boundaries in practice

### `admin`

Shared routing and identity:

- Traefik edge
- Portainer
- directory service
- OIDC provider
- oauth2-proxy

### `monitor`

Monitoring is split deliberately:

- Beszel hub on `pecan`
- Beszel agent as a global Swarm service
- oauth2-proxy for the public dashboard path

### `platform`

Shared runtime services:

- Postgres
- Redis
- LiteLLM
- Ollama
- private registry
- Playwright run-server

### application stacks

Each app deploys as its own stack with its own env, configs, secrets, and Traefik labels.

That part now looks simple because the layers underneath it are doing their job.

## The commands I actually use

Cluster bring-up and management stack:

```bash
./setup.sh preflight
./setup.sh bootstrap-swarm
./setup.sh deploy-stack
./setup.sh configure-ingress
./setup.sh verify
```

Application deploys:

```bash
./deploy/emanate/scripts/deploy.sh --build
./deploy/doughray/scripts/deploy.sh --build
./deploy/firefly/scripts/deploy.sh --build
./deploy/leankernel/scripts/deploy.sh --build
```

Monitoring:

```bash
./deploy/monitor/scripts/set-agent-creds.sh --bootstrap-admin
./deploy/monitor/scripts/deploy.sh
```

## What changed after the stack model settled down

Before the split:

- routing changes leaked into app work
- auth behavior was hard to reason about
- rollouts were inconsistent
- network failures were harder to isolate

After the split:

- failures had a place to live
- docs mapped to real files and scripts
- deploy scripts became reusable instead of ceremonial
- the old Surface Pros stopped feeling fragile

That last point was the fun part.

Repurposed hardware is exciting right up until it misbehaves. Then it becomes engineering again.

The rest of this series covers the specific layers, the failure modes inside each one, and the fixes that made the cluster predictable.
