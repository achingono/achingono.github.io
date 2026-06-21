---
author: "Alfero Chingono"
title: "Browser Automation Via a Webwright Sidecar: Contracts, Defaults, and Cancelability"
date: 2026-06-20T09:00:00Z
draft: true
description: "How LeanKernel keeps browser automation out of the .NET process by wrapping Webwright in a sidecar, exposing async browser_* tools with strict security defaults and an operational contract."
slug: browser-automation-via-webwright-sidecar-leankernel
tags: [
  "LeanKernel",
  "Platform Engineering",
  "AI Agents",
  "Webwright",
  "Playwright",
  "Security"
]
categories: [
  "Platform Engineering",
  "Software Engineering"
]
image: "cover.png"
---

The moment you let an agent drive a real browser, you inherit the browser’s problems: heavy processes, brittle DOM timing, long-running steps, and failure modes that look like “the model is stuck.”

LeanKernel’s answer is to keep the browser runtime out of the .NET process. It talks to a Webwright sidecar over authenticated HTTP, and it exposes a small tool contract that matches how humans debug outages: submit, poll, inspect artifacts, cancel.

## The interface: browser_* tools are async by design

Browser automation registers as an optional tool family. When `LeanKernel:Webwright:Enabled=true`, the runtime exposes these tools under the `browser` category:

| Tool | What it returns |
| --- | --- |
| `browser_run_task` | `runId` |
| `browser_get_run` | status, final datum, error payload, and an artifact manifest |
| `browser_get_artifact` | one artifact fetched by opaque ID (base64 for binaries) |
| `browser_cancel_run` | a cancellation request that Webwright treats as idempotent |

The critical detail is that `browser_run_task` never blocks on browser completion. Callers submit work, then poll `browser_get_run` until the run reaches a terminal state: `succeeded`, `failed`, `cancelled`, or `timed_out`.

That sounds obvious, but it changes everything in practice. Your agent turn stays responsive. Your app stays operable. You can put backpressure around the polling loop without guessing how Webwright is behaving today.

## Security defaults: disabled first, then narrow

LeanKernel keeps browser automation off by default. Turning it on means you provide two pieces of secret material and a few operational limits.

The sidecar validates more than “is the token correct.” It checks task size, start URLs, bearer auth, queue limits, domain allow/deny rules, private IP ranges, and artifact confinement. It also refuses to hand your caller raw file paths. Artifacts show up only through the run’s manifest.

On the engine side, that maps to the tool contract, not ad-hoc checks inside the prompt layer. The governance pass already decides what tools the caller can see. Once the caller sees `browser_run_task`, the runtime routes the request into the sidecar client.

## Cancelability: make it a first-class operational primitive

Long browser tasks fail in messy ways. A page may hang. A selector may never match. A login may trigger a captcha loop.

If you only implement “wait until done,” your UI ends up showing the same story for every failure: a spinner that never resolves.

LeanKernel avoids that by wiring cancellation to `DELETE /runs/{runId}` in the sidecar. The engine exposes `browser_cancel_run` so the agent (or the caller) can request idempotent cancellation for a queued or running run.

This gives you a simple debugging loop:

| Step | Why you do it |
| --- | --- |
| submit | creates `runId` you can reference |
| poll | gets a status you can act on |
| fetch artifacts | confirms what the browser actually did |
| cancel | stops wasted spend and unblocks the turn |

That sequence matters because it separates “task intent” from “browser wall clock.” When the browser takes longer than you want, you cancel. You do not wait for the system to time out and pretend that is the same outcome.

## Spend and model governance: browser traffic routes through LiteLLM

Webwright still needs a model. If you route it directly to a provider with a shared key, you lose observability and you lose control over what models the browser is allowed to use.

LeanKernel routes Webwright’s model calls through LiteLLM inside the sidecar container. It uses `LITELLM_BASE_URL` and `LITELLM_API_KEY`, but it does not send provider keys to the browser service.

Instead, it expects a dedicated LiteLLM virtual key for the browser workload via `WEBWRIGHT_LITELLM_KEY`. That gives you a clean separation: browser-driven spend can carry its own budget, tags, and model allow list at the LiteLLM layer.

One more operational win: you can set reasonable caps for browser output without changing the browser runtime. The sidecar config includes:

| Setting | Default |
| --- | --- |
| `RequestTimeoutSeconds` | 15 |
| `MaxArtifactBytes` | 2000000 |
| `MaxOutputChars` | 12000 |

Those limits shape what you can store and what you can safely render back to a user. You get predictable failure behavior and you avoid surprise memory pressure.

## Health checks: probe readiness, not just liveness

Compose liveness is not enough for a sidecar that depends on model routing. LeanKernel uses an authenticated readiness endpoint (`/ready`) so the engine can decide whether the browser service is actually usable.

The sidecar also exposes an unauthenticated `/health` for Compose-level liveness.

This split keeps your runtime honest. If the sidecar process exists but it cannot reach LiteLLM, your engine should treat it as unavailable and stop dispatching new `browser_run_task` calls.

## The takeaway

Browser automation fails in ways that look like model failure unless you draw a hard boundary. LeanKernel does that boundary work up front.

Async tools give you cancelability. Opaque artifact access gives you confinement. Governance keeps the tool surface narrow. Routed model calls give you spend control.

Once you treat “browser execution” like an external capability with a contract, the agent can stay simple. The complexity moves into the sidecar, where it belongs.
