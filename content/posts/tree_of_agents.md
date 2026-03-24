---
title: "Tree of Agents"
date: 2026-03-23
draft: true
tags: ["exomonad", "agent-orchestration", "ai", "architecture"]
categories: ["ai"]
description: "Trees as topology, forking as mechanism — why agentic AI needs both"
---

Two ideas changed how I build agent swarms: trees as topology, and forking as mechanism.

<!--more-->

## The problem with loops

A single context window accumulates everything — tool outputs, failed attempts, dead branches, sibling task noise. Every token of irrelevant history competes with the tokens that matter. Performance degrades as the window fills. Compression loses signal. The model spends increasing effort navigating its own accumulated context instead of solving the problem.

Loops work fine for simple tasks. But the more complex the task, the more context accumulates, the worse the agent performs — exactly when you need it most.

## Trees as natural decomposition

Software engineering tasks are trees. You break a problem into parts, solve the parts, and merge the solutions. The agent topology should match the task topology.

A tree-of-agents architecture does this literally: a parent agent decomposes a task, spawns child agents for each subtask, and merges their results. Each child carries only the context it needs. Sibling noise doesn't exist — child A never sees child B's failed attempts, dead-end explorations, or irrelevant tool outputs.

This isn't a metaphor. It's optimal allocation of a finite resource (context window tokens) across a parallelizable workload.

## Context windows as finite resource

Every token in a context window has a cost: it competes with every other token for the model's attention. In a loop, you pay this cost for every piece of accumulated history, whether relevant or not. In a tree:

- **Children are isolated.** Each branch carries only the context relevant to its subtask. A frontend agent doesn't see backend test failures. A CSS agent doesn't see database migration errors.
- **Failure is contained.** If a branch fails, it doesn't pollute its siblings. The parent retries or re-decomposes that subtask alone. In a loop, failure injects confusing context that degrades all subsequent attempts.
- **Compression is hierarchical.** Parents summarize children's results before passing them up. Each level of the tree compresses irrelevant detail. By the time results reach the root, they're maximally relevant.

## Forking beats spawning

The tree architecture has a cold-start problem. You spawn a sub-TL, it reads the repo, builds a mental model, discovers the scaffolding the parent already wrote, figures out what it means, then starts working. All that context-building is redundant — the parent already did it.

Session forking eliminates the redundancy. Instead of spawning a fresh agent, you fork the parent's full context window into a new worktree. The child starts at the parent's level of understanding. Parent wrote scaffolding → child already understands scaffolding → child writes more scaffolding → its children understand that. Recursive context inheritance down the tree.

The qualitative difference is startling. Before forking: spawn a sub-TL, wait while it orients, watch it rediscover what you already know. With forking: you have your tech lead. It built some scaffolding, you fork, you fork nodes. They already understand the scaffolding, they build more scaffolding, they fork — it's done before you know it.

This isn't just faster. It's a different kind of thing. Not a tree of separate agents coordinating through messages — a single agent's understanding, forked and diverged across branches. An agentic tree instead of an agentic loop.

The implementation: [ExoMonad](/posts/exomonad/) uses `--fork-session` with symlink magic to fork a context window into an isolated git worktree — context + filesystem state, diverged cleanly.

## Natural supervision

Parents review children's work at merge boundaries — the same pattern as human engineering teams. This creates a natural quality assurance hierarchy:

- Children submit PRs (pull requests, literally)
- Parents review diffs before merging
- Each level of the tree adds a layer of review
- Failures are caught at the narrowest scope possible

This isn't imposed process. It falls out of the architecture. When your merge mechanism is `git merge`, code review happens automatically.

## The cost model

Tree-of-agents has an explicit cost model: parent tokens are expensive (planning, decomposition, review), child tokens are cheap (implementation, execution). This maps directly to frontier model pricing — an Opus root coordinating Haiku or Gemini leaves.

The root's job is to never touch implementation. Every file read for implementation detail, every line of code the root writes, is wasted budget. Decompose, spec, spawn. That's it.

## Links

- [ExoMonad](/posts/exomonad/) — the implementation: how the tree is built
- [Tidepool](/posts/tidepool/) — the build case study: what a devswarm produced
- [Gemini retrospective](/posts/gemini_jailbreak_retrospective/) — the red-team case study: the same tree, different leaves

[inanna@recursion.wtf](mailto:inanna@recursion.wtf)
