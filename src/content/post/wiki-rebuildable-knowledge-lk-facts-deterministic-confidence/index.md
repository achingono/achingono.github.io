---
author: "Alfero Chingono"
title: "A Rebuildable Wiki for Agent Systems: lk-facts, Deterministic Confidence, and Sidecar-owned Indexing"
date: 2026-08-13T09:00:00Z
draft: true
description: "How LeanKernel keeps wiki knowledge rebuildable and honest: canonical markdown records with fenced lk-facts YAML, deterministic confidence, and an index you can regenerate from source."
slug: wiki-rebuildable-knowledge-lk-facts-deterministic-confidence
tags: [
  "LeanKernel",
  "Knowledge",
  "AI Agents",
  "Reliability",
  "Qdrant"
]
categories: [
  "Platform Engineering",
  "Software Engineering"
]
image: "cover.png"
---

Most agent “memory” systems fail in the same quiet way. You store data, you query it, and then you discover you cannot rebuild it.

When you cannot rebuild, you also cannot validate. You can’t tell whether a model upgrade changed the meaning of your knowledge, or whether your pipeline started drifting.

LeanKernel’s wiki pipeline takes a different posture. It treats markdown as the source of truth and makes every projection rebuildable.

## Canonical records live in markdown

LeanKernel’s wiki store writes one canonical markdown file per subject and primary dimension.

The canonical form stays readable in GitHub and terminal views. You can open it, skim it, and understand what the system thinks it knows.

The storage contract looks like this:

```text
data/wiki/{dimension}/{subject-slug}.md
```

Each file carries two layers:

1. Human prose (frontmatter, `# {Subject}`, `## Summary`, and a `## Facts` section for readable bullets).
2. Machine-tracked facts inside a fenced YAML block: a single ` ```yaml lk-facts ` section that holds the structured per-fact records.

This is not a preference for neatness. It is a safety move. The fenced block becomes the canonical structured data for round-tripping. The prose becomes the human view.

## Don’t trust an LLM’s confidence field

If you ask an LLM for confidence, you get a number that looks scientific. It also tracks the model’s self-narrative, not your system’s actual evidence.

LeanKernel’s extraction contract explicitly avoids that. The LLM emits the extracted 5W1H structure and quotes, but it does not decide confidence.

Confidence gets computed deterministically later in `WikiFactMapper`. That keeps your “how sure are we” logic inside code, not inside sampled text.

The result is boring, and boring is good. When the mapper changes, you can rerun the pipeline and compare outcomes.

## The index is rebuildable, not magical

Markdown is the source. The index is a projection.

LeanKernel stores generated metadata under `data/wiki/.LeanKernel/`, including:

- `index.json`
- an append-only `migration.json` ledger
- a `migration.completed` sentinel for one-shot migration gates

On startup, if the index is missing or version-mismatched, the runtime rebuilds synchronously from markdown before serving queries.

That forces a painful but useful discipline. If your wiki facts are corrupt, the index rebuild will fail early. You catch the problem before you let a query return confident-sounding output based on stale or drifting projections.

LeanKernel also keeps index writes safe under concurrency. It serializes mutations in-process and merges by re-reading the latest persisted index before writing.

## Cross-dimensional retrieval without folder scans

The wiki uses 5W1H fields and supports retrieval by any populated dimension.

If you implement cross-dimensional listing by scanning every folder, you eventually hit a scaling wall. The fix is an index-level data structure: `factPointers`.

When a fact populates a non-primary dimension, the index records pointers so `ListByDimension` can return it without scanning every markdown file.

This keeps retrieval predictable. It also keeps it inspectable. You can open the index and see which dimensions point where.

## Sidecar owns Qdrant fact indexing

Here’s where a lot of projects get sloppy. They “help” by writing into the vector store from multiple places.

LeanKernel avoids that by giving the indexing responsibility to the Python sidecar. The C# runtime owns markdown writes and retrieval. The sidecar owns the Qdrant fact points.

That separation reduces schema drift risk. It also makes the system rebuild story coherent: markdown changes feed the sidecar, and the sidecar regenerates the Qdrant projection.

## Migration stays idempotent

Backfills are where trust breaks.

LeanKernel treats migration as a controlled one-shot. A separate `wiki-migrate` task moves valid legacy records into the canonical format and writes a `migration.completed` sentinel at the end.

Because the migration is idempotent, reruns don’t double-apply data. If the pipeline needs to rerun after a partial failure, it can. If it needs to rerun after a code change, it can.

## LLM failures never advance checkpoints

One rule protects your memory integrity: never advance scrub checkpoints when extraction fails.

The PRD calls this out because it sounds like an implementation detail until you hit the failure case. If you move a checkpoint forward after a partial extraction, you create a permanent blind spot.

So even when the pipeline schedules background extraction, it treats failures as “retry later,” not “we tried, good enough.”

## What you should copy

If you’re building an agent system with persistent knowledge, you do not need LeanKernel’s exact models.

You do need the shape of the contracts:

- Markdown is the source of truth.
- Structured facts live in a machine-readable fenced block.
- Confidence and normalization are deterministic code, not LLM self-report.
- Every index and vector projection is rebuildable from source.

When you build memory this way, you can upgrade models without pretending your past knowledge stayed the same.
