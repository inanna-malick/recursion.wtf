
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


This is the second in a series of blog posts on recursion in rust. The previous blog post introduced a method for writing performant stack safe recursion in Rust, but it was scoped to a single recursive data structure - a simple expression language. This post shows how to write _one_ recursive toolkit that can handle expanding and collapsing any recursive data structure. This has been implemented, with examples, in the crate here and the repo here.

<!--more--> 

# A Recap

​We're going to start by looking at our `ExprTopo::fold` function from the last post. Note that it's specialized for use with `ExprLayer`. We explored the impl in detail last time, so we're going to elide that, and just look at the types:

```rust
impl ExprTopo {
    fn fold<F: FnMut(ExprLayer<A>) -> A>(self, mut fold_layer: F) -> A {
        /* see previous post for full impl */
    }
}
```

It's generic in one way - it can take any function `ExprLayer<A> -> A` and use it to collapse an `ExprTopo` down to a single value. But it's still tied, explicitly, to `ExprTopo`. We want to use this, with all the complex internal machinery (that we really don't want to have to duplicate everywhere), written only once.

# A more Generic collapse 

We're going to start by renaming `fold` to `collapse`[^why_rename_collapse] and defining a trait `Collapse`:


[^why_rename_collapse]: because the function name `fold` is already used by the core `core::std::Iterator` trait, which results in confusing type errors. Thi looks like: 

```rust
/// Support for collapsing a structure into a single value, one layer at a time
pub trait Collapse<A, Wrapped> {
    fn collapse_layers<F: FnMut(Wrapped) -> A>(self, collapse_layer: F) -> A;
}
```

This should be read as being parameterized over some type `Layer`, with `Wrapped` being `Layer<A>`[^wrapped_excuse]. `A` is just the type we're collapsing the structure down into. Here's a link to this trait in the crate documentation.

[^wrapped_excuse]: In a perfect word, we could parameterize this over the `Layer` type (such as `ExprLayer`), but in Rust all types need to be fully applied, so we need to use `Wrapped` instead, to represent the fully applied `Layer<A>` type.

While we're at it, let's put together a trait for expanding a recursive structure from some seed - this corresponds to `generate` from the last post, but `collapse` and `expand` work really well together so we'll use `Expand` for our trait name and `expand_layers` for our function[^cata_ana]:

[^cata_ana]: If you're a recursion schemes nerd like me, you might notice that these correspond to catamorphism (a collapsing change) and anamorphism (an expanding change), but with less greek. They don't sound as cool, but I think they're more readable representations of the same concept.

```rust
/// Support for expanding a structure from a seed value, one layer at a time
pub trait Expand<A, Wrapped> {
    fn expand_layers<F: Fn(A) -> Wrapped>(a: A, expand_layer: F) -> Self;
}
```

Here it is - this trait represents the ability to expand a recursive data structure out from some seed.


# But how?

Here's why I kept talking about `Functor` in the last post: we can write `Collapse` and `Expand` implementations over some `Layer` type given only a `Functor` instance for that type. This makes sense, given that the `ExprLayer`-specific `expand` and `collapse` implementations both use `fmap` as a core component.
```rust
/// The ability to map over some structure
pub trait Functor<B> {
    type Unwrapped;
    type To;
    /// fmap over an owned value
    fn fmap<F: FnMut(Self::Unwrapped) -> B>(self, f: F) -> Self::To;
}
```

And here it is [^functor_disclaimer]


[^functor_disclaimer]: Rust's type system isn't _quite_ up to expressing all the required `Functor` constraints but it's enough for our purpose

Here's what it looks like as implemented for `ExprLayer`


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

# Implementing the Expand and Collapse traits

## Recursive Tree

We'll be implementing `Expand` and `Collapse` for this structure: 

```rust
/// Recursive tree of some structure of type 'Layer'
pub struct RecursiveTree<Wrapped, Index> {
    /// nonempty, in topological-sorted order
    elems: Vec<Wrapped>, // Layer<Index>
    /// the index type over which 'Layer' is parameterized
    _underlying: std::marker::PhantomData<Index>,
}
```

It's pretty much the same as `ExprTopo`, with some extra type parameters for bookkeeping.

Here's what `ExprTopo` would look like using `RecursiveTree`:

```rust
struct ArenaIndex(usize);
type ExprTopo = RecursiveTree<ExprLayer, ArenaIndex>
```

## Expand

Here's what expanding this structure looks like. It's the same as the `ExprLayer`-specific implementation, but we're using a _generic_ `fmap` instead of a `map` function specific to `ExprLayer`:


```rust
impl<A, Underlying, Wrapped> Expand<A, Wrapped> for RecursiveTree<Underlying, ArenaIndex>
where
    Wrapped: Functor<ArenaIndex, Unwrapped = A, To = Underlying>,
{
    fn expand_layers<F: Fn(A) -> Wrapped>(a: A, expand_layer: F) -> Self {
        let mut frontier = VecDeque::from([a]);
        let mut elems = vec![];

        // unfold to build a vec of elems while preserving topo order
        while let Some(seed) = frontier.pop_front() {
            let layer = expand_layer(seed);

            let layer = layer.fmap(|aa| {
                frontier.push_back(aa);
                // idx of pointed-to element determined from frontier + elems size
                ArenaIndex(elems.len() + frontier.len())
            });

            elems.push(layer);
        }

        Self {
            elems,
            _underlying: std::marker::PhantomData,
        }
    }
}
```

Very similar to the `generate` function in the previous post (link), but over _some generic structure_. Just to verify, let's write a function to expand out an `ExprTopo` from a boxed expression, just as we did in the last post. But this time, it's generic:

```rust
pub fn expand_from_boxed(ast: &ExprBoxed) -> ExprTopo {
    ExprTopo::expand(ast, |seed| match seed {
        ExprBoxed::Add { a, b } => ExprLayer::Add { a, b },
        ExprBoxed::Sub { a, b } => ExprLayer::Sub { a, b },
        ExprBoxed::Mul { a, b } => ExprLayer::Mul { a, b },
        ExprBoxed::LiteralInt { literal } => ExprLayer::LiteralInt { literal: *literal },
    })
}
```

Nice.

## Collapse

Here's what collapse looks like. Just as before, it's fully generic: all it needs is a `Functor` to handle the internal bookkeeping. Feel free to skim the impl, since it's fully generic you can just use the crate, there's no need to ever impl it yourself.

```rust
impl<A, Wrapped, Underlying> Collapse<A, Wrapped>
    for RecursiveTree<Underlying, ArenaIndex>
    where
            Underlying: Functor<A, To = Wrapped, Unwrapped = ArenaIndex>
{
    fn collapse_layers<F: FnMut(Wrapped) -> A>(self, mut fold_layer: F) -> A {
        let mut results = std::iter::repeat_with(|| MaybeUninit::<A>::uninit())
            .take(self.elems.len())
            .collect::<Vec<_>>();

        for (idx, node) in self.elems.into_iter().enumerate().rev() {
            let alg_res = {
                let node = node.fmap(|ArenaIndex(x)| unsafe {
                    let maybe_uninit =
                        std::mem::replace(results.get_unchecked_mut(x), MaybeUninit::uninit());
                    maybe_uninit.assume_init()
                });
                fold_layer(node)
            };
            results[idx].write(alg_res);
        }

        unsafe {
            let maybe_uninit = std::mem::replace(
                results.get_unchecked_mut(ArenaIndex::head().0),
                MaybeUninit::uninit(),
            );
            maybe_uninit.assume_init()
        }
    }
}
```

Here's what it looks like in use:

```rust
pub fn eval(expr: ExprTopo) -> i64 {
    self.collapse(|expr| match expr {
        ExprLayer::Add { a, b } => a + b,
        ExprLayer::Sub { a, b } => a - b,
        ExprLayer::Mul { a, b } => a * b,
        ExprLayer::LiteralInt { literal } => literal,
    })
}
```

These pass all the same tests as the `Expr`-specialized `fold` function, but we only have to write the machinery of recursion once! Not once per recursive data structure, just once! [^drama]

[^drama]: a bolt of lightning strikes behind me. I am momentarily shown silhouetted by the actinic blue light. It is very dramatic

Just as before, I want to emphasize that this is fully generic over _any layer type_:

# A minimal example

Here's what an N-tree looks like using this idiom (an N-tree is a tree where each node can have any number of child nodes, and where some value `V` is stored at each node of the tree. Nodes with no child nodes are leaf nodes)

```rust
pub struct NTreeLayer<Val, A> {
    val: Val,
    children: Vec<A>,
}

pub type RecursiveNTree<V> = RecursiveTree<NTreeLayer<V, ArenaIndex>, ArenaIndex>;

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

Cool, right? No need to mess about with the visitor pattern, just directly expressing recursive algorithms. Neat.

# More examples

## Async IO

There's a small but fully functional filetree reader/file contents search tool here, in the top-level example directory. It's fully async, with all the bells and whistles one might expect. [^grep_thingy_note] 

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

If you're curious how it's implemented, the source code is here (link).

# To be continued

My next post will show how to implement different recursion backends (yes, this is where we finally get to see stack machines in action), along with some cool stuff with fused `expand` and `collapse` operations such that we can construct a recursive data structure and consume it at the same time, without having to allocate a `RecursiveTree`.


# Thank you

Thank you to [Fiona](https://twitter.com/munin), [Rain](https://twitter.com/sunshowers6), [Eliza](https://twitter.com/mycoliza), among others, for reviewing drafts of this post.
