---
title: How I Split Planning, Execution, and Review Across AI Coding Tools
date: "2026-06-14"
draft: true
summary: A practical workflow for using one tool to plan, another to execute, and a final pass to review without burning through limits.
tags: ai, coding-tools, workflows, agentic-ai
---

Recent social posts keep pointing at the same workflow: people plan with one model, execute with another, and then review the result before shipping.

That pattern is useful because it matches how the work actually feels. Planning needs breadth. Execution needs focus. Review needs distance.

I started noticing this after watching a run of posts about Claude Code, Codex, and Fable 5. The details changed, but the shape stayed the same. People were getting better results when they stopped asking one tool to do every part of the job.

In this post, I want to break that pattern into something concrete:

- use one tool to sketch the problem and outline the steps
- use another tool to do the work in smaller slices
- use a final pass to check for mistakes, edge cases, and awkward phrasing

That sounds simple, but it changes how much friction you feel while working. You spend less time fighting the model and more time steering it.

I have seen the same idea show up in the rest of my writing too. A lot of my recent posts are about systems: multi-agent setups, platform engineering, and the small choices that make workflows easier to run again tomorrow. This is the same thing, just at the level of day-to-day coding.

The goal is not to collect more tools. The goal is to give each tool a narrower job so the output gets better.
