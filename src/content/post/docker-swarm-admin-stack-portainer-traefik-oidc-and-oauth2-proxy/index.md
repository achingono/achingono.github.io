---
author: "Alfero Chingono"
title: "Docker Swarm Admin Stack: Portainer + Traefik Edge + OIDC + oauth2-proxy"
date: 2026-06-24T09:00:00Z
draft: true
description: "The practical control-plane writeup: how the management stack is wired, which failures showed up first, and what the final routing and identity model looks like in code and scripts." 
slug: docker-swarm-admin-stack-portainer-traefik-oidc-and-oauth2-proxy
tags: [
  "docker",
  "swarm",
  "traefik",
  "oidc",
  "oauth2",
  "portainer"
]
categories: [
  "Infrastructure",
  "Security"
]
image: "cover.png"
---

The management stack is the part of the cluster that has to be right before anything else matters.

In this repo it is defined primarily by:

- `docker-stack.yml`
- `docker-stack.ingress.yml`
- `setup.sh`
- `scripts/remote/*.sh`
- `templates/traefik.yml.tpl`
- `templates/management.yml.tpl`

The final result is compact:

- in-Swarm Traefik edge
- Portainer
- directory service
- OIDC provider
- oauth2-proxy
- optional public edge VM for TLS termination

What took time was not deciding to use those parts. It was getting their boundaries right.

## Why the edge proxy lives inside Swarm

The in-Swarm edge is the service named `edge` in `docker-stack.yml`.

Important settings:

- Swarm provider enabled
- `--providers.swarm.exposedByDefault=false`
- healthcheck via `traefik healthcheck --ping`
- port `80` published in host mode

The host-mode publish is easy to overlook. It matters because the tunnel peer's source address stays visible, which makes forwarded-header trust workable.

Before that detail was right, auth behavior was inconsistent. Requests would arrive, but the downstream system had an incomplete or misleading picture of the original scheme.

In practical terms, the check path after fixes looked like this:

```bash
docker stack services management
docker service logs management_edge --tail 50
curl -Ik https://<admin-host>
```

## Why Portainer is not behind oauth2-proxy

This took a few iterations.

Portainer and oauth2-proxy solve different problems.

Putting oauth2-proxy in front of Portainer did not create real single sign-on. It created two auth boundaries in sequence.

The working design was:

- let Traefik route Portainer directly
- let Portainer act as its own OAuth client
- use the shared OIDC provider as the identity source

That logic lives in `scripts/remote/configure-portainer-sso.sh`.

## The first Portainer failure: setup timeout

Portainer expects the initial admin to be created quickly after first startup.

That assumption is fine in a simple local install. It breaks down when the service is waiting on routing, auth wiring, DNS, and certificates.

The symptom was the security-timeout screen.

The fix was to stop relying on the interactive first-run path and pre-seed the admin password via Docker secret:

```yaml
--admin-password-file=/run/secrets/portainer_admin_password
```

That one change removed an entire class of fragile timing behavior.

## The second Portainer failure: successful SSO, zero access

This one is less obvious.

If you pre-seed the admin password, you skip Portainer's setup wizard. That wizard is also where the local Docker environment usually gets registered.

So you can end up with a working login and nothing to manage.

The repo now handles that in the SSO configuration script by:

- creating the local Docker environment if it does not exist
- creating the default OAuth team
- granting that team access to the local environment
- adding existing OAuth users to the team if needed

That turned SSO from “authentication works” into “operators can actually use the UI.”

## Directory, OIDC provider, and oauth2-proxy

The auth services live in `docker-stack.ingress.yml`.

The split is straightforward:

- directory service stores users and seeds the initial admin account
- OIDC provider talks to the directory service internally
- oauth2-proxy fronts the directory UI for browser access

Representative internal URLs from the stack file:

- `http://directory:8080`
- `http://oidc-provider:8080/token`
- `http://oidc-provider:8080/jwks`
- `http://oidc-provider:8080/me`

These are useful details because they explain why the system can come up before public ingress is fully ready. Internal service-to-service auth calls do not need the public edge path.

## Reverse-proxy mode was the right choice

One design choice shows up repeatedly across the repo: use oauth2-proxy as a reverse proxy for browser-facing paths instead of only as forward-auth middleware.

That improved two things:

- fewer awkward browser flows
- clearer ownership of the authenticated upstream path

You can see the same preference later in the application stacks.

## Public edge VM: only for TLS and forwarding

The public edge VM has a narrow job:

- terminate TLS on `:443`
- hold ACME state
- forward over the private tunnel to the in-Swarm edge

The setup flow is implemented in `setup.sh` and remote helpers such as:

- `configure-ingress-wireguard.sh`
- `configure-manager-wireguard.sh`
- `deploy-traefik-ingress.sh`
- `list-ingress-domains.sh`

The interesting part is how certificate domains are discovered.

`list-ingress-domains.sh` does not maintain a separate list by hand. It scans deployed services for Traefik router rules using `Host(...)`. That means the routing labels on services become the source of truth for public certificate coverage.

That was a good correction. Manual domain lists drift.

## The ingress failures worth remembering

Three failures showed up often enough that they ended up documented explicitly.

### 1. Tunnel is up in one direction only

Symptom:

- bytes sent, zero received
- tunnel handshake missing or stale

The lesson was to separate host firewall checks, tunnel config checks, and upstream firewall checks. They are different problems.

### 2. ACME state is stale

Symptom:

- certificate issuance keeps failing even though the host now resolves correctly

The fix in practice was to reset Traefik's ACME storage and let it register a fresh account.

### 3. DNS is late

Symptom:

- management routes are deployed
- tunnel is healthy
- certs still do not issue

That failure is boring, but common. The route exists before the public name is ready.

## Why `setup.sh` is split into phases

The current phases are not ceremony:

```bash
./setup.sh preflight
./setup.sh bootstrap-swarm
./setup.sh deploy-stack
./setup.sh configure-ingress
./setup.sh verify
```

That split came directly from debugging experience.

When routing, node setup, secret creation, tunnel wiring, and TLS are all blended into one long shell script, recovery gets slow. The phased design made it much easier to rerun one layer of work without redoing everything else.

That is the main pattern of the admin stack: move implicit coupling into explicit files, explicit scripts, and explicit phases.
