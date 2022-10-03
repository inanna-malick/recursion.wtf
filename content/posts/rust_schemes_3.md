
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

%%People say recursive algorithms are a bad fit for rust, but it's not a big deal - just write your own replacement for the call stack. That leads us to the second problem: this is complex and results in bug-prone and ugly code.%% 

FIXME: people will be, like, wtf is a stack machine?

This is the third post in a three-post series. In the first post (link) we developed a stack-safe, ergonomic, and concise method for working with recursive data structures (using a simple expression language as an example). In the second post (link) we made it fully generic, providing a set of generic tools for expanding and collapsing _any_ recursive data structure in Rust.

In this post we will see how to _combine_ these two things - expanding a structure and collapsing it at the same time. In the process, we will create a fully generic stack machine back end, allowing us to write arbitrary recursive functions in Rust while retaining stack safety.

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_eval.gif" alt="execution graph for simultaneously expanding and collapsing a simple expression" position="center" style="border-radius: 8px;" >}}

<!--more--> 

# A Recap

TODO: REMOVE?

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

Here's a visualization of what this data structure might look like for the expression `5 - 3 * 3 + 12`.

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_structure.png" alt="simple expr data at rest" position="center" style="border-radius: 8px;" >}}

## Expand

Expanding out the data structure takes the form of a depth-first traversal. Layers are generated and pushed onto a vector of elements, which then forms the `RecursiveTree`. The full implementation is elided, but if you're curious you can see the full source code here (todo github link https://github.com/inanna-malick/recursion/blob/18c96463b19c12f7a106e02938f01d6d40ac5d9d/src/recursive_tree/stack_machine_eval.rs#L18-L46):

```rust
impl<A, Underlying, O: MapLayer<(), Unwrapped = A, To = U>> Expand<A, O>
    for RecursiveTree<U>
{
    fn expand_layers<F: Fn(A) -> O>(a: A, generate_layer: F) -> Self {
       ...
    }
}
```

Let's see what this looks like when expanding  `5 - 3 * 3 + 12` into a recursive tree. Since it's already made up of layers, we can just dereference the boxed value.

```rust
let expr_tree = RecursiveTree::expand(boxed_expr_tree, |ExprBoxed(boxed)| *boxed)
```

### Expand Visualization

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_expand_only.gif" alt="execution graph for expansion of a simple expression" position="center" style="border-radius: 8px;" >}}

## Collapse

Here's how we collapse a recursive tree. As with the previous `expand_layers` implementation, the full implementation is elided. If you're curious you can see the full source code (todo github link https://github.com/inanna-malick/recursion/blob/18c96463b19c12f7a106e02938f01d6d40ac5d9d/src/recursive_tree/stack_machine_eval.rs#L48-L63):

```rust
impl<A, O, U: MapLayer<A, To = O, Unwrapped = ()>> Collapse<A, O>
    for RecursiveTree<U>
{
    fn collapse_layers<F: FnMut(O) -> A>(self, mut collapse_layer: F) -> A {
        ...
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

### Collapse Visualization

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_collapse_only.gif" alt="execution graph for collapsing of a simple expression" position="center" style="border-radius: 8px;" >}}

# Combining Expand and Collapse
(TODO: replace 'data at rest' with something else)
As a reminder, the data at rest for the expression evaluation shown above looks like this:

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_structure.png" alt="simple expr data at rest" position="center" style="border-radius: 8px;" >}}

Here's what the full evaluation looks like, if we run expand immediately followed by collapse

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_eval_sorted.gif" alt="execution graph for expanding and then collapsing a simple expression" position="center" style="border-radius: 8px;" >}}


## Expand And Collapse

But what if we could avoid holding all of this in memory? It should be possible to evaluate the expression like this, one branch at a time:

{{< image src="/img/rust_schemes/stack_machines_1/simple_expr_eval.gif" alt="execution graph for simultaneously expanding and collapsing a simple expression" position="center" style="border-radius: 8px;" >}}

The code here is a bit complex, so don't worry about understanding all of the internals. At the conceptual level, it's just a stack machine built by fusing the expand and collapse stages defined earlier in this post. If you're curious, the full implementation is here (todo link https://github.com/inanna-malick/recursion/blob/18c96463b19c12f7a106e02938f01d6d40ac5d9d/src/stack_machine.rs#L53-L95)

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
  ...
}
```

This is _fully generic_, and can be used with any recursive structure. In the next post, I'll show how to use it to build an expression language for matching filesystem entities, with features like short-circuiting to minimize syscalls, arena allocation (as a flex), and, of course, many more execution graphs.

# Thank you

Thank you to [Fiona](https://twitter.com/munin) and [Rain](https://twitter.com/sunshowers6), among others, for reviewing drafts of this post.
