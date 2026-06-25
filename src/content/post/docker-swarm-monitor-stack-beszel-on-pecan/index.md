---
author: "Alfero Chingono"
title: "Docker Swarm Monitor Stack: Beszel on a Dedicated `pecan` Host"
date: 2026-09-03T09:00:00Z
draft: true
description: "The monitoring writeup: why the Beszel hub lives off-cluster, how the global agent stack is wired, and which operational shortcuts were worth automating." 
slug: docker-swarm-monitor-stack-beszel-on-pecan
tags: [
  "docker",
  "swarm",
  "monitoring",
  "beszel",
  "oidc",
  "oauth2-proxy"
]
categories: [
  "Infrastructure",
  "Observability"
]
image: "cover.png"
---

I did not want the monitoring dashboard to disappear just because the cluster itself was having trouble.

That decision is the whole shape of this stack.

The Beszel hub runs on `pecan` as a native systemd-managed service. Swarm only runs:

- the Beszel agent as a global service
- oauth2-proxy for the public dashboard path

## Why the hub is outside the cluster

There are two practical reasons.

First, monitoring should not share every failure domain with the thing it is monitoring.

Second, the hub is not a placement problem. It is a stable service problem. A small dedicated host is a better fit.

That is why the repo has a separate install path for `pecan`:

- `deploy/monitor/scripts/install-pecan.sh`

## What the Swarm stack actually does

The stack file is `deploy/monitor/docker-stack.yml`.

It has only two services.

### `beszel-agent`

The agent runs with:

- `network_mode: host`
- read-only Docker socket mount
- local state volume
- `deploy.mode: global`

That last setting matters more than replicas would. I do not want “one metrics agent somewhere.” I want one agent per Swarm node.

The core runtime inputs are:

- `HUB_URL`
- `LISTEN`
- `KEY`
- `TOKEN`

### `monitor-oauth2-proxy`

This service is attached to the external `management` overlay network and exposed through Traefik with a standard `Host()` rule.

It reuses the same auth model as the rest of the cluster:

- shared OIDC provider
- shared client secret
- shared cookie secret pattern

That reuse was worth it. Monitoring is already operationally special. It did not need to become identity-special too.

## Static addressing still matters here

`pecan` is outside Swarm, but it is still part of the design and still needs stable network identity.

The networking docs in the repo make the larger point clearly: once addressing drifts, cluster behavior gets harder to trust.

That same lesson applies to the monitoring host. The Beszel hub is referenced by other systems. It is not the place for dynamic guesswork.

## The credential workflow that saved time

One useful bit of engineering in this stack is `deploy/monitor/scripts/set-agent-creds.sh`.

It does two jobs:

- accept a known Beszel agent key/token pair
- bootstrap an admin account on the hub and capture the generated credentials automatically

The bootstrap path matters because it removes a fragile manual loop:

1. open the UI
2. create admin
3. copy generated values
4. paste them into an env file
5. hope you did not transpose anything

The script closes that loop and writes `deploy/monitor/.env` directly.

That is a small improvement, but it is exactly the kind that makes a homelab cluster feel maintained instead of improvised.

## The deployment sequence

This is the path that actually works:

```bash
./deploy/monitor/scripts/install-pecan.sh
./deploy/monitor/scripts/set-agent-creds.sh --bootstrap-admin
./deploy/monitor/scripts/deploy.sh
```

After that:

```bash
docker stack services monitor
docker service ps monitor_beszel-agent --no-trunc
docker service logs monitor_monitor-oauth2-proxy --tail 50
```

## Why host networking is fine here

I am usually happy to keep things on overlay networks.

Monitoring agents are one of the exceptions.

They care about the host. They inspect the host. They expose host-relevant state. Running them with host networking keeps the path shorter and the assumptions simpler.

That tradeoff is worth stating plainly because “put everything on overlays” sounds cleaner than it actually is.

## What this stack avoids

By keeping the hub off-cluster, the monitor stack avoids three common problems:

- monitoring disappears with the cluster
- dashboard placement becomes part of Swarm scheduling
- auth for monitoring becomes a separate ad hoc system

Instead, the monitor layer ends up with a narrow contract:

- Swarm must run one agent per node
- the hub must remain reachable on the private network
- the public dashboard path must authenticate through the shared OIDC flow

That is a small surface area, and small surface areas are easier to keep alive.
