---
author: "Alfero Chingono"
title: "OAuth2 and OIDC for Solo Developers: A Practical Setup With Docker"
date: 2025-09-15T09:00:00Z
draft: false
description: "Auth is always harder than it looks. Here is how I set up OIDC for my personal projects without spending a fortune on managed services."
slug: oauth2-and-oidc-for-solo-developers-a-practical-setup-with-docker
tags: [
"OAuth2",
"OIDC",
"Auth",
"Docker",
"Security",
"Self-Hosted",
"Famorize",
"FireFly"
]
categories: [
"Craft",
"Security"
]
image: "cover.png"
---

Authentication is the classic "black hole" of side projects. You start with a simple idea, and four days later you're still reading RFC 6749 and wondering if you should just use a password in a plain text file (don't).

For my personal projects like [Famorize](/blog/2025/10/22/building-a-multi-currency-app-the-edge-cases-nobody-warns-you-about/) and [FireFly](/blog/2025-05-08-teaching-kids-to-code-with-bayesian-knowledge-tracing-why-i-built-firefly/), I needed something standards-compliant and easy to manage across multiple services. It also had to be dependable. I didn't want to pay $50/month for a managed identity provider (IdP) for projects that were still in development, but I also didn't want to roll my own crypto.

The solution was a self-hosted OIDC setup using Docker.

## Why OIDC?

If you're building a "modern" app, you're likely using a decoupled architecture: a frontend (React/Next.js/Hugo) and one or more backend APIs. You need a way to:

1.  Log a user in.
2.  Prove to the API that the user is who they say they are.
3.  Do it securely without sharing passwords everywhere.

OpenID Connect (OIDC) is the standard way to do this. It sits on top of OAuth2 and provides the "identity" layer.

## The Stack

For my local and production environments, I settled on a "trio" of tools that play very well together in Docker Compose.

*   **Authelia:** A lightweight, self-hosted IdP. It's fast, has a great UI, and supports multi-factor authentication (MFA) out of the box.
*   **Redis:** For session storage and rate-limiting.
*   **Nginx Proxy Manager (or Traefik):** To handle the routing and SSL termination.

## The "Local First" Workflow

The biggest pain point with OIDC is setting it up locally. Most managed services require a public redirect URI, which is a nightmare for `localhost` development.

With a self-hosted setup, I can use a local domain (like `auth.local.chingono.com`) and point it to my Docker container. This means my local development environment is *identical* to production. If it works on my machine, the OIDC flow will work in the cloud.

## Key Lessons Learned

Building this into my projects taught me a few hard-won lessons:

1.  **Scopes and Claims are everything.** Spend the time to understand the difference. Don't just request `openid profile email` and hope for the best. Define only what your app actually needs.
2.  **HTTPS is non-negotiable.** Even for local development, OIDC requires secure cookies. Use `mkcert` to generate local SSL certificates and save yourself the headache of "Insecure Connection" errors.
3.  **Token rotation matters.** If you're building a long-lived app (like a mobile client for [Famorize](/blog/2025/10/22/building-a-multi-currency-app-the-edge-cases-nobody-warns-you-about/)), you need to handle refresh tokens correctly. Don't just set a 30-day expiration on your access tokens.

## Why Not Just Use Auth0 or Clerk?

Managed services are great for teams with a budget and no time. But as a solo developer who enjoys the "Build in Public" ethos, I find that owning my identity layer is worth the extra effort. It gives me complete control over the user experience, total data sovereignty, and a much deeper understanding of the security architecture.

If you're interested in the specific Docker Compose configurations I use, stay tuned for the next post in this "Craft" series.

---
*Related reading:*
- [Building a Multi-Currency App: The Edge Cases Nobody Warns You About](/blog/2025/10/22/building-a-multi-currency-app-the-edge-cases-nobody-warns-you-about/)
- [Why I Run OpenClaw in Docker on My Own Machine](/blog/why-i-run-openclaw-in-docker-on-my-own-machine/)
