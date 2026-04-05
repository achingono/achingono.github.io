---
author: "Alfero Chingono"
title: "Fixing Deep Nesting in TypeScript: A Real Refactor From CueMarshal"
date: 2026-02-10T09:00:00Z
draft: true
description: "How to handle deeply nested conditional logic without losing your mind. A deep dive into a real refactor of CueMarshal's Gitea MCP server."
slug: fixing-deep-nesting-in-typescript-a-real-refactor-from-cuemarshal
tags: [
"TypeScript",
"Refactoring",
"Clean Code",
"CueMarshal",
"MCP",
"Software Engineering"
]
categories: [
"Craft",
"Software Engineering"
]
image: "cover.png"
---

There's a point in the lifecycle of any project where the "quick fix" logic starts to pile up. For the [Gitea MCP server](/blog/2025/11/27/mcp-at-scale-how-i-used-model-context-protocol-to-connect-ai-agents-to-gitea/) in [CueMarshal](https://www.cuemarshal.com), that point was the `get_pr_diff` tool.

What started as a simple API call had grown into a 150-line function with four levels of nested `if/else` statements. It was classic "Arrow Code": the logic was pointed so far to the right that it was getting hard to read on a standard monitor.

Here's how I refactored it and the TypeScript patterns I used to make it maintainable.

## The Problem: The "Arrow Code" Trap

The original function had to handle multiple failure modes:

1.  Check if the repository exists.
2.  Check if the pull request exists.
3.  Check if the diff is too large for the LLM's context.
4.  Handle the Gitea API's specific error formats.

If any of these failed, I had to return a structured error message. This led to a pattern of:

```typescript
if (repoExists) {
  if (prExists) {
    const diff = await getDiff();
    if (diff.length < LIMIT) {
      // Success!
    } else {
      // Error: Too large
    }
  } else {
    // Error: PR missing
  }
} else {
  // Error: Repo missing
}
```

This is hard to follow because the "Success" case is buried in the middle, and the "Error" cases are disconnected from their triggers.

## The Solution: Guard Clauses and "Return Early"

The first step was to flip the logic. Instead of nesting for success, I used **Guard Clauses** to return early on failure.

```typescript
if (!repoExists) return { error: "Repo missing" };
if (!prExists) return { error: "PR missing" };

const diff = await getDiff();
if (diff.length >= LIMIT) return { error: "Diff too large" };

// Success! (Now at the top-level indentation)
return { diff };
```

This immediately flattened the function and made it much more readable.

## Leveling Up with TypeScript's `Result` Type

To make this even more robust, I introduced a `Result` type pattern (inspired by Rust or F#). Instead of returning `null` or throwing an error, every step of the process returns an object that explicitly states if it succeeded or failed.

```typescript
type Result<T, E = string> = 
  | { ok: true; value: T } 
  | { ok: false; error: E };
```

Using this pattern, I could chain the operations together. If any step returns `{ ok: false }`, the whole chain stops and returns that error.

## The "Functional" Refactor

The final version of the `get_pr_diff` tool now looks like this:

1.  **Validate Inputs:** A clean, typed function that ensures the repo and PR IDs are valid.
2.  **Fetch PR Data:** A separate function that handles the Gitea API call and maps the error codes to human-readable messages.
3.  **Process Diff:** A dedicated function for the logic of truncation and formatting.

By splitting these into small, testable functions, the main `execute` method for the tool became a simple 10-line orchestrator.

## Why This Matters for Agentic AI

Refactoring for "Clean Code" isn't just for human developers anymore. When you are building [Agentic Orchestras](/blog/2025/08/28/designing-multi-agent-systems-lessons-from-building-an-8-agent-engineering-orchestra/), your code *is* the environment the agent has to reason about.

If your codebase is full of deep nesting and "spaghetti" logic, the AI is more likely to make mistakes when trying to refactor or extend it. Clean, flat, typed code makes your system more "agent-friendly."

The refactor of the Gitea MCP server didn't just make my life easier; it made [CueMarshal's agents](/blog/2025/11/27/mcp-at-scale-how-i-used-model-context-protocol-to-connect-ai-agents-to-gitea/) faster and more reliable.

---
*Related reading:*
- [MCP at Scale: Connecting AI Agents to Gitea](/blog/2025/11/27/mcp-at-scale-how-i-used-model-context-protocol-to-connect-ai-agents-to-gitea/)
- [Designing Multi-Agent Systems: Lessons from an 8-Agent Orchestra](/blog/2025/08/28/designing-multi-agent-systems-lessons-from-building-an-8-agent-engineering-orchestra/)
