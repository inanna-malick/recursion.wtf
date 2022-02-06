+++
title = "Transitive Frontier"
date = "2020-12-30"
author = "Inanna Malick"
authorTwitter = "inanna_malick"
cover = ""
tags = ["guppy", "rust"]
keywords = ["rust", "guppy"]
description = "Using `guppy` to build a tool for auditing Cargo dependency graphs"
showFullContent = false
+++

## Querying Cargo Dependency DAGs with Guppy

{{< image src="/img/transitive_frontier/project_with_compat_transitive_frontier.svg" alt="dependency graph" position="center" style="border-radius: 8px;" >}}

[`guppy`](https://crates.io/crates/guppy) is a rust crate that provides tools for working with cargo dependency graphs using the [`petgraph`](https://crates.io/crates/petgraph) graph data structure crate. It's used by Facebook to audit a high-security subset of the cargo dependency graph for some of their more high-visibility projects. Treating the dependency graph resulting from a cargo build operation as a DAG lets us draw on the well-studied field of graph algorithms to answer complex questions about our build without resorting to ad-hoc traversals or re-implementation of common graph primitives.

For my first project using `guppy`, I decided to build a tool to produce machine-readable summaries describing why some target dependency is included in a cargo workspace's build graph. My motivation was to support projects that are migrating from `futures 0.1` to `futures 0.3`. Many rust projects started using `futures 0.1` for their initial async implementation, and are still in the process of switching over to `futures 0.3`. If you're interested in learning more about the differences between the two packages, [this blog post by ncameron](https://www.ncameron.org/blog/migrating-a-crate-from-futures-0-1-to-0-3/) is a great resource. Being able to easily generate machine-readable reports opens up new possibilities - for example, you could use the output of this tool to build a linter that asserts that no new transitive dependencies on `futures 0.1` are introduced into a workspace, to provide tooling-backed assurances that usage of `futures 0.1` only ever decreases.

<!--more--> 

The technical name for this is _reverse transitive dependency_, which is just the name for the subset of the dependency graph that leads back to some target dependency.
I decided to focus on the _workspace frontier_, which I've defined as the set of edges in a dependency graph that originate from within a cargo workspace but do not terminate in that cargo workspace. These are the edges in the dependency graph via which dependencies on external packages are introduced into the workspace, which makes them the natural focus for my use case. Since I'm specifically interested in the places where some dependencies on some specific package make their way into the workspace, I can narrow my search to the subset of the dependency graph that leads back to that package, otherwise known as the _reverse transitive dependencies_ of that package.


I've called this tool `transitive-frontier`, because it finds the intersection of the workspace frontier and the reverse transitive dependency graph - that is, the places where dependencies on some target package are introduced into a cargo workspace. on some package are introduced. Building it has been a great way to familiarize myself with `guppy`. You can find it on github [here](https://github.com/inanna-malick/transitive-frontier), if you'd like to follow along or use it to audit dependencies in one of your projects.

## Using Guppy

Before describing the `transitive_frontier` tool, I'm going to go over how I use `guppy`: these examples are run against some simple strawman workspaces, which you can find in the `example_workspaces` directory in the `transitive_frontier` repo. Here's a simplified representation of the dependency graph for the first workspace:

{{< image src="/img/transitive_frontier/project.svg" alt="dependency graph" position="center" style="border-radius: 8px;" >}}

The main entry point into `guppy` is the `PackageGraph`, which holds information about the dependency graph resulting from a cargo build operation (as generated via `cargo metadata`). Here's how to construct a `PackageGraph` instance for your workspace:

```rust
let mut cmd = MetadataCommand::new();
if let Some(workspace) = &opt.workspace {
    cmd.current_dir(workspace);
}
let package_graph = PackageGraph::from_command(&mut cmd)?;
```

Next, let's construct a query to find all the packages that depend on `futures 0.1` in this package graph. You can find package ids in the output of `cargo metadata`, but the `transitive_frontier` tool supports providing a substring of the target package id for convenience.


```rust
let futures =
  "futures 0.1.30 (registry+https://github.com/rust-lang/crates.io-index)";
let target_dependency = PackageId::new(futures);
let package_query = package_graph.query_reverse(iter::once(id))?;
```

After building a query, we resolve it, yielding a `PackageSet`. It's possible to add filters at this stage, to avoid traversing certain dependency edges.

```rust
let package_set = package_query.resolve();
```

After we've constructed the query, we need to resolve it - this step is where we actually traverse edges to build a package set. Here we're just resolving every edge that the query matches, but `guppy` also supports resolving queries with a filter function, to allow for skipping some edges.

Here's a visualization of the package set this query produces, with the reverse transitive dependencies of `futures 0.1` highlighted:


{{< image src="/img/transitive_frontier/project_with_transitive_deps.svg" alt="dependency graph" position="center" style="border-radius: 8px;" >}}

However, we don't want a package set - we want the frontier via which dependencies `futures 0.1` are introduced into our workspace, shown further highlighted here:


{{< image src="/img/transitive_frontier/project_with_transitive_frontier.svg" alt="dependency graph" position="center" style="border-radius: 8px;" >}}

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
target_dependency = "futures 0.1.30"

[frontier]
capabilities-old = ["library_old 0.1.0", "futures 0.1.30"]
server = ["futures 0.1.30"]
```

You can reproduce this by running the following command in the `transitive_frontier` repo: `cargo run -- -p "futures 0.1" -- example_workspaces/project`. This example, and those that will follow, highlight snippets of code similar to those used to implement the functionality of the `transitive_frontier` tool.


### Ignoring Packages

In a real project that's still migrating off `futures 0.1` usage, `futures 0.3` would likely be compiled with the `compat` feature enabled, to better support interop. `example_packages/project_with_compat` provides an example of this. Let's take a look at the transitive frontier for `futures 0.1` for that workspace, via `cargo run -- -p "futures 0.1" -- example_workspaces/project_with_compat`:

```toml
target_dependency = "futures 0.1.30"

[frontier]
capabilities-old = ["futures 0.1.30", "library_old 0.1.0"]
server = ["futures 0.3.8", "futures 0.1.30"]
capabilities-new = ["library_new 0.1.0", "futures 0.3.8"]
```

Unfortunately, the `compat` dependency in `futures_util` makes `futures 0.1` a transitive dependency of `futures 0.3`, which causes every edge that introduces a dependency on `futures 0.3` to be included in the report. This is not the desired behavior. Here's a visualization of the problem, showing the path via which `futures 0.3` depends on `futures 0.1`:

{{< image src="/img/transitive_frontier/project_futures_util_compat.svg" alt="dependency graph" position="center" style="border-radius: 8px;" >}}

To solve this issue, we're going to have to skip some edges:

```rust
let package_set = package_query
      .resolve_with_fn(|_, link| !opt.skip.iter().any(|s| link.to().id().repr().contains(s)));
```

This code block takes a provided list of substrings to ignore and skips edges to any package with an id containing an ignored substring. Let's try regenerating the transitive frontier with that enabled, via `cargo run -- -p "futures 0.1" --skip "futures-util" -- example_workspaces/project_with_compat`:

```toml
target_dependency = "futures 0.1.30 (registry+https://github.com/rust-lang/crates.io-index)"

[frontier]
capabilities-old = ["library_old 0.1.0", "futures 0.1.30"]
server = ["futures 0.1.30"]
```

That fixes the problem by skipping `futures-util`, as you can see in this visualization:

{{< image src="/img/transitive_frontier/project_with_compat_transitive_frontier.svg" alt="dependency graph" position="center" style="border-radius: 8px;" >}}

Now we have the ability to generate machine-readable reports on where transitive dependencies on `futures 0.1` enter the workspace. For this workspace, that doesn't mean that much, but for larger workspaces with tens or even hundreds of crates automating processes like this can be critical.


## Summary

Guppy has been really great to work with. It provides a great set of tools for querying cargo dependency graphs, the docs are really comprehensive, and the maintainers are really responsive and helpful (thanks @rain!). I particularly appreciated being able to explore the problem by writing code to query the graph instead of grepping`Cargo.toml` files.  I was able to leverage all this to build the `transitive_frontier` tool really quickly. The abstractions provided by `guppy` let me spend most of my development time exploring the problem space and thinking about output formats.

`transitive_frontier` supports outputting two machine readable formats (JSON, TOML) and one human readable one (HTML). You can fetch it via `cargo install transitive_frontier`.

## Example Invocations

- Find the transitive frontier for `futures 0.1` for a workspace and output TOML:
```shell
transitive_frontier -p "futures 0.1" --skip "futures-util" \
                    -f toml -- ~/target/dir > summary.toml
```

- Find the transitive frontier for `futures 0.1` for a workspace and output JSON:
```shell
transitive_frontier -p "futures 0.1" --skip "futures-util" \
                    -f json -- ~/target/dir > summary.json
```

- Find the transitive frontier for `futures 0.1` for a workspace and output HTML:
```shell
transitive_frontier -p "futures 0.1" --skip "futures-util" \
                    -f html -- ~/target/dir > summary.html
```



<!-- snippets: -->

<!-- ## Implementation -->

<!-- My initial plan was to output graphviz dot files showing the entire reverse transitive dependency graph, but when run against an IRL project the resulting graph had too many nodes and edges to be readable. To produce a usable artifact I'd have to find a way to reduce the size of the output. I spent some time using different dependency graph combinators in guppy to omit edges and nodes that were less relevant, but the output was still too large to easily visualize. Eventually, I decided to only look at the places in the graph where a dependency on the futures 0.1 was introduced into the cargo workspace - edges with one end inside the workspace and the other outside it. This reduced the size of the report down to something manageable, even for large workspaces with nontrivial complexity. -->
