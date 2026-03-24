---
title: "Tree of Agents"
date: 2026-03-23
draft: true
tags: ["exomonad", "agent-orchestration", "ai", "architecture"]
categories: ["ai"]
description: "Why agent topology should match task topology — and how git gives you the tree for free"
---

An agentic loop has a simple structure: observe, think, act, repeat. The agent accumulates state in a single context window, iterating until the task is done or the window fills up. Most agentic frameworks are built around this loop. It works. But it's a special case of something more general.

<!--more-->

## The shape

The natural structure of work is not a loop — it's a tree. A task decomposes into subtasks. Subtasks decompose further. Leaves execute. Results merge back up.

This gives a three-phase cycle: **scaffold** the shared foundation and build context, **fork** into parallel agents, and **converge** by merging their results. The parent scaffolds — writing types, stubs, shared state — then forks children to implement the pieces in parallel. When children finish, the parent merges their work, verifies integration, and decides whether another wave of decomposition is needed.

This is recursive. A child that receives a complex subtask becomes a parent itself: scaffold, fork, converge. The tree grows to match the shape of the problem.

## Agent state

An agent's state is two things: what's on disk and what's in the context window. The repository is the persistent artifact — code, config, documentation. The context window is the ephemeral reasoning state — every file read, every decision made, every tool output accumulated during the session.

In a loop, both are singular. One repo, one context window, one agent iterating in place.

In a tree, both fork. Git worktrees give each child agent an isolated copy of the repository. Claude Code's `--fork-session` clones the parent's entire conversation history into the child. The child starts with everything the parent knew at the moment of forking — every decision, every file read, the full reasoning chain — plus an isolated working directory where its changes can't interfere with siblings.

Forking the repo and forking the context window is the same operation as forking the agent. The agent IS the pair of (repo state, context window). Fork both, and you've turned one agent into many, each carrying the context it needs and nothing it doesn't.

## Loops as degenerate trees

A loop is a tree with branching factor one. One agent, one branch, iterating sequentially. This framing makes the limitation obvious: a loop is a tree that never parallelizes and never isolates.

Everything a loop accumulates stays in a single context window. Dead branches from failed attempts. Irrelevant tool outputs from earlier iterations. Noise from subtasks that have nothing to do with the current one. Every token of history competes with the tokens that matter. As the window fills, the agent drifts — it starts pursuing easier alternatives because the hard task is buried under accumulated noise.

A tree with branching factor greater than one solves this structurally. Independent subtasks run in parallel, in isolated contexts, with no cross-contamination. A failure in one branch doesn't pollute its siblings. The parent's context stays clean because children handle the implementation noise.

Loops aren't wrong. They're the simplest possible tree — and for simple tasks, that's fine. But they're a strict subset of what a tree can do, and for complex tasks, the restriction costs you.

## Supervision

In a tree, every child agent has exactly one task. Leaves implement — they write code, run tests, produce artifacts. Interior nodes plan and orchestrate — they decompose tasks, write specs, and review results. No agent does both.

This single-task constraint is what prevents drift. A leaf agent's context window contains only what's relevant to its task. It can't silently pivot to something easier because there's nothing else in its context to pivot to. It doesn't know about the easier alternatives — they're in a sibling's worktree, invisible.

At merge boundaries, the parent reviews what the child produced. This happens naturally: when the merge mechanism is git, code review is the merge. The parent reads the diff, verifies it matches the spec, and either merges or rejects. This is the same pattern as human engineering teams — not because it's imposed as process, but because tree-shaped work produces tree-shaped review. Each level of the tree adds a layer of verification. Failures get caught at the narrowest scope possible.

The combination matters: narrow mandate, narrow context, review at merge. Any one layer might fail to catch drift. All three together make it structurally difficult for work to silently deviate from plan.

## Proof

This isn't speculative architecture. [ExoMonad](/posts/exomonad/) implements the scaffold-fork-converge cycle over git worktrees, with swappable agent behavior at each node. [Tidepool](/posts/tidepool/) is a full application built by an agent tree — decomposed, parallelized, merged, and shipped.

## Trees are natural

Tasks decompose recursively. They always have — every project plan, every work breakdown structure, every recursive function call is a tree. Agents that iterate in loops are flattening a structure that was already there.

The loop became dominant because it's easy to implement. One agent, one context window, one iteration at a time. But the tools for trees already exist. Git branches are task decomposition. Worktrees are context isolation. Merge is convergence. The tree was always available — it just wasn't the default.

It should be.

[inanna@recursion.wtf](mailto:inanna@recursion.wtf)
