---
author: "Alfero Chingono"
title: "Beyond CI/CD: Why AI Agents Are the Next Layer of Software Delivery"
date: 2025-06-18T09:00:00Z
draft: true
description: "Traditional CI/CD is about deterministic pipelines. The next layer is agentic: handling the non-deterministic coordination that actually slows down engineering teams."
slug: beyond-ci-cd-why-ai-agents-are-the-next-layer-of-software-delivery
tags: [
"AI Agents",
"DevOps",
"Software Delivery",
"CI/CD",
"Agentic AI",
"CueMarshal"
]
categories: [
"Agentic AI",
"DevOps"
]
image: "cover.png"
---

For the last decade, "modern" software delivery has been defined by CI/CD. We spent years perfecting the art of the deterministic pipeline: code goes in, tests run, artifacts build, and deployments happen. If a human pushes a button (or a branch), the machine follows a script.

It works. It is stable. It is the foundation we all build on.

But CI/CD has a ceiling. It handles the *delivery* of the code, but it is blind to the *coordination* of the work. The bottleneck for most engineering teams is no longer how fast the build runs; it is how long an issue sits unassigned, how many times a PR bounces back for minor linting errors, and how much context is lost between a design doc and an implementation.

The next layer of software delivery isn't just "faster CI." It is **Agentic Delivery**.

## The Coordination Gap

If you look at a typical engineering day, the actual "delivery" (the pipeline run) is a tiny fraction of the cycle. Most of the time is spent in the coordination gap:

1.  **Context Assembly:** Figuring out which files matter for a task and what the existing patterns are.
2.  **Quality Gating:** Waiting for a human to notice a typo, a missing test, or a broken internal convention.
3.  **Context Handoff:** Writing the PR description, explaining the *why* to a reviewer, and updating the docs.

Traditional CI/CD tools cannot help here because they are stateless and reactive. They don't "understand" the task; they only know if the exit code was zero.

## Enter the Agentic Layer

When I started building [CueMarshal](/blog/2025/02/15/why-i-started-building-my-own-devops-platform-and-what-i-learned/), I realized that the real value of AI agents in DevOps isn't just generating code. It is handling the non-deterministic parts of the workflow that humans find tedious but necessary.

An agentic delivery layer lives *above* your CI/CD. It doesn't replace Jenkins or GitHub Actions; it uses them.

### What an Agentic Layer actually does:

*   **Active Monitoring:** Instead of waiting for a manual trigger, agents can watch for new issues, draft an implementation plan, and open a PR before a human even context-switches.
*   **Semantic Quality Gates:** While a linter checks syntax, an agent can check if the implementation actually matches the requirements in the issue.
*   **Continuous Context Maintenance:** Agents can keep documentation in sync with code changes in real-time, effectively eliminating "doc debt."

## Lessons from the Trenches

Building this into [my own platform](/blog/2025/02/15/why-i-started-building-my-own-devops-platform-and-what-i-learned/) taught me that the transition from CI/CD to Agentic Delivery requires a shift in how we think about automation.

1.  **From Scripts to Tools:** In CI/CD, we write shell scripts. In agentic systems, we define [Model Context Protocol (MCP)](/blog/2025/03/20/mcp-in-practice-what-anthropics-model-context-protocol-actually-means-for-developers/) tools. The agent needs a structured way to interact with the world, not just a line of bash.
2.  **From Success/Failure to "Looks Good to Me":** We are moving from binary pass/fail gates to probabilistic evaluation. That is why [specialized agent roles](/blog/2025/08/28/designing-multi-agent-systems-lessons-from-building-an-8-agent-engineering-orchestra/) matter so much: you want a dedicated "Reviewer" agent whose only job is to be skeptical.
3.  **Git is still the anchor:** Even with agents doing the heavy lifting, the final source of truth must remain the repository. If it's not in Git, it didn't happen.

## The 2025 Reality

We are past the "AI hype" phase where we wonder if agents *can* do this. In 2025, the question for engineering leaders is how to integrate them without breaking the trust and safety of the delivery pipeline.

The teams that win won't be the ones that let AI run wild. They will be the ones that build an agentic layer that respects the rigor of Platform Engineering while reclaiming the hours lost to the coordination gap.

---
*Related reading:*
- [Why I Started Building My Own DevOps Platform](/blog/2025/02/15/why-i-started-building-my-own-devops-platform-and-what-i-learned/)
- [Designing Multi-Agent Systems: Lessons from an 8-Agent Orchestra](/blog/2025/08/28/designing-multi-agent-systems-lessons-from-building-an-8-agent-engineering-orchestra/)
