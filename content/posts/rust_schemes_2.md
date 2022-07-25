
+++
title = "Fully generic recursion in Rust"
date = "2002-07-24"
author = "Inanna Malick"
authorTwitter = "inanna_malick"
tags = ["recursion schemes", "rust", "code", "generic"]
keywords = ["recursion schemes", "rust", "code"]
showFullContent = false
images = ["/img/rust_schemes/criterion_screenshot_preview.png"]
feature = "/img/rust_schemes/criterion_screenshot_preview.png"
thumbnail = "/img/rust_schemes/criterion_screenshot_preview.png"
+++


This is the second in a series of blog posts. Previously, we introduced a method for writing performant stack safe recursion in Rust for a recursive data structure. This post shows the core ideas I used to implement a _single_ recursive backend that can handle collapsing or expanding any user-defined recursive data structure. I have a [crate here](https://crates.io/crates/recursion), with the [source code here](https://github.com/inanna-malick/recursion).

<!--more--> 

# A Recap

[Last time](https://recursion.wtf/posts/rust_schemes/), we implemented `collapse_layers` and `expand_layers`[^cata_ana] functions, specialized to `ExprTopo`.

[^cata_ana]: If you're a recursion schemes nerd like me, you might notice that these correspond to catamorphism (a collapsing change) and anamorphism (an expanding change), but with less greek. They don't sound as cool, but I think they're a more readable representation of the same concept.

​We're going to start by looking at our `ExprTopo::collapse_layers` function from the last post. Note that it's specialized for use with `ExprLayer`. We explored the impl in detail last time, so we're going to elide that, and just look at the types:

```rust
pub struct ExprIdx(usize);

pub struct ExprTopo {
    // nonempty, in topological-sorted order. guaranteed via construction.
    elems: Vec<ExprLayer<ExprIdx>>,
}

impl ExprTopo {
    fn collapse_layers<F: FnMut(ExprLayer<A>) -> A>(self, mut collapse_layer: F) -> A {
        /* see previous post for full impl */
    }
}
```

It's generic in one way - it can take any function `ExprLayer<A> -> A` and use it to collapse an `ExprTopo` down to a single value. But it's still tied, explicitly, to `ExprTopo`. It would be better if we only had to write this function once, instead of once per data structure. That way, we can focus on optimizing that function, and can closely analyze it for correctness (instead of having to write it multiple times, at the risk of introducing subtle errors in boilerplate code).

# A more Generic collapse 

Here's what we want our generic `collapse_layers` function to look like:

```rust
/// Support for collapsing a data structure into a single value, one layer at a time
pub trait Collapse<A, Wrapped> {
    fn collapse_layers<F: FnMut(Wrapped) -> A>(self, collapse_layer: F) -> A;
}
```

This should be read as being parameterized over some type `Layer`, with `Wrapped` being `Layer<A>`[^wrapped_excuse]. `A` is just the type we're collapsing the data structure down into. Here's a link to this trait in the crate documentation.

[^wrapped_excuse]: In a perfect word, we could parameterize this over the `Layer` type (such as `ExprLayer`), but in Rust all types need to be fully applied, so we need to use `Wrapped` instead, to represent the fully applied `Layer<A>` type.

While we're at it, let's put together a trait for expanding a recursive data structure from some seed:


```rust
/// Support for expanding a data structure from a seed value, one layer at a time
pub trait Expand<A, Wrapped> {
    fn expand_layers<F: Fn(A) -> Wrapped>(a: A, expand_layer: F) -> Self;
}
```

Now that we have these traits, we can represent the generic ability to collapse or expand _arbitrary_ recursive data structures, we just need to figure out how to implement them.

# But how?

[In the last post](https://recursion.wtf/posts/rust_schemes/#factoring-out-duplicated-code), we saw how `ExprLayer::map` could be used to factor out the core logic of `collapse_layers` and `expand_layers`. We're going to abstract over _map_ so we can map over any data structure, instead of just `ExprLayer<_>`.

This is why I kept talking about `Functor` in the last post: we can write `Collapse` and `Expand` implementations over some `Layer<_>` type given only a `Functor` instance for that type. This makes sense, given that the `ExprLayer<_>`-specific expand and collapse functions both use `fmap` as a core component.

You should read this as being implemented for some `Layer<_>` (eg `ExprLayer<_>`) with the (pseudocode) type signature `fn fmap(self: Layer<A>, f: Fn(A) -> B) -> Layer<B>`.

```rust
/// The ability to map over some a single layer of some structure
/// 'Layer<_>', eg 'ExprLayer<_>'.
pub trait Functor<B> { // where Self is Layer<A>
    type Unwrapped;
    type To;
    /// fn fmap(Self: Layer<A>, f: Fn(A) -> B) -> Layer<B>
    fn fmap<F: FnMut(Self::Unwrapped) -> B>(self, f: F) -> Self::To;
}
```

And here it is [^functor_disclaimer]


[^functor_disclaimer]: Rust's type system isn't _quite_ up to expressing all the required `Functor` constraints but it's enough for our purpose

Here's what it looks like as implemented for `ExprLayer<_>`


```rust
impl<A, B> Functor<B> for ExprLayer<A> {
    type To = ExprLayer<B>;
    type Unwrapped = A;

    fn fmap<F: FnMut(Self::Unwrapped) -> B>(self, mut f: F) -> Self::To {
        match self {
            Expr::Add(a, b) => Expr::Add(f(a), f(b)),
            Expr::Sub(a, b) => Expr::Sub(f(a), f(b)),
            Expr::Mul(a, b) => Expr::Mul(f(a), f(b)),
            Expr::LiteralInt(x) => Expr::LiteralInt(x),
        }
    }
}
```

Nothing too complex, but by making it generic we've unlocked an _incredible power_, power _to rival the gods_.

# Implementing the Collapse and Expand traits

We'll be implementing `Collapse` and `Expand` for this data structure: 

```rust
/// Index into the vector of elements
struct Index(usize);

/// Recursive tree made up of layers of some type 'Layer<_>', eg `ExprLayer<_>`
pub struct RecursiveTree<Wrapped> {
    /// nonempty, in topological-sorted order
    elems: Vec<Wrapped>, // Layer<Index> (eg `Layer<_>`)
}
```

`Index` is a generic version of the `ExprIndex` used in the previous post - it's an internal index used to track links between different layers of the `RecursiveTree`.

Here's what `ExprTopo` would look like using `RecursiveTree`:

```rust
type ExprTopo = RecursiveTree<ExprLayer<Index>>
```

## Collapse

Here's what the implementation of `Collapse` looks like. I've elided the internal machinery so we can focus on the types - the full source code is [here](https://github.com/inanna-malick/recursion/blob/41b40a89d5f20fa86990db5d75f588d0e4674317/src/recursive_tree/arena_eval.rs#L96-L127), but since it's fully generic you can just use the crate, there's no need to ever implement it yourself.



```rust
impl<A, Wrapped, Underlying> Collapse<A, Wrapped>
    for RecursiveTree<Underlying>
    where
            Underlying: Functor<A, To = Wrapped, Unwrapped = Index>
{
    fn collapse_layers<F: FnMut(Wrapped) -> A>(self, mut collapse_layer: F) -> A {
        /* elided */
    }
}
```


This is very similar to the `ExprTopo::collapse_layers` function [in the previous post](https://recursion.wtf/posts/rust_schemes/#making-it-generic), and all we need is a `Functor`!

Here's what it looks like in use:

```rust
pub fn eval(expr: ExprTopo) -> i64 {
    self.collapse_layers(|expr| match expr {
        ExprLayer::Add { a, b } => a + b,
        ExprLayer::Sub { a, b } => a - b,
        ExprLayer::Mul { a, b } => a * b,
        ExprLayer::LiteralInt { literal } => literal,
    })
}
```


## Expand

Here's what expanding this data structure from a seed value looks like. As before, I've elided the implementation. If you're curious, the full source code is [here](https://github.com/inanna-malick/recursion/blob/41b40a89d5f20fa86990db5d75f588d0e4674317/src/recursive_tree/arena_eval.rs#L26-L52).

```rust
impl<A, Underlying, Wrapped> Expand<A, Wrapped> for RecursiveTree<Underlying>
where
    Wrapped: Functor<Index, Unwrapped = A, To = Underlying>,
{
    fn expand_layers<F: Fn(A) -> Wrapped>(a: A, expand_layer: F) -> Self {
        /* elided */
    }
}
```

Very similar to the `ExprTopo::expand_layers` function [in the previous post](https://recursion.wtf/posts/rust_schemes/#making-it-generic-1), but over _some generic structure_. Just to verify, let's write a function to expand out an `ExprTopo` from a boxed expression, just as we did in the last post. But this time, it's generic:

```rust
pub fn from_boxed(ast: &ExprBoxed) -> ExprTopo {
    ExprTopo::expand(ast, |seed| match seed {
        ExprBoxed::Add { a, b } => ExprLayer::Add { a, b },
        ExprBoxed::Sub { a, b } => ExprLayer::Sub { a, b },
        ExprBoxed::Mul { a, b } => ExprLayer::Mul { a, b },
        ExprBoxed::LiteralInt { literal } => ExprLayer::LiteralInt { literal: *literal },
    })
}
```

Nice.


These pass all the same tests as the `Expr`-specialized `collapse_layers`/`expand_layers` functions, but we only have to write the machinery of recursion once! Not once per recursive data structure, just once! [^drama]

[^drama]: a bolt of lightning strikes behind me. I am momentarily shown silhouetted by the actinic blue light. It is very dramatic

Just as before, I want to emphasize that this is fully generic over _any_ recursive data structure:

# A minimal example

Here's what an N-tree looks like using this idiom (an N-tree is a tree where each node can have any number of child nodes, and where some value `V` is stored at each node of the tree. Nodes with no child nodes are leaf nodes)

```rust
pub struct NTreeLayer<Val, A> {
    val: Val,
    children: Vec<A>,
}

pub type RecursiveNTree<V> = RecursiveTree<NTreeLayer<V, Index>>;

impl<A, B, V> Functor<B> for NTreeLayer<V, A> {
    type To = NTreeLayer<V, B>;
    type Unwrapped = A;

    fn fmap<F: FnMut(Self::Unwrapped) -> B>(self, f: F) -> Self::To {
        Self::To {
            val: self.val,
            children: self.children.into_iter().map(f).collect(),
        }
    }
}
```

This is pretty good, not much boilerplate! 

Here's some simple functions over our N-tree:

```rust
pub fn depth<V>(r: RecursiveNTree<V>) -> usize {
    r.collapse_layers(|layer| layer.children.iter().max().map_or(1, |n| n + 1))
}

pub fn max<V: Ord>(r: RecursiveNTree<V>) -> Option<V> {
    r.collapse_layers(|layer| layer.children.into_iter().filter_map(|x| x).max())
}
```

Cool, right?

# Async IO

I've used my crate to implement a small but fully functional filetree reader/file contents search tool example. It's fully async, with all the bells and whistles one might expect. [^grep_thingy_note]. [You can find the source code here.](https://github.com/inanna-malick/recursion/blob/41b40a89d5f20fa86990db5d75f588d0e4674317/examples/grep.rs).

[^grep_thingy_note]: ok, so it just uses `tokio::fs` (which just calls std lib blocking functions to work with the file system) instead of something hand crafted with manual file handle management, but there's a good reason I didn't implement that: I didn't want to

<pre><font color="#A6CC70"><b>➜ </b></font>./my_grep -- --regex &quot;Expr.*Sub&quot; --paths-to-ignore .git -p target
<font color="#95E6CB">sparse filetree depth:</font> 4
<font color="#95E6CB">file:</font> &quot;/home/inanna/dev/rust-schemes/src/examples/expr.rs&quot;
<font color="#95E6CB">permissions</font> Permissions(FilePermissions { mode: 33204 })
<font color="#95E6CB">modified</font> Ok(SystemTime { tv_sec: 1658530322, tv_nsec: 155879740 })
<font color="#B48EAD">29::</font>	            Expr::Sub(a, b) =&gt; Expr::Sub(f(a), f(b)),
<font color="#B48EAD">45::</font>	            Expr::Sub(a, b) =&gt; Expr::Sub(f(*a), f(*b)),


<font color="#95E6CB">file:</font> &quot;/home/inanna/dev/rust-schemes/src/examples/expr/eval.rs&quot;
<font color="#95E6CB">permissions</font> Permissions(FilePermissions { mode: 33204 })
<font color="#95E6CB">modified</font> Ok(SystemTime { tv_sec: 1658527781, tv_nsec: 256109881 })
<font color="#B48EAD">54::</font>	        Expr::Sub(a, b) =&gt; Ok(CompiledExpr::Sub(a, b)),
<font color="#B48EAD">70::</font>	        CompiledExpr::Sub(a, b) =&gt; a - b,
<font color="#B48EAD">80::</font>	        Expr::Sub(a, b) =&gt; a - b,
<font color="#B48EAD">89::</font>	        ExprAST::Sub(a, b) =&gt; naive_eval(a) - naive_eval(b),
</pre>

# To be continued

My next post will show how to implement different recursion backends (yes, this is where we finally get to see stack machines in action), along with some cool stuff with fused `expand_layers` and `collapse_layers` operations such that we can construct a recursive data structure and consume it at the same time, without having to allocate a `RecursiveTree`.


# Thank you

Thank you to [Fiona](https://twitter.com/munin), [Rain](https://twitter.com/sunshowers6), [Eliza](https://twitter.com/mycoliza), among others, for reviewing drafts of this post.
