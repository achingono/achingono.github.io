---
author: "Alfero Chingono"
title: "The Junior/Senior Gap Widens With AI — Here's How I'd Fix Onboarding"
date: 2026-07-09T09:00:00Z
draft: true
description: "AI makes senior engineers faster and juniors confidently wrong. Onboarding that assumes a shared baseline is now the single biggest underestimated risk in engineering teams."
slug: the-junior-senior-gap-widens-with-ai-heres-how-to-fix-onboarding
tags: [
"Software Engineering",
"Mentorship",
"AI Agents",
"FireFly",
"Platform Engineering",
"Onboarding"
]
categories: [
"Software Engineering",
"AI Agents"
]
image: "cover.png"
---

The most repeated take on my timeline this month: AI tools widen the gap between senior and junior engineers. I think that is right, but most of the conversation stops at the diagnosis. The harder question is what onboarding looks like when the new-hire default is an AI-assisted first draft they cannot evaluate.

## The mechanism, stated plainly

Seniors recognize when code is wrong-shaped before they can articulate why. They have a system model. Something comes out of the autocomplete that does not match how this codebase handles errors, or how this team names things, or how this database actually behaves under concurrent writes, and the senior's hand pauses before the enter key. That pause is expensive to train. It took them years.

A junior using the same tool gets the same output. What they do not have is the pause. The code compiles. The tests pass, because the model also wrote the tests and they assert the same assumptions. The PR goes up. The reviewer catches it, or they don't, and either way the junior has now shipped code whose correctness they were not in a position to evaluate.

This is not a skill gap that existed before and is merely being revealed by AI. It is a gap that AI is actively widening, because the senior's productivity multiplier from the tool is larger than the junior's. The senior uses AI to skip typing. The junior uses it to skip thinking, not because they are lazy but because they do not yet know which thinking they are skipping.

That is the actual problem. Everything else is downstream of it.

## Why "just turn off Copilot for juniors" is not the fix

I have heard this suggestion in at least six different engineering orgs, usually proposed by someone who has not spoken to a junior in a while.

It is not going to happen. The tool is free or close to it, the value is obvious on day one, and telling a new hire they cannot use something their peers use is going to push it underground rather than out of their workflow. The policy survives about two weeks.

Even if it did survive, it would not solve the right problem. The goal is not to protect juniors from AI assistance; it is to protect their learning from being short-circuited by it. Those sound similar. They are not.

The useful frame is AI as tutor, not author.

## Onboarding patterns that survive AI assistance

The things I have seen work share a common property. They make the junior's thinking visible, and they do not accept the AI's output as a substitute for that thinking.

**Code review as curriculum, not as a gate.** This is the one I would change first in most orgs. Most review processes treat the review as a quality check: does this pass. Onboarding review needs to treat it as teaching: does this person now understand something they did not before. That means longer comments, more "why" and less "nit," and explicit pointers to the parts of the system the junior did not know existed. The review becomes the curriculum. The PR is just the homework.

**Socratic mentor loops.** When a junior asks for help, the senior's first move should be a question, not an answer. What happens if this input is empty. What does this function do when the database is slow. Why is this file in this folder and not the other one. This sounds like a rhetorical trick and it is not. The question forces the junior to build the system model the tool would otherwise skip past. I borrowed a lot of this pattern from what I built in [FireFly's Socratic tutor](../how-i-wired-up-an-ai-tutor-to-teach-like-a-socratic-mentor-not-a-cheater/), because the underlying problem is the same: how do you teach someone a thing when another system is willing to just hand them the answer.

**Short, well-scoped first issues with an explicit expected-failure list.** Onboarding issues should be small, yes. The less common move is to tell the junior, up front, where you expect them to get stuck. "You will probably hit a weird interaction with the auth middleware here. When you do, come find me, do not ask Copilot." That does two things. It signals that getting stuck is the expected path, not a failure. And it gives them permission to notice the stuck, instead of papering over it with whatever the autocomplete offers.

**"Explain the diff in your own words" as a required PR section.** This is the highest-signal change I have seen any team make. Every PR from an onboarding engineer has a section, maybe 150 words, explaining what the change does, why it was done this way, and what alternatives were considered. The model can write this for them. The seniors reviewing it can tell when it has, and the conversation that follows is exactly the teaching moment the team needs to have.

None of this scales for free. All of it takes senior time. The good news is that the teams I have seen do it well end up with juniors who get productive faster, not slower, because they are actually learning the system instead of learning how to prompt their way around it.

## The platform-engineering angle

Onboarding is a product. AI pressure is a forcing function to treat it like one.

Before AI, a mediocre onboarding experience produced a slow new hire. They ramped up eventually because the work itself taught them. After AI, a mediocre onboarding experience produces a new hire who looks productive on week two and is shipping subtle bugs by week six, because the tool hid the ramp-up that was supposed to teach them the system.

The fix is the same fix platform teams apply to everything else they own. Instrument it. What does the first-month PR review burden look like for each new hire? How many of their PRs get reverted in the following quarter? What is the ratio of senior-mentor hours to junior-shipped-code hours, and is it going up or down? You do not need a dashboard. You need someone whose job includes noticing.

I talked about this dynamic in [how IDPs actually improve team productivity](../the-dora-report-was-right-idps-improve-team-productivity-by-10-percent-heres-how-ive-seen-it/) — the same logic applies here. Platform investment in onboarding is leverage, and the teams that treat it as overhead are the teams that will feel this gap widen the fastest.

## What does not work

Pairing where the senior quietly rewrites whatever the junior produced with the help of AI, and ships it under the junior's name. Everyone walks away thinking something productive happened. Nothing was learned. The junior now has the model-output-with-senior-polish pattern as their mental template, which is about the worst thing you can give someone who is trying to build a system model.

Turning PR review into a rubber stamp because the AI-assisted volume is too high to review properly. If the team's reviewers cannot keep up, the review is not the thing to compromise. The volume is. Fewer PRs, reviewed carefully, will produce better engineers than more PRs waved through.

And the one I see most often: assuming that because the junior can ship code that works, they have learned to engineer. Shipping and learning were never the same thing. AI has widened the gap between them into something large enough that you can lose people inside it.

## The payoff

Juniors onboarded this way take a little longer to look productive and get much more productive after. That is the trade. Teams that refuse to make it will end up with a bimodal workforce: senior engineers who are faster than ever, and mid-level engineers who plateaued in year two because they learned the tool instead of the craft.

The gap will not close on its own. Someone on your team has to decide to close it. The good news is that the work is not exotic. Thoughtful review. Good questions. Scoped problems. Writing, by the junior, in their own words.

The same things that made onboarding work before. Now done with more intention, because the pressure on them is higher.
