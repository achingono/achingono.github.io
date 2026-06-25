---
author: "Alfero Chingono"
title: "SKILL.md to Runtime Tools: How LeanKernel Loads Skills Without Turning Security into a Guess"
date: 2026-08-06T09:00:00Z
draft: true
description: "A practical look at LeanKernel’s SKILL.md contract: parsing, validation, quarantine for invalid skills, egress allow lists for HTTP skills, and hot reload behavior for dynamic tool registration."
slug: skill-definition-format-quarantine-runtime-egress-allowlists
tags: [
  "LeanKernel",
  "Security",
  "AI Agents",
  "Tooling",
  "Runtime Skills"
]
categories: [
  "Software Engineering",
  "Platform Engineering"
]
image: "cover.png"
---

Dynamic tool loading is one of those features that sounds safe until you ship it.

If you let an agent discover and execute “whatever the repo contains,” you end up with an unbounded attack surface. It might be malicious. It might just be broken. Either way, your agent now calls code you didn’t intend.

LeanKernel’s SKILL.md system attacks the problem by moving the decision boundary earlier than most projects do.

It parses skill metadata, validates it, and refuses to register invalid skills.

## The contract: runtime.type, runtime.command, and egress policy

LeanKernel dynamic skills come from `SKILL.md` files. A `SkillParser` loads and interprets a small frontmatter schema.

The minimum required frontmatter includes `name`, `description`, and a `runtime` block.

The runtime block decides what the skill can do at execution time. It supports `type: cli`, `http`, and `composite`.

For CLI skills you must provide a `runtime.command` plus optional binary requirements.

For HTTP skills you must declare an egress allow list under `runtime.egress.allowHosts`. That list must be non-empty, and the runtime uses it to keep outbound calls inside your intended boundaries.

The contract also defines operations. Each operation has an `id`, a short `summary`, and an `invoke` section describing how the tool should run.

The practical point is that SKILL.md is not “free-form instructions.” It is a machine-checked description of what the runtime can execute.

## Validation: quarantine invalid skills instead of best-effort registration

LeanKernel does not pretend that every `SKILL.md` is valid.

If a skill fails parsing or fails validation rules, LeanKernel quarantines it. Quarantined skills stay out of the active tool registry.

The runtime still tracks quarantined skills in memory and emits logs so you can find the problem. It does not silently fall back to partial behavior.

This is the difference between “dynamic” and “chaotic.”

## Security posture: SSRF risk belongs in a hard allow list

The worst failure mode for HTTP skills is not “the call fails.” The worst mode is “the call reaches a place you did not intend.”

LeanKernel’s design forces you to declare allowed hosts. HTTP skills require non-empty `runtime.egress.allowHosts`, and the egress policy layer uses the declared allow list.

That means your security posture lives in metadata that you can review like code.

It also means you can write tests around the contract without executing the skill.

## Hot reload: SKILL.md changes refresh the registry quickly

Dynamic skills are only useful if they update without restarting the whole service.

LeanKernel adds file watchers with a debounce window of 250ms. When `SKILL.md` changes, the runtime refreshes the registry and the plugin host.

That behavior matters operationally. You can fix a broken skill definition and see the runtime register it after the debounce, instead of waiting for a restart cycle.

## Binary availability: skip tools that cannot run

CLI skills can declare `runtime.requires.bins`.

LeanKernel checks binary availability through an `IBinaryResolver`. Skills that require missing binaries get marked unavailable and are skipped during registration.

That keeps the agent from discovering a tool that always fails at runtime.

## How governance fits in

LeanKernel also applies tool governance. A turn flow constructs a visible tool set first, and then it exposes those tool names into the prompt context.

When you add dynamic skills, you want that visibility pass to reflect only what has passed validation.

SKILL.md quarantine makes that easy: invalid skills never enter the registry, so governance does not need extra “trust me” checks.

## A concrete workflow for adding a new skill

If you add a new dynamic tool, the boring sequence matters:

| Step | What you do |
| --- | --- |
| 1 | Create `data/skills/<name>/SKILL.md` with `runtime.type`, `runtime.command` (for CLI), and declared `operations`. |
| 2 | For HTTP skills, set `runtime.egress.allowHosts` to the exact hosts you expect. |
| 3 | If the skill declares required bins, place the binaries so the resolver can find them. |
| 4 | Verify the runtime logs show the skill registered, not quarantined. |

Once the skill registers, governance can expose it to the agent for the turns where the caller allows the relevant tool names or categories.

## The takeaway

If you want dynamic tools without dynamic risk, you need a tight loop:

- metadata-first contracts
- strict validation
- quarantine for failures
- egress allow lists for outbound calls

That’s how you make “skills” feel like code, not like vibes.
