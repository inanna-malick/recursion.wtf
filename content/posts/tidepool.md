---
title: "Tidepool"
date: 2026-03-12
draft: false
tags: ["tidepool", "haskell", "rust", "cranelift", "jit", "llm"]
categories: ["projects"]
description: "A lazily evaluated Haskell-in-Rust runtime with native interop, built by an ExoMonad agent swarm"
---

[Tidepool](https://github.com/tidepool-heavy-industries/tidepool) is a lazily evaluated Haskell-in-Rust runtime with native interop. It's very similar to the WASM sandbox used by [ExoMonad](/posts/exomonad/), except instead of WebAssembly it uses Rust's Cranelift JIT crate directly, operating on the intermediate representation Core used by the Haskell compiler.

I built Tidepool using [ExoMonad](/posts/exomonad/) over about two weeks, using less than 50% of my Claude Max and Gemini Ultra subscription plans. For the build story — how an agent swarm blasted out 32,000 lines across 61 PRs in 4 days — see the [ExoMonad post](/posts/exomonad/).

<!--more-->

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

Tidepool also supports live compilation, in cases where the user has the Haskell compiler available. To demonstrate this, I've built an MCP server that exposes this functionality. It's a bit like GHCI (Haskell's REPL), but specialized for the monadic composition of pure effects that are executed by Rust code.

Here are a few examples of what it can do:

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

## Links

- [Tidepool on GitHub](https://github.com/tidepool-heavy-industries/tidepool)
- [ExoMonad](/posts/exomonad/) — the agent orchestrator used to build it
