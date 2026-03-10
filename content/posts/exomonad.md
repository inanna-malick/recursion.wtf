---
title: "Exomonad"
date: 2026-03-08T12:00:00-08:00
draft: true
tags: []
categories: []
summary: ""
description: ""
---

# §1 — ExoMonad


ExoMonad is a highly reconfigurable agent orchestration platform that incorporates the best features of Claude Agent Teams and Gastown. It doesn't try to replace Claude Code or Gemini-cli (or kimi, or letta code), it stitches them together, along with Copilot (for Github PR reviews), all with their existing binaries and your existing subscription plan.

(screenshot) 
caption: "claude spawns gemini workers" -> "Gemini calls notify_parent → ExoMonad writes to Claude's Teams inbox → Claude sees a native `<teammate-message>`"

It hooks natively into Claude Agent Team's messaging bus, allowing for seamless integration of agent team members running other frontier model architectures. It builds on Gastown's use of worktrees, replacing the "swarm of agents ramming PRs into main" model with a tree-of-worktrees model.

It's traditional, when debuting a new orchestration engine, to build something insane and overambitious, like a cleanroom reimplementation of the [C compiler](https://www.anthropic.com/engineering/building-c-compiler). I don't have 20k to burn on tokens, so I built something a bit more modest: a lazily evaluated cranelift JIT compiled haskell-in-rust runtime with native interop.


## Heterogeneous agent orchestration system

Other open source agent orchestration systems notionally support multiple LLM agents (gemini-cli, claude code, etc), but they expect users to go all-in on their preferred model. Closed source orchestration systems like Claude Agent Teams are designed from the ground up to support a single model family, and are unlikely to open up any time soon.

Opus is smart but expensive. Gemini is flaky but cheap and fast. Copilot is integrated with GitHub PR review and heavily subsidized. ExoMonad lets you use all of them — Claude writes the plan, Gemini does the work, Copilot reviews the PR.
Different model families have different blind spots and different price points. The landscape shifts weekly. ExoMonad is built to shift with it — reconfiguring its own orchestration to match whatever just dropped.

(screenshot/ascii diagram showing single 'opus spawns gemini worktree, worktree iterates w/ copilot, opus gets notified after iteration complete and can either send a msg to the gemini agent w/ change requests or merge' workflow)

todo: description of above flow (text)

ExoMonad supports any agent binary with hooks and MCP support, with all logic evaluated in a shared server via Unix Socket at repo root. This server handles exomonad MCP tool invocation, agent hooks, message routing, and event-based logic, serving as the glue layer between local agent binaries and cloud-based agentic processes (Github Copilot PR review).



## Radically Reconfigurable

Earlier iterations used YAML config, but I kept re-implementing subsets of Haskell — if/then/else, map, pattern matching — and decided to just use the thing itself.

All orchestration logic — routing, hooks, tool dispatch, event handling — is defined in Haskell effects executed by the ExoMonad server. You don't need to know Haskell. LLMs are great at working with the strongly-typed pure logic core of Haskell. ExoMonad only asks for the composition part. Use the default devswarm config and let them modify it as needed.

The type system bounds what agents can touch — they can reconfigure orchestration but can't escape into arbitrary IO. Self-modification is a PR where the compiler already verified the wiring. The failure mode is "doesn't compile," not "silently routes tasks to the wrong model for three hours."

- TODO: example — a short DSL snippet showing a hook/tool definition, demonstrating density + type safety

TODO: example — git log or PR showing an agent reconfiguring the DSL mid-session. The diff should be legible Haskell that a non-Haskell reader can squint at and grasp the shape of.

# §2 — How it Works

## Tree of Worktrees

### Unfold

- TL writes scaffolding. Commits. Creates child branches off that commit, each in its own worktree.
- Child agents land in those worktrees with the scaffolding already there — that's their context.
- Child TLs repeat: write scaffolding, commit, branch, spawn. Gemini devs at the leaves execute, can't spawn.
- The tree grows through commits. Each branch point is a committed scaffold. Context propagates via git.
- Further from root = narrower scope. Opus TLs where judgment matters, Gemini devs where throughput matters.
- Tree shape: preplanned for big pushes, TLs add leaves + replan during execution.

### Iterate

- Each TL breaks work into subunits, dispatches as parallel tasks (conflict-aware — parallel tasks touch different areas)
- Dispatch wave → leaves do work, make commits, file PRs against parent's branch
- TL merges, commits new scaffolding, dispatches next wave
- Pattern: dispatch → merge → scaffold → dispatch
- Within each wave, leaf review ratchet:
	- Gemini submits PR → Copilot reviews → Gemini fixes → repeat until merge
