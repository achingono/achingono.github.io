---
author: "Alfero Chingono"
title: "Building a Microsoft To Do CLI and Wiring It into OpenClaw"
date: 2026-04-16T09:00:00Z
draft: false
description: "How I built a TypeScript CLI for Microsoft To Do in a single session with Copilot, packaged it with GitHub Actions, and integrated it as a shared OpenClaw skill so every agent can manage my task lists."
slug: building-a-microsoft-todo-cli-and-wiring-it-into-openclaw
tags: [
"OpenClaw",
"Microsoft To Do",
"CLI",
"TypeScript",
"GitHub Actions",
"AI Agents"
]
categories: [
"Agentic AI",
"Platform Engineering",
"Build in Public"
]
image: "cover.png"
---

I wanted my OpenClaw agents to manage my Microsoft To Do lists: create tasks, track follow-ups, and check things off. The problem was that Microsoft doesn't ship a To Do CLI. There's the Graph API, but nothing that an agent can call from a shell prompt with a simple command.

So I built one.

## Why a CLI and not a direct API integration

OpenClaw agents execute tools as shell commands. That's the integration surface. If I wanted all five agents to manage To Do items, I needed something that could be invoked as a binary, accepted flags, and returned structured output. A CLI fit that model perfectly.

I also wanted the tool to exist independently of OpenClaw. It should work on any machine with Node.js and an Azure app registration. OpenClaw would consume it as a skill, but the CLI itself would be a standalone, open-source project.

## From zero to v0.0.1 in one session

The entire CLI was built in a single Copilot coding agent session. I started with `npm init`, defined the command structure with Commander.js, and layered on MSAL for auth and Axios for the Graph API. The session produced:

- **Auth commands:** `login` (device-code flow), `status`, `logout`, `print-account`
- **List commands:** `create`, `list`
- **Task commands:** `create`, `update`, `complete`, `list`, `get`
- **Step commands:** `create`, `update`, `complete`, `delete`, `list` for checklist items within a task
- **Short-form aliases:** `ms-todo-cli create` as a shortcut for `ms-todo-cli task create`

Everything outputs valid JSON. No plain-text error messages, no Commander help text leaking to stdout. Every response includes `ok: true` or `ok: false` with a structured error code. That was deliberate. Agents need parseable output, not human-friendly paragraphs.

### The error code system

Instead of generic error messages, every failure carries a typed code:

```typescript
export const ErrorCodes = {
  AUTH_REQUIRED: 'AUTH_REQUIRED',
  AUTH_EXPIRED: 'AUTH_EXPIRED',
  LIST_NOT_FOUND: 'LIST_NOT_FOUND',
  TASK_NOT_FOUND: 'TASK_NOT_FOUND',
  VALIDATION_ERROR: 'VALIDATION_ERROR',
  GRAPH_ERROR: 'GRAPH_ERROR',
  RATE_LIMITED: 'RATE_LIMITED',
} as const;
```

This lets an agent react to `AUTH_EXPIRED` differently from `LIST_NOT_FOUND`. In a skill prompt, I can say "if you see `AUTH_REQUIRED`, ask the user to run the login flow," and the agent can pattern-match on the JSON output reliably.

### Avoiding the O(N) list scan

The Microsoft Graph To Do API is list-scoped. To operate on a task, you need the list ID. If you only have the task ID, you have to enumerate all lists and probe each one until you find the right task.

The initial implementation did this naively. Every `task get` or `task update` call fetched all lists first. I added `--list-id` as an optional flag on every task command. When provided, it skips the list scan entirely. The skill prompt teaches agents to cache and reuse list IDs:

```bash
# Without list-id: O(N) list scan
ms-todo-cli task get --task-id "abc123"

# With list-id: single API call
ms-todo-cli task get --task-id "abc123" --list-id "def456"
```

## Packaging with GitHub Actions

I wanted the CLI installable from GitHub Releases, not from npm, at least not yet. The CI pipeline builds on three platforms, Ubuntu, macOS, and Windows, runs lint and tests on each, then packages the compiled output as a tarball:

```yaml
- name: Create artifact
  run: tar -czf ms-todo-cli-${{ matrix.os }}.tar.gz dist package.json package-lock.json README.md LICENSE
```

On a tagged push (`v*.*.*`), the release job uploads all three tarballs to a GitHub Release with auto-generated release notes.

The resulting tarball isn't a standalone binary. It is a packaged Node CLI with a `#!/usr/bin/env node` shebang. That works well for the OpenClaw gateway, which already has Node.js installed. For distribution outside Docker, a proper `npx`-compatible package or a pkg-compiled binary would be the next step.

## Installing in the gateway Docker image

The gateway Dockerfile downloads the Ubuntu release tarball and installs it globally:

