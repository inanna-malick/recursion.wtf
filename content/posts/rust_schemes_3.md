
+++
title = "Stack Machines for Free"
date = "2022-11-22"
author = "Inanna Malick"
authorTwitter = "inanna_malick"
tags = ["recursion schemes", "rust", "code", "generic", "stack_machines"]
keywords = ["recursion schemes", "rust", "code", "stack_machines"]
showFullContent = false
images = ["/img/rust_schemes/.png"]
feature = "/img/rust_schemes/stack_machines_1/simple_expr_eval.gif"
thumbnail = "/img/rust_schemes/stack_machines_1/simple_expr_eval.gif"
+++


TODO REWRITE

%%[Previously](https://recursion.wtf/posts/rust_schemes/), we introduced a method for writing performant stack safe recursion in Rust for a single recursive data structure. This post uses the same ideas to implement a _single_ recursion backend that can collapse or expand any recursive data structure.%%

<!--more--> 

This generic recursion backend is implemented in my new [recursion](https://crates.io/crates/recursion) crate. Docs are [here](https://docs.rs/recursion). Source code, along with examples and benchmarks, [can be found here](https://github.com/inanna-malick/recursion).

# A Recap

## Expand and Collapse

TODO: show expand/collapse interfaces, show expr language we'll be using, show algebras

## Our Expression Language

show expr layer, also show expr boxed _replaced with fixed-point repr, yay me._

# Stack Machines
- note that the stack machine _can be thought of as a purpose-specific impl of the call stack_ . for a specific recursive function.

## Data At Rest
- show stack marker (replacing idx) and impl of same, note that it's just a positional marker that says 'pop one element off the stack'. 
Here's what the data structure looks like

```rust
/// Recursive tree made up of layers of some type 'Layer<_>', eg `ExprLayer<_>`
pub struct RecursiveTree<Wrapped> {
    /// nonempty, in topological-sorted order
    elems: Vec<Wrapped>, // Layer<Index> (eg `ExprLayer<()>`)
}
```

## Expand

```rust
impl<A, Underlying, O: MapLayer<(), Unwrapped = A, To = U>> Expand<A, O>
    for RecursiveTree<U>
{
    fn expand_layers<F: Fn(A) -> O>(a: A, generate_layer: F) -> Self {
        let mut frontier = Vec::from([a]);
        let mut elems = vec![];

        // expand to build a vec of elems while preserving depth-first topo order
        while let Some(seed) = frontier.pop() {
            let layer = generate_layer(seed);

            let mut topush = Vec::new();
            let layer = layer.map_layer(|aa| {
                topush.push(aa);
                ()
            });
            frontier.extend(topush.into_iter().rev());

            elems.push(layer);
        }

        elems.reverse();

        Self {elems}
    }
}
```

Let's see what this looks like when expanding a simple expression (TODO: here).

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_expand_only.gif" alt="execution graph for expansion of a simple expression" position="center" style="border-radius: 8px;" >}}

## Collapse


{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_collapse_only.gif" alt="execution graph for collapsing of a simple expression" position="center" style="border-radius: 8px;" >}}

# Fusing the stages

## Why  Bother?

As a reminder, the data at rest for the expression evaluation shown above looks like this:

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_structure.png" alt="execution graph for collapsing of a simple expression" position="center" style="border-radius: 8px;" >}}

And here's what the full evaluation looks like, if we run expand immediately followed by collapse

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_eval_sorted.gif" alt="execution graph for collapsing of a simple expression" position="center" style="border-radius: 8px;" >}}

But what if we could avoid holding all of this in memory? It should be possible to evaluate the expression like this, one branch at a time:

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_eval.gif" alt="execution graph for collapsing of a simple expression" position="center" style="border-radius: 8px;" >}}


## Expand And Collapse

```rust
pub fn expand_and_collapse<Seed, Out, Expandable, Collapsable>(
    seed: Seed,
    mut expand_layer: impl FnMut(Seed) -> Expandable,
    mut collapse_layer: impl FnMut(Collapsable) -> Out,
) -> Out
where
    Expandable: MapLayer<(), Unwrapped = Seed>,
    <Expandable as MapLayer<()>>::To: MapLayer<Out, Unwrapped = (), To = Collapsable>,
{
    enum State<Seed, CollapsableInternal> {
        Expand(Seed),
        Collapse(CollapsableInternal),
    }

    let mut vals: Vec<Out> = vec![];
    let mut stack = vec![State::Expand(seed)];

    while let Some(item) = stack.pop() {
        match item {
            State::Expand(seed) => {
                let node = expand_layer(seed);
                let mut seeds = Vec::new();
                let node = node.map_layer(|seed| {
                    seeds.push(seed);
                    ()
                });

                stack.push(State::Collapse(node));
                stack.extend(seeds.into_iter().map(State::Expand));
            }
            State::Collapse(node) => {
                let node = node.map_layer(|_: ()| vals.pop().unwrap());
                vals.push(collapse_layer(node))
            }
        };
    }
    vals.pop().unwrap()
}
```


# Thank you

Thank you to [Fiona](https://twitter.com/munin), [Rain](https://twitter.com/sunshowers6), [Eliza](https://twitter.com/mycoliza), among others, for reviewing drafts of this post.
