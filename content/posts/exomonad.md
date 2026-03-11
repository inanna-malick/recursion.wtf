---
title: "Exomonad"
date: 2026-03-08T12:00:00-08:00
draft: true
tags: ["exomonad", "tidepool", "agent-orchestration", "haskell", "rust", "llm"]
categories: ["projects"]
description: "ExoMonad stitches frontier model binaries together into reconfigurable agent swarms"
---

# §1 — ExoMonad

[ExoMonad](https://github.com/tidepool-heavy-industries/exomonad) is the first agent orchestration system designed for heterogeneous model composition. It stitches together Claude Code, Gemini-cli, kimi, letta code, Copilot — using their existing binaries and your existing subscription plan. It hooks into Claude Agent Teams' messaging bus, so agents running other model architectures show up as native team members. It builds on Gastown's worktree model, replacing "swarm of agents ramming PRs into main" with a tree-of-worktrees.

ExoMonad is designed to be radically reconfigurable. The Rust binary handles interaction with the real world, but all logic is defined in a flexible config-as-code model. It ships with a default devswarm configuration that handles worktrees, coordination, iterating on pull requests, but [when I needed a red team test harness](https://recursion.wtf/posts/vibe_coding_critical_infrastructure/), I just wrote another exomonad config, and threw one together overnight [^1]

As a proof of capabilities, I used ExoMonad to implement a new Haskell compiler backend using Rust. More on that later in this post.

<!--more-->

To follow along, follow these steps
```bash
git checkout https://github.com/tidepool-heavy-industries/exomonad/
try-exomonad/run.sh https://github.com/BurntSushi/ripgrep # or any other repo
```


## Heterogeneous agent orchestration

Opus is smart but expensive. Gemini is flaky but cheap and fast. Copilot is integrated with GitHub PR review and heavily subsidized. ExoMonad lets you use all of them — Claude writes the plan, Gemini does the work, Copilot reviews the PR. It's designed for token arbitrage: using each LLM model for what it's best at, shifting based on who's subsidizing usage in any given week.

This is the workflow I found myself using naturally. Over time, I found myself mainly orchestrating agents and steering planning processes, so I built ExoMonad to automate that. Here's an example:

{{< figure src="/img/exomonad_zellij_devswarm.png" caption="Zellij devswarm — TL dispatching Wave 4 to three Gemini workers in parallel, each in its own worktree. Bottom panes show workers mid-execution." >}}

Here you can see multiple gemini-cli agents working with a Claude Code tech lead in the same tab, with another tab showing an in-progress worktree owned by another Gemini agent. All agents use the Claude Agent Teams message bus (it uses on-disk mailboxes and jsonl, it was trivial to reverse engineer).

## Radically reconfigurable

All logic is evaluated in a shared server at the repo root. ExoMonad config, unix sockets, and worktrees are stored in the `.exo` dir used by this server.

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

This haskell code matches tool use, and - if it's the builtin `Replace` tool, applies a quick heuristic check to catch a common Gemini failure mode.

Let's have another example: here's what PR review routing looks like.

```haskell
prReviewHandler (ReviewReceived n comments) =
  pure (InjectMessage (copilotReviewReceived n comments))

prReviewHandler (ReviewApproved n) =
  pure (NotifyParentAction (prReady n) n)
```

This results in extremely token dense, expressive, and type safe code-as-config, all executed in a WASM sandbox using predefined effects executed by the Rust host binary. This plays to both language's strengths: Haskell for elegant composition of effects and expressive use of the type system to minimize boilerplate, Rust for interacting with the real world at the systems programming level.

# §2 — ExoMonad in Action

The default devswarm config defines three roles.
- Tech Leads (TLs) are planners and project managers: they orchestrate work, spawn child agents, review PRs.
- Devs write code: they write code, file PRs, and iterate with Copilot on review feedback.
- Workers are like Devs, but ephemeral: they run in the same worktree as a TL

ExoMonad makes heavy use of Git worktrees. Worktrees are full clones of a working repository that share that repository's .git object store. They're great for working with agent swarms.

Gastown uses worktrees with a single merge queue for each repo. ExoMonad's tree decomposition model uses nested worktrees: the system starts with a TL owning the root branch. It splits features into subtasks, and spawns subagents for each. Small tasks are assigned to Devs, who implement them as a single PR. Larger tasks are assigned to TLs, which (with human steering and guidance as needed) repeat this process, breaking features up into smaller units until they're small enough for a coding agent to implement unsupervised.

This results in a tree of worktrees, mirroring the structure of the project itself.

Here's how it works:

- The tech lead writes scaffolding (types, tests, stub files, planning docs) as an initial commit.
- It then spawns waves of subagents (TL or Dev as needed), each in their own worktree branching off of that scaffolding commit.
- Subagents eventually complete work, file PRs, iterate with Copilot, and send a message to the agent that spawned them
- It then launches followup waves of subagents, iterating until the feature is complete. 
- Then, the tech lead files a PR against its parent.

This repeats at every recursive level - `main.foo.bar` and `main.foo.baz` file PRs against `main.foo`, which then files a PR against `main`.

## ExoMonad in action

It's traditional, when debuting a new orchestration engine, to build something insane and overambitious, like a cleanroom reimplementation of the [C compiler](https://www.anthropic.com/engineering/building-c-compiler). I don't have 20k to burn on tokens, so I built something a bit more modest: a lazily evaluated haskell-in-rust runtime with native interop. It's very similar to the WASM sandbox used by ExoMonad, except instead of web assembly it uses Rust's cranelift JIT crate directly, operating on the intermediate representation Core used by the Haskell compiler.

It's called [Tidepool](https://github.com/tidepool-heavy-industries/tidepool).

[PLACEHOLDER: screenshot of tidepool build story viz]

Here's a visualization showing how I used the tree-of-worktrees model to build it: each node is a worktree, each of which resulted in one or more commits and a PR against its parent worktree. You can see a visualization [with links to various PRs here](/viz/tidepool-build-story.html).

# §3 — Tidepool

Tidepool is a lazily evaluated Haskell-in-Rust runtime that's tightly integrated with the Rust host, using the interpreter pattern as glue. Haskell composes and sequences effects using monadic sequencing; Rust effect interpreters execute them against the real world. The Haskell code is run directly in Rust, with full control over what operations are allowable, so there's no escape hatch - not even `unsafePerformIO`.

## Compile Time

Pipeline: GHC compiles Haskell to Core (GHC's intermediate representation) → serialized to CBOR → Cranelift JIT-compiles to native code with full laziness, tail call optimization, and a copying GC. Binaries have zero runtime dependency on Haskell infrastructure.

<details>
<summary>Minimal working example</summary>

```rust
#[derive(FromCore)]
enum ConsoleReq {
    #[core(name = "Emit")]
    Emit(String),
    #[core(name = "AwaitInt")]
    AwaitInt,
}

struct ConsoleHandler;

impl EffectHandler for ConsoleHandler {
    type Request = ConsoleReq;

    fn handle(&mut self, req: ConsoleReq, cx: &EffectContext) -> Result<Value, EffectError> {
        match req {
            ConsoleReq::Emit(s) => { println!("{}", s); cx.respond(()) }
            ConsoleReq::AwaitInt => {
                let mut input = String::new();
                std::io::stdin().read_line(&mut input).unwrap();
                cx.respond(input.trim().parse::<i64>().unwrap_or(0))
            }
        }
    }
}

fn main() {
    let (expr, table) = haskell_inline! {
        target = "game",
        include = "haskell",
        r#"
game :: Eff '[Console, Rng] ()
game = do
  target <- randInt 1 100
  emit "I'm thinking of a number between 1 and 100."
  guessLoop target

guessLoop :: Int -> Eff '[Console, Rng] ()
guessLoop target = do
  emit "Your guess? "
  guess <- awaitInt
  if guess == target
    then emit "Correct!"
    else do
      emit (if guess < target then "Too low!" else "Too high!")
      guessLoop target
        "#
    };

    let mut vm = JitEffectMachine::compile(&expr, &table, 1 << 20)
        .expect("JIT compilation failed");
    let mut handlers = frunk::hlist![ConsoleHandler, RngHandler(rand::thread_rng())];
    vm.run(&table, &mut handlers, &()).unwrap();
}
```

`haskell_inline!` compiles Haskell to GHC Core at build time. Cranelift JIT-compiles Core to native code inside the Rust process — no GHC runtime, no FFI.

</details>

## Live Compilation

Tidepool also supports live compilation, in cases where the user has the Haskell compiler available. To demonstrate this, I have provided an MCP server that provides this functionality. It's a bit like GHCI (Haskell's REPL), but specialized for the monadic composition of pure effects that are executed by Rust code.

More on this later, but here are a few examples of what it can do:


Sequencing monadic effects


Free pagination via continuation


Complex effect sequences

# Conclusion


[ExoMonad](https://github.com/inannamalick/exomonad). [Tidepool](https://github.com/inannamalick/tidepool).


[^1]: Auto-bootstrapped jailbreak support, siloed Opus workers with templates based on various elicitation strategies, misc other features. No, I haven't published this one, write your own.