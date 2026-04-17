---
author: "Alfero Chingono"
title: "Why \"Vibecoding\" Hits a Wall: A Debrief From the Other Side"
date: 2026-07-23T09:00:00Z
draft: true
description: "LLMs close the first-draft gap but widen the integration, debugging, and security gap. Here is where non-engineers actually get stuck — and where AI genuinely helps them past it."
slug: why-vibecoding-hits-a-wall-a-debrief-from-the-other-side
tags: [
"AI Agents",
"Software Engineering",
"Developer Experience",
"Build in Public",
"OpenClaw"
]
categories: [
"Software Engineering",
"AI Agents"
]
image: "cover.png"
---

The promise of "anyone can build software now" has had a year to play out. What I see in practice is more interesting than either the hype or the backlash: non-engineers absolutely can ship a first version, and they absolutely do hit a wall. The wall is not coding. The wall is everything around the code.

## The first-draft gap really did close

I want to say this up front, because the honest version of the vibecoding conversation has to start here. The people who sneered at non-engineers shipping software with Cursor and Claude were wrong. A motivated non-engineer, with a clear idea and a willingness to iterate, can get a functioning first version of a non-trivial app online inside a weekend. I have watched it happen enough times that the novelty has worn off.

What I have also watched happen, enough times to see the pattern, is the wall. Usually around week two. The app that worked on the demo starts behaving strangely in ways the vibecoder cannot diagnose. They describe the symptom to the model. The model proposes a fix. The fix kind of works, or introduces a new symptom, or sort of does both. A week later the project is either abandoned or rewritten by someone they hired on Upwork.

This is not a failure of AI. It is a failure of a specific story about AI. The story is that coding was the hard part. Coding was never the hard part. Coding was the visible part.

## The five walls

I have watched these show up in enough projects, from enough different non-engineer builders, that I have stopped thinking of them as surprises.

**Environment setup.** The first wall is usually boring and usually fatal. A runtime version mismatch. A package manager that is installing into the wrong location. OAuth scopes that work in development and not in production because the redirect URL is off by one slash. These problems do not feel like the project's problem; they feel like the computer's problem. The model is great at describing what they are and mediocre at fixing them without context, because the fix depends on things the model cannot see: the vibecoder's shell config, their path, which installer they ran six months ago and forgot about.

**State and persistence.** The second wall is when the app starts forgetting things. Data that should be there is not. The form submitted, the response came back, and now on refresh the state is gone. Non-engineers hit this because the default shape the model produces often uses in-memory state or localStorage when the actual need is a database. The model is not wrong, exactly; it is answering the question that was asked, which was "make this work," not "make this persist correctly across sessions and processes." The vibecoder does not yet know those are different questions.

**Async and concurrency.** Every serious vibecoded app I have seen has a concurrency bug by week three. Race conditions between two API calls. A webhook that fires twice. State that was correct when one user was testing and corrupts when two are. These are bugs that do not show up in the demo and do show up the moment the app has actual users. The model can describe these bugs in the abstract beautifully. It struggles to find them in the specific codebase because they are not in any one file; they are in the timing between files.

**Security and secrets.** Credentials in source control. API keys in client-side bundles. Auth flows that sort of work but leak tokens in URLs. These are the ones that scare me, because the vibecoder does not know they have happened. There is no error, no crash, no broken behavior. The app works. It also has the master Stripe key embedded in a React component. The model generated the code that way because that is the pattern that produces a working local demo, and nobody was there to stop the pattern from shipping.

**Deployment, observability, and incident recovery.** The last wall is the one that looks least technical and is actually the most expensive. The app is live. Something breaks. The vibecoder cannot tell what broke, because there are no logs they know how to read, no metrics they set up, no staging environment to reproduce against. So they describe the symptom to the model, and the model starts guessing. Sometimes it guesses right. Often enough, it proposes changes that paper over the symptom and leave the underlying problem to recur in a different shape.

These five walls share a property. They are all problems of context. The code is fine. The system the code runs in is what has gone wrong.

## Where LLMs are great and where they are not

The pattern I keep seeing: models get the first 80% of each wall right and the last 20% catastrophically wrong.

