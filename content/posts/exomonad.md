---
title: "ExoMonad"
date: 2026-03-12
draft: false
tags: ["exomonad", "tidepool", "agent-orchestration", "haskell", "rust", "llm"]
categories: ["projects"]
description: "ExoMonad stitches frontier model binaries together into reconfigurable agent swarms"
---


[ExoMonad](https://github.com/tidepool-heavy-industries/exomonad) builds on [Gastown's worktree model](https://steve-yegge.medium.com/welcome-to-gas-town-4f25ee16dd04), replacing "swarm of agents ramming PRs into main" with a tree of worktrees. It hooks into Claude Agent Teams' messaging bus, so agents running other model architectures show up as native team members. It integrates with Copilot for Github PR reviews. It stitches together Claude, Gemini, Kimi, Letta Code, Copilot — using their existing binaries and your existing subscription plan. 

ExoMonad is radically reconfigurable. It ships with a default devswarm configuration that handles worktrees, coordination, iterating on pull requests, but [when I needed a red team test harness](https://recursion.wtf/posts/vibe_coding_critical_infrastructure/), I just wrote another exomonad config, and threw one together overnight [^1].

I started by using ExoMonad to build itself over 700 PRs. A few weeks ago, it felt ready to test out on other projects, so I used it to implement a new Haskell compiler backend using Rust.

<!--more-->

To get started:
```bash
git clone https://github.com/tidepool-heavy-industries/exomonad
cd exomonad
./try-exomonad/run.sh https://github.com/BurntSushi/ripgrep # or any other repo
```


## Heterogeneous agent orchestration

Opus is smart but expensive. Gemini is flaky but cheap and fast. Copilot is integrated with GitHub PR review and heavily subsidized. ExoMonad lets you use all of them — Claude writes the plan, Gemini does the work, Copilot reviews the PR. It's designed for token arbitrage: using each model for what it's best at, shifting based on who's subsidizing usage in any given week.

This is the workflow I found myself using naturally. This orchestration gets complex, so I built ExoMonad to automate it. Here's an example:

{{< figure src="/img/exomonad_zellij_devswarm.png" caption="Zellij devswarm — TL dispatching Wave 4 to three Gemini workers in parallel, each in its own worktree. Bottom panes show workers mid-execution." >}}

Here you can see multiple gemini-cli agents working with a Claude Code tech lead in the same tab, with another tab showing an in-progress worktree owned by another Gemini agent. All agents use the Claude Agent Teams message bus (on-disk mailboxes and JSONL — straightforward to integrate with).

The default devswarm config defines three agent roles.
- Tech Leads (TLs) run Claude Opus. They're planners and project managers: they orchestrate work, spawn child agents, review PRs.
- Devs run Gemini CLI. They write code, file PRs, and iterate with Copilot on review feedback.
- Workers are like Devs, and also run on Gemini, but they're ephemeral: they run in the same directory as the TL that spawned them.

## Tree of Worktrees

ExoMonad makes heavy use of Git worktrees. Worktrees are working copies of a repository that share its .git object store. They're great for working with agent swarms.

Gastown uses worktrees with a single merge queue for each repo. There's one main branch, and a list of worktrees branched off of it, filing PRs against the main branch. This naturally results in contention, which Gastown solves by assigning a dedicated worker to manage each merge queue.

ExoMonad's tree decomposition model uses nested worktrees: the system starts with a TL owning the root branch. It splits features into subtasks, and spawns subagents for each. Small tasks are assigned to Devs, who implement them as a single PR. Larger tasks are assigned to TLs, which (with human steering and guidance as needed) repeat this process, breaking features up into smaller units until they're small enough for a coding agent to implement unsupervised.

This results in a tree of worktrees, mirroring the structure of the project itself.

Here's how it works:

{{< figure src="/img/worktree-1-initial.svg" >}}

A tech lead owns a worktree and writes scaffolding — types, tests, stub files, planning docs — as an initial commit. This scaffolding commit becomes the base that all subtask branches fork from.

{{< figure src="/img/worktree-2-unfold.svg" >}}

The TL then decomposes the work into subtasks, spawning waves of subagents (TL or Dev as needed), each in their own worktree branching off of that scaffolding commit. Large subtasks get their own TL, which repeats this decomposition recursively — `core-eval.heap` spawns `gc-trace` and `arena` the same way `core-eval` spawned `heap`, `codegen`, and `optimize`.

{{< figure src="/img/worktree-3-fold.svg" >}}

As subagents complete work, they file PRs against their parent branch, iterate with Copilot on review feedback, and notify the TL that spawned them. The TL merges completed PRs and launches followup waves as needed.

{{< figure src="/img/worktree-4-wave2.svg" >}}

Work naturally decomposes into multiple waves — the TL front-loads independent subtasks into wave 1, then uses the merged result as the scaffolding commit for wave 2. Followup tasks can reference code that didn't exist when the first wave launched, and since every agent in a wave forks from the same commit, there's no rebase burden within a wave. Eventually the TL files its own PR against *its* parent, and the same pattern repeats up the tree: `main.foo.bar` and `main.foo.baz` merge into `main.foo`, which merges into `main`.

## Radically reconfigurable

All logic is evaluated in a shared server at the repo root. ExoMonad config, Unix sockets, and worktrees are stored in the `.exo` dir used by this server.

Early iterations used YAML with custom expression languages for config, but I kept re-implementing subsets of Haskell — mostly various forms of effect composition — so I decided to just use the thing itself. All orchestration logic — routing, hooks, tool dispatch, event handling — is defined in Haskell effects executed by the ExoMonad server.

This is less scary than it sounds: it's a restricted subset of Haskell that leverages Haskell's type system for expressiveness and safety. It's designed for coding agents to modify (with humans in the loop for code review), so it's ok if you don't know exactly what a Monad is and don't really care to learn.

Let's look at how this works in practice: Gemini keeps stripping the dash from `#-}` in Haskell LANGUAGE pragmas. So I added a hook:

```haskell
checkPragmaCorruption (Replace fp old new)
  | ".hs" `T.isSuffixOf` fp
  , "#-}" `T.isInfixOf` old
  , not ("#-}" `T.isInfixOf` new)
  , "#}" `T.isInfixOf` new
  = Just "BLOCKED: Your replacement corrupts Haskell LANGUAGE pragmas..."
checkPragmaCorruption _ = Nothing
```

This Haskell code matches tool use, and — if it's the builtin `Replace` tool — applies a quick heuristic check to catch a common Gemini failure mode. Instead of spending tokens on prompting every Gemini agent (even those that never touch Haskell code), the guidance is encoded as code. This is just one tweak, but the idea is Kaizen: a continuous process of finding opportunities to make 1% improvements. When an agent produces garbage — and they will — the hook catches it before it hits a PR, and the fix becomes permanent infrastructure rather than a one-off prompt tweak. Over time, these compound. Your LLM swarm evolves, adapts to your project and context.

Here's another example: PR review routing.

```haskell
prReviewHandler (ReviewReceived n comments) =
  pure (InjectMessage (copilotReviewReceived n comments))

prReviewHandler (ReviewApproved n) =
  pure (NotifyParentAction (prReady n) n)
```

This results in extremely token-dense, expressive, and type-safe code-as-config, all executed in a WASM sandbox using predefined effects executed by the Rust host binary. This plays to both languages' strengths: Haskell for elegant composition of effects and expressive use of the type system to minimize boilerplate, Rust for interacting with the real world at the systems programming level.

## ExoMonad in Action

It's traditional, when debuting a new orchestration engine, to build something insane and overambitious, like [a cleanroom reimplementation of the C compiler](https://www.anthropic.com/engineering/building-c-compiler). To prove ExoMonad could handle diverse projects, I used it to build something nontrivial from scratch.

I don't have $20k to burn on tokens, so I built something a bit more modest: [Tidepool](/posts/tidepool/), a lazily evaluated Haskell-in-Rust runtime with native interop. It's very similar to the WASM sandbox used by ExoMonad, except instead of WebAssembly it uses Rust's Cranelift JIT crate directly, operating on the intermediate representation Core used by the Haskell compiler.

I built it over about two weeks, using less than 50% of my Claude Max and Gemini Ultra subscription plans.

{{< figure src="/img/exomonad_tidepool_build_story.png" >}}

This visualization shows how I used the tree-of-worktrees model to build the core of it. The swarm blasted out the foundational architecture — over 32,000 lines of code across 61 PRs — in just 4 days of active waves. You can see an interactive version [with links to various PRs here](/viz/tidepool-build-story.html).

The remaining week and a half was spent doing human-in-the-loop bug squashing, adding feature support, and general polish. For a deep dive into Tidepool itself — the Cranelift pipeline, the MCP server, live compilation examples — see the [Tidepool post](/posts/tidepool/).

## Conclusion

ExoMonad is open source and ready for early adopters. If you run into bugs or missing features, please file an issue.

- [ExoMonad](https://github.com/tidepool-heavy-industries/exomonad)
- [Tidepool](https://github.com/tidepool-heavy-industries/tidepool)

If you'd like to hear more over coffee, I'm in SF/Oakland, or happy to hop on a call.

[^1]: Auto-bootstrapped jailbreak support, siloed Opus workers with templates based on various elicitation strategies, misc other features. No, I haven't published this. Write your own.