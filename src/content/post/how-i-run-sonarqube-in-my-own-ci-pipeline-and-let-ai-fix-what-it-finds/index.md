---
author: "Alfero Chingono"
title: "How I Run SonarQube in My Own CI Pipeline (And Let AI Fix What It Finds)"
date: 2026-03-05T09:00:00Z
draft: false
description: "The pattern I use to turn SonarQube from passive reporting into an active remediation loop with labels, issues, AI-assisted fixes, and human-controlled pull requests."
slug: how-i-run-sonarqube-in-my-own-ci-pipeline-and-let-ai-fix-what-it-finds
tags: [
"SonarQube",
"DevSecOps",
"CI/CD",
"Automation",
"AI",
"Code Quality"
]
categories: [
"Agentic AI",
"DevSecOps",
"Automation"
]
image: "cover.png"
---

I wrote in 2024 about [automating OWASP scan reports in Azure DevOps](/blog/2024/09/05/automating-owasp-scan-reports-in-azure-devops/) because I wanted security scanning to become part of the delivery flow instead of an afterthought.

This post is the next step in that same direction.

The thing I wanted from SonarQube was not another dashboard full of guilt. I wanted a loop that could actually create work, route it, fix it, and come back cleaner on the next scan.

That changed the design completely.

## The real goal was not "run SonarQube"

Running SonarQube is easy.

Turning findings into a useful engineering loop is the hard part.

The pattern I have found most practical looks like this:

1. run the scan on a schedule
2. translate findings into issues with enough structure to act on
3. let AI or agents handle the obvious remediation work
4. keep human review as the merge gate
5. rescan and repeat

That is what I have been doing across FireFly and CueMarshal.

## The FireFly version: temporary SonarQube, durable issues

In [FireFly](https://github.com/achingono/firefly), the workflow is intentionally self-contained.

The scheduled GitHub Action spins up a SonarQube Community service container, sets the admin password, creates the project, generates an analysis token, runs the scanner in Docker, and then uses the SonarQube API to fetch open issues.

From there, the workflow does something I think is more useful than just failing the pipeline: it turns findings into **GitHub issues with meaningful labels**.

The labels encode both issue type and severity:

- `sonar`
- `sonar: bug`
- `sonar: vulnerability`
- `sonar: security hotspot`
- `sonar: blocker`
- `sonar: critical`
- `sonar: major`

That small step matters a lot. Once the findings live as first-class issues in the repo, they stop being hidden inside a scan report and start participating in the normal engineering workflow.

The FireFly workflow also keeps the body format clean: key, severity, type, rule, file, line, and the actual message. That makes the issue understandable without forcing someone to click back into SonarQube every time.

## The CueMarshal version: findings re-enter the agent loop

CueMarshal takes the pattern further.

There, SonarQube is not just a quality gate. It is a signal source for the self-improvement system.

The scan runs on a schedule, the quality gate is checked, and when issues remain, they are picked up by the self-improvement workflow. That workflow runs deterministic scanners, produces a findings JSON file, and lets AI select the high-value, automation-friendly items to turn into actual repository work.

At that point the flow becomes very CueMarshal-like:

- finding becomes issue
- issue gets labels such as `self-improvement` and `source:sonar`
- developer agent works the task
- reviewer agent reviews it
- human still controls the merge

That is the part I care about most. Static analysis becomes part of an operational loop instead of a reporting loop.

## What AI actually fixed

This pattern became more convincing to me once I could see it in the commit history instead of just in a diagram.

In FireFly, the SonarQube-driven fixes moved through recognizable stages:

- critical auth and data exposure issues
- medium-severity issues in the LLM, tracer, and execution paths
- blocker and critical tracer problems
- remaining major issues in non-UI files

In CueMarshal, the same loop showed up in a different form:

- bug-class findings resolved
- cognitive-complexity hotspots refactored
- scan-flow issues fixed so the SonarQube pipeline itself became more reliable

That is the detail that made the whole approach feel real to me. The AI was not "doing security" in some theatrical sense. It was participating in a bounded remediation loop with concrete input, reviewable output, and a cleaner next scan.

## What I still keep human

I do not think static analysis findings should all be auto-fixed blindly.

Some changes affect security-sensitive behavior. Some touch core orchestration logic. Some need architectural judgment more than mechanical cleanup.

That is why I still care so much about review gates, protected areas, and explicit pull requests. AI can do triage. AI can do a surprising amount of repair work. But the system becomes trustworthy only when people retain approval authority over the consequential parts.

This is the same design instinct behind CueMarshal more broadly: automate aggressively, but make the control points obvious.

## Why I like this pattern

The more repositories I maintain, the less patience I have for passive quality tooling.

If a scan only tells me what is wrong, it is useful.
If a scan creates the next actionable task, it is much more useful.
If that task can be routed through an AI-assisted workflow and still land in a human-reviewed PR, then the tool has become part of delivery rather than commentary on delivery.

That is the threshold I care about now.

I still think DAST and pipeline security automation matter deeply; that earlier OWASP post still reflects that. But SonarQube plus an AI remediation loop feels like the next generation of the same idea: make quality signals operational, not ornamental.

If you want the broader architecture around this, [Designing Multi-Agent Systems: Lessons from Building an 8-Agent Engineering Orchestra](/blog/2025/08/28/designing-multi-agent-systems-lessons-from-building-an-8-agent-engineering-orchestra/) covers the orchestration side, and [Why I Started Building My Own DevOps Platform](/blog/2025/02/15/why-i-started-building-my-own-devops-platform-and-what-i-learned/) covers the bigger motivation.

References:

- [FireFly scheduled SonarQube workflow](https://github.com/achingono/firefly/blob/main/.github/workflows/scheduled-scan.yml)
- [FireFly SonarQube project configuration](https://github.com/achingono/firefly/blob/main/sonar-project.properties)
- [CueMarshal self-improvement design](https://github.com/cuemarshal/cuemarshal/blob/main/docs/operations/self-improvement.md)
- [CueMarshal architecture overview](https://github.com/cuemarshal/cuemarshal/blob/main/docs/architecture/overview.md)
