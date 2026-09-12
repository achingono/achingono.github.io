---
title: From Highway Exits to Guided Express Lanes
date: "2026-06-23"
draft: true
summary: A practical platform engineering piece on turning standards, pipelines, and automation into guided paths for repeatable work.
tags: platform engineering, developer experience, automation, thought leadership
---

# From Highway Exits to Guided Express Lanes

A few months ago, we watched a team try to get a small service through a deployment that should have been boring.

They had the right repo. The right chart. The right environment. But the path still required too many decisions in the wrong order: pick the right pipeline template, remember the policy annotations, wire up the secrets path, choose the right deployment target, then check whether the service needed a manual approval. The work was technically possible, but it kept pulling people out of the flow of building the product.

That kind of friction is what pushed us toward standards, pipelines, and automation in the first place.

But a platform can be well-instrumented and still feel like a highway interchange.

The difference shows up when a path is repeatable. Novel work can tolerate complexity. Repeatable work should not force people to relearn the same set of turns every time.

That is where the guided express lane idea matters.

For the common cases, we started by turning the steps into a paved path: a smaller number of choices, a default configuration, and a workflow that makes the next correct action obvious. The goal was simple: let developers spend time on business logic instead of platform mechanics.

We used TSTV as a test bed because it had just enough real traffic to expose the pain, but not so much surface area that we could not reshape the path. That made the tradeoff clear. We could either keep the full flexibility of the generic platform flow, or we could constrain the common path and make it dramatically easier to use.

We chose the second option.

Concretely, that meant:

- standardizing the deployment pattern for the common case
- baking policy and environment selection into the pipeline instead of asking people to remember them
- removing a few manual approval steps where the risk was already covered by automation
- keeping an escape hatch for exceptions instead of trying to flatten every edge case

That last point mattered. The paved road did not replace experts. It scaled them.

The expert path was still there for migrations, unusual data flows, and services with special operational constraints. But for the recurring cases, the platform now captured the judgment that an experienced engineer would otherwise apply by memory.

A simple sketch of the pattern looks like this:

```text
request -> guided workflow -> validated config -> automated pipeline -> deploy
             |                                        |
             |                                        +--> policy checks / approvals when needed
             +--> fallback to expert path for exceptions
```

This is usually what people miss when they hear “platform standardization.” The point is not to eliminate decisions. It is to move the decisions to the place where they can be encoded once and reused many times.

Hemal was a good example of why that matters. Instead of spending time figuring out platform mechanics, he could stay focused on the thing the team actually needed: the business value in the service itself. That is the real win. Faster delivery is nice. Less context switching is nicer. But the biggest gain is when the platform gets out of the way of the work.

The first batch of guided paths came out of that effort, and they were useful for reasons that were visible immediately: fewer handoffs, fewer “how do I do this again?” questions, and fewer places where a routine change could stall.

The next lanes usually show up the same way. Someone hits the same friction twice. A teammate asks the same question three times. A workflow that should be ordinary turns into a scavenger hunt.

That is the signal.

If you are building platforms, the challenge is not to make every path identical. It is to recognize which paths are already repeatable and then make those paths obvious, safe, and fast. The people using the system will tell you where the next lane belongs.
