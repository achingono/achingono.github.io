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

## Why per-seat worked for SaaS

Per-seat pricing earned its place. For classic SaaS, it mapped to something measurable. One human, one browser tab, bounded throughput. A Salesforce seat represented roughly the work one salesperson could do in a day, which was also roughly the load they put on the system. The pricing model matched the capacity model, and procurement could forecast reliably by counting badges.

That alignment is what made the model durable. It was not because "per seat" is intrinsically elegant. It was because the seat was a decent proxy for both value and load at the same time.

Agents break that proxy in both directions at once.

## Why it does not port to agents

An agent can run for 30 seconds or for eight hours. It can call one tool or two hundred. It can sit idle overnight or hammer a queue at 400 concurrent jobs. Two "seats" in the same vendor's catalogue can produce wildly different load profiles and wildly different amounts of work. The seat has stopped being a proxy for anything.

So when a vendor says an agent seat costs $80 a month, the first question is: what is included. Sometimes it is a token budget. Sometimes it is a job count. Sometimes it is a concurrency limit that is not documented on the pricing page but shows up in the SOC 2 annex. Sometimes, and this is the one to watch for, it is unlimited-until-it-isn't, where the vendor reserves the right to throttle or renegotiate if your usage exceeds what they expected.

None of this is unreasonable from the vendor's side. It is a genuinely hard pricing problem. What is unreasonable is asking a platform team to do capacity planning against that shape while calling it "per seat" as if it were a solved category.

## The questions I would ask before signing

If someone pitches agent-seat pricing to you, these are the questions that separate a real product from a marketing exercise.

What does one seat actually include, in units you can count? Not "a seat is an agent." A seat is how many tokens, how many tool calls, how many concurrent jobs, at what latency. If the sales engineer cannot give you numbers on the call, the pricing model is not finished yet, and you are being asked to absorb that uncertainty for them.

How is overage priced, and at what granularity? This is where the real bill lives. A seat that looks cheap on the sticker and meters aggressively on overage is a cloud-bill-in-disguise. You want the overage curve in writing, with the unit economics exposed, so you can model your actual workload against it instead of guessing.

How do you attribute usage back to a team or workload? If the answer is "you will get one invoice per month," that is not attribution; that is an invoice. Without per-team, per-workload, ideally per-task attribution, you cannot do any of the cost work I talked about in [cost visibility comes before cost optimization](../cost-visibility-comes-before-cost-optimization/). You are back to total-spend-as-the-only-number, which is where FinOps conversations go to die.

What happens during a regression or an outage? Vendors ship model upgrades. Sometimes those upgrades are worse for your specific workload. If the model behind your agent seats degrades next quarter, what is the refund mechanism? What is the SLA? If the answer is vague, assume there is no answer, and price the risk accordingly.

Can you bring your own model routing? This one matters more than most people realize. A vendor that insists on their model backend is selling you a seat and a lock-in simultaneously. The gap between "we host the model" and "you can route to your own LiteLLM tier" is the difference between a vendor you can walk away from and one you cannot.

And one question I do not see asked enough: what observability do you get, and is it yours or theirs? If the vendor owns the logs, you do not own the diagnostic picture when something goes wrong. That has downstream consequences well beyond cost.

## The capacity-planning anti-patterns this encourages

Per-seat pricing does not just mislead the bill. It misleads how teams plan.

The first anti-pattern: headcount-parity buying. Someone in leadership decides the engineering team should have "one agent seat per human engineer," as if that were a sensible unit. It is not. Some engineers will run agents constantly. Others will run them weekly. Some workloads need zero seats and a batch job. You end up with either unused capacity you are paying for or a shortage you cannot articulate because the shortage is measured in tokens and you bought seats.

The second: role-based seats. "A review-agent seat, an implementation-agent seat, a docs-agent seat." This is the per-seat model pretending the agent is a person, when what you really have is a shared queue of heterogeneous jobs hitting a shared model backend. The role is a label, not a capacity unit.

The third, and most expensive: forecasting growth in seats. If next year's plan says "we will grow from 50 to 100 agent seats," you are forecasting nothing. The real forecast is jobs per day, tokens per job, concurrency peak. Seats are downstream of all of that.

## A cost-to-serve reframing

The model I have been using internally, and that has survived contact with actual invoices, is cost-to-serve by task class.

Pick the things your agents actually do. Code review, test generation, docs cleanup, incident triage, whatever your cast looks like (mine is roughly [the engineering orchestra shape](../designing-multi-agent-systems-lessons-from-building-an-8-agent-engineering-orchestra/)). For each class, measure the real numbers: tokens in, tokens out, tool calls, model tier used, latency, and cost per completed task. Now you have a unit economics view. Now the question "should we pay for 20 more seats next quarter" becomes "our review throughput is constrained and the cost-per-review is $0.14; is that spend justified by the throughput we would gain."

That is a real procurement conversation. "Seats" is not.

This is also, incidentally, why I keep pushing on the idea that [API dashboards are only useful if they change decisions](../api-dashboards-are-only-useful-if-they-change-decisions/). A cost-to-serve view either drives a decision about scaling, task mix, or model tier, or it is wallpaper.

## How to negotiate

Three things to push on, assuming you are committing real money.

Insist on a usage floor rather than a seat count. Commit to a token or task volume that matches your forecast, with a real discount for the commitment, and let the vendor meter against it however they like. You are buying capacity; buy it in capacity units.

Demand portability language in the contract. Written confirmation that your prompts, your agent definitions, and your routing configuration remain yours and exportable. If you ever want to move to a different backend, the cost should be migration effort, not the vendor's pricing model.

Require observability rights. Full access to logs, token accounting, latency metrics, and tool-call success rates at the workload level. If the vendor resists, they are selling you opacity, and opacity is the most expensive line item on an AI bill.

## The short version

Agent-seat pricing is a familiar pattern being stretched over an unfamiliar shape. That does not make it inherently bad. It makes it inherently misleading if you let the familiarity do the work of the analysis.

Ask what one seat actually is. Price the overage, not the sticker. Attribute by workload, not by invoice. And forecast in tokens and tasks, never in seats.

If your vendor cannot answer those questions cleanly, you are not too skeptical. You are exactly skeptical enough.