- PR review failure is the happy path — cheap iteration
- TODO: example — a real PR review cycle: Gemini's initial PR, copilot's review comment, Gemini's fix commit. Shows the ratchet in action — cheap iteration, not expensive correctness.
- During the long tail of bugfixes, spawn 5 Gemini agents each investigating a different hypothesis with their full 1M context window, feeding results back to the Opus coordinator
- TODO: inline screenshot — Zellij showing 5 Gemini workers (h1-thunk-sharing, h3-float-out, h4-bytearray, h5-varid-collision, h2-case-binder) investigating hypotheses in parallel. All responses route back via Claude Teams channels as synthetic workers. TL pane shows rolling scorecard: H1 disproven, H3 partially plausible, H5 disproven, H2/H4 still running. Opus coordinator on right already synthesizing results, pointing toward binder aliasing. Screenshot exists.

### Fold

- Leaves finish → commits → PR to parent
- Copilot reviews. TL merges, HEAD advances.
- TL PRs to its parent. Same cycle.
- Continues until root PRs to main.

# §3 — What I Built: Tidepool

Tidepool is a lazily evaluated Haskell-in-Rust runtime: it provides a susbset of Haskell that includes lenses, monadic composition of effects, and the full power of the type system, without any access to IO. Since there's no ability to interact with the outside world, it's a perfect sandbox. Effects are composed and executed by haskell code, and interpreted by Rust handlers. Both languages get to do what they're best at: Haskell for beautiful concise code composing and sequencing effects, Rust for systems-programming level code that interacts with the real world.

## How it works

Tidepool uses GHC to compile Haskell code to Core, an intermediate representation used internally by GHC. This is then fed to Cranelift and JIT-compiled (with full laziness support, tail call optimizations, garbage collection, etc) and executed inline inside a Rust binary.


I have some examples of this pattern in the `examples` directory. These examples use the Haskell compiler (GHC) as a compile-time dependency and produce binaries that have no runtime dependency on any Haskell infrastructure or libraries.

tidepool-tide: a minimal embedded language with parsing via Rust PEG grammar (pest) and evaluation in Haskell:

```
tide> let add = \a b -> a + b
<function>
tide> let inc = \a -> add(a, 1)
<function>
tide> inc(2)
3
```

You can also embed Haskell code directly in Rust via macro:

TODO example w/ link

## How it was built

viz story

## Tidepool MCP Server

Tidepool also provides an MCP server binary that uses GHC at runtime to compile 


Tidepool MCP runtime section, from everything we've discussed:

Tidepool provides an MCP server. Two tools: eval and resume. That's the entire surface.
eval: agent sends a Haskell do-block. Tidepool JIT-compiles via Cranelift, runs it. 10 effects available: Console, KV, Fs, SG (ast-grep, 14 languages), Http, Exec, Git (native libgit2), Llm (Haiku), Meta (runtime introspection), Ask (suspend/resume).
resume: agent sends a continuation_id + response. Blocked eval thread unblocks, continues from where it suspended.
Continuations: ask "prompt" suspends the JIT mid-execution, returns {suspended: true, continuation_id, prompt} to the calling LLM. The LLM can go do other work — grep, run other evals, think — then resume with an informed choice. The suspended eval holds all gathered data in the JIT heap.
Tree-structure-aware pagination: large structured results (500 code matches) get truncated into navigable stubs preserving JSON shape. Agent drills into subtrees via resume. Not string truncation — tree navigation.
llm/llmJson: inline Haiku calls as pure effects. llmJson forces structured output via typed schema ADT. Fixed model (Haiku), fixed temperature — these are micro-judgments inside data pipelines, not conversations.
Composition demo: sgFind all panic!/todo!/unreachable! → fsRead surrounding context → llmJson scores each with {reachable_in_production: bool, risk: 1-5, reason: text} → ask surfaces ranked results → calling LLM drills into the scariest one.
Or: crate census demo — fsGlob + fsMetadata across all source files → foldl' into per-crate stats → ask with formatted menu → LLM picks a crate → resume returns filtered detail listing.
The pattern: gather wide (structural search) → judge (inline LLM) → surface (continuation) → steer (resume). Effects compose, the calling LLM steers between suspensions.



# Conclusion

ExoMonad and Tidepool look like two projects. They're one idea. Haskell composes effects. Rust executes IO. The effect boundary is the interface. It's the same architecture at two scales — orchestration and runtime.

The computer use model is all about composition. Right now agents compose with bash: `&` and `|` and janky sed pipelines. Haskell is a radically more powerful composition system — aeson-lens, effects, GADTs, type-level constraints. And agents already know how to write it, because it's the Haskell the internet writes about.

- TODO: example — one aeson-lens or GADT snippet showing agent-written Haskell doing something bash can't express cleanly. Should demonstrate composition payoff, not just type-level cleverness.

check out exomonad try it out file issues pls pls pls


[1] 	I needed a red team testbench with auto-bootstrapped jailbreak support and siloed Opus workers w/ templates based on various elicitation strategies, so I had an agent build one. This took only a few hours, for a radically different use case.