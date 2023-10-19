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

The [recursion crate](https://crates.io/crates/recursion) provides abstractions for separating the _machinery_ of recursion from the _logic_ of recursion. This is similar to how iterators separate the _machinery_ of iteration from the _logic_ of iteration, allowing us to replace  {{< highlight c "hl_inline=true">}} for (int i = 0; i < 10; ++i) {} {{< /highlight >}} with {{< highlight rust "hl_inline=true">}}  for item in items.iter() {}{{< /highlight >}} 

This post doesn't attempt to explain every feature of the recursion crate (that's what the [cargo docs](https://docs.rs/recursion/0.5.1/recursion/) are for), but instead aims to spark your curiosity and to motivate you to use the tools provided by the recursion crate.

<!--more--> 


# Motivation: a file matching tool


I often use the `find` tool to find files matching some set of criteria - name, metadata, file contents, that kind of thing. Almost every time I use it for something nontrivial, I end up having to google the right combination of command line flags to express my query. For example, let's say we want to list all files that fit one of the two following criteria: 
1. the file is executable and has 'detect' in its name
2. the filename includes '.rs' and the file contents include the string 'map_frame'

It's important to note that it _is_ possible to express this query using `find`, but due to the inherent constraints of the tools involved doing so is nontrivially complex and a bit hard to read.

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

That's why I wrote `detect`. It's like `find`, but instead of flags it uses an expression language with a series of predicates. Since some matching criteria are cheap and some are expensive, it runs in multiple stages - first filename criteria, then metadata criteria, and only then criteria that require looking at the full file contents.

```bash
detect 'filename(detect) && executable() || filename(.rs) && contains(map_frame)'
```


Here's a visualization of `detect` running this expression against three different files in its own repository (link):

## README.md

short circuits as `false` using only filename matcher

{{< figure src="/img/recursion_crate/detect_example_readme.gif" alt="visualization showing evaluation of simple boolean expression" position="center" style="border-radius: 8px;" >}}

## target/debug/detect

short circuits as `true` after reading metadata:

{{< figure src="/img/recursion_crate/detect_example_target_detect.gif" alt="visualization showing evaluation of simple boolean expression" position="center" style="border-radius: 8px;" >}}

## src/expr/frame.rs

evaluates to `true` after reading metadata and file contents

{{< figure src="/img/recursion_crate/detect_example_expr_frame.gif" alt="visualization showing evaluation of simple boolean expression" position="center" style="border-radius: 8px;" >}}

These GIFs show the process of running partial evaluation with short circuiting. Now we turn our eye to the implementation of this functionality.
# Why the Recursion Crate

 The `recursion` crate is a great fit for problems involving expression languages. That's why it was used it to implement `detect`, and why it's used to implement nextest's test filtering expression language, as implemented in [nextest-filtering](https://github.com/nextest-rs/nextest/tree/main/nextest-filtering).
## An expression language

The structure of the expression language is the same throughout each phase, with only the type representing still-unevaluated predicates changing as we step through the phases.

```rust
enum Expr<Predicate> {
    And(Box<Expr>, Box<Expr>),
    Or(Box<Expr>, Box<Expr>),
    Not(Box<Expr>),
    Literal(bool),
    Predicate(Predicate),
}
```

The core of `detect` is a function called `reduce_and_short_circuit` that shrinks the set of available predicates, short circuiting where possible and substituting in some new predicate type where not.

```rust
// apply some potentially short circuiting transformation to all predicates in
// this expression, with the goal of eventually eliminating all predicate cases
fn reduce_and_short_circuit<A, B>(
	&e: Expr<A>,
	f: impl Fn(A) -> ShortCircuit<B>,
) -> Expr<B> { /* etc */ }


enum ShortCircuit<A> {
  Unknown<A>,
  Known(bool),
}
```

Let's see what the `reduce_and_short_circuit` function looks like, with and without the recursion crate.
## the old ways

The traditional way to write this function is both hard to read and can cause a stack overflow if an expression is too deeply nested. Feel free to let your eyes slide over it - the nested matches, the `&**` invocations, the verbose recursive calls - terrible. We can do better.

```rust
pub fn reduce_and_short_circuit<A, B>(
	&e: Expr<A>,
	f: impl Fn(A) -> ShortCircuit<B>,
) -> Expr<B> {
	match self {
		// Literal expressions are unchanged
		Expr::Literal(x) => Expr::Literal(*x),
		// apply 'f' to Predicate expressions
		Expr::Predicate(p) => match f(p.clone()) {
			ShortCircuit::Known(bool) => Expr::Literal(bool),
			ShortCircuit::Unknown(p) => Expr::Predicate(p),
		},
		// reduce And expressions
		Expr::And(a, b) => match (&**a, &**b) {
			(Expr::Literal(false), _) => Expr::Literal(false),
			(_, Expr::Literal(false)) => Expr::Literal(false),
			(x, Expr::Literal(true)) => x.reduce_and_short_circuit(f),
			(Expr::Literal(true), x) => x.reduce_and_short_circuit(f),
			(a, b) => Expr::And(
				Box::new(a.reduce_and_short_circuit(&f)),
				Box::new(b.reduce_and_short_circuit(&f)),
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
	f: impl Fn(A) -> ShortCircuit<B>,
) -> Expr<B> {
	self.collapse_frames(|e| match e {
		// Literal expressions are unchanged
		ExprFrame::Literal(x) => Expr::Literal(x),
		// apply 'f' to Predicate expressions
		ExprFrame::Predicate(p) => match f(p) {
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

Note that this is not just less verbose, it also _does not use the call stack_. The `collapse_frames` function uses an internal heap-allocated stack to manage the bookkeeping of recursion, and thus is not susceptible to call stack overflows.

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

The `Collapsible` trait provides the ability to write recursive algorithms that _collapse_ some structure into a single value, frame by frame. Let's look at some examples:

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

Here's a GIF showing each stage of the evaluation of `false && true || true` using this function

{{< figure src="/img/recursion_crate/eval_simple.gif" alt="visualization showing evaluation of simple boolean expression" position="center" style="border-radius: 8px;" >}}

This GIF was generated by a special _instrumented_ version of the recursive machinery used by `collapse_frames`  that automatically generates animated 'stack traces'u.

## Link to full impl

You can find a frozen version of the detect crate implementing the functionality described in this crate [here]("img/recursion_crate/detect_example_expr_frame.gif"])


link to frozen version of detect w/ only the fn from the intro, no viz, etc.
https://github.com/inanna-malick/detect/blob/version_for_blog_post/src/expr.rs#L32-L60

link to main branch of detect with etc etc etc
# The Detect crate

The `reduce_predicate_and_short_circuit` function is taken from my `detect` crate, which provides tools for finding files on the filesystem using an expression language matcher. It supports multiple predicate stages (filename, file metadata, file contents, arbitrary subprocesses), and runs evaluation in multiple stages to minimize syscall use.

## Another example

The most recent version of `detect` also supports spawning subprocesses - for fun, I tried using it with `file` to look for ASCII files, and I used short circuiting to ensure the subprocess is only run on X