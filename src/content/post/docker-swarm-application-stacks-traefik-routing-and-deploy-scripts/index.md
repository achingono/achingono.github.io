---
author: "Alfero Chingono"
title: "Docker Swarm App Stacks: Traefik Host Routing + Per-App Deploy Scripts"
date: 2026-09-17T09:00:00Z
draft: true
description: "The application deployment writeup: consistent Traefik routing, shared auth assumptions, build helpers, and the Swarm-specific failure modes that changed how I roll out updates." 
slug: docker-swarm-application-stacks-traefik-routing-and-deploy-scripts
tags: [
  "docker",
  "swarm",
  "traefik",
  "deployment",
  "oidc",
  "oauth2-proxy"
]
categories: [
  "Infrastructure",
  "Platform Engineering"
]
image: "cover.png"
---

The application layer only became manageable after the rest of the cluster stopped changing shape.

Each app now follows the same broad pattern:

- stack-local services for that app
- Traefik labels for public exposure
- shared OIDC assumptions
- shared platform dependencies
- a deploy script under `deploy/<app>/scripts/deploy.sh`

That consistency matters because Docker Swarm has a few behaviors that are easy to misread if every app is doing something different.

## Routing is declared where the service lives

The public routing model is label-based and Host-based.

Typical pattern:

```yaml
traefik.enable: "true"
traefik.http.routers.<name>.rule: "Host(`<app-host>`)"
traefik.http.routers.<name>.entrypoints: "web"
traefik.http.services.<name>.loadbalancer.server.port: "<container-port>"
```

That keeps the public contract close to the service that owns it.

I prefer this to maintaining a separate routing registry because the stack file already answers the question, “is this service public?”

## Shared auth, app-local edges

Most browser-facing apps still use stack-local oauth2-proxy services while depending on the shared OIDC provider.

That is a useful compromise.

Shared identity gives me one place to manage user auth.

Stack-local proxies let each app own its public browser path cleanly, especially when cookie scope or callback behavior should stay tied to one hostname.

The per-stack docs in `swarm/docs/deployment/stacks/` make this very clear for DoughRay, Emanate, Firefly, and LeanKernel.

## What the deploy scripts actually do

The deploy scripts are not wrappers for the sake of wrappers.

They encode the operational steps that turned out to matter.

Using Emanate as a representative example, the deploy script:

- loads root `.env`
- loads stack-specific `.env` if present
- optionally builds and pushes images with `--build`
- warns if secret files are empty
- deploys remotely with `DOCKER_HOST=ssh://...`

Representative pattern:

```bash
DOCKER_HOST="ssh://${target}" \
  docker stack deploy --detach=true --prune -c docker-stack.yml emanate
```

LeanKernel adds one more layer because it syncs shared CLI state and mounted directories before deploy. That logic lives in `deploy/leankernel/scripts/deploy.sh`.

## The build helper fixed a real rollout problem

`deploy/scripts/build.sh` became necessary once I stopped trusting `:latest`.

The script:

- builds from each app's compose file
- tags images with a unique `IMAGE_TAG`
- pushes to the private registry
- prints the env values needed for deployment

That is the right level of automation here. It standardizes the dangerous parts without pretending release engineering needs a giant pipeline just to function.

## The Swarm behaviors that changed my rollout habits

### 1. `:latest` is not a deployment strategy

On this cluster, `:latest` caused enough ambiguity that it stopped being worth the convenience.

If one node has the old image cached and another pulls the new one, you can get a mixed rollout while telling yourself the deploy was successful.

Unique tags fixed that.

### 2. `docker stack rm` is not a clean slate

One of the better troubleshooting notes in the repo is the warning that removing a stack definition does not always kill already-running task containers the way people assume.

That matters because those orphaned containers can keep running with stale mounts or broken config references.

The fix was procedural:

- prefer controlled redeploys
- use `docker service update --force` when you need a fresh task
- do not assume “definition removed” means “old runtime state is gone”

### 3. Swarm configs are immutable

If an entrypoint script is shipped as a config and the content changes, the config name often needs to change too.

That sounds annoying because it is.

But it is better to encode that reality into deployment workflow than to pretend the old config mutated in place.

## Deployment order is part of the system

The clean order is:

1. admin layer
2. platform layer
3. app stacks

The reason is simple.

If routing or identity is broken, app deploy symptoms are misleading.

If platform is broken, app health checks do not mean much.

That order is reflected across the repo docs and scripts.

## What I verify after an app deploy

There is no magic smoke test, but there is a stable checklist:

```bash
docker stack services <stack>
docker service ps <stack>_<service> --no-trunc
docker service logs <stack>_<service> --tail 50
curl -Ik https://<app-host>
```

Then I verify the dependency-specific path:

- OIDC redirect works
- app health endpoint responds
- model traffic reaches LiteLLM
- browser automation reaches the shared Playwright service if the app uses it

## Why the app layer feels calmer now

The app layer did not become simpler because the apps got smaller.

It became simpler because the cluster finally had conventions that matched the problems it was actually having.

That is the pattern running through the whole series.

The fixes were rarely glamorous. They were mostly about turning accidental behavior into explicit engineering.
