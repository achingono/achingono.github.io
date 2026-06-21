---
author: Alfero Chingono
title: A Good Cloud Diagram Should Expose Trust Boundaries
date: 2026-05-28T09:00:00.000Z
description: A useful Azure reference architecture is less about listing services and more about making trust boundaries, entry points, and data paths obvious.
slug: a-good-cloud-diagram-should-expose-trust-boundaries
tags:
  - Azure
  - Cloud Architecture
  - Security
  - Reference Architecture
  - Platform Engineering
categories:
  - Azure
  - Architecture
image: cover.png
---

A few times now, I’ve looked at a cloud diagram and felt the same thing: the boxes were neat, but the architecture still wasn’t telling the truth.

That’s usually the moment I start looking for trust boundaries.

Not because trust boundaries are some abstract security slogan, but because they’re the fastest way I know to see whether a diagram is actually describing how a system behaves. A good diagram makes it obvious where traffic enters, which component makes the trust decision, and where sensitive data stops being public.

## The first thing I check is the edge

When I’m reviewing a diagram, I usually start at the public edge.

If there is an application gateway, API gateway, or some other front door, I want to know exactly what it protects and what it does not. I want to see whether the edge is just decorative or whether it is actually doing something useful like terminating TLS, filtering traffic, or separating public access from private workloads.

If the diagram draws a single cloud icon and then several internal services with no obvious boundary between them, that’s already a warning sign. It usually means the author is describing deployment shape, not trust shape.

## Then I follow the data path

The next thing I trace is the data path.

A diagram can look secure and still hide a bad assumption. I have seen that happen when secrets, certificates, or application settings are drawn as if they just “exist” somewhere in the platform.

That is where a concrete example helps.

In **How to Enable VM Insights on an Azure Virtual Machine Using Bicep**, the path is not just “enable monitoring.” The actual flow is more useful than that:

- the VM gets the extension
- telemetry moves through the storage account / monitoring plumbing
- Application Insights becomes the place where the signals end up

That is the kind of detail I want a diagram to expose. Not just that monitoring exists, but how the machine, the extension, and the telemetry destination relate to one another.

The same thing shows up in **Consume Secrets in Azure Key vault From Kubernetes On-prem**.

If the diagram only says “Key Vault,” it is too vague to help me reason about risk. If it shows:

- a service principal or identity
- a Key Vault policy or access path
- the CSI driver
- the pod mounting the secret

then I can actually see where the trust decisions happen. I can tell whether the secret is being fetched at runtime, where identity is enforced, and what the cluster needs in order to touch that data.

That is the difference between a diagram that looks complete and a diagram that lets you review an implementation.

## I care about what stays public and what does not

A lot of diagram problems come from mixing public and private concerns in the same visual layer.

If user traffic is supposed to stop at one component and backend services stay private, I want the diagram to make that separation unavoidable. If every box is drawn with the same visual weight, people tend to assume everything is equally exposed.

That is where teams get themselves into trouble.

The diagram should make it easy to answer questions like:

- What can the internet reach directly?
- What only lives inside the private network?
- What has to be authenticated before it can talk to anything else?
- What is just internal plumbing?

If I can’t answer those questions from the diagram, I usually don’t trust the diagram yet.

## Supporting services should not disappear into the background

Another thing I look for is whether the diagram hides the supporting services.

Things like monitoring, secret stores, identity providers, certificate handling, and storage are often drawn as little side notes. But in real systems, those pieces are part of the trust story.

That is why I liked the two concrete examples above. They remind me that support services are not incidental:

- VM Insights depends on an actual path for telemetry
- Key Vault depends on identity, policy, and runtime access

If those pieces are invisible, the diagram is pretending the system is simpler than it really is.

## What I mean when I say a diagram should expose trust boundaries

I do not mean every diagram has to look like a security architecture poster.

I mean a reviewer should be able to glance at it and understand where the interesting decisions are.

For me, that usually means the diagram answers three things clearly:

1. where the boundary is
2. what crosses it
3. who or what is allowed to cross it

If the picture does that, it is useful.
If it doesn’t, the picture is mostly decoration.

## The best diagrams feel like a review, not a presentation

The strongest diagrams I’ve seen are the ones that make review easy.

They do not try to impress you with symmetry. They make it obvious what is exposed, what is internal, and what needs to be trusted carefully.

That is what I was aiming for when I wrote this post in the first place, and it is still the standard I use when I look at my own architecture sketches.

A good cloud diagram should not just show what exists.

It should show where the trust changes.

