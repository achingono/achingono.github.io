---
author: "Alfero Chingono"
title: "How I Wired Up an AI Tutor to Teach Like a Socratic Mentor — Not a Cheater"
date: 2025-08-05T09:00:00Z
draft: true
description: "Why most AI tutors are just homework-solvers, and how I used specific prompting and Bayesian Knowledge Tracing to make FireFly a real teacher."
slug: how-i-wired-up-an-ai-tutor-to-teach-like-a-socratic-mentor-not-a-cheater
tags: [
"FireFly",
"EdTech",
"AI",
"Learning",
"Prompt Engineering",
"Build in Public"
]
categories: [
"Build in Public",
"AI"
]
image: "cover.png"
---

The problem with most "AI tutors" today is simple: they are too helpful. If a student asks "How do I solve for x?", a generic LLM will often just show the steps and give the answer. In an educational context, that isn't teaching; it is a shortcut to a finished worksheet with zero retention.

When I started building [FireFly](/blog/2025-05-08-teaching-kids-to-code-with-bayesian-knowledge-tracing-why-i-built-firefly/), my goal was the opposite. I wanted an AI that would act like a **Socratic Mentor**. It should never give the answer directly. It should only ask the next right question to help the student find the answer themselves.

That turned out to be a surprisingly hard technical challenge.

## The "Helpful Assistant" Bias

Large Language Models (LLMs) are trained to be helpful assistants. Their default behavior is to minimize the "effort" for the user. In education, you actually want to *maximize* the student's cognitive effort within a safe range (the Zone of Proximal Development).

To break the LLM's habit of just giving the answer, I had to move beyond simple system prompts and build a more structured interaction loop.

## Three Layers of a Socratic AI

In FireFly, the "AI Tutor" isn't just one prompt. It is three distinct layers working together.

### 1. The Knowledge Layer (BKT)

Before the AI says a word, the system checks the student's current mastery using [Bayesian Knowledge Tracing (BKT)](/blog/2025-05-08-teaching-kids-to-code-with-bayesian-knowledge-tracing-why-i-built-firefly/). If the system knows the student is 90% likely to understand `loops` but only 10% likely to understand `nested loops`, it passes that context to the LLM.

The prompt becomes: "The student understands X, but is struggling with Y. Ask a question that bridges the gap."

### 2. The Socratic Constraint

The core prompt for the FireFly tutor is built around strict negative constraints:

*   **NEVER** provide the full solution.
*   **NEVER** point out the exact line of the error.
*   **ALWAYS** ask a leading question.
*   **ALWAYS** validate the student's *process*, not just their output.

If the student is stuck on a syntax error, the AI might say: "I see you're trying to repeat a block of code. Have you looked at where your curly braces are starting and ending?"

### 3. The Age-Adapted Tone

Teaching a 7-year-old is different from teaching a 15-year-old. FireFly uses different "personas" depending on the user's profile. For younger kids, the tone is more encouraging and uses metaphors (like "the computer is a very literal robot"). For older students, the tone is more technical and precise.

## Dealing with the "Just Tell Me" Frustration

One of the biggest challenges in Socratic teaching is student frustration. When an AI keeps asking questions instead of giving answers, some students will just keep asking "Just tell me the answer."

To handle this, FireFly has a "Frustration Fuse." If the student asks for the answer three times in a row, the AI is allowed to provide a *hint* that is slightly more direct, or it can offer to "reset" the problem to an easier version. This keeps the student engaged without breaking the pedagogical goal.

## Why This Matters

We are entering an era where AI-powered personalized learning will be the norm. But if we just build "answer machines," we are doing a disservice to the next generation of learners.

Building FireFly taught me that the most powerful use of AI in education isn't knowing everything. It's being a patient, persistent, and occasionally annoying mentor who refuses to let the student take the easy way out.

---
*Related reading:*
- [Teaching Kids to Code With BKT: Why I Built FireFly](/blog/2025-05-08-teaching-kids-to-code-with-bayesian-knowledge-tracing-why-i-built-firefly/)
- [Sandboxed Code Execution for Kids: How Judge0 and Python sys.settrace Power FireFly](/blog/2026/01/15/sandboxed-code-execution-for-kids-how-judge0-and-python-sys-settrace-power-firefly/)
