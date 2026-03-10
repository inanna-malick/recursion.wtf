---
title: "Exomonad"
date: 2026-03-08T12:00:00-08:00
draft: true
tags: ["exomonad", "tidepool", "agent-orchestration", "haskell", "rust", "llm"]
categories: ["projects"]
summary: ""
description: "ExoMonad stitches frontier model binaries together into reconfigurable agent swarms. Tidepool gives them a Haskell effect system compiled to native code via Cranelift."
---

# §1 — ExoMonad

No shipping agent orchestration system is designed for heterogeneous model composition. Open source systems expect you to go all-in on one model family. Closed source systems like Claude Agent Teams are single-vendor by design. ExoMonad stitches them together — Claude Code, Gemini-cli, kimi, letta code, Copilot — all using their existing binaries and your existing subscription plan.

It hooks into Claude Agent Teams' messaging bus, so agents running other model architectures show up as native team members. It builds on Gastown's worktree model, replacing "swarm of agents ramming PRs into main" with a tree-of-worktrees.

It's traditional, when debuting a new orchestration engine, to build something insane and overambitious, like a cleanroom reimplementation of the [C compiler](https://www.anthropic.com/engineering/building-c-compiler). I don't have 20k to burn on tokens, so I built something a bit more modest: a lazily evaluated cranelift JIT compiled haskell-in-rust runtime with native interop.

It's also been used to build a [red team testbench](/posts/vibe_coding_critical_infrastructure/) in a few hours — radically different use case, same platform.[^1]

<!--more-->

## Heterogeneous agent orchestration

Opus is smart but expensive. Gemini is flaky but cheap and fast. Copilot is integrated with GitHub PR review and heavily subsidized. ExoMonad lets you use all of them — Claude writes the plan, Gemini does the work, Copilot reviews the PR.

The flow: Opus TL writes scaffolding and spawns Gemini devs into child worktrees. Each dev works against the scaffolding, commits, and files a PR back to the TL's branch. Copilot automatically reviews the PR — comments get injected into the Gemini agent's pane. Gemini addresses the feedback, pushes fixes, and the TL gets notified to merge or send change requests. Cheap iteration, expensive judgment only at merge points.

{{< figure src="/img/exomonad_zellij_devswarm.png" caption="Zellij devswarm — TL dispatching Wave 4 to three Gemini workers in parallel, each in its own worktree. Bottom panes show workers mid-execution." >}}

Token arbitrage: instead of spending Opus tokens stitching tool calls together, have Claude write tickets and use Gemini for tasks. Opportunities open up when heterogeneous model use is trivial. Different model families have different blind spots and different price points. The landscape shifts weekly. ExoMonad is built to shift with it — reconfiguring its own orchestration to match whatever just dropped.

ExoMonad supports any agent binary with tool-use hooks and MCP support. All logic evaluates in a shared server via Unix socket at repo root — MCP tool invocation, agent hooks, message routing, event dispatch.

## Radically reconfigurable

Earlier iterations used YAML config, but I kept re-implementing subsets of Haskell — if/then/else, map, pattern matching — and decided to just use the thing itself.

All orchestration logic — routing, hooks, tool dispatch, event handling — is defined in Haskell effects executed by the ExoMonad server. Agents write the Haskell, not humans, so you don't need to learn Haskell: your agents already know it, and it's readable so you can review their self-modification PRs. Use the default devswarm config and let agents modify it as needed.

The type system bounds what agents can touch — they can reconfigure orchestration but can't escape into arbitrary IO. Self-modification is a PR where the compiler already verified the wiring. The failure mode is "doesn't compile," not "silently routes tasks to the wrong model for three hours."

The devswarm config defines three roles. TLs spawn child agents into worktrees, merge their PRs, and scaffold the next wave. Devs work inside their worktree, file PRs, and iterate with Copilot on review feedback. Workers investigate and report back — no commits, just structured results piped to whoever dispatched them.

Gemini keeps stripping the dash from `#-}` in Haskell LANGUAGE pragmas. So there's a hook:

```haskell
checkPragmaCorruption (Replace fp old new)
  | ".hs" `T.isSuffixOf` fp
  , "#-}" `T.isInfixOf` old
  , not ("#-}" `T.isInfixOf` new)
  , "#}" `T.isInfixOf` new
  = Just "BLOCKED: Your replacement corrupts Haskell LANGUAGE pragmas..."
checkPragmaCorruption _ = Nothing
```

And PR review routing — when a review comes in, inject the comments into the agent's pane. When approved, notify the TL:

```haskell
prReviewHandler (ReviewReceived n comments) =
  pure (InjectMessage (copilotReviewReceived n comments))

prReviewHandler (ReviewApproved n) =
  pure (NotifyParentAction (prReady n) n)
```

# §2 — How it Works

## Tree of Worktrees

Flat dispatch (Agent Teams, Gastown) rams everything into one merge queue. Tree decomposition scopes integration to each subtree — conflicts stay local, and each TL only merges work it scaffolded.

### Each Worktree is an Agent

- TL writes scaffolding, commits, creates child branches off that commit — each in its own worktree.
- Child agents land in those worktrees with the scaffolding already there. That's their context.
- Child TLs repeat: scaffold, commit, branch, spawn. Gemini devs at the leaves execute, can't spawn.
- The tree grows through commits. Each branch point is a committed scaffold. Context propagates via git.
- Further from root = narrower scope. Opus TLs where judgment matters, Gemini devs where throughput matters.
- Tree shape: preplanned for big pushes, TLs add leaves and replan during execution.

