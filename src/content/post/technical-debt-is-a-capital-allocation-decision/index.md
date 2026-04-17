---
author: "Alfero Chingono"
title: "Technical Debt Is a Capital Allocation Decision"
date: 2023-12-07T09:00:00Z
draft: true
description: "The hardest part of technical debt is not identifying it. It is helping leadership see remediation as business investment instead of a stream of annoying IT costs."
slug: technical-debt-is-a-capital-allocation-decision
tags: [
"Technical Debt",
"Leadership",
"Modernization",
"Platform Engineering",
"Software Delivery"
]
categories: [
"Platform Engineering",
"Leadership"
]
image: "cover.png"
---

By the time technical debt is obvious, the engineering argument is usually no longer the hard part.

People can see the outages.
They can see the missed commitments.
They can see the manual work, the duplicated effort, the slow onboarding, the operational fragility.

The harder part is helping the business interpret what it is seeing.

Is this just a run of unfortunate IT costs?

Or is it a sign that the company needs to reallocate capital toward the platform underneath the business?

I think the second framing is usually the correct one.

## Technical debt is not only a code smell

One reason these conversations go sideways is that technical debt is still too often described as if it lives mostly in source code.

Sometimes it does.

But in practice, the more expensive debt usually spans:

- code and schema variation
- outdated libraries and protocols
- manual deployment behavior
- undocumented custom features
- weak monitoring and recovery capability
- poor separation of configuration and secrets
- process models that rely on individuals more than systems

That matters because the remediation strategy changes depending on the frame.

If leadership hears "developers want to clean up code," the work sounds optional.

If leadership hears "the current operating model is reducing delivery capacity, increasing risk, and slowing revenue growth," the conversation becomes more honest.

## Some remediation costs are avoidable. Some are not

I think this distinction is useful, especially for leadership and finance conversations.

There are costs the business should absolutely try to eliminate.

Things like:

- repeated manual intervention on unstable components
- professional services spent compensating for poor operational control
- paying for underperforming service providers without getting the contracted value
- troubleshooting time caused by undocumented variance
- support volume created by preventable deployment mistakes

Those are avoidable costs, or at least reducible ones.

But there are also costs that are simply part of maturing a software business.

Things like:

- replacing deprecated dependencies
- redesigning deployment paths
- introducing monitoring and alerting
- centralizing source control and environment management
- modernizing critical parts of the architecture
- training teams to work in a safer, more scalable way

Those are not accidental expenses.

They are the price of remaining operable as the company grows.

Treating them as surprising overhead is how organizations stay trapped.

## The most expensive choice is often delay

This is the part I think many businesses discover too late.

When leaders postpone remediation because the spending feels large, they are still making an investment decision. They are investing in continued fragility.

Delay has a return profile too.

It buys:

- more downtime risk
- more customer frustration
- more operational labor
- more difficulty launching strategic work
- more reluctance inside the team to make changes confidently

In other words, inaction is not free.

It is just financed through slower delivery, preventable incidents, and missed opportunity.

## Why phased remediation works better than heroic transformation

I am skeptical of remediation plans that read like one dramatic act.

The better pattern is usually phased, especially in risk-sensitive organizations.

Small steps.
High value.
Deliberate sequencing.
Maximum de-risking.

That sounds conservative. In practice, it is what allows momentum to survive.

I would rather see a company do five grounded things well than announce one total reinvention and spend a year deepening confusion.

For older systems, the practical sequence is often something like:

1. make the current state more visible
2. centralize source and dependency management
3. reduce environmental variance
4. create safer development and staging loops
5. automate the repeatable paths
6. modernize the architecture in bounded slices

That sequence is not exciting, but it respects reality.

## Leadership has to fund the future, not just the present

There is a mindset shift here that matters.

If leadership sees modernization only as cost containment, the ambition will usually be too small.

The stronger framing is that remediation buys capacity.

It buys the ability to:

- keep commitments more reliably
- onboard customers with less operational stress
- release with less fear
- reduce reliance on hard-to-replace tribal knowledge
- give product and revenue work a healthier platform underneath it

That is not just maintenance.

That is business enablement.

## The IT treadmill is real

There is one more uncomfortable truth here.

Even after a serious remediation program, the work is not over forever.

Protocols change.
Vendors deprecate features.
Libraries age out.
Security expectations rise.
Operating systems move on.

This is normal.

I think organizations get into trouble when they keep treating updates as exceptional disruptions instead of ongoing budget realities. A healthy platform budget should assume that some amount of modernization is continuous.

Not because the team failed.

Because software lives in a moving environment.

## My takeaway

Technical debt remediation should not be sold as housekeeping.

It is a capital allocation decision about what kind of business the company wants to be able to run two years from now.

If the current platform is constraining delivery, trust, and growth, then remediation is not a side project competing with the business. It is part of the business finally funding the conditions it needs to keep scaling.

That is the real argument.

Not "we should clean this up someday."

But "we need to invest in a platform that stops charging us interest on every move."

This is the last post in the series. The first two were [When Product Velocity Becomes a Growth Tax](/blog/2023/12/05/when-product-velocity-becomes-a-growth-tax/) and [Why Manual Deployments Keep Legacy Teams Stuck](/blog/2023/12/06/why-manual-deployments-keep-legacy-teams-stuck/).
