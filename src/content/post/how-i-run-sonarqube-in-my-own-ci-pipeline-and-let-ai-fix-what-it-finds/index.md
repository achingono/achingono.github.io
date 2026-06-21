---
author: Alfero Chingono
title: How I Run SonarQube in My Own CI Pipeline (And Let AI Fix What It Finds)
date: 2026-03-05T09:00:00.000Z
description: The pattern I use to turn SonarQube from passive reporting into an active remediation loop with labels, issues, AI-assisted fixes, and human-controlled pull requests.
slug: how-i-run-sonarqube-in-my-own-ci-pipeline-and-let-ai-fix-what-it-finds
tags:
  - SonarQube
  - DevSecOps
  - CI/CD
  - Automation
  - AI
  - Code Quality
categories:
  - Agentic AI
  - DevSecOps
  - Automation
image: cover.png
---

I did not want SonarQube to become another dashboard that told me what was wrong and then sat there waiting for a human to do all the work.

What I wanted was a loop.

Run the scan. Turn findings into something actionable. Let AI handle the obvious remediation. Keep a human in charge of the merge. Scan again.

That is the difference between reporting and operations.

## Running SonarQube is easy. Making it useful is the hard part.

A scan by itself is not the interesting problem. The interesting problem is whether the findings actually change what happens next.

The pattern I use is pretty simple:

1. run the scan on a schedule
2. translate findings into issues with enough structure to act on
3. let AI or agents handle the obvious remediation work
4. keep human review as the merge gate
5. rescan and repeat

That is the loop I have been using across FireFly and CueMarshal.

## FireFly: findings become issues, not just report output

In FireFly, the scheduled GitHub Action spins up SonarQube Community as a service container, sets the admin password, creates the project, generates an analysis token, runs the scanner in Docker, and then fetches open issues through the SonarQube API.

The useful part is what happens after that.

Instead of treating the scan as the end of the process, the workflow turns findings into GitHub issues with labels like:

- `sonar`
- `sonar: bug`
- `sonar: vulnerability`
- `sonar: security hotspot`
- `sonar: blocker`
- `sonar: critical`
- `sonar: major`

That labeling step is small, but it changes the feel of the system. The finding moves out of a report and into the normal engineering flow.

The issue body is also intentionally plain: key, severity, type, rule, file, line, and message. That is enough context to act without bouncing back and forth to SonarQube every time.

## CueMarshal: the findings re-enter the agent loop

CueMarshal uses the same basic idea, but the loop is tighter.

There, SonarQube is not just a quality gate. It is a signal source for the self-improvement workflow.

The scan runs on a schedule, the quality gate gets checked, and any remaining issues are routed into the agent system. That workflow runs deterministic scanners, produces a findings JSON file, and lets AI choose the items that are actually worth turning into repository work.

After that, the path is familiar:

- finding becomes issue
- issue gets labels like `self-improvement` and `source:sonar`
- developer agent works the task
- reviewer agent reviews it
- human still controls the merge

That is the part that makes the whole thing feel operational instead of theoretical.

## The useful part is seeing what actually gets fixed

This pattern felt more real to me once I could see it in commits instead of just in a diagram.

In FireFly, the SonarQube-driven fixes included things like:

- critical auth and data exposure issues
- medium-severity issues in the LLM, tracer, and execution paths
- blocker and critical tracer problems
- remaining major issues in non-UI files

In CueMarshal, the same loop showed up differently:

- bug-class findings resolved
- cognitive-complexity hotspots refactored
- scan-flow issues fixed so the pipeline itself became more reliable

That is what convinced me this was worth doing. The AI was not magically “doing security.” It was working inside a bounded remediation process with real input, reviewable output, and a cleaner next scan.

## What I still keep human

I do not want all findings auto-fixed blindly.

Some changes affect security-sensitive behavior. Some touch orchestration logic. Some need architectural judgment more than mechanical cleanup.

That is why I still care about review gates, protected areas, and explicit pull requests. AI can triage. AI can repair a lot. But the system only stays trustworthy when people keep approval authority over the consequential parts.

## Why this pattern matters to me

I have less patience for passive quality tools every year.

If a scan only tells me what is wrong, that is useful.
If it creates the next actionable task, that is better.
If that task can go through an AI-assisted workflow and still land in a human-reviewed PR, then the tool is part of delivery instead of commentary on delivery.

That is the threshold I care about.

I still think DAST and pipeline security automation matter deeply. The earlier OWASP post was about that. But SonarQube plus an AI remediation loop feels like the next step: make quality signals operational, not ornamental.

References:

- [FireFly scheduled SonarQube workflow](https://github.com/achingono/firefly/blob/main/.github/workflows/scheduled-scan.yml)
- [FireFly SonarQube project configuration](https://github.com/achingono/firefly/blob/main/sonar-project.properties)
- [CueMarshal self-improvement design](https://github.com/cuemarshal/cuemarshal/blob/main/docs/operations/self-improvement.md)
- [CueMarshal architecture overview](https://github.com/cuemarshal/cuemarshal/blob/main/docs/architecture/overview.md)