```dockerfile
ARG MS_TODO_CLI_RELEASE=v0.0.1
RUN tmpdir=$(mktemp -d) \
 && install -d /opt/ms-todo-cli \
 && curl -fsSL -o "$tmpdir/ms-todo-cli.tar.gz" \
      "https://github.com/achingono/ms-todo-cli/releases/download/${MS_TODO_CLI_RELEASE}/ms-todo-cli-ubuntu-latest.tar.gz" \
 && tar -xzf "$tmpdir/ms-todo-cli.tar.gz" -C /opt/ms-todo-cli \
 && cd /opt/ms-todo-cli \
 && npm install --omit=dev --omit=optional --ignore-scripts \
 && chmod +x /opt/ms-todo-cli/dist/cli.js \
 && ln -sf /opt/ms-todo-cli/dist/cli.js /usr/local/bin/ms-todo-cli \
 && rm -rf "$tmpdir"
```

The `--omit=optional` flag skips `keytar`, a native module for OS keychain access that isn't needed in a headless container. The `--ignore-scripts` flag avoids post-install compilation surprises.

Two things make this work across container rebuilds:

1. **Token cache volume:** `./data/ms-todo-cli:/home/node/.ms-todo-cli` in `docker-compose.yml` persists the MSAL token cache on the host. Without this, every image rebuild would require re-authenticating.

2. **Client ID environment variable:** `MS_TODO_CLIENT_ID` is set in the `docker-compose.yml` environment for both the `gateway` and `cli` services. The CLI fails fast if this isn't set; there is no silent fallback to a broken state.

## The OpenClaw skill

With the binary installed, the last piece was teaching the agents how to use it. OpenClaw skills are markdown files with a YAML front matter block:

```yaml
---
name: ms-todo
description: "Manage Microsoft To Do lists, tasks, and checklist steps..."
metadata:
  openclaw:
    emoji: "✅"
    homepage: "https://github.com/achingono/ms-todo-cli"
    requires: { bins: ["ms-todo-cli"] }
---
```

The skill file at `data/config/skills/ms-todo/SKILL.md` covers:

- **Auth check first:** always run `ms-todo-cli auth status` before any operation
- **Duplicate avoidance:** list tasks in the target list before creating a new one
- **List hygiene:** prefer existing catch-all lists (`Tasks`, `Inbox`) over creating new ones; ask before creating a new list
- **Response format:** report back with list name, task title, due date, and any steps created

The skill is enabled in `openclaw.json` under `skills.entries` and accessible to all five agents without any per-agent allowlist.

## Authentication: the device-code flow

The CLI uses Microsoft's device-code flow, the only OAuth flow that works without a redirect URI, which makes it a good fit for a headless container:

1. `ms-todo-cli auth login` contacts Azure AD and receives a device code
2. It prints the code and URL to stderr (not stdout, which keeps stdout clean for JSON)
3. The user opens the URL in a browser, enters the code, and completes login
4. The MSAL library caches the refresh token at `~/.ms-todo-cli/msal-cache.json`

After the initial login, subsequent commands use silent token refresh. The cache file is mode `0o600` for minimal exposure.

The Azure app registration needs three delegated permissions: `Tasks.ReadWrite`, `User.Read`, and `offline_access`. The app must support "Accounts in any organizational directory and personal Microsoft accounts" to work with both work/school and personal Microsoft accounts.

## What this unlocks

With the skill active, any agent can now handle requests like:

> "Add 'Review the quarterly report' to my Tasks list, due Friday, high priority."

The agent runs `ms-todo-cli list list` to find the right list, creates the task with `ms-todo-cli task create`, and reports back with the task title and due date. If I later say "mark that task as done," the agent can complete it by ID.

Checklist steps work the same way. For multi-part follow-ups such as "gather the documents, review the draft, submit the form," the agent creates a parent task and adds each item as a step.

The JSON output means the agent always knows whether the operation succeeded, what error occurred if it didn't, and what IDs to reference for follow-up operations.

## What I'd improve

A few things are on the list for future iterations:

- **Pagination:** the current implementation doesn't handle paginated Graph API responses. If a list has more than 100 tasks, it only returns the first page. Fine for now, but it'll bite eventually.
- **npm publish:** publishing to npm would make `npx ms-todo-cli` work anywhere, not just in Docker images that install from GitHub Releases.
- **Recurrence and reminders:** the Graph API supports these, but the CLI doesn't expose them yet.
- **Linked resources:** To Do tasks can link to emails, URLs, and other Microsoft 365 items. That's a natural extension for agent workflows.

For now, though, the basic CRUD loop of create, list, update, and complete covers most of what I actually need from a task manager integration.

---

*This is part of an ongoing series about running OpenClaw as a self-hosted AI agent stack. Previous posts:*
- *[Why I Run OpenClaw in Docker on My Own Machine](/blog/2026/03/05/why-i-run-openclaw-in-docker-on-my-own-machine/)*
- *[Inside the Dockerfile Behind My OpenClaw Gateway](/blog/2026/03/09/inside-the-dockerfile-behind-my-openclaw-gateway/)*
- *[How I Wired Signal and Microsoft Teams into a Custom OpenClaw Image](/blog/2026/03/15/how-i-wired-signal-and-microsoft-teams-into-a-custom-openclaw-image/)*
- *[How I Split OpenClaw into Main and Personal Agents](/blog/2026/04/03/how-i-split-openclaw-into-main-and-personal-agents/)*
- *[How I Configured Five AI Agents on One Teams Bot](/blog/2026/04/05/how-i-configured-five-ai-agents-on-one-teams-bot/)*
