
+++
title = "Stack Machines for Free"
date = "2000-09-01"
author = "Inanna Malick"
authorTwitter = "inanna_malick"
tags = ["recursion schemes", "rust", "code", "generic", "stack_machines"]
keywords = ["recursion schemes", "rust", "code", "stack_machines"]
showFullContent = false
images = ["/img/rust_schemes/.png"]
feature = "/img/rust_schemes/stack_machines_1/simple_expr_eval.gif"
thumbnail = "/img/rust_schemes/stack_machines_1/simple_expr_eval.gif"
+++

Previously we saw how to expand seed values into recursive structures given a function that expands a single layer of structure, and how to collapse recursive structures into a single value given a function that consumes a single layer of structure. Here we'll see how to fuse those two steps, to generate a stack machine that simultaneously expands and collapses recursive structures, given just a function for expanding layers and a function for collapsing layers.

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_eval.gif" alt="execution graph for simultaneously expanding and collapsing a simple expression" position="center" style="border-radius: 8px;" >}}

<!--more--> 

# A Recap
In the last two posts, we introduced a stack safe and ergonomic method for expressing recursion in Rust, and then made a generic implementation of same. This generic recursion backend is implemented in my [recursion](https://crates.io/crates/recursion) crate. Docs are [here](https://docs.rs/recursion). Source code, along with examples and benchmarks, [can be found here](https://github.com/inanna-malick/recursion).

## Our Expression Language

Earlier in this series we defined a simple expression language for math, using an enum to represent _a single layer_ of an expression tree:

```rust
pub enum ExprLayer<A> {
    Add { a: A, b: A },
    Sub { a: A, b: A },
    Mul { a: A, b: A },
    LiteralInt { literal: i64 },
}
```

So that we can easily write expressions out by hand (for test cases and examples), let's define a simple boxed pointer expression tree:

```rust
struct ExprBoxed(Box<ExprLayer<ExprBoxed>>);

// some simple utility functions for creating boxed expressions
impl ExprBoxed {
    fn add(a: Self, b: Self) -> Self { ... }
    fn sub(a: Self, b: Self) -> Self { ... }
    fn mul(a: Self, b: Self) -> Self { ... }
    fn literal_int(x: i64) -> Self { ... }
}
```



In this post we'll be working with the expression `5 - 3 * 3 + 12`. Here's what that looks like as code:

```rust
use ExprBoxed::*;
let expr = mul(
	sub(literal_int(4), literal_int(3)),
	add(literal_int(2), literal_int(12)),
);
```


## Expand and Collapse
Previously, we defined two core traits representing the ability to expand a recursive structure out from some seed, and to collapse a recursive structure down into a single value:

```rust
/// Support for expanding a data structure from a seed value, one layer at a time
pub trait Expand<A, Wrapped> {
    fn expand_layers<F: Fn(A) -> Wrapped>(a: A, expand_layer: F) -> Self;
}

/// Support for collapsing a data structure into a single value, one layer at a time
pub trait Collapse<A, Wrapped> {
    fn collapse_layers<F: FnMut(Wrapped) -> A>(self, collapse_layer: F) -> A;
}
```

We'll be using those traits in this post, to introduce a new recursion backend.


# Stack Machines
Stack machines can be thought of as a _purpose-specific implementation of the call stack_  for a specific recursive function.

## Data At Rest

Instead of using a vector of elements linked by vector indices, we construct a vector where linkages are defined only by the _position_ of elements. This lets us replace the `usize` vector indices with zero-size markers that, instead of representing a specific index, just indicate that we should pop one element off the stack.  This results in an extremely compact representation of recursive structures.


Here's what the data structure looks like

```rust
/// Recursive tree made up of layers of some type 'Layer<_>', eg `ExprLayer<_>`
pub struct RecursiveTree<Wrapped> {
    /// nonempty, in topological-sorted order
    elems: Vec<Wrapped>, // Layer<Index> (eg `ExprLayer<()>`)
}
```

Here's a visualization of what this might look like for the expression `5 - 3 * 3 + 12`.

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_structure.png" alt="simple expr data at rest" position="center" style="border-radius: 8px;" >}}

## Expand

Expanding out the data structure takes the form of a depth-first traversal. Layers are generated and pushed onto a vector of elements, which then forms the `RecursiveTree`. Feel free to skim the implementation:

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

Let's see what this looks like when expanding  `5 - 3 * 3 + 12` into a recursive tree. Since it's already made up of layers, we can just dereference the boxed value.

```rust
let expr_tree = RecursiveTree::expand(boxed_expr_tree, |ExprBoxed(boxed)| *boxed)
```

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_expand_only.gif" alt="execution graph for expansion of a simple expression" position="center" style="border-radius: 8px;" >}}

## Collapse

Here's how we collapse a recursive tree. As with the previous `expand_layers` implementation, it's fine to just skim this code:

```rust
impl<A, O, U: MapLayer<A, To = O, Unwrapped = ()>> Collapse<A, O>
    for RecursiveTree<U>
{
    fn collapse_layers<F: FnMut(O) -> A>(self, mut collapse_layer: F) -> A {
        let mut result_stack = Vec::new();

        for layer in self.elems.into_iter() {
            let layer = layer.map_layer(|_| result_stack.pop().unwrap());

            result_stack.push(collapse_layer(layer));
        }

        result_stack.pop().unwrap()
    }
}
```

Let's see what this looks like when collapsing  `5 - 3 * 3 + 12`.

```rust
let result = recursive_tree.collapse_layers(|expr| {
    use ExprLayer::*;
	match expr {
        Add { a, b } => a + b,
        Sub { a, b } => a - b,
        Mul { a, b } => a * b,
        LiteralInt { literal } => literal,
	}
})
```

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_collapse_only.gif" alt="execution graph for collapsing of a simple expression" position="center" style="border-radius: 8px;" >}}

# Fusing the stages

## Why  Bother?

As a reminder, the data at rest for the expression evaluation shown above looks like this:

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_structure.png" alt="simple expr data at rest" position="center" style="border-radius: 8px;" >}}

Here's what the full evaluation looks like, if we run expand immediately followed by collapse

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_eval_sorted.gif" alt="execution graph for expanding and then collapsing a simple expression" position="center" style="border-radius: 8px;" >}}



## Expand And Collapse

But what if we could avoid holding all of this in memory? It should be possible to evaluate the expression like this, one branch at a time:

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_eval.gif" alt="execution graph for simultaneously expanding and collapsing a simple expression" position="center" style="border-radius: 8px;" >}}

The code here is a bit complex, so don't worry about understanding all of the internals. At the conceptual level, it's just a stack machine built by fusing the expand and collapse stages defined earlier in this post.

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

    let mut result_stack: Vec<Out> = vec![];
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
                let node = node.map_layer(|_: ()| result_stack.pop().unwrap());
                result_stack.push(collapse_layer(node))
            }
        };
    }
    result_stack.pop().unwrap()
}
```

This is _fully generic_, and can be used with any recursive structure. In the next post, I'll show how to use it to build an expression language for matching filesystem entities, with features like short-circuiting to minimize syscalls, arena allocation (as a flex), and, of course, many more execution graphs.

# Thank you

Thank you to [Fiona](https://twitter.com/munin) and [Rain](https://twitter.com/sunshowers6), among others, for reviewing drafts of this post.