They will correctly identify that you need a database. They will scaffold a schema. They will write the migration. They will not catch that your hosting provider's Postgres instance has a connection pool size of 3 and your serverless function is opening a new connection on every invocation. The first 80% got you to something that works in development. The last 20% is what breaks in production.

This is the gap the current generation of tools has not closed. It is not a prompt-engineering problem. It is a problem of the model not being able to observe the running system, react to what it sees, and change its own output accordingly. The model writes code. The world runs code. The round trip between those two is where software engineering actually lives.

## What actually helps vibecoders past the walls

Not more chat. More execution.

The tool that helps a vibecoder is not the one that can describe the fix. It is the one that can apply the fix, watch what happens, and try something else if it did not work. The shift from "AI that suggests code" to "AI that executes, observes, and adjusts" is the interesting transition, and it maps cleanly onto agent patterns rather than chat patterns.

A few things that I have seen move the needle for non-engineers specifically.

Scaffolded starter kits with fewer choices. The more decisions the model has to make about the shape of the app, the more ways the shape can be wrong in production. Opinionated kits that pre-decide the database, the hosting platform, the auth provider, and the deployment pipeline take four of the five walls off the table before the first line of code. You lose some flexibility. You gain a project that survives contact with real users.

Tool-calling agents that actually deploy. An agent that has permission to run `git push`, invoke the hosting provider's CLI, tail logs, and commit follow-up fixes behaves very differently from an assistant that hands the vibecoder a wall of instructions. The difference is not the model; it is the tool surface. The MCP-based agents I have been building (I wrote about [the Dockerfile behind the OpenClaw gateway](../inside-the-dockerfile-behind-my-openclaw-gateway/)) start to hit this shape. They are not there yet for a non-engineer audience; they are getting closer.

Error-loop agents that run, fail, read the log, fix. This one matters more than it sounds. Most vibecoder debugging today looks like: run, fail, copy the error, paste into chat, apply suggestion, run. Each round trip goes through a human who does not fully understand what they are seeing. If the agent closes that loop itself, running the command, reading the output, and proposing a fix against the actual failure mode rather than a guess, the vibecoder's job becomes approving changes rather than mediating between two systems that cannot see each other.

For the auth wall specifically, I think [OAuth2 and OIDC for solo developers](../oauth2-and-oidc-for-solo-developers-a-practical-setup-with-docker/) is the pattern non-engineers need most and have the fewest examples of, because the canonical auth tutorials assume familiarity with infrastructure they do not have.

## The honest limit

Here is the part I do not want to soften. There is a skill floor below which AI assistance cannot compensate for the absence of a system model, and pretending otherwise sets people up to fail publicly.

If you cannot read the output of a failing deployment and form a hypothesis about what might have caused it, no amount of chatting with a model will give you production software that stays up. The model can fix individual symptoms. It cannot build the diagnostic intuition you need to distinguish "the app is down" from "my ISP is flaky" from "the database provider had a regional incident" from "my last deploy introduced a memory leak that takes 47 minutes to crash the container."

That intuition takes time. It is teachable, and AI might eventually teach it well, but right now it is the thing separating people who ship and maintain software from people who ship software that eventually stops working while they try to get support on a Discord server.

So the honest answer to "can anyone build software now" is: yes, anyone can build a first version. Maintaining the second version requires something the tool does not yet provide.

## Where this leaves the agent stack

The design implication, for me at least, is clear. Chat-first interfaces have taken vibecoding as far as they can. The next layer is execution-first.

Less back-and-forth. More "the agent did the thing, observed the result, and either shipped or rolled back." The vibecoder's role moves from typing prompts to approving changes, with the agent handling the round trips they cannot yet see.

That is what [OpenClaw in Docker](../why-i-run-openclaw-in-docker-on-my-own-machine/) is really about, underneath the specific tech choices. An agent system that closes the loop on its own work, not one that suggests work for a human to close the loop on.

The vibecoder wall is not permanent. It is just further out than the first-draft gap, and nobody has built the tools that cross it yet at a consumer level. The people who do will have a much larger audience than the ones selling yet another chat interface.

I would bet on that audience existing. The wall proves they want to be on the other side.