### Waves

- Each TL breaks work into subunits, dispatches as parallel tasks (conflict-aware — parallel tasks touch different areas).
- Dispatch wave → leaves do work, make commits, file PRs against parent's branch.
- TL merges, commits new scaffolding, dispatches next wave.
- Pattern: dispatch → merge → scaffold → dispatch.
- Within each wave: Gemini submits a PR, Copilot reviews (free, subsidized), review comments get injected into Gemini's pane via `InjectMessage`, Gemini addresses feedback and pushes fixes, `NotifyParentAction` tells the TL the PR is ready.
- PR review failure is the happy path — cheap iteration, not expensive correctness.
- During the long tail of bugfixes, spawn 5 Gemini agents each investigating a different hypothesis with their full 1M context window, feeding results back to the Opus coordinator.
- Leaves finish → commits → PR to parent → Copilot reviews → TL merges. Same cycle upward until root PRs to main.

# §3 — Tidepool

## The Runtime

Tidepool is a lazily evaluated Haskell-in-Rust runtime. Haskell composes and sequences effects; Rust handlers interpret them against the real world. No IO in the Haskell layer — it's a perfect sandbox.

Pipeline: GHC compiles Haskell to Core (GHC's intermediate representation) → serialized to CBOR → Cranelift JIT-compiles to native code with full laziness, tail call optimization, and a copying GC. Binaries have zero runtime dependency on Haskell infrastructure.

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

## How it was built

[PLACEHOLDER: screenshot of tidepool build story viz]

[Explore the interactive build story →](/viz/tidepool-build-story.html)

## Tidepool MCP Server

Two tools: `eval` and `resume`. That's the entire surface.

`eval` takes a Haskell do-block. Tidepool JIT-compiles it via Cranelift and runs it. 10 effects available:

- **Console** — print output
- **KV** — key-value store (persists across evals)
- **Fs** — read/write files (sandboxed)
- **SG** — ast-grep structural search (14 languages)
- **Http** — fetch JSON from endpoints
- **Exec** — run shell commands
- **Git** — native libgit2 repo access
- **Llm** — inline Haiku calls (classification, extraction, judgment)
- **Meta** — runtime introspection
- **Ask** — suspend execution, return control to calling LLM

`resume` takes a `continuation_id` and a response. The blocked eval thread unblocks and continues from where it suspended.

`ask "prompt"` suspends the JIT mid-execution — the entire call stack, all gathered data, sitting in the JIT heap — and returns `{suspended: true, continuation_id, prompt}` to the calling LLM. The LLM can go do other work, think, run more evals, then `resume` with an informed answer. Execution picks up exactly where it left off.

Effects compose:

Bytes of Rust per crate, sorted (Fs only):

```haskell
files <- fsGlob "tidepool-*/src/**/*.rs"
fmap (sortBy (flip $ comparing snd) . Map.toList . Map.fromListWith (+))
  (forM files $ \f -> do { (sz,_,_) <- fsMetadata f; pure (head (splitOn "/" f), sz) })
```
```json
[["tidepool-codegen", 444878], ["tidepool-repr", 139678], ...]
```

Cross-reference `unsafe` blocks with git churn (Fs + SG + Exec, no LLM):

```haskell
unsafeFiles <- filterM (\f -> not . null <$> sgFind Rust "unsafe { $$B }" [f])
                 =<< fsGlob "tidepool-*/src/**/*.rs"
forM unsafeFiles $ \f -> do
  n <- length <$> sgFind Rust "unsafe { $$B }" [f]
  (_, log, _) <- run ("git log --oneline -30 -- " <> f)
  pure (object ["file" .= f, "unsafe_blocks" .= n,
                "commits" .= length (filter (not . isNull) (lines log))])
```
```json
[{"file": "src/host_fns.rs", "unsafe_blocks": 93, "commits": 30}, ...]
```

Scan `panic!` sites, score with Haiku, suspend for triage (SG + Llm + Ask):

```haskell
hits <- stake 5 <$> sgFind Rust "panic!($$A)" []
scored <- forM hits $ \m -> do
  v <- llmJson ("Risk 1-5, one sentence.\n" <> mFile m <> ":" <> show (mLine m) <> " " <> mText m)
         (SObj [("risk", SNum), ("reason", SStr)])
  pure (mFile m, mLine m, v ^? key "risk" . _Number, v ^? key "reason" . _String)
let ranked = sortBy (flip $ comparing (\(_,_,r,_) -> r)) scored
ask (unlines (imap (\i (f,l,r,s) ->
  show i <> ". [" <> maybe "?" show r <> "] " <> f <> ":" <> show l
  <> " " <> maybe "" id s) ranked))
```

`ask` suspends. The calling LLM gets the ranked list, can scout independently, then `resume` with a pick. Execution continues from where it left off.

Each example is ≤10 lines. Without tidepool, each is ~6-12 LLM-orchestrated tool calls where intermediate results vanish between rounds. With tidepool: 1 eval, everything in scope.

Large structured results get tree-structure-aware pagination — not string truncation, but navigable stubs preserving JSON shape. The agent drills into subtrees via `resume`.

# Conclusion

I plan to eventually replace ExoMonad's WASM sandbox with Tidepool.

[ExoMonad](https://github.com/inannamalick/exomonad). [Tidepool](https://github.com/inannamalick/tidepool).

[^1]: Auto-bootstrapped jailbreak support, siloed Opus workers with templates based on various elicitation strategies. Took a few hours to reconfigure ExoMonad for an adversarial use case.
