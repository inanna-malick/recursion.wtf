+++
title = "Transitive Frontier"
date = "2020-12-30"
author = "Inanna Malick"
authorTwitter = "inanna_malick"
cover = ""
tags = ["guppy", "rust"]
keywords = ["rust", "guppy"]
description = "[`guppy`](https://crates.io/crates/guppy) is a rust crate that provides tools for working with cargo build graphs using the full power of the [`petgraph`](https://crates.io/crates/petgraph) graph data structure crate. It's used by Facebook to audit a high-security subset of the cargo build graph for their digital currency, Diem (formerly Libra). Treating the dependency graph resulting from a cargo build operation as a DAG lets us draw on the well-studied field of graph algorithms to answer complex questions about our build without resorting to ad-hoc traversals or re-implementation of common graph primitives."
showFullContent = false
+++


## Querying Cargo Build DAGs with Guppy

[`guppy`](https://crates.io/crates/guppy) is a rust crate that provides tools for working with cargo build graphs using the full power of the [`petgraph`](https://crates.io/crates/petgraph) graph data structure crate. It's used by Facebook to audit a high-security subset of the cargo build graph for their digital currency, Diem (formerly Libra). Treating the dependency graph resulting from a cargo build operation as a DAG lets us draw on the well-studied field of graph algorithms to answer complex questions about our build without resorting to ad-hoc traversals or re-implementation of common graph primitives.

## Transitive Frontier

I've been meaning to find an excuse to build something using [`guppy`](https://crates.io/crates/guppy) and tooling to support refactoring large projects moving from `futures01` to `futures03` seems like a good place to start. With that in mind, I built a tool to audit the places where some dependency (in this case, `futures01`) finds its way into a cargo workspace, either directly or transitively. I've called this tool `transitive-frontier`, because it finds the places along the workspace frontier where transitive dependencies on some crate are introduced.

I've used graphviz to render a simple build graph with dependencies on both `futures01` and `futures03` to help visualize the concept. Every edge that introduces a transitive dependency on `futures01` is highlighted (with the exception of the futures_util compatibility crate, which is intentionally ignored - `transitive-futures` supports skipping such compat crates)

The edges along the workspace frontier that introduce a `futures01` dependency into the workspace have been further highlighted. These edges make up the transitive frontier for that workspace with regard to `futures01`.

{{< image src="/img/transitive_frontier.svg" alt="build dag for a simple workspace" position="center" style="border-radius: 8px;" >}}

Given the above build DAG, this invocation of `transitive_frontier` can be used to find the edges that introduce a `futures01` dependency into the workspace

```shell
transitive_frontier.rs ~/dev/my-workspace -p 'futures 0.1' -s 'futures_util'
```

results in:

```toml
package_id = "futures 0.1.29 (registry+...)"

[frontier]
server = ["futures 0.1.0",]
capabilities_old = ["futures 0.1.0", "library_old 0.1.0"]
```

## Applications

In practice, it's usually not that hard to write up a list of workspace crates that need to be refactored to not use futures 01, but a machine-readable report opens up new possibilities - for example, you could use the output of this tool to build a linter that asserts that no new transitive dependencies on `futures01` have been introduced into a workspace.
            
## Implementation Details

I used [cargo-script](https://github.com/DanielKeep/cargo-script) to build a single-file tool, [structopt](https://crates.io/crates/structopt) to parse arguments and flags, and [serde](https://crates.io/crates/serde) to serialize the output as JSON or TOML.

The full `transitive_frontier.rs` gist can be found [here](https://gist.github.com/inanna-malick/2c828815eda194c652e2eb0fea74b3d5).
