
+++
title = "Fully Generic Recursion in Rust: Part 2"
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


This is the third post in a three-post series. In the [first post](https://recursion.wtf/posts/rust_schemes) we developed a stack-safe, ergonomic, and concise method for working with recursive data structures (using a simple expression language as an example). In the [second post](https://recursion.wtf/posts/rust_schemes_2) we made it fully generic, providing a set of generic tools for expanding and collapsing _any_ recursive data structure in Rust.

In this post we will see how to _combine_ these two things - expanding a structure and collapsing it at the same time. In the process, we will create a fully generic stack machine back end, allowing us to write arbitrary recursive functions in Rust while retaining stack safety.

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_eval.gif" alt="execution graph for simultaneously expanding and collapsing a simple expression" position="center" style="border-radius: 8px;" >}}

<!--more--> 

A more full-featured version of the generic recursion machinery discussed in these posts is implemented in my [recursion](https://crates.io/crates/recursion) crate. Docs are [here](https://docs.rs/recursion). Source code, along with examples and benchmarks, [can be found here](https://github.com/inanna-malick/recursion).

## Our Expression Language

Earlier in this series we defined a simple expression language for math, using this enum to represent _a single layer_ of an expression tree:

```rust
pub enum ExprLayer<A> {
    Add { a: A, b: A },
    Sub { a: A, b: A },
    Mul { a: A, b: A },
    LiteralInt { literal: i64 },
}

struct ExprBoxed(Box<ExprLayer<ExprBoxed>>);
```

So that we can easily write expressions out by hand (for test cases and examples), let's define some helper functions (specifics omitted).

```rust
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

We'll be using those traits in this post to introduce a new recursion backend.

# Stack Machines

A stack machine is similar to the call stack used by languages like Rust (if you've seen a stack overflow error, it's the call stack that's overflowing). The stack machines we'll be using in this post will be slightly more complex, but the implementation details aren't that important. We'll be using stack machines to provide _purpose-specific implementation of the call stack_  for our recursive functions. We'll implement our stack machines _once_, as `Collapse` and `Expand` instances, such that users of the abstractions described in this post need not worry about the specifics. By using our own stack instead of the Rust call stack, we don't need to worry about stack overflow errors.


## Recursive Data Structure

Instead of using a vector of elements linked by vector indices, we'll construct a vector where linkages are defined only by the _position_ of elements. This lets us replace the `usize` vector indices used in the previous posts with zero-size markers that, instead of representing a specific index, just indicate that we should pop one element off the stack.  This results in an extremely compact representation of recursive structures.


Here's what the data structure looks like

```rust
/// Recursive tree made up of layers of some type 'Layer<_>', eg `ExprLayer<_>`
pub struct RecursiveTree<Wrapped> {
    /// nonempty, in topological-sorted order
    elems: Vec<Wrapped>, // Layer<Index> (eg `ExprLayer<()>`)
}
```

Here's a visualization of what this data structure looks like for the expression `5 - 3 * 3 + 12`.

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_structure.png" alt="simple expr data at rest" position="center" style="border-radius: 8px;" >}}

# Expand

Let's see what expanding a boxed expression tree for `5 - 3 * 3 + 12` into a `RecursiveTree` looks like. We'll look at the implementation soon, but this visualization shows what the computation looks like.

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_expand_only.gif" alt="execution graph for expansion of a simple expression" position="center" style="border-radius: 8px;" >}}


Expanding out the data structure takes the form of a depth-first traversal. Layers are generated and pushed onto a vector of elements, which then forms the `RecursiveTree`. Feel free to skim the implementation of `expand_layers`, if you're curious, but the important thing is _that_ we have an implementation of `Expand`, not _how_ we've implemented.

<details>
      <summary>Click to expand code sample</summary>
      
```rust
impl<A, Underlying, O: MapLayer<(), Unwrapped = A, To = U>> Expand<A, O>
    for RecursiveTree<U>
{
    fn expand_layers<F: Fn(A) -> O>(a: A, generate_layer: F) -> Self {
        let mut frontier = Vec::from([a]);
        let mut elems = vec![];

        // expand to build a vec of elems while preserving topo order
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

        Self {
            elems,
            _underlying: std::marker::PhantomData,
        }
    }
}
```
</details>

Let's see what this looks like when expanding  `5 - 3 * 3 + 12` into a recursive tree. Since it's already made up of layers, we can just dereference the boxed value.

```rust
let expr_tree = RecursiveTree::expand(expr, |ExprBoxed(boxed)| *boxed)
```


# Collapse

Let's see what this looks like when collapsing  the `RecursiveTree` for `5 - 3 * 3 + 12` that we constructed in the last section. As in the last section, we'll look at the implementation soon, but this visualization shows what the computation looks like.

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_collapse_only.gif" alt="execution graph for collapsing of a simple expression" position="center" style="border-radius: 8px;" >}}


We collapse a recursive tree by traversing the stack of layers, applying a `collapse_layers` function, and pushing the results to a results stack. As in the last section, feel free to skim the implementation of `collapse_layers` if you're curious, but the implementation details aren't as important as the fact that we have an implementation of `Collapse` for `RecursiveTree`.

<details>
      <summary>Click to expand code sample</summary>

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
</details>

Here's how we can use this to collapse our expression:

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

# Combining Expand and Collapse

As a reminder, the `RecursiveTree` representing `5 - 3 * 3 + 12` looks like this:

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_structure.png" alt="simple expr data at rest" position="center" style="border-radius: 8px;" >}}

Here's a visualization showing what the full evaluation looks like, if we run expand immediately followed by collapse

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_eval_sorted.gif" alt="execution graph for expanding and then collapsing a simple expression" position="center" style="border-radius: 8px;" >}}

But what if we could avoid holding the full tree in memory? It should be possible to evaluate the expression like this, one branch at a time:

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_eval.gif" alt="execution graph for simultaneously expanding and collapsing a simple expression" position="center" style="border-radius: 8px;" >}}


As before, the type signature is the important part - the full implementation is below, if you're curious, but you don't need to understand the internals to use the function.

```rust
/// For some Layer type, Expandable is Layer<Seed> and Collapsable is Layer<Out>
pub fn expand_and_collapse<Seed, Out, Expandable, Collapsable>(
    seed: Seed,
    mut expand_layer: impl FnMut(Seed) -> Expandable,
    mut collapse_layer: impl FnMut(Collapsable) -> Out,
) -> Out
```

<details>
      <summary>Click to expand code sample</summary>

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
                    seeds.push(seed)
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
</details>


This is _fully generic_, and can be used with any recursive structure, not just the simple expression language used in this post. In the next post, I'll show how to use it to build an expression language for matching filesystem entities, with features like short-circuiting to minimize syscalls, arena allocation (as a flex), and, of course, many more execution graphs.

# Haskell Bullshit (feel free to skip)

If you're familiar with Haskell and Recursion schemes, you might recognize this as a hylomorphism. If you don't know what the hell a hylomorphism is and are interested in learning, there's an [excellent blog post here](https://blog.sumtypeofway.com/posts/recursion-schemes-part-5.html):  

# Thank you

Thank you to [Fiona](https://twitter.com/munin), [Rain](https://twitter.com/sunshowers6), [Eliza](https://twitter.com/mycoliza), among others, for reviewing drafts of this post.