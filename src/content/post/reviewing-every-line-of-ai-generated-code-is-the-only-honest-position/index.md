---
author: Alfero Chingono
title: Reviewing Every Line of AI-Generated Code Is the Only Honest Position
date: 2026-06-11T09:00:00.000Z
draft: true
description: Treating AI code generation as a 'higher level of abstraction' is a dodge. If your name is on the PR, the review bar is the same as code you typed yourself.
slug: reviewing-every-line-of-ai-generated-code-is-the-only-honest-position
tags:
  - AI Agents
  - Software Engineering
  - Code Review
  - CI/CD
  - SonarQube
  - OpenClaw
categories:
  - Software Engineering
  - AI Agents
image: cover.png
---

A lot of smart people are arguing that reviewing AI-generated code line-by-line is unnecessary — that AI code generation is "just a higher level of abstraction," and reading every line is as pointless as reading the assembly your compiler emits. I disagree, and I think the argument sneaks in an assumption that does not hold up under a real incident.

The assumption is that the abstraction is well-specified. A compiler is a function. Claude is not.

## The compiler analogy is doing too much work

The argument goes like this. We do not read the assembly that gcc emits. We trust the compiler because it is a deterministic, specified, well-tested function from source to machine code. AI code generation, so the claim goes, is simply the next rung: we will soon trust the model the same way, and line-by-line review will start to look quaint.

I understand the appeal. I also think the analogy is wrong in a way that matters.

A compiler is a function with a published specification and a decades-long track record of being fixed when it violates that specification. If gcc miscompiles something, the response is an issue, a regression test, a patch. The next release does not silently change its behavior on the same input.

Claude does. GPT does. Gemini does. They all do.

The mapping from prompt to code is many-to-many. The same prompt produces different code across sessions, across model versions, and sometimes across retries five seconds apart. The model is not a function; it is a sampled distribution over plausible continuations. That distribution contains plenty of plausible-but-wrong patterns that a compiler would never emit, because a compiler does not invent APIs, misuse library primitives, or quietly reinvent a weaker version of something your code already does.

So when someone says "reading AI code is like reading assembly," the honest rewrite is: reading AI code is like reading the output of a brilliant, motivated contractor who also hallucinates sometimes, disagrees with the style guide when they feel like it, and occasionally cites a function that was removed two major versions ago.

You would review that contractor's PRs. Closely.

## The position I keep landing on

If my name is on the pull request, I own every line. Same bar as code I typed myself.

I have tried the softer version of this. It does not hold up under an incident. When something broke in production, nobody in the postmortem cared which lines I had typed and which ones came out of an autocomplete. The blast radius did not care either. The customer definitely did not care.

"The model wrote it" is not a defense. It is a confession.

## What the review bar actually looks like in my workflow

The way I stay honest about this without drowning in diffs is to change the shape of the work, not the shape of the review.

First, I keep generations small. If I ask for a 2,000-line change, I am not going to review it properly; I am going to skim it and lie to myself about having reviewed it. So I scope prompts to things I would be willing to write by hand in one sitting. A function. A config loader. A migration. One concern at a time.

Second, static analysis runs before I do. I wrote about [how I run SonarQube in my own CI pipeline](../how-i-run-sonarqube-in-my-own-ci-pipeline-and-let-ai-fix-what-it-finds/) and let the AI take a first pass at fixing what it finds. That catches a depressingly high fraction of the AI-shaped bugs: dead branches, off-by-one, unused variables, resources not closed, null checks that exist twice. The point is not that SonarQube is smart. The point is that the class of mistakes AI makes overlaps heavily with the class of mistakes a linter already recognizes, and making the tool catch them first shrinks what I have to read with my brain.

Third, there is a fix loop. The model proposes fixes for the findings, I accept or reject each one, and only then does a human review start. This is where the review bar actually lands. By the time the diff hits me, it has survived a linter, a type checker, and a round of automated fixes. What is left is the part that needs judgment.

Fourth, tests the model did not write. This is the single highest-leverage habit I have picked up. Models tend to write tests that mirror the implementation they just produced, which means the tests pass for exactly the wrong reason: they encode the same assumptions. So I write at least one test that the model never saw the prompt for. Adversarial cases. Empty inputs. The weird boundary you would not mention in a spec. Those find the interesting failures.

None of this is novel. None of it requires special tooling. It is discipline, not infrastructure.

## The counter-argument I take seriously

Reviewer fatigue is real. This is the part where I think some of the "just review everything" crowd is being a bit glib.

If the team's expectation is that AI doubles output but review throughput stays the same, the review is going to get worse. Humans do not scale linearly under volume; they start waving things through. At some point the review becomes a ritual rather than a check, and the "every line" policy is a fiction that people stop enforcing in private before they stop enforcing it in public.

So the honest version of the position is not "review harder." It is: shrink the review surface to something you can actually carry.

That means narrower prompts, stronger types, more constraints in the project itself so the model has fewer ways to go wrong, and smaller diffs. It means saying no to the 2,000-line refactor the model is eager to produce. It means building what [the OpenClaw gateway](../inside-the-dockerfile-behind-my-openclaw-gateway/) does under the hood: scope work so tightly that review is still a feasible human activity.

If you cannot review it, you cannot ship it. That is not a statement about AI. That has always been true.

## Accountability, not distrust

The framing I keep seeing, that demanding line-by-line review reflects a lack of trust in AI, gets the direction backward.

Review is not about the tool. It never was.

When a senior engineer reviews a junior's code, it is not because the junior is untrustworthy. It is because the senior is accountable for the outcome and needs to understand what went in. The review is the mechanism by which accountability is real rather than theatrical. Swap the junior for a model and the accountability does not move. It still sits with whoever presses merge.

So the review bar is the same. The tools that help me hit it have changed a lot. The bar has not.

If you want AI to write more of your code, fine. I do too. But the deal is this: you read it, you own it, and when it breaks at 3am, you are the one on the bridge call, not the model.

That is the only version of this story that holds together.
