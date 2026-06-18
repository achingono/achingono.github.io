---
author: Alfero Chingono
title: "Local-First Agents: Running Qwen 3.6 and Gemma 4 Inside OpenClaw"
date: 2026-07-02T09:00:00.000Z
draft: true
description: Wiring a local model into OpenClaw without breaking tool-calling contracts — and a frank look at where local still loses.
slug: local-first-agents-running-qwen-and-gemma-inside-openclaw
tags:
  - OpenClaw
  - Local AI
  - Ollama
  - MCP
  - AI Agents
  - docker
categories:
  - AI Agents
  - OpenClaw
image: cover.png
---

Running a local model is easy. Running a local model as the brain of an agent that has to call tools, honor schemas, and hand off to other agents is where it gets interesting. This is what I actually had to change inside OpenClaw to make Qwen 3.6 and Gemma 4 work as first-class agent backends — and where I still route to a frontier model on purpose.

## Why I bothered with local at all

The case for local-first agents reads differently depending on what you are optimizing for. For me, the ranking was privacy first, cost second, dev-loop speed third, and rate-limit immunity a close fourth.

Privacy is the one I will not compromise on. A fair chunk of what OpenClaw touches is client code under NDA. Sending that through a hosted endpoint, even one with a clean data-handling policy, is a contract conversation I would rather not have. Local sidesteps it entirely.

Cost follows. I am not running a hyperscale product; I am running a handful of long-running agent loops. Even so, a month of letting a review agent chew through pull requests on a frontier endpoint had me staring at a bill that made me reconsider my life choices. The dev-loop argument is quieter but real: when the model lives on the same machine as the code, the round trip from edit-prompt to see-result is under a second, and that changes how I iterate on agent behavior.

Rate-limit immunity is the last one and the most underrated. No local model has ever returned 429 at me in the middle of a ten-step tool sequence.

## The three things that usually break

You cannot just point your agent framework at a local endpoint and expect the same behavior. I wish that were true. It is not.

**Tool-calling schema adherence.** This is the big one. Frontier models have been trained hard on function-calling formats, and they get the JSON right almost every time. Open-weight models at the 7B to 32B size class get it right most of the time, which is a very different thing. A tool-calling agent that works 95% of the time is not a 95% agent; it is an agent that wedges on roughly every twentieth step of a multi-step task, which means a 10-step workflow fails about 40% of the time if you do the math. Qwen 3.6 is noticeably better at this than previous-generation Qwen. Gemma 4 is closer to the old line, but still recoverable with a bit of scaffolding.

**Long-context reasoning past the effective window.** Open-weight models advertise context windows that are mostly fiction past the first 32k or so. The attention is there; the reasoning quality drops off a cliff well before the advertised maximum. I have seen Qwen 3.6 stay coherent to about 48k on code-heavy inputs before it starts losing track of which file it was supposed to edit. Gemma 4 hits the wall a little earlier. If your agent relies on stuffing a whole repo into context and reasoning over it, local is going to disappoint you silently.

**Silent degradation on multimodal inputs.** If your agent ever handles screenshots, diagrams, or PDFs with layout that matters, the frontier multimodal models are still ahead by a meaningful margin. Local vision-language models are improving fast, but "improving fast" is not the same as "ready for a production tool call that depends on reading a config screenshot correctly."

Once you know these three, the architecture question stops being "local or cloud" and becomes "which tasks route where."

## The router that made it work

The change I made inside OpenClaw was not swapping the model. It was adding a thin routing layer in front of the existing agent surface.

The gateway (I wrote about [the Dockerfile that sits behind it](../inside-the-dockerfile-behind-my-openclaw-gateway/)) already fronted every agent call. What I added was a task-signature classifier that picks the backend before the call ever goes out. The classifier is boring on purpose: it keys on the agent identity, the estimated input size, whether the call includes images, and whether the task has a strict schema.

The rules in practice look roughly like this:

