---
author: "Alfero Chingono"
title: "When Product Velocity Becomes a Growth Tax"
date: 2023-12-05T09:00:00Z
draft: true
description: "What I have learned from legacy software teams that optimized for speed for years, only to find that the same choices eventually made growth slower, riskier, and more expensive."
slug: when-product-velocity-becomes-a-growth-tax
tags: [
"Technical Debt",
"Legacy Systems",
"Platform Engineering",
"Software Delivery",
"Modernization"
]
categories: [
"Platform Engineering",
"Software Delivery"
]
image: ""
---

One of the more expensive patterns in software is this: a company grows because it moves quickly, then later struggles because it kept moving the same way for too long.

I have seen this most clearly in older product companies that were built by smart, determined people making rational decisions under pressure. A founder writes the first version. Customers arrive. Custom features get added. A few shortcuts are taken because the business needs momentum more than elegance. Then the team grows, the customer base grows, the environments multiply, and the operational model quietly stops scaling.

That does not usually look like a strategy problem at first.

It looks like missed deadlines.
It looks like frustrated leadership.
It looks like a team that supposedly needs to estimate better.

But very often the deeper issue is simpler: the organization is still trying to buy velocity using methods that now create drag.

## The original shortcuts were often reasonable

I think this matters, because teams are too quick to moralize technical debt.

Many of the decisions that later become expensive started as good business decisions.

If you are a growing software company in the early years, shipping quickly is not optional. You need customers. You need proof. You need cash flow. You need the product to keep moving.

That usually leads to choices like:

- customer-specific customizations
- infrastructure built one environment at a time
- manual deployment habits that rely on trusted individuals
- configuration embedded too close to code
- process light enough not to slow urgent work down

That can work for a while.

The problem is that those decisions compound. And once they compound far enough, the organization stops getting speed from them.

It starts paying interest instead.

## The warning sign is usually operational, not architectural

When a company says, "we are struggling to keep commitments," the instinct is often to inspect planning first.

Sometimes that is correct.

But if the team is supporting product issues, customer-funded custom work, roadmap initiatives, and modernization efforts all at the same time, planning is only one layer of the problem. If every environment is a little different, if releases are manual, if production fixes depend on tribal knowledge, and if custom features are poorly mapped, then the delivery system is already carrying too much hidden variance.

That kind of variance does not show up neatly in a sprint board.

It shows up as:

- longer troubleshooting cycles
- defensive estimation
- support interruptions destroying planned work
- leadership hearing commitments that were never structurally realistic

This is one reason I do not believe an agile coach, by themselves, can solve a system like that.

Agile can improve visibility. It can improve flow. It can improve conversations.

But it cannot cancel out environment drift.

## Growth gets harder when every customer becomes a partial fork

One of the hardest scaling traps for older products is customer-specific drift.

At first, it feels like responsiveness.

You say yes to an enterprise request. You implement something valuable. You help close or retain revenue. That can be exactly the right choice.

But if the implementation model becomes "add one more custom behavior, one more schema difference, one more environment-specific deployment path," then eventually the product stops behaving like a product and starts behaving like a family of loosely related installations.

That changes everything.

Testing gets harder because you are not really testing one system anymore.

Releases get harder because every change has to survive a wider, murkier set of permutations.

Onboarding gets harder because new engineers are learning not just a codebase, but a historical record of customer-by-customer divergence.

At that point, the company may still be calling itself fast. But it is doing fast-looking work on top of a slow-moving substrate.

## The team is not slow. The system is expensive

This is the sentence I wish more leaders were willing to say out loud.

When a development team looks inconsistent, the cause is not always capability. Sometimes the system they are operating inside is charging them too much for every change.

If developers are working across environments that do not match, touching production-like systems to understand basic behavior, or spending days reconciling code and database mismatches, that is not a motivation problem.

It is a cost-of-change problem.

The same is true when direct server access is part of the normal workflow.

That setup may look faster because it removes ceremony. In reality, it pushes quality control, traceability, and operational safety back onto individual memory and caution. That trade gets worse as the business grows.

## The real ceiling is not technical. It is commercial

What makes this pattern serious is that it does not stay inside engineering.

Eventually it reaches the commercial edge of the company.

Customers who were used to frequent updates start feeling instability.
Custom work takes longer to deliver.
Roadmap work competes with support work more aggressively.
Expansion into new markets slows because the platform underneath the business is too fragile to carry more load confidently.

That is when technical debt stops being a developer complaint and becomes a growth constraint.

I think that is the frame leaders need most.

Not "the code is old."

Not "the team needs to work differently."

But: "the current operating model is now taxing every future move the business wants to make."

## What I would focus on first

If a company finds itself here, I would resist the urge to start with a grand rewrite story.

The first job is to reduce variance and recover control.

That usually means:

- understanding how environments actually differ
- centralizing source control and deployment logic
- separating configuration and secrets from code
- limiting direct production changes
- making customer-specific differences more explicit than accidental

Those are not glamorous steps.

They are foundational ones.

You do not modernize a delivery system by declaring a new architecture and hoping the old operational mess becomes irrelevant. You modernize it by making the current system more knowable, more repeatable, and less dependent on memory.

## My takeaway

Velocity is one of the best things a young software business can have.

It is also one of the easiest things to counterfeit.

If speed comes from undocumented variation, manual heroics, and an ever-growing number of exceptions, that speed has an expiry date. Eventually it becomes a tax on the very growth it once enabled.

That is the trap.

The goal is not to regret the early decisions forever. The goal is to recognize when the business has outgrown them and to invest accordingly.

This is the first post in a short series. The next one is more operational: [Why Manual Deployments Keep Legacy Teams Stuck](/blog/2023/12/06/why-manual-deployments-keep-legacy-teams-stuck/).
