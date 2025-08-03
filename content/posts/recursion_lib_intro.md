+++
title = "Recursion: a quick introduction"
date = "2023-10-16"
author = "Inanna Malick"
authorTwitter = "inanna_malick"
tags = ["recursion", "rust", "code"]
keywords = ["recursion", "rust", "code"]
showFullContent = false
images = ["img/recursion_crate/detect_example_expr_frame.gif"]
feature = "img/recursion_crate/detect_example_expr_frame.gif"
thumbnail = "img/recursion_crate/detect_example_expr_frame.gif"
+++


In traditional low level languages such as C iteration is implemented manually, with users writing out {{< highlight c "hl_inline=true">}} for (int idx = 0; idx < items_len; ++idx) { do_thing(items[idx] } {{< /highlight >}}, every time they want to iterate over a list. Newer languages like Rust provide abstractions - iterators - that separate the _machinery_ of recursion from the logic: {{< highlight rust "hl_inline=true">}}  for item in items.iter() { do_thing(item) }{{< /highlight >}} .

The [recursion crate](https://crates.io/crates/recursion) does the same thing for recursive data structures. This post is an introduction to the new version of the `recursion` crate, but you don't have to read [my](https://recursion.wtf/posts/rust_schemes/) [earlier](https://recursion.wtf/posts/rust_schemes_2/) [posts](https://recursion.wtf/posts/rust_schemes_3/) to understand it.


<!--more--> 


# Motivation: a file matching tool


I often use the `find` tool to find files matching some set of criteria - name, metadata, file contents, that kind of thing. Almost every time I use it for something nontrivial, I end up having to google the right combination of command line flags to express my query. For example, let's say we want to list all files that fits one of the two following criteria: 
1. the file is executable and has 'detect' in its name
2. the filename includes '.rs' and the file contents include the string 'map_frame'

While you can still express this query using `find`, due to the inherent constraints of the tools involved doing so is nontrivially complex.

```bash
{
  # files executable by the current user, with "detect" in the filename
  find . -type f -perm -u=x -name '*detect*'

  # files with a ".rs" extension, containing the string "map_frame"
  find . -type f -name "*.rs" -exec grep --files-with-matches map_frame '{}' ';'
} |
  # remove duplicate entries
  sort -u
```

(credit for this impressive bash hack: [Josh Cheek](https://twitter.com/josh_cheek/status/1582178039907069952))

## Detect

That's why I wrote `detect`. It's like `find`, but instead of flags it uses an expression language with a series of predicates. 

```bash
detect 'filename(detect) && executable() || filename(.rs) && contains(map_frame)'
```

Since some matching criteria are cheap and some are expensive, the `detect` tools is clever enough to run them in multiple stages - first filename criteria, then metadata criteria, and only then criteria that require looking at the full file contents. At each stage, it attempts to short circuit before running the next, more expensive, stage.

For the expression above, here are some visualizations of `detect` , as run against three different files in its own [repository](https://github.com/inanna-malick/detect):


`README.md`: short circuits as `false` using only filename matcher

{{< figure src="/img/recursion_crate/detect_example_readme.gif" alt="visualization showing evaluation of simple boolean expression" position="center" style="border-radius: 8px;" >}}


`target/debug/detect`: short circuits as `true` after reading metadata:

{{< figure src="/img/recursion_crate/detect_example_target_detect.gif" alt="visualization showing evaluation of simple boolean expression" position="center" style="border-radius: 8px;" >}}


`src/expr/frame.rs`: evaluates to `true` after reading metadata and file contents

{{< figure src="/img/recursion_crate/detect_example_expr_frame.gif" alt="visualization showing evaluation of simple boolean expression" position="center" style="border-radius: 8px;" >}}

These GIFs show the process of running partial evaluation with short circuiting. Now we turn our eye to the implementation of this functionality.

# Predicates

We evaluate expressions in three phases, and we have three types of predicate: Name, Metadata, and Content:

```rust
enum Predicate<Name, Metadata, Content> {
    Name(Name),
    Metadata(Metadata),
    Content(Content),
}
```

The type parameters let us change the type of a `Predicate` to indicate what stage we're on. As stages are eliminated we will use `Done` - an uninhabited type - to mark those branches as eliminated.

```rust
enum Done {}
```

Because `Done` is an enum with zero branches, it cannot exist at runtime. That means any branch of an enum containing it cannot exist either. For example, a `Predicate<Done, Done, Content>` can _only_ contain `Predicate::Content` branches, because all the other branches are marked as uninhabited.

We will evaluate predicates by eliminating predicate type parameters to mark the evaluation of the corresponding phase, short circuiting where evaluation is possible:

```rust
enum ShortCircuit<A> {
  Unknown<A>,
  Known(bool),
}


impl<A, B> Predicate<NamePredicate, A, B> {
    fn eval_name_predicate(self, filename: String)
        -> ShortCircuit<Predicate<Done, A, B>>  {}
}

impl<A> Predicate<Done, MetadataPredicate, B> {
    fn eval_metadata_predicate(self, metadata: Metadata)
        -> ShortCircuit<Predicate<Done, Done, B>>  {}
}

impl Predicate<Done, Done, ContentPredicate> {
    fn eval_content_predicate(self, contents: String)
        -> ShortCircuit<Predicate<Done, Done, Done>>  {}
}
```

At each stage, one type of predicate is eliminated as a possibility until none are left.

# The expression language

The structure of the expression language is the same throughout each phase, with only the predicate type changing as we step through each phase.

```rust
enum Expr<Predicate> {
    And(Box<Expr>, Box<Expr>),
    Or(Box<Expr>, Box<Expr>),
    Not(Box<Expr>),
    Literal(bool),
    Predicate(Predicate),
}
```

The core of `detect` is a function called `reduce_and_short_circuit` that shrinks the set of available predicates, short circuiting where possible and otherwise substituting in the predicate type for the next phase of evaluation.

```rust
// apply some potentially short circuiting transformation to all predicates in
// this expression, with the goal of eventually eliminating all predicate cases
fn reduce_and_short_circuit<A, B>(
	&e: Expr<A>,
	eval_predicate: impl Fn(A) -> ShortCircuit<B>,
) -> Expr<B> { /* etc */ }
```

Let's see what the `reduce_and_short_circuit` function looks like, with and without the recursion crate.
## The old ways

The traditional way to write this function is both hard to read and can cause a stack overflow if an expression is too deeply nested. Feel free to let your eyes slide over it - the nested matches, the `&**` invocations, the verbose recursive calls - terrible. We can do better.

```rust
pub fn reduce_and_short_circuit<A, B>(
	&e: Expr<A>,
	eval_predicate: impl Fn(A) -> ShortCircuit<B>,
) -> Expr<B> {
	match self {
		// Literal expressions are unchanged
		Expr::Literal(x) => Expr::Literal(*x),
		// apply 'f' to Predicate expressions
		Expr::Predicate(p) => match eval_predicate(p.clone()) {
			ShortCircuit::Known(bool) => Expr::Literal(bool),
			ShortCircuit::Unknown(p) => Expr::Predicate(p),
		},
		// reduce And expressions
		Expr::And(a, b) => match (&**a, &**b) {
			(Expr::Literal(false), _) => Expr::Literal(false),
			(_, Expr::Literal(false)) => Expr::Literal(false),
			(x, Expr::Literal(true)) => x.reduce_and_short_circuit(eval_predicate),
			(Expr::Literal(true), x) => x.reduce_and_short_circuit(eval_predicate),
			(a, b) => Expr::And(
				Box::new(a.reduce_and_short_circuit(&eval_predicate)),
				Box::new(b.reduce_and_short_circuit(&eval_predicate)),
			),
		},
        /* ...and so on for Or and Not */
	}
}
```

## A shiny new future

With the recursion crate, this can be simplified dramatically. The idioms used below (`collapse_frames`, the `ExprFrame` type) will be explained in this blog post, but you should be able to understand approximately what's going on here by just skimming the function definition:

```rust
pub fn reduce_and_short_circuit<A, B>(
	&e: Expr<A>,
	eval_predicate: impl Fn(A) -> ShortCircuit<B>,
) -> Expr<B> {
	self.collapse_frames(|e| match e {
		// Literal expressions are unchanged
		ExprFrame::Literal(x) => Expr::Literal(x),
		// apply 'f' to Predicate expressions
		ExprFrame::Predicate(p) => match eval_predicate(p) {
			ShortCircuit::Known(b) => Expr::Literal(b),
			ShortCircuit::Unknown(p) => Expr::Predicate(p),
		},
		// reduce And expressions
		ExprFrame::And(Expr::Literal(false), _) => Expr::Literal(false),
		ExprFrame::And(_, Expr::Literal(false)) => Expr::Literal(false),
		ExprFrame::And(x, Expr::Literal(true)) => x,
		ExprFrame::And(Expr::Literal(true), x) => x,
		ExprFrame::And(a, b) => Expr::And(Box::new(a), Box::new(b)),
        /* ...and so on for Or and Not */
	})
}
```

Note that this is not just , it also more concise and easier to read, it _does not use the call stack_. The `collapse_frames` function uses an internal heap-allocated stack to manage the bookkeeping of recursion, and thus is not susceptible to call stack overflows.

# How?

Let's start with a simplified `Expr` type with no predicate param:

```rust
enum Expr {
    And(Box<Expr>, Box<Expr>),
    Or(Box<Expr>, Box<Expr>),
    Not(Box<Expr>),
    Literal(bool),
}
```

For working with our `Expr` type we'll define a _frame_ type `ExprFrame<A>`. It's exactly the same as `Expr`, except the recursive self-reference `Box<Self>` is replaced with `A`. This may be a bit confusing at first, but this idiom unlocks a lot of potential (expressiveness, stack safety, etc). You can think of `ExprFrame` as representing a single _stack frame_ in a recursive algorithm.

```rust
enum ExprFrame<A> {
    And(A, A),
    Or(A, A),
    Not(A),
    Literal(bool),
}
```

Now we define the relationship between `Expr` and `ExprFrame`. This requires us to implement two traits: `MappableFrame` and `Collapsible`. The actual implementation of these traits is fairly mechanical (and the details are discussed at more length in the docs), so feel free to skim:

```rust
impl MappableFrame for ExprFrame<PartiallyApplied> {
    type Frame<X> = ExprFrame<X>;
    
    fn map_frame<A, B>(input: Self::Frame<A>, mut f: impl FnMut(A) -> B) -> Self::Frame<B> {
        match input {
            ExprFrame::And(a, b) => ExprFrame::And(f(a), f(b)),
            ExprFrame::Or(a, b) => ExprFrame::Or(f(a), f(b)),
            ExprFrame::Not(a) => ExprFrame::Not(f(a)),
            ExprFrame::Literal(x) => ExprFrame::Literal(x),
        }
    }
}

impl<'a> Collapsible for &'a Expr {
    type FrameToken = ExprFrame<PartiallyApplied>;
    
    fn into_frame(self) -> <Self::FrameToken as MappableFrame>::Frame<Self> {
        match self {
            Expr::And(a, b) => ExprFrame::And(a, b),
            Expr::Or(a, b) => ExprFrame::Or(a, b),
            Expr::Not(a) => ExprFrame::Not(a),
            Expr::Literal(x) => ExprFrame::Literal(*x),
        }
    }
}
```


Generally speaking, you won't use these traits yourself - they're used by the internals of the recursion crate. The recursion crate docs contain full explanations of these traits, and how to implement them.

# Writing Recursive Algorithms

The `Collapsible` trait provides the ability to write recursive algorithms that _collapse_ some structure into a single value, frame by frame. Let's look at an examples:

### Evaluating simple expressions

Here's what evaluating an `Expr` with `collapse_frames` looks like:

```rust
fn eval(e: &Expr) -> bool {  
    e.collapse_frames(|frame| match frame {
        ExprFrame::And(a, b) => a && b,
        ExprFrame::Or(a, b) => a || b,
        ExprFrame::Not(a) => !a,
        ExprFrame::Literal(x) => x,
    })
}
```

Note that the function passed to `collapse_frames` only collapses a single frame of type `ExprFrame<bool>` - `collapse_frames` handles the rest.

Here's a GIF showing each step in the evaluation of `false && true || true` using this function:

{{< figure src="/img/recursion_crate/eval_simple.gif" alt="visualization showing evaluation of simple boolean expression" position="center" style="border-radius: 8px;" >}}

# Conclusions

## Use Recursion

The recursion crate is a perfect fit for working with recursive data structures. We've seen how it can used to implement a lazily evaluated expression language, and how expression languages can be a powerful tool for expressing arbitrary logic (for example, [test filtering expressions in nextest](https://github.com/nextest-rs/nextest/tree/main/nextest-filtering) are implemented using the recursion crate). Next time you're developing a tool, consider providing users with an expression language - it's easier than you might think.

The crate is [here](https://crates.io/crates/recursion), docs are [here](https://docs.rs/recursion/0.5.1/recursion/), the github repository is [here](https://github.com/inanna-malick/recursion).

## Use Detect

Detect is on [crates.io](https://crates.io/crates/detect_rs). The most recent version of it supports multiple predicate stages (filename, file metadata, file contents, and even arbitrary subprocesses), and runs evaluation in multiple stages to minimize syscall use.

```shell
➜  cargo install detect_rs
➜  detect 'filename(detect) && executable() || filename(.rs) && contains(map_frame)'
...
```


## Generating Visualizations

All GIFs in this post were generated by a special _instrumented_ version of the recursive machinery used by `collapse_frames`  that automatically generates animated 'stack traces'. Source code for this is [here](https://github.com/inanna-malick/detect/blob/94bcff1593c685dfee019c5caffbff6fa7ad6477/src/eval.rs).

This is a great example of the kind of thing that you can do once you've separated the _logic_ of recursion from the _machinery_.