- Linting, docs generation, simple summarization, and any call under 8k tokens with a loose output shape: route to local Gemma 4 on Ollama.
- Implementation, test generation, and review on typical-sized diffs, with strict JSON output: route to local Qwen 3.6.
- Anything with images, anything above 48k tokens, anything labeled "architecture": route to a frontier model, currently Claude Sonnet.
- Fallback: if local fails schema validation twice, escalate to the frontier tier and log it.

That last rule is what made the system trustworthy. Local is the default; frontier is the escape hatch. Every escape gets logged, and the log is the signal for whether a new local model is ready to promote.

## The stack, concretely

Ollama serves the models. I picked it over vLLM for this specific use case because I wanted fast model swapping during development and did not need the throughput ceiling vLLM offers. On a Mac Studio with 64GB unified memory, Qwen 3.6 at Q4_K_M quantization and Gemma 4 at the same level both fit comfortably with room for context. Swap time between the two is a couple of seconds, which is fine for how I work.

LiteLLM is the uniform call surface. Every agent in OpenClaw thinks it is talking to an OpenAI-shaped endpoint. LiteLLM translates the call to Ollama, Anthropic, or Google depending on the model name, and the retry and fallback logic lives in its config. I wrote more about [how I use LiteLLM's tier aliases and fallbacks](../when-the-model-regresses-building-for-provider-drift/) when provider drift is the concern; the same machinery does the local-to-cloud handoff.

The MCP tool surface did not change. This is the part I am happiest about. Every tool the agents call goes through the same Model Context Protocol layer, regardless of whether the reasoning model is running on my laptop or in Anthropic's datacenter. The agent does not know and does not need to know. When I eventually swap Qwen 3.6 for whatever comes next, the tool contracts do not move.

## A worked example

The task I used to gate the local promotion was deliberately not a toy. I pulled a real pull request out of my own repo, one that added a new MCP tool endpoint, and handed it to the review agent with the standard prompt: identify issues, propose fixes as a structured patch, flag anything that needs human judgment.

On Claude, that PR produced a clean structured review in one shot. On Qwen 3.6 with no adjustments, the first attempt returned valid JSON but missed a subtle issue with how the tool handler serialized errors. The second attempt, after I tightened the prompt to require the model to enumerate every public function before commenting, caught it. The third run, with that prompt now baked into the agent's system message, matched the Claude output on substance, with a few stylistic differences in how the rationale was phrased.

Was it as good as Claude? On that specific task, essentially yes. Does it generalize? The data I have says Qwen 3.6 handles about 80% of review tasks at acceptable quality; the 20% that needs a better model is heavily weighted toward cross-file refactors and anything involving more than a couple of hundred lines of diff.

Which is exactly what the router is for.

## The honest failure log

Things I tried that did not work, in case they save someone else the time.

Running everything through local, even with a generous escape hatch, broke down on architecture tasks. The escape hatch triggered too often, the logs got noisy, and I stopped trusting the default. Promoting "architecture" to a separate tier with frontier as the default fixed it.

Asking Gemma 4 to emit strict JSON for tool calls was a losing fight. The model wants to narrate. I moved Gemma to tasks where the output is prose (docs, summaries, commit messages) and stopped trying to force it into a tool-calling seat. Qwen 3.6 holds schemas much better.

Quantizing more aggressively to fit a larger model did not pay off. Q3 Qwen 3.6 at a larger parameter count lost more on reasoning quality than it gained from the bigger base. Q4 at the smaller size was consistently better on my tasks.

## What this changes

The interesting claim is not that local models can replace frontier ones. They cannot, not yet, not on everything.

The interesting claim is that if you design the tool surface correctly, the model becomes a routing decision rather than an architectural one. Local wins the tasks it is good at, frontier catches the ones it is not, and the agents above the gateway do not care which is running underneath.

That is what local-first actually means, at least for the way I build agents. Local by default, cloud on purpose, with an honest log of which is which.
