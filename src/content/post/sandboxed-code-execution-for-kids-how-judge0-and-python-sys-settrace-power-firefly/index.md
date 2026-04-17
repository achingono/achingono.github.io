---
author: "Alfero Chingono"
title: "Sandboxed Code Execution for Kids: How Judge0 and Python sys.settrace Power FireFly"
date: 2026-01-15T09:00:00Z
draft: false
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

When you build a platform for kids to learn to code, like [FireFly](/blog/2025-05-08-teaching-kids-to-code-with-bayesian-knowledge-tracing-why-i-built-firefly/), the hard problem is **safety**.

Letting a developer run arbitrary code in a container is one thing. Letting a 7-year-old, who might accidentally or intentionally write an infinite loop or a memory-hogging script, run code on your servers is another.

For FireFly, I needed a solution that was:

1.  **Secure:** No "breakouts" to the host machine.
2.  **Fast:** Near-instant execution so the learning flow isn't broken.
3.  **Traceable:** I needed to know exactly which line was running at any moment so the AI tutor could give grounded feedback.

The solution ended up being a combination of **Judge0** and **Python's `sys.settrace()`**.

## Layer 1: The Hard Sandbox (Judge0)

The first line of defense is **Judge0**, an open-source online code execution system. I run Judge0 in a set of Docker containers. When a student in FireFly clicks "Run," their code is sent to the Judge0 API, which:

*   Creates a temporary, isolated worker.
*   Enforces strict CPU and memory limits.
*   Limits the execution time to a few seconds.
*   Returns the output (or the error).

This handles the outer safety boundary. Even if a student tries to `import os; os.system('rm -rf /')`, Judge0 catches it or confines the damage to a disposable container.

## Layer 2: The Soft Sandbox (Python `sys.settrace`)

Judge0 keeps the system safe, but it does not tell me why a student got stuck. To power a [Socratic AI Tutor](/blog/2025/08/05/how-i-wired-up-an-ai-tutor-to-teach-like-a-socratic-mentor-not-a-cheater/), the system needs to see the internal state of execution: which variables change, and which lines get hit.

To do that, I wrap the student's Python code in a tracer script that uses `sys.settrace()`. It is a built-in Python hook that lets you run a function for each executed line.

### How the Tracer Works:

1.  **Line-by-line tracking:** As the code runs, the tracer records the current line number and the values of local variables.
2.  **Instruction limit:** If the code takes too many steps, as in an infinite loop, the tracer raises a custom exception and stops execution before Judge0 has to step in.
3.  **State snapshot:** At the end of the run, the tracer returns a breadcrumb trail of the execution.

## Layer 3: The AI Tutor Feedback Loop

The "breadcrumb" from the tracer is what makes the [FireFly AI Tutor](/blog/2025/08/05/how-i-wired-up-an-ai-tutor-to-teach-like-a-socratic-mentor-not-a-cheater/) so effective. Instead of just seeing "Error: NameError: name 'x' is not defined," the AI can see: "The student defined `x` on line 2, but they are trying to use it on line 5 inside a function where it's not in scope."

This level of detail allows the AI to ask much better [Socratic questions](/blog/2025/08/05/how-i-wired-up-an-ai-tutor-to-teach-like-a-socratic-mentor-not-a-cheater/).

## Why This Matters for EdTech

We often think of "sandboxing" as a security feature for protecting servers. In EdTech, it is also a **pedagogical feature**. A safe, observable environment gives kids room to experiment, break things, and learn from their mistakes without real-world consequences.

Building this part of FireFly has been one of the most satisfying engineering problems in the project. It sits right where security requirements and teaching goals meet.

---
*Related reading:*
- [How I Wired Up an AI Tutor to Teach Like a Socratic Mentor](/blog/2025/08/05/how-i-wired-up-an-ai-tutor-to-teach-like-a-socratic-mentor-not-a-cheater/)
- [Teaching Kids to Code With BKT: Why I Built FireFly](/blog/2025-05-08-teaching-kids-to-code-with-bayesian-knowledge-tracing-why-i-built-firefly/)
