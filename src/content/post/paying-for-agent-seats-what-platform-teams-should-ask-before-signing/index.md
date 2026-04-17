---
author: "Alfero Chingono"
title: "Paying for Agent Seats: What Platform Teams Should Ask Before Signing"
date: 2026-07-16T09:00:00Z
draft: true
description: "Per-seat pricing for AI agents breaks capacity planning. Frame the buying conversation around utilization, attribution, and cost-to-serve — not sticker price."
slug: paying-for-agent-seats-what-platform-teams-should-ask-before-signing
tags: [
"Platform Engineering",
"FinOps",
"AI Agents",
"Procurement",
"Enterprise",
"Cloud Costs"
]
categories: [
"Platform Engineering",
"FinOps"
]
image: ""
---

Enterprise vendors are converging on a simple pricing story: charge for AI agent seats the same way you already charge for humans. It reads well in a board deck. It falls apart the moment a platform team tries to forecast capacity, because an "agent seat" is not a unit of anything consistent.

## Outline

- Why per-seat pricing worked for SaaS: one human, one browser tab, bounded throughput
- Why it doesn't cleanly port to agents: throughput, concurrency, and unit economics are detached from "seats"
- The questions I would ask any vendor pitching agent-seat pricing:
  - What does one seat actually include (tokens? tool calls? concurrent jobs?)
  - How is overage priced, and at what granularity?
  - How do I attribute usage back to a team or workload?
  - What happens during a regression or outage — credits, refunds, SLA?
  - Can I bring my own model routing, or am I locked into their backend?
- Capacity-planning anti-patterns this pricing encourages
- A cost-to-serve reframing: forecast by task class, not seat count
- How to negotiate: usage floors, portability, observability rights

## Artifacts to reference

- [Cost Visibility Comes Before Cost Optimization](../cost-visibility-comes-before-cost-optimization/)
- [API Dashboards Are Only Useful If They Change Decisions](../api-dashboards-are-only-useful-if-they-change-decisions/)
- Internal Emanate/OpenClaw per-task cost breakdown as a concrete shape
