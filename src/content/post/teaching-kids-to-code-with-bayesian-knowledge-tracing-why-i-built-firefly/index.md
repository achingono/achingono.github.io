---
author: "Alfero Chingono"
title: "Teaching Kids to Code With Bayesian Knowledge Tracing: Why I Built FireFly"
date: 2025-05-08T09:00:00Z
draft: false
description: "Why I built FireFly around mastery, not streaks: using Bayesian Knowledge Tracing, visual execution, and age-adaptive tutoring to help kids actually understand code."
slug: teaching-kids-to-code-with-bayesian-knowledge-tracing-why-i-built-firefly
tags: [
"FireFly",
"Education",
"Bayesian Knowledge Tracing",
"Python",
"AI Tutor",
"Learning"
]
categories: [
"Build in Public",
"AI",
"Education"
]
image: "cover.png"
---

Most kids' coding products are better at rewarding momentum than understanding.

They make it easy to complete the next exercise, collect the badge, and keep moving. What they do not always make easy is answering the question that actually matters: **does the learner understand what the code is doing?**

That question is the reason I started building [FireFly](https://github.com/achingono/firefly).

I wanted something that combined three ideas:

1. kids should write **real code**
2. they should be able to **see it execute step by step**
3. progression should depend on **demonstrated understanding**, not just content completion

That third requirement is what led me to Bayesian Knowledge Tracing.

## Why I did not want a purely linear curriculum

One thing I dislike about a lot of beginner coding experiences is that they quietly assume exposure equals mastery.

Finish the lesson? Great, on to the next topic.

But anyone who has taught programming knows that this is not how understanding works. A learner can complete an exercise and still have a shaky mental model. They may get the right answer for the wrong reason. They may be copying a pattern without being able to transfer it.

I wanted FireFly to behave more like a patient tutor than a content playlist.

That meant the platform needed a way to estimate whether a concept was becoming solid over time, not just whether one attempt happened to be correct.

## Why Bayesian Knowledge Tracing felt like the right fit

Bayesian Knowledge Tracing (BKT) gave me something I liked immediately: it is simple enough to reason about, but still more honest than a naive pass/fail progression model.

In FireFly, each concept gets a mastery probability. After every attempt, the platform updates that probability and decides whether the learner has actually crossed the mastery threshold.

The model in the current implementation uses fixed, interpretable parameters:

- initial knowledge (`pL0`) = `0.10`
- learning transition (`pT`) = `0.20`
- guess probability (`pG`) = `0.25`
- slip probability (`pS`) = `0.10`
- mastery threshold = `0.80`

I like this because it mirrors how real learning feels. One correct answer can move a student forward meaningfully, but not crown them an expert. A wrong answer does not erase everything, but it does lower confidence. Consistent performance matters more than isolated wins.

That is much closer to how a thoughtful teacher actually thinks.

## Why BKT alone is not enough

A mastery model is only as good as the learning experience feeding it.

That is why FireFly is not just a scoring engine. The heart of the product is still the visual code stepper.

Kids write Python in a Monaco editor, run it through a sandboxed execution engine, and watch the program unfold line by line with stack, heap, variables, and output. Under the hood, the backend wraps Python code with a `sys.settrace()` tracer before submitting it to Judge0. The result is a structured trace that the UI can render frame by frame.

That matters because novices do not just need answers. They need a way to build mental models.

If a child can see:

- which line ran
- what changed in local variables
- what got printed
- where a loop is getting stuck

then the feedback stops being mysterious.

Once you combine that with mastery tracking, you get something more interesting than "did the code run?" You get a system that can ask, "is this learner genuinely building understanding over time?"

## Why I made the experience age-adaptive

Another thing I did not want was one aesthetic and one teaching voice for everybody.

An eight-year-old and a fifteen-year-old can both be beginners, but they do not need the same interface. FireFly uses three age modes:

- **Fun** for younger learners
- **Balanced** for middle learners
- **Pro** for older learners

The execution engine stays the same, but the tone, visuals, density, and celebration style change. I wanted the underlying system to stay serious while the presentation adapts to the learner.

That principle carries into the AI tutor as well. The goal is not to hand out solutions. The goal is to provide age-appropriate hints, explanations, and nudges that keep the learner thinking.

## What I learned while building it

The more I worked on FireFly, the more I became convinced that the real bottleneck in beginner programming is not lack of content. It is lack of visible causality.

Kids get stuck because code feels magical until it does not work. Then it feels hostile.

The combination of execution tracing and mastery tracking is my attempt to close that gap:

- the **trace** makes program behavior visible
- the **tutor** makes the explanation conversational
- the **BKT layer** prevents false confidence from becoming progression

That combination feels much closer to how I would want a patient, thoughtful mentor to teach.

## Why I still think this matters

I did not build FireFly because kids need more gamification. They already have plenty of that.

I built it because I think programming is one of the few subjects where you can watch thought become behavior almost immediately, and that is incredibly powerful when it is made visible.

If a learner can write code, see it run, get unstuck without being spoon-fed, and unlock the next idea only after they really understand the current one, then we are not just teaching syntax. We are teaching how to think.

That is the kind of learning experience I wanted to build.

References:

- [FireFly repository](https://github.com/achingono/firefly)
- [FireFly product specification](https://github.com/achingono/firefly/blob/main/docs/SPEC.md)
- [FireFly architecture overview](https://github.com/achingono/firefly/blob/main/docs/architecture/overview.md)
- [FireFly mastery API](https://github.com/achingono/firefly/blob/main/docs/api/mastery.md)
