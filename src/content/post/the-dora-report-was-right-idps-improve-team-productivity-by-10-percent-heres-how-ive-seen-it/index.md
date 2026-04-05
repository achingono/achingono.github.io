---
author: "Alfero Chingono"
title: "The DORA Report Was Right: IDPs Improve Team Productivity by 10% — Here's How I've Seen It"
date: 2025-04-10T09:00:00Z
draft: false
description: "Why the DORA platform engineering findings rang true to me, and what I have learned from building delivery standards, templates, and paved roads that actually help teams move faster."
slug: the-dora-report-was-right-idps-improve-team-productivity-by-10-percent-heres-how-ive-seen-it
tags: [
"DORA",
"Internal Developer Platforms",
"Platform Engineering",
"Developer Experience",
"DevOps",
"Productivity"
]
categories: [
"Platform Engineering",
"Developer Experience"
]
image: "cover.png"
---

When the DORA research started surfacing stronger evidence around internal developer platforms, the headline did not surprise me nearly as much as the reactions did.

Some people still hear "platform engineering" and imagine more process, more gates, and another internal team inventing obstacles. That risk is real. But it is only one version of the story.

The version I have seen in practice is much simpler: when you reduce cognitive load, standardize the boring parts well, and make the safe path the easy path, teams move faster.

That is why the widely shared DORA finding about internal developer platforms improving team performance felt directionally right to me. I have seen that pattern from multiple angles: CI/CD modernization that improved project velocity, reusable delivery templates that removed duplication, feature-flag and configuration patterns that reduced rollout risk, and platform standards that made decision-making less expensive for product teams.

I would not claim every platform effort automatically produces a neat percentage uplift. But I do believe the mechanism is real.

## What platform engineering is actually buying you

At its best, an internal developer platform is not a control tower. It is a **friction reducer**.

It helps teams spend less time answering questions like:

- How should we structure this pipeline?
- Which security checks are required?
- What is the approved deployment pattern?
- How do we manage runtime configuration safely?
- How do we release gradually without gambling in production?

If every team answers those questions from scratch, you get inconsistency, duplicated effort, and unnecessary risk. If the platform team answers them once, clearly, and with enough flexibility, you get leverage.

That leverage is where the productivity gain comes from.

Not from a portal.
Not from a dashboard.
Not from a maturity model.

From fewer repeated decisions.

## Where I have seen the gains show up

One of the more durable lessons in my career is that developer productivity is usually downstream of environment design.

At VCA Software, the measurable win was CI/CD modernization. We saw delivery speed improve because the path from code to release became more repeatable and less person-dependent. That did not happen because engineers suddenly became more talented. It happened because the delivery system stopped making them re-solve the same operational problems over and over.

In more platform-oriented work, I have seen the same principle show up differently:

- a reusable feature-flag framework that makes progressive delivery safer
- standardized pipeline templates that reduce copy-paste infrastructure
- better observability defaults so teams are not blind after deployment
- configuration-management patterns that reduce drift and remove manual setup

That last point is one reason I still like the pattern I wrote about in [Conditionally Deploying Resources in Azure App Configuration Using Deployment Scripts](/blog/2025/01/10/conditionally-deploying-resources-azure-app-configuration-using-deployment-scripts/). It is not glamorous, but it is exactly the kind of operational sharp edge a good platform should smooth out.

## The DORA nuance matters too

What I appreciate about the DORA research is that it does not treat platform engineering as universally positive in every implementation.

That matches my experience.

A platform helps when it gives teams **self-service with sensible defaults**.

A platform hurts when it becomes:

- mandatory ceremony
- an opaque ticket queue
- a rigid abstraction over real team needs
- a place where local context goes to die

This is where some platform efforts go sideways. They optimize for governance theater instead of developer flow. Then leaders conclude that platform engineering is slow, when the real problem is that the platform is not being run like a product.

## What good platform teams do differently

The best platform work I have seen has a few traits in common.

### 1. It starts with repeated pain, not abstract ambition

Good platform teams do not begin with "let's build an internal developer portal." They begin with "teams keep tripping over the same deployment, security, configuration, or release problems."

That difference matters because it keeps the work grounded in actual developer friction.

### 2. It productizes standards

A standard that lives in a slide deck does almost nothing.

A standard that shows up as a reusable template, a safe default, a documented flow, a working example, and a supportable paved road actually changes behavior.

### 3. It respects local autonomy

The best platforms do not remove all choice. They remove the expensive choices that most teams should not have to make repeatedly.

That is a very different posture from central control.

### 4. It measures adoption, not just existence

If nobody uses the platform voluntarily, that is feedback.

Platform teams need to care about usability the same way product teams do. A paved road that engineers avoid is not a paved road.

## My working definition now

I increasingly think of platform engineering as the discipline of making good engineering behavior easier to repeat.

That includes technology, of course, but also language, templates, defaults, and trust.

Done well, it gives teams more autonomy because it lowers the cost of doing the right thing. Done badly, it creates another dependency.

That is why the DORA finding resonated with me. Not because I am attached to the term, but because I have seen the underlying dynamic up close. When teams have usable internal platforms, they make better decisions faster.

That is not magic. It is what happens when you turn institutional knowledge into operable systems.

If this topic interests you, the closest companion piece here is [Why I Started Building My Own DevOps Platform](/blog/2025/02/15/why-i-started-building-my-own-devops-platform-and-what-i-learned/), which comes at the same problem from the builder side rather than the organizational one.

References:

- [2024 DORA Report](https://dora.dev/research/2024/dora-report/)
- [OpsLevel summary of the 2024 Google Cloud DORA Report](https://www.opslevel.com/resources/tl-dr-key-takeaways-from-the-2024-google-cloud-dora-report)
- [Conditionally Deploying Resources in Azure App Configuration Using Deployment Scripts](/blog/2025/01/10/conditionally-deploying-resources-azure-app-configuration-using-deployment-scripts/)
