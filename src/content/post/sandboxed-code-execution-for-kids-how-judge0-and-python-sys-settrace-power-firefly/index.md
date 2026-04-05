---
author: "Alfero Chingono"
title: "Sandboxed Code Execution for Kids: How Judge0 and Python sys.settrace Power FireFly"
date: 2026-01-15T09:00:00Z
draft: true
description: "Why standard code execution isn't safe for kids, and how I built a secure, traceable environment for FireFly using Judge0 and Python's tracing hooks."
slug: sandboxed-code-execution-for-kids-how-judge0-and-python-sys-settrace-power-firefly
tags: [
"FireFly",
"Python",
"Judge0",
"Security",
"EdTech",
"Sandboxing",
"Build in Public"
]
categories: [
"Craft",
"Build in Public"
]
image: "cover.png"
---

When you build a platform for kids to learn to code, like [FireFly](/blog/2025-05-08-teaching-kids-to-code-with-bayesian-knowledge-tracing-why-i-built-firefly/), you face a unique technical challenge: **safety**.

It's one thing to let a developer run arbitrary code in a container. It's another thing to let a 7-year-old—who might accidentally (or intentionally) write an infinite loop or a memory-hogging script—run code on your servers.

For FireFly, I needed a solution that was:

1.  **Secure:** No "breakouts" to the host machine.
2.  **Performant:** Near-instant execution so the learning flow isn't broken.
3.  **Traceable:** I needed to know *exactly* which line of code was being executed at any given time to power the "AI Tutor's" feedback.

The solution was a combination of **Judge0** and **Python's `sys.settrace`**.

## Layer 1: The Hard Sandbox (Judge0)

The first line of defense is **Judge0**, an open-source online code execution system. I run Judge0 in a set of Docker containers. When a student in FireFly clicks "Run," their code is sent to the Judge0 API, which:

*   Creates a temporary, isolated worker.
*   Enforces strict CPU and memory limits.
*   Limits the execution time to a few seconds.
*   Returns the output (or the error).

This handles the "macro" safety. Even if a student tries to `import os; os.system('rm -rf /')`, Judge0 will catch it or isolate the damage to a disposable container.

## Layer 2: The Soft Sandbox (Python `sys.settrace`)

Judge0 is great for safety, but it doesn't give me the "why" behind a student's code. To power a [Socratic AI Tutor](/blog/2025/08/05/how-i-wired-up-an-ai-tutor-to-teach-like-a-socratic-mentor-not-a-cheater/), the system needs to see the internal state of the execution: which variables are changing, and which lines are being hit.

To do this, I wrap the student's Python code in a "tracer" script that uses `sys.settrace()`. This is a built-in Python hook that allows you to run a function for every single line of code executed.

### How the Tracer Works:

1.  **Line-by-Line Tracking:** As the code runs, the tracer records the line number and the values of all local variables.
2.  **Instruction Limit:** If the code takes too many "steps" (like an infinite loop), the tracer raises a custom exception and stops the execution before Judge0 even has to intervene.
3.  **State Snapshot:** At the end of the run, the tracer returns a "breadcrumb" of the entire execution.

## Layer 3: The AI Tutor Feedback Loop

The "breadcrumb" from the tracer is what makes the [FireFly AI Tutor](/blog/2025/08/05/how-i-wired-up-an-ai-tutor-to-teach-like-a-socratic-mentor-not-a-cheater/) so effective. Instead of just seeing "Error: NameError: name 'x' is not defined," the AI can see: "The student defined `x` on line 2, but they are trying to use it on line 5 inside a function where it's not in scope."

This level of detail allows the AI to ask much better [Socratic questions](/blog/2025/08/05/how-i-wired-up-an-ai-tutor-to-teach-like-a-socratic-mentor-not-a-cheater/).

## Why This Matters for EdTech

We often think of "sandboxing" as a security feature for protecting servers. But in EdTech, sandboxing is a **pedagogical feature**. By creating a safe, observable environment, we give kids the freedom to experiment, break things, and learn from their mistakes without any real-world consequences.

Building this infrastructure for FireFly has been one of the most rewarding "Craft" challenges of the project. It's where security meets education, and where complex systems make simple, delightful learning experiences possible.

---
*Related reading:*
- [How I Wired Up an AI Tutor to Teach Like a Socratic Mentor](/blog/2025/08/05/how-i-wired-up-an-ai-tutor-to-teach-like-a-socratic-mentor-not-a-cheater/)
- [Teaching Kids to Code With BKT: Why I Built FireFly](/blog/2025-05-08-teaching-kids-to-code-with-bayesian-knowledge-tracing-why-i-built-firefly/)
