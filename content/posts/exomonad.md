---
title: "Exomonad"
date: 2026-03-08T12:00:00-08:00
draft: true
tags: ["exomonad", "tidepool", "agent-orchestration", "haskell", "rust", "llm"]
categories: ["projects"]
summary: "Heterogeneous agent orchestration — Claude writes the plan, Gemini does the work, Copilot reviews the PR. Plus a lazily evaluated Haskell-in-Rust JIT runtime, because why not."
description: "ExoMonad stitches frontier model binaries together into reconfigurable agent swarms. Tidepool gives them a Haskell effect system compiled to native code via Cranelift."
---

# §1 — ExoMonad

No shipping agent orchestration system is designed for heterogeneous model composition. Open source systems expect you to go all-in on one model family. Closed source systems like Claude Agent Teams are single-vendor by design. ExoMonad stitches them together — Claude Code, Gemini-cli, kimi, letta code, Copilot — all using their existing binaries and your existing subscription plan.

It hooks into Claude Agent Teams' messaging bus, so agents running other model architectures show up as native team members. It builds on Gastown's worktree model, replacing "swarm of agents ramming PRs into main" with a tree-of-worktrees.

It's traditional, when debuting a new orchestration engine, to build something insane and overambitious, like a cleanroom reimplementation of the [C compiler](https://www.anthropic.com/engineering/building-c-compiler). I don't have 20k to burn on tokens, so I built something a bit more modest: a lazily evaluated cranelift JIT compiled haskell-in-rust runtime with native interop.

I also needed a [red team testbench](/posts/vibe_coding_critical_infrastructure/) with auto-bootstrapped jailbreak support and siloed Opus workers running elicitation strategies — so I had an agent build one. That took a few hours, for a radically different use case. Same platform.

## Heterogeneous agent orchestration

Opus is smart but expensive. Gemini is flaky but cheap and fast. Copilot is integrated with GitHub PR review and heavily subsidized. ExoMonad lets you use all of them — Claude writes the plan, Gemini does the work, Copilot reviews the PR.

The flow: Opus TL writes scaffolding and spawns Gemini devs into child worktrees. Each dev works against the scaffolding, commits, and files a PR back to the TL's branch. Copilot automatically reviews the PR — comments get injected into the Gemini agent's pane. Gemini fixes, Copilot re-reviews. When approved, the TL gets notified and either merges or sends change requests. Cheap iteration on the review loop, expensive judgment only at merge points.

[PLACEHOLDER: screenshot/diagram — Opus TL spawning Gemini worktrees, PR review cycle with Copilot, merge back to TL branch]

Different model families have different blind spots and different price points. The landscape shifts weekly. ExoMonad is built to shift with it — reconfiguring its own orchestration to match whatever just dropped.

ExoMonad supports any agent binary with tool-use hooks and MCP support, with all logic evaluated in a shared server via Unix socket at repo root. This server handles MCP tool invocation, agent hooks, message routing, and event-based logic — the glue layer between local agent binaries and cloud-based agentic processes like Copilot PR review.

## Radically reconfigurable

Earlier iterations used YAML config, but I kept re-implementing subsets of Haskell — if/then/else, map, pattern matching — and decided to just use the thing itself.

All orchestration logic — routing, hooks, tool dispatch, event handling — is defined in Haskell effects executed by the ExoMonad server. You don't need to know Haskell. LLMs are fluent in the strongly-typed pure logic core of Haskell. ExoMonad only asks for the composition part. Use the default devswarm config and let agents modify it as needed.

The type system bounds what agents can touch — they can reconfigure orchestration but can't escape into arbitrary IO. Self-modification is a PR where the compiler already verified the wiring. The failure mode is "doesn't compile," not "silently routes tasks to the wrong model for three hours."

The devswarm config defines three roles. TLs spawn child agents into worktrees, merge their PRs, and scaffold the next wave. Devs work inside their worktree, file PRs, and iterate with Copilot on review feedback. Workers investigate and report back — no commits, just structured results piped to whoever dispatched them.

Here's what reconfiguration looks like in practice. Gemini keeps stripping the dash from `#-}` in Haskell LANGUAGE pragmas. So there's a hook:

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

Both are plain Haskell, pattern matching on events, returning effects. Agents can add new guards, new routing rules, new event handlers — all type-checked before deployment.

# §2 — How it Works

## Tree of Worktrees

Flat dispatch (Agent Teams, Gastown) rams everything into one merge queue. Tree decomposition scopes integration to each subtree — conflicts stay local, and each TL only merges work it scaffolded.

### Unfold

- TL writes scaffolding, commits, creates child branches off that commit — each in its own worktree.
- Child agents land in those worktrees with the scaffolding already there. That's their context.
- Child TLs repeat: scaffold, commit, branch, spawn. Gemini devs at the leaves execute, can't spawn.
- The tree grows through commits. Each branch point is a committed scaffold. Context propagates via git.
- Further from root = narrower scope. Opus TLs where judgment matters, Gemini devs where throughput matters.
- Tree shape: preplanned for big pushes, TLs add leaves and replan during execution.

### Iterate

- Each TL breaks work into subunits, dispatches as parallel tasks (conflict-aware — parallel tasks touch different areas).
- Dispatch wave → leaves do work, make commits, file PRs against parent's branch.
- TL merges, commits new scaffolding, dispatches next wave.
- Pattern: dispatch → merge → scaffold → dispatch.
- Within each wave, the PR review ratchet: Gemini submits a PR. Copilot reviews it — free, subsidized. The review comments get injected directly into Gemini's Zellij pane via `InjectMessage`. Gemini reads the feedback, fixes, pushes. Copilot re-reviews. This loops until approval, at which point `NotifyParentAction` tells the TL the PR is ready. Cheap iteration — the expensive model only gets involved at merge decisions.
- PR review failure is the happy path — cheap iteration, not expensive correctness.
- During the long tail of bugfixes, spawn 5 Gemini agents each investigating a different hypothesis with their full 1M context window, feeding results back to the Opus coordinator.

