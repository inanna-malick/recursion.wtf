---
title: "Exomonad"
date: 2026-03-08T12:00:00-08:00
draft: true
tags: ["exomonad", "tidepool", "agent-orchestration", "haskell", "rust", "llm"]
categories: ["projects"]
description: "ExoMonad stitches frontier model binaries together into reconfigurable agent swarms"
---

[ExoMonad](https://github.com/tidepool-heavy-industries/exomonad) is the first agent orchestration system designed for heterogeneous model composition. It hooks into Claude Agent Teams' messaging bus, so agents running other model architectures show up as native team members. It integrates with Copilot for Github PR reviews. It stitches together Claude, Gemini, kimi, letta code, Copilot — using their existing binaries and your existing subscription plan. 

ExoMonad builds on Gastown's worktree model, replacing "swarm of agents ramming PRs into main" with what I call the 'Tree of Worktrees' model, in which each unit of work is recursively unfolded into subtasks (with human assistance during the planning phase) until the subtasks are small and well specified enough for autonomous agents to complete. 

ExoMonad is designed to be radically reconfigurable. The Rust binary handles interaction with the real world, but all logic is defined in a flexible config-as-code model. It ships with a default devswarm configuration that handles worktrees, coordination, iterating on pull requests, but [when I needed a red team test harness](https://recursion.wtf/posts/vibe_coding_critical_infrastructure/), I just wrote another exomonad config, and threw one together overnight [^1]

As a proof of capabilities, I used ExoMonad to implement a new Haskell compiler backend using Rust.

<!--more-->

To follow along, follow these steps
```bash
git checkout https://github.com/tidepool-heavy-industries/exomonad/
try-exomonad/run.sh https://github.com/BurntSushi/ripgrep # or any other repo
```


## Heterogeneous agent orchestration

Opus is smart but expensive. Gemini is flaky but cheap and fast. Copilot is integrated with GitHub PR review and heavily subsidized. ExoMonad lets you use all of them — Claude writes the plan, Gemini does the work, Copilot reviews the PR. It's designed for token arbitrage: using each LLM model for what it's best at, shifting based on who's subsidizing usage in any given week.

This is the workflow I found myself using naturally. Over time, I found spending a lot of time orchestrating this process, so I built ExoMonad to automate that. Here's an example:

{{< figure src="/img/exomonad_zellij_devswarm.png" caption="Zellij devswarm — TL dispatching Wave 4 to three Gemini workers in parallel, each in its own worktree. Bottom panes show workers mid-execution." >}}

Here you can see multiple gemini-cli agents working with a Claude Code tech lead in the same tab, with another tab showing an in-progress worktree owned by another Gemini agent. All agents use the Claude Agent Teams message bus (it uses on-disk mailboxes and jsonl, it was trivial to reverse engineer).

The default devswarm config defines three roles.
- Tech Leads (TLs) run Claude Opus. They're planners and project managers: they orchestrate work, spawn child agents, review PRs.
- Devs run Gemini CLI. They write code, file PRs, and iterate with Copilot on review feedback.
- Workers are like Devs, and also run on Gemini, but they're ephemeral: they run in the same directory as the TL that spawned them.

# Tree of Worktrees

ExoMonad makes heavy use of Git worktrees. Worktrees are full clones of a working repository that share that repository's .git object store. They're great for working with agent swarms.

Gastown uses worktrees with a single merge queue for each repo. There's one main branch, and a list of worktrees branched off of it, filing PRs against the main branch. This naturally results in contention, which Gastown solves by assigning a dedicated worker to manage each merge queue.

ExoMonad's tree decomposition model uses nested worktrees: the system starts with a TL owning the root branch. It splits features into subtasks, and spawns subagents for each. Small tasks are assigned to Devs, who implement them as a single PR. Larger tasks are assigned to TLs, which (with human steering and guidance as needed) repeat this process, breaking features up into smaller units until they're small enough for a coding agent to implement unsupervised.

This results in a tree of worktrees, mirroring the structure of the project itself.

Here's how it works:

- The tech lead writes scaffolding (types, tests, stub files, planning docs) as an initial commit.
- It then spawns waves of subagents (TL or Dev as needed), each in their own worktree branching off of that scaffolding commit.
- Subagents eventually complete work, file PRs, iterate with Copilot, and send a message to the TL that spawned them.
- That TL then launches followup waves of subagents, iterating until the feature is complete. 
- The TL then files a PR against its parent.

This repeats at every recursive level - `main.foo.bar` and `main.foo.baz` file PRs against `main.foo`, which then files a PR against `main`.


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

## ExoMonad in action

It's traditional, when debuting a new orchestration engine, to build something insane and overambitious, like a cleanroom reimplementation of the [C compiler](https://www.anthropic.com/engineering/building-c-compiler). I don't have 20k to burn on tokens, so I built something a bit more modest: a lazily evaluated haskell-in-rust runtime with native interop. It's very similar to the WASM sandbox used by ExoMonad, except instead of web assembly it uses Rust's cranelift JIT crate directly, operating on the intermediate representation Core used by the Haskell compiler.

It's called [Tidepool](https://github.com/tidepool-heavy-industries/tidepool).

{{< figure src="/img/exomonad_tidepool_build_story.png" >}}

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

### Sequencing monadic effects

Each effect operation is a monadic action. Chain them with do-notation — read files, run shell commands, query a KV store, all in one program:

```haskell
content <- fsRead "Cargo.toml"
let lineCount = len (lines content)
say ("Cargo.toml has " <> pack (show lineCount) <> " lines")
pure lineCount
```
```
## Output
Cargo.toml has 29 lines

## Result
29
```

A more involved example — glob for files, gather metadata, return structured JSON. One eval replaces many tool calls:

```haskell
files <- fsGlob "tidepool-*/Cargo.toml"
sizes <- mapM (\f -> do
  (sz, _, _) <- fsMetadata f
  pure (object ["file" .= f, "bytes" .= sz])) files
pure (toJSON sizes)
```
```json
[
  {"bytes": 482, "file": "tidepool-bridge/Cargo.toml"},
  {"bytes": 921, "file": "tidepool-codegen/Cargo.toml"},
  ...
]
```

### Free pagination via continuation

When a return value is too large for the context window, the server automatically truncates it in a tree-structure-aware way. Arrays get their tails replaced with stubs; objects get large fields replaced. The truncated result is returned as a suspended continuation — the LLM can resume with a stub ID to drill into the elided subtree:

```haskell
files <- fsGlob "tidepool-*/src/**/*.rs"
sizes <- mapM (\f -> do
  (sz, _, _) <- fsMetadata f
  pure (object ["file" .= f, "bytes" .= sz])) files
pure (toJSON sizes)
```

The result is an 84-element array. Too large — so the server returns the first 8 elements inline and stubs the rest:

```json
[
  {"bytes": 2073, "file": "tidepool-bridge/src/error.rs"},
  {"bytes": 34539, "file": "tidepool-bridge/src/impls.rs"},
  ...
  "[76 more, ~4781 chars -> stub_0]"
]
stubs: [{"id": "stub_0", "size": 4785}]
```

The calling LLM resumes with `stub_0` to page forward. The next page returns more elements and a new stub for the remainder. This happens transparently — the Haskell code just returns `pure (toJSON sizes)` and the pagination machinery wraps it in a suspend/resume cycle automatically.

This is "free" in two senses: the user code doesn't implement it, and it works on any JSON shape — nested objects, arrays of arrays, mixed structures. The truncation walks the Value tree and makes local decisions about what to stub based on a character budget.

### Complex effect sequences

Combine structural code search (ast-grep), file I/O, LLM classification, and persistent state in one program:

```haskell
-- Find all struct definitions in the codegen crate
matches <- sgFind Rust "struct $NAME { $$$FIELDS }"
                  ["tidepool-codegen/src/"]
let summary = map (\m -> case m of
      Match t f l _ _ -> object ["file" .= f, "line" .= l])
      matches

-- Present results, suspend for human steering
answer <- ask ("Found " <> pack (show (length matches))
  <> " structs.\n" <> "Classify all, or pick a file?\n"
  <> "1. Classify all\n2. Pick a file")

-- Resume: classify the selected structs with a fast LLM
let selected = case answer of
      String "2" -> filter (\m -> case m of
        Match _ f _ _ _ -> isInfixOf "emit" f) matches
      _           -> take 5 matches
results <- mapM (\m -> do
  let text = case m of Match t _ _ _ _ -> t
  category <- llm ("Classify this Rust struct as \
                    \'data', 'config', or 'handler': " <> text)
  pure (object ["struct" .= text,
                "category" .= category])) selected

-- Persist for later evals
kvSet "struct_analysis" (toJSON results)
pure (toJSON results)
```

This chains five effects in one computation: ast-grep search, continuation (`ask` suspends, the LLM scouts independently, then resumes with a decision), LLM classification, KV persistence, and structured JSON output. The Haskell code describes the sequence; Rust executes each step.

# Conclusion


[ExoMonad](https://github.com/inannamalick/exomonad). [Tidepool](https://github.com/inannamalick/tidepool).


[^1]: Auto-bootstrapped jailbreak support, siloed Opus workers with templates based on various elicitation strategies, misc other features. No, I haven't published this. Write your own.