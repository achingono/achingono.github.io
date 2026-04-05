---
author: "Alfero Chingono"
title: "Adding simplefin-cli to OpenClaw and Scoping It to Dot"
date: 2026-04-23T09:00:00Z
draft: true
description: "How I installed simplefin-cli in my custom OpenClaw image, persisted its SimpleFin Bridge credentials, and exposed it as a workspace skill that only Dot, my financial agent, can use."
slug: adding-simplefin-cli-to-openclaw-and-scoping-it-to-dot
tags: [
"OpenClaw",
"SimpleFin",
"CLI",
"Docker",
"AI Agents",
"Personal Finance"
]
categories: [
"Agentic AI",
"Platform Engineering",
"Build in Public"
]
image: ""
---

I had already wired Microsoft To Do into OpenClaw as a shared skill for all of my agents. The next integration was different.

I wanted **Dot**, my financial agent, to inspect account balances and transaction history through [SimpleFin Bridge](https://beta-bridge.simplefin.org/). But I did **not** want every other agent to see that capability. This was a better test of how cleanly OpenClaw could support **agent-scoped tools**, not just globally shared ones.

So I added `simplefin-cli` to the container image and exposed it only to Dot.

## Why a CLI still makes sense

OpenClaw agents execute tools through shell commands. That means the lowest-friction integration surface is still a CLI:

- it can be invoked from the agent runtime without custom plugin code
- it can return structured JSON to the model
- it remains independently useful outside OpenClaw

In this case I already had a local `simplefin-cli` repository, so the job was not building the tool from scratch. The job was packaging it for the gateway image and teaching only the right agent how to use it.

## Installing `simplefin-cli` in the gateway image

The install pattern mirrors the one I used for `ms-todo-cli`: publish a GitHub Release tarball, download it in the Dockerfile, install production dependencies, and symlink the CLI onto `PATH`.

```dockerfile
ARG SIMPLEFIN_CLI_RELEASE=v0.0.1
RUN tmpdir=$(mktemp -d) \
 && install -d /opt/simplefin-cli \
 && curl -fsSL -o "$tmpdir/simplefin-cli.tar.gz" \
      "https://github.com/achingono/simplefin-cli/releases/download/${SIMPLEFIN_CLI_RELEASE}/simplefin-cli-ubuntu-latest.tar.gz" \
 && tar -xzf "$tmpdir/simplefin-cli.tar.gz" -C /opt/simplefin-cli \
 && cd /opt/simplefin-cli \
 && npm ci --omit=dev --omit=optional --no-fund --no-audit \
 && chmod +x /opt/simplefin-cli/dist/cli.js \
 && ln -sf /opt/simplefin-cli/dist/cli.js /usr/local/bin/simplefin-cli \
 && rm -rf "$tmpdir"
```

Like the To Do CLI, this is not a native binary. It is a packaged Node CLI with a `#!/usr/bin/env node` shebang. That is exactly fine for my OpenClaw image because Node is already there.

## The persistence detail that matters

`simplefin-cli` exchanges a one-time setup token for a persistent access URL and stores it locally at:

```text
~/.simplefin-cli/config.json
```

That means the important operational detail is not an environment variable. It is the **volume mount**:

```yaml
- ./data/simplefin-cli:/home/node/.simplefin-cli
```

I mounted that into both the `gateway` and `cli` services so rebuilds do not wipe the saved access URL. Without that mount, every rebuild would force the setup flow again.

This is one of those small Docker details that matters more than the tool itself. The integration is not really done until the credential state survives a rebuild.

## The bigger design question: shared skill or agent-only skill?

This was the interesting part.

For `ms-todo-cli`, a shared skill made sense. Any of my agents might legitimately create or complete tasks.

For financial data, that is a bad default. I wanted:

- Dot to have the capability
- the rest of the agents to stay unaware of it
- no custom plugin code
- no brittle prompt surgery

My first instinct was to look for a per-agent allowlist in `openclaw.json`. But OpenClaw's skill loading model is actually cleaner than that.

## The right pattern: workspace-scoped skills

OpenClaw already supports **per-agent skills** through workspace layout.

Instead of putting the skill in the shared directory:

```text
data/config/skills/simplefin/
```

I placed it in Dot's workspace:

```text
data/workspaces/financial/skills/simplefin/SKILL.md
```

That one choice did most of the work.

Because each agent has its own workspace, a skill under `financial/skills/` is naturally visible only to the financial agent. No global config toggle was needed. No other agent saw it.

That is an important pattern: if a tool is domain-specific or sensitive, **scope it by workspace first** and only use shared skills when broad visibility is actually desirable.

## What the skill teaches Dot

The `SKILL.md` file is intentionally practical. It teaches Dot to:

1. Run `simplefin-cli status` first
2. Ask for a setup token if the CLI is not configured
3. Use `simplefin-cli account list` to resolve account IDs before filtering transactions
4. Use ISO 8601 dates for transaction filters
5. Treat balances, payees, memos, and descriptions as sensitive financial data

The core commands are simple:

```bash
simplefin-cli status | jq
simplefin-cli setup "<base64-token>" | jq
simplefin-cli account list | jq
simplefin-cli transaction list --account-id "ACCOUNT_ID" --start-date "2026-01-01" --end-date "2026-02-01" | jq
```

That is enough for requests like:

- "Show me the balances across all linked accounts"
- "What transactions hit my chequing account last month?"
- "Pull everything from January for this account and summarize the major outflows"

It is deliberately read-only. No money movement, no account changes, no pretending the tool does more than it actually does.

## The validation insight I did not expect

The most useful lesson from this integration was how to verify agent-scoped skills correctly.

I initially reached for:

```bash
openclaw skills info simplefin
```

That was misleading for this use case, because the standalone `openclaw skills` command reflects the caller's **active workspace**, not an arbitrary agent context.

The more reliable validation path was to run a real agent turn and inspect the `systemPromptReport` in the JSON response:

```bash
openclaw agent --agent financial --message "reply with ok" --json
```

Then look at:

```json
.result.meta.systemPromptReport.skills.entries
```

For Dot, `simplefin` appeared in that list. For the business agent, it did not.

That is a much stronger validation than checking the filesystem alone. It proves the running agent prompt sees the skill the way I intended.

## What this unlocks

With this in place, Dot can now act more like a real financial assistant inside my OpenClaw setup:

- inspect linked accounts
- retrieve transaction history
- filter by account and date range
- summarize patterns from structured JSON output

That is a meaningful step toward giving each agent a real domain instead of just a different personality.

It also reinforces a design principle I keep coming back to: **multi-agent systems stay cleaner when capabilities are scoped as tightly as possible**. Shared tools are powerful, but agent-specific tools are often the better architecture.

## What I would improve next

There are a few obvious next steps:

- add richer transaction analysis helpers on top of the raw JSON
- normalize merchant naming for better summaries
- layer budgeting or recurring-expense detection on top of the transaction feed
- decide whether `simplefin-cli` should eventually publish to npm as well as GitHub Releases

For now, though, I have the piece I wanted: a financial data tool in the image, with the right state persisted, and only the right agent aware of it.

---

*This is part of an ongoing series about running OpenClaw as a self-hosted AI agent stack. Related posts:*
- *[Why I Run OpenClaw in Docker on My Own Machine](/blog/2026/03/05/why-i-run-openclaw-in-docker-on-my-own-machine/)*
- *[Inside the Dockerfile Behind My OpenClaw Gateway](/blog/2026/03/09/inside-the-dockerfile-behind-my-openclaw-gateway/)*
- *[How I Wired Signal and Microsoft Teams into a Custom OpenClaw Image](/blog/2026/03/15/how-i-wired-signal-and-microsoft-teams-into-a-custom-openclaw-image/)*
- *[How I Configured Five AI Agents on One Teams Bot](/blog/2026/04/05/how-i-configured-five-ai-agents-on-one-teams-bot/)*
- *[Building a Microsoft To Do CLI and Wiring It into OpenClaw](/blog/2026/04/05/building-a-microsoft-todo-cli-and-wiring-it-into-openclaw/)*
