+++
title = "Transitive Frontier"
date = "2020-12-30"
author = "Inanna Malick"
authorTwitter = "inanna_malick"
cover = ""
tags = ["guppy", "rust"]
keywords = ["rust", "guppy"]
description = "TODO"
showFullContent = false
+++


## Querying Cargo Build DAGs with Guppy

<!-- TODO: add a bit on rain here, mostly their work, mb describe as partner? I'd like that. -->

[guppy](https://crates.io/crates/guppy) is a rust crate that provides tools for working with cargo build graphs using the full power of the [petgraph](https://crates.io/crates/petgraph) graph data structure crate. It's used by Facebook to audit a high-security subset of the cargo build graph for some of their more high-visibility projects. Treating the dependency graph resulting from a cargo build operation as a DAG lets us draw on the well-studied field of graph algorithms to answer complex questions about our build without resorting to ad-hoc traversals or re-implementation of common graph primitives.

I've been meaning to find an excuse to build something using [guppy](https://crates.io/crates/guppy), and decided on a simple tool to audit the places where some dependency finds its way into a cargo workspace, either directly or transitively. I've called this tool `transitive-frontier`, because it finds the places along the workspace frontier where transitive dependencies on some crate are introduced.

The use case I had in mind was large projects moving from futures 0.1 to futures 0.3. In practice, it's usually not that hard to write up a list of workspace crates that need to be refactored to not use futures 0.1, but a machine-readable report opens up new possibilities - for example, you could use the output of this tool to build a linter that asserts that no new transitive dependencies on futures 0.1 have been introduced into a workspace. This project has also been a great way to familiarize myself with guppy. You can find the project [here](https://github.com/inanna-malick/transitive-frontier), if you'd like to follow along (currently private, will publish after review).

## Using Guppy

Before describing the `transitive_frontier` tool, I'm going to go over how I use guppy: these examples are run against some simple strawman workspaces, which you can find in the `example_workspaces` directory in the `transitive_frontier` repo. Here's a simplified representation of the dependency graph for the first workspace:

{{< image src="/img/project.svg" alt="TODO" position="center" style="border-radius: 8px;" >}}

The main entry point into guppy is the `PackageGraph`, which holds information about the dependency graph resulting from a cargo build operation (as generated via `cargo metadata`). Here's how to construct a `PackageGraph` instance for your workspace:

```rust
let mut cmd = MetadataCommand::new();
if let Some(workspace) = &opt.workspace {
    cmd.current_dir(workspace);
}
let package_graph = PackageGraph::from_command(&mut cmd)?;
```

Next, let's construct a query to find all the packages that depend on futures 0.1 in this package graph. You can find package ids in the output of `cargo metadata`, but the `transitive_frontier` tool supports providing a substring of the target package id for convenience.


```rust
let futures =
  "futures 0.1.30 (registry+https://github.com/rust-lang/crates.io-index)";
let package_id = PackageId::new(futures);
let package_query = package_graph.query_reverse(iter::once(id))?;
```

After building a query, we resolve it, yielding a `PackageSet`. It's possible to add filters at this stage, to avoid traversing certain dependency edges.

```rust
let package_set = package_query.resolve();
```

After we've constructed the query, we need to resolve it - this step is where we actually traverse edges to build a package set. Here we're just resolving every edge that the query matches, but guppy also supports resolving queries with a filter function, to allow for skipping some edges.

Here's a visualization of the package set this query produces, with the reverse transitive dependencies of futures 0.1 highlighted:


{{< image src="/img/project_with_transitive_deps.svg" alt="TODO" position="center" style="border-radius: 8px;" >}}

However, we don't want a package set - we want the frontier via which dependencies futures 0.1 are introduced into our workspace, shown further highlighted here:


{{< image src="/img/project_with_transitive_frontier.svg" alt="TODO" position="center" style="border-radius: 8px;" >}}

For this step, we need to traverse the set of edges included in the `PackageSet`. Since it's a _package_ set, it only tracks packages, so what we're actually doing is iterating over every edge in the package graph for where `to` and `from` are members of the package set.

```rust
let mut frontier = HashMap::new();

for link in package_set.links(DependencyDirection::Reverse) {
    // != implements logical xor
    if link.to().in_workspace() != link.from().in_workspace() {
        let entry = frontier
            .entry(link.from().name().to_string())
            .or_insert_with(Vec::new);
        entry.push(format!("{} {}", link.to().name(), link.to().version()))
    }
}
```

This code block iterates over every edge in the `PackageSet`, taking edges with one end inside the workspace and the other outside it and storing them in a `HashMap` keyed by the name of the package introducing the dependency, resulting in this output:

```toml
package_id = "futures 0.1.30 (registry+https://github.com/rust-lang/crates.io-index)"

[frontier]
capabilities_old = ["library_old 0.1.0", "futures 0.1.30"]
server = ["futures 0.1.30"]
```

You can reproduce this by running the following command in the `transitive_frontier` repo: `cargo run -- -p "futures 0.1" -- example_workspaces/project`. This example, and those that will follow, highlight snippets of code similar to those used to implement the functionality of the `transitive_frontier` tool.


### Ignoring Packages

In a real project with heterogenous futures 0.1/futures 0.3 usage futures 0.3 would likely be compiled with the `compat` feature enabled, to better support interop. `example_packages/project_with_compat` provides an example of this. Let's take a look at the transitive frontier for futures 0.1 for that workspace, via `cargo run -- -p "futures 0.1" -- example_workspaces/project_with_compat`:

```toml
package_id = "futures 0.1.30 (registry+https://github.com/rust-lang/crates.io-index)"

[frontier]
capabilities_old = ["futures 0.1.30", "library_old 0.1.0"]
server = ["futures 0.3.8", "futures 0.1.30"]
capabilities_new = ["library_new 0.1.0", "futures 0.3.8"]
```

Unfortunately, the `compat` dependency in `futures_util` makes futures 0.1 a transitive dependency of futures 0.3, which causes every edge that introduces a dependency on futures 0.3 to be included in the report. This is not the desired behavior. Here's a visualization of the problem, showing the path via which futures 0.3 depends on futures 0.1:

{{< image src="/img/project_futures_util_compat.svg" alt="TODO" position="center" style="border-radius: 8px;" >}}

To solve this issue, we're going to have to skip some edges:

```rust
let package_set = package_query
      .resolve_with_fn(|_, link| !opt.skip.iter().any(|s| link.to().id().repr().contains(s)));
```

This code block takes a provided list of substrings to ignore and skips edges to any package with an id containing an ignored substring. Let's try regenerating the transitive frontier with that enabled, via `cargo run -- -p "futures 0.1" --skip "futures-util" -- example_workspaces/project_with_compat`:

```toml
package_id = "futures 0.1.30 (registry+https://github.com/rust-lang/crates.io-index)"

[frontier]
capabilities_old = ["library_old 0.1.0", "futures 0.1.30"]
server = ["futures 0.1.30"]
```

That fixes the problem by skipping `futures-util`, as you can see in this visualization:

{{< image src="/img/project_with_compat_transitive_frontier.svg" alt="TODO" position="center" style="border-radius: 8px;" >}}

Now we have the ability to generate machine-readable reports on where transitive dependencies on futures 0.1 enter the workspace. For this workspace, that doesn't mean that much, but for larger workspaces with tens or even hundreds of crates automating processes like this can be critical.

## Other Applications

TODO: expand here



<!-- snippets: -->

<!-- I've been meaning to find an excuse to build something using [guppy](https://crates.io/crates/guppy) and tooling to support refactoring large projects moving from futures 0.1 to futures 0.3 seems like a good place to start. With that in mind, I built a tool to audit the places where some dependency (in this case, futures 0.1) finds its way into a cargo workspace, either directly or transitively. I've called this tool `transitive-frontier`, because it finds the places along the workspace frontier where transitive dependencies on some crate are introduced. -->

<!-- I've used graphviz to render a simple build graph with dependencies on both futures 0.1 and futures 0.3 to help visualize the concept. Every edge that introduces a transitive dependency on futures 0.1 is highlighted (with the exception of the futures_util compatibility crate, which is intentionally ignored - `transitive-frontier` supports skipping such compat crates) -->

<!-- The edges along the workspace frontier that introduce a futures 0.1 dependency into the workspace have been further highlighted. These edges make up the transitive frontier for that workspace with regard to futures 0.1. -->
