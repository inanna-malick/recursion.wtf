---
title: "Tree of Agents"
date: 2026-03-23
draft: true
tags: ["exomonad", "agent-orchestration", "ai", "architecture"]
categories: ["ai"]
description: "Why trees beat loops for agentic AI — and why the same topology that builds software can red-team it"
---

The dominant paradigm for AI agents is a loop: observe, think, act, repeat. The agent accumulates context in a single window until it solves the problem or runs out of room. This is wrong.

<!--more-->

## The problem with loops

A single context window accumulates everything — tool outputs, failed attempts, dead branches, sibling task noise. Every token of irrelevant history competes with the tokens that matter. Performance degrades as the window fills. Compression loses signal. The model spends increasing effort navigating its own accumulated context instead of solving the problem.

This is the fundamental scaling limit of loop-based agents. The more complex the task, the more context accumulates, the worse the agent performs — exactly when you need it most.

## Trees as natural decomposition

Software engineering tasks are trees. You break a problem into parts, solve the parts, and merge the solutions. The agent topology should match the task topology.

A tree-of-agents architecture does this literally: a parent agent decomposes a task, spawns child agents for each subtask, and merges their results. Each child carries only the context it needs. Sibling noise doesn't exist — child A never sees child B's failed attempts, dead-end explorations, or irrelevant tool outputs.

This isn't a metaphor. It's optimal allocation of a finite resource (context window tokens) across a parallelizable workload.

## Context windows as finite resource

Every token in a context window has a cost: it competes with every other token for the model's attention. In a loop, you pay this cost for every piece of accumulated history, whether relevant or not. In a tree:

- **Children are isolated.** Each branch carries only the context relevant to its subtask. A frontend agent doesn't see backend test failures. A CSS agent doesn't see database migration errors.
- **Failure is contained.** If a branch fails, it doesn't pollute its siblings. The parent retries or re-decomposes that subtask alone. In a loop, failure injects confusing context that degrades all subsequent attempts.
- **Compression is hierarchical.** Parents summarize children's results before passing them up. Each level of the tree compresses irrelevant detail. By the time results reach the root, they're maximally relevant.

## Natural supervision

Parents review children's work at merge boundaries — the same pattern as human engineering teams. This creates a natural quality assurance hierarchy:

- Children submit PRs (pull requests, literally)
- Parents review diffs before merging
- Each level of the tree adds a layer of review
- Failures are caught at the narrowest scope possible

This isn't imposed process. It falls out of the architecture. When your merge mechanism is `git merge`, code review happens automatically.

## Reconfigurability

The tree topology is independent of node behavior. The same parent-decompose-child-merge pattern works whether the nodes are:

- Software engineers building features
- Red teamers probing attack surfaces
- Researchers exploring a literature
- Security auditors reviewing code

[ExoMonad](/posts/exomonad/) implements this by defining agent behavior as swappable Haskell effect definitions. Change the effect handlers and the devswarm becomes a red-team harness overnight — same tree, same coordination protocol, different leaf behavior.

The [Gemini retrospective](/posts/gemini_jailbreak_retrospective/) demonstrates this directly: the same ExoMonad swarm architecture that [built Tidepool](/posts/tidepool/) was reconfigured to run parallel jailbroken Gemini instances against CTF targets. The tree didn't care what its leaves were doing. It cared that tasks decomposed, executed, and merged.

## The cost model

Tree-of-agents has an explicit cost model: parent tokens are expensive (planning, decomposition, review), child tokens are cheap (implementation, execution). This maps directly to frontier model pricing — an Opus root coordinating Haiku or Gemini leaves.

The root's job is to never touch implementation. Every file read for implementation detail, every line of code the root writes, is wasted budget. Decompose, spec, spawn. That's it.

This is also why it works for red-teaming: the expensive model (Opus) does strategy and review. The cheap model (jailbroken Gemini) does execution. The tree structure means the expensive model's context window stays clean — it never sees the noise of individual exploit attempts.

## Links

- [ExoMonad](/posts/exomonad/) — the implementation: how the tree is built
- [Tidepool](/posts/tidepool/) — the build case study: what a devswarm produced
- [Gemini retrospective](/posts/gemini_jailbreak_retrospective/) — the red-team case study: the same tree, different leaves

[inanna@recursion.wtf](mailto:inanna@recursion.wtf)