[PLACEHOLDER: Zellij screenshot — 5 Gemini workers (h1-thunk-sharing, h3-float-out, h4-bytearray, h5-varid-collision, h2-case-binder) investigating hypotheses in parallel. All responses route back via Claude Teams channels as synthetic workers. TL pane shows rolling scorecard: H1 disproven, H3 partially plausible, H5 disproven, H2/H4 still running. Opus coordinator on right already synthesizing results, pointing toward binder aliasing.]

### Fold

- Leaves finish → commits → PR to parent.
- Copilot reviews. TL merges, HEAD advances.
- TL PRs to its parent. Same cycle.
- Continues until root PRs to main.

# §3 — Tidepool

## The Runtime

Tidepool is a lazily evaluated Haskell-in-Rust runtime. Haskell composes and sequences effects; Rust handlers interpret them against the real world. No IO in the Haskell layer — it's a perfect sandbox.

Pipeline: GHC compiles Haskell to Core (GHC's intermediate representation) → serialized to CBOR → Cranelift JIT-compiles to native code with full laziness, tail call optimization, and a copying GC. Binaries have zero runtime dependency on Haskell infrastructure.

Trivial example to show the FFI bridge — the interesting cases are the MCP effects below:

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

`haskell_inline!` compiles at build time via GHC. `#[derive(FromCore)]` auto-generates the bridge between Haskell GADT constructors and Rust enums. The Haskell code is pure effect composition — `emit`, `awaitInt`, `randInt` — and the Rust handlers do the IO. Type-safe FFI across a JIT boundary.

## How it was built

[PLACEHOLDER: Interactive visualization of hylo tree / nested worktree model, hosted as static page.]

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

Effects compose. `sgFind` pattern-matches across the codebase → `fsRead` pulls surrounding context → `llmJson` scores each hit with a typed schema → `ask` surfaces the ranked results → the calling LLM drills into whatever looks scariest, steers via `resume`. The calling LLM steers between suspensions.

Concrete example — audit all `todo!`/`panic!`/`unreachable!` in a Rust codebase, score by risk, surface for triage:

**eval** — helpers:
```haskell
data Score = Score { reachable :: Bool, risk :: Int, reason :: Text }

enrich :: Match -> M (Match, Text)
enrich m = do
  src <- fsRead (mFile m)
  let ctx = unlines . stake 11 . sdrop (max' 0 (mLine m - 6)) . lines $ src
  pure (m, ctx)

judge :: (Match, Text) -> M (Match, Score)
judge (m, ctx) = do
  v <- llmJson ("Rate this Rust code location.\nFile: " <> mFile m
                <> "\n```rust\n" <> ctx <> "\n```\n"
                <> "Is this reachable in production? How risky (1=fine, 5=will bite you)?")
        (SObj [("reachable_in_production", SBool), ("risk", SNum), ("reason", SStr)])
  pure (m, Score
    (maybe False id $ v ^? key "reachable_in_production" . _Bool)
    (maybe 1 truncate $ v ^? key "risk" . _Number)
    (maybe "" id $ v ^? key "reason" . _String))
```

**eval** — code:
```haskell
hits <- concatMap (sgFind Rust) ["todo!($$A)", "panic!($$A)", "unreachable!($$A)"]
          >>= mapM enrich
ranked <- sortBy (flip $ comparing risk) <$> mapM judge hits
let menu = imap (\i (m,j) -> show (i+1) <> ". [" <> show (risk j) <> "/5] "
             <> mFile m <> ":" <> show (mLine m) <> "  " <> reason j) ranked
ask ("Pick a number for detail:\n" <> unlines menu)
```

`sgFind` is AST-aware, not regex. `enrich` and `judge` are Kleisli arrows (`a -> M b`), composable via `>>=`/`mapM`. `sortBy (flip $ comparing risk)` is point-free descending sort.

**Response** (suspended):
```json
{
  "text": "Pick a number for detail:\n1. [5/5] tidepool-codegen/src/emit/expr.rs:847  Panic in live code path — if the case scrutinee has an unexpected tag, this fires at runtime with no recovery\n2. [4/5] tidepool-effect/src/dispatch.rs:234  todo! blocks the unhandled-effect fallback; any new effect type will hit this\n3. [3/5] tidepool-runtime/src/render.rs:112  unreachable! in JSON rendering — technically guarded by the match above, but refactors could expose it\n4. [2/5] tidepool-codegen/src/emit/primop.rs:1203  todo! for a niche Float128 primop — no GHC backend emits this yet\n5. [1/5] tidepool-heap/src/gc/trace.rs:45  unreachable! inside debug_assert — compiled out in release",
  "suspended": true,
  "continuation_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

The calling LLM now has the ranked list. It can scout independently — grep for callers, check git blame, read tests — before deciding.

**resume:**
```json
{
  "continuation_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "response": "1"
}
```

Execution picks up exactly where it left off with the selected item.

Without tidepool, this is ~12 LLM-orchestrated tool calls — grep, cat each file, reason about each hit in prose, synthesize a ranking. With tidepool: 1 eval + 1 resume. The Haskell does the grunt work; `ask` suspends at exactly the point where human judgment matters.

Large structured results get tree-structure-aware pagination — not string truncation, but navigable stubs preserving JSON shape. The agent drills into subtrees via `resume`.

# Conclusion

[ExoMonad](https://github.com/inannamalick/exomonad). [Tidepool](https://github.com/inannamalick/tidepool).
