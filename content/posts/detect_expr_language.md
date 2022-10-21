+++
title = "Expression languages in Rust"
date = "2022-10-20"
author = "Inanna Malick"
authorTwitter = "inanna_malick"
tags = ["recursion schemes", "rust", "code"]
keywords = ["recursion schemes", "rust", "code"]
showFullContent = false
images = ["/img/rust_schemes/stack_machines_1/simple_expr_eval.gif"]
feature = "/img/rust_schemes/stack_machines_1/simple_expr_eval.gif"
thumbnail = "/img/rust_schemes/stack_machines_1/simple_expr_eval.gif"
+++


- Blog title: "expression languages in rust ", twitter post with 4x gif of my find tool collapsing a nontrivial expr, preceded by 1x screenshot of shell examination of file via ls/cat
	- subheading: concise, performant, easy
Rust has been used to implement many fast modern command line tools with low overhead, quick start times, and extremely high performance. Most of these tools use command line flags to express query and filtering rules. General purpose expression languages are traditionally seen as nice to have but hard to implement, and are thus not common in the Rust ecosystem. I think it would be nice to change that. Expression languages are useful tools for expressing complex logic, and I think they're worth adding.

This blog post shows how to implement a nontrivially complex expression language for filtering filesystem entries: it's similar to the `fd` and `find` tools, but instead of command line flags it uses expressions like `extension(.rs) && contents(eval) || filesize(0..1024) && executable()`.

<!--more--> 

This is the 4th post in a series, but it's intended to be an entry point to motivate reading the previous posts: if you haven't read them, some patterns may be unfamiliar, but I've provided visualizations of each stage so you can follow along.

TODO: gif here

# A file matching expression language


This tool is a proof of concept and does not provide feature parity with `find` or `fd`, but it does represent a serious attempt to demonstrate how such a tool might look. It is written to avoid stack overflows and to minimize syscalls. Expressions are evaluated in multiple stages, using boolean operator short circuiting wherever possible.

## An example expression

Here's an example of the kind of expression it takes as input: `extension(.rs) && contents(eval) || filesize(0..1024) && executable()`. It has 4 different predicates and 2 different operators. Here are the predicates, in order of evaluation cost:
- `extension(.rs)`is a name predicate. It examines a file's name. This is pretty much free, because we start with the file's path.
- `filesize(0..1024)` and  `executable()` are metadata predicates. Both of these are a bit more expensive, because they require reading a file's metadata using the path.
- Finally, we have the `contents(eval)` predicate. It's a contents predicate, meaning it runs against the full on-disc contents of a file. These are the most expensive, because they require reading the full contents of a file from the filesystem.

We would like to evaluate the least expensive predicates first, and to skip evaluation of the more expensive predicates wherever possible. Here's how: boolean operators allow for short circuiting.
- The Or operator, `||`, evaluates to true if _either side_ is true. 
- The And operator, `&&`, evaluates to false if _either side_ is false.

## Evaluating expressions

Let's see what this looks like in practice. First, imagine we're evaluating this expression for a small executable named `my_executable`. It's very small, only 64 bytes.
- First, we start with`extension(.rs) && contents(eval) || filesize(0..1024) && executable() ` , the full expression.
- First we evaluate the name predicate, `extension(.rs)`: the filename `my_executable` doesn't have the .rs extension, so it evaluates to `false`.  That false can be substituted in, resulting in `false && contents(eval) || filesize(0..1024) && executable()` which can then be collapsed down to `false || filesize(0..1024) && executable()` via the short circuiting rule for `&&`.
- We still don't have a boolean result, so we read the metadata and evaluate the metadata predicates `filesize(0..1024` and `executable()`. Doing so results in `false || true && true` . Since every remaining predicate in the expression has been evaluated, we can return `true` and entirely skip looking at the contents of the file.

Another example: this time we're looking at a rust file, `eval.rs`, that has contents `fn() eval { todo!() }`.
- As before, we start with the full expression: `extension(.rs) && contents(eval) || filesize(0..1024) && executable() `
- First we evaluate the name predicate, `extension(.rs)`: the filename `eval.rs` has the .rs extension, so it evaluates to `true`.  That  is then substituted in, resulting in `true && contents(eval) || filesize(0..1024) && executable()`.
- We still don't have a boolean result for the expression, so next we read the metadata and evaluate the metadata predicates `filesize(0..1024` and `executable()`. This results in `true && contents(eval) || true && false`. This is then short circuited to `true && contents(eval) || false` via the short circuiting rule for `&&`
- We still don't have a boolean result, so we run the last and most expensive predicate: `contents(eval)`. Since the file contains the substring `eval`, it evaluates to `true`. Substituting that result in, we get `true && true || false`. This then evaluates to `true`, which is returned.

These examples illustrate how short circuiting works, and how predicates are run in multiple stages. Now we can move on to the implementation:

## Predicate implementation

Here's our predicates:
```rust

pub enum NamePredicate {
	Regex(Regex),
	Extension(String),
}

pub enum MetadataPredicate {
	Filesize(Range<u64>),
	Executable(),
}
pub enum ContentPredicate {
	Regex(Regex),
	Utf8,
}
```

They each implement an `is_match` function.
```rust
impl NamePredicate {
    pub fn is_match(&self, path: &Path) -> bool {...}
}

impl MetadataPredicate {
    pub fn is_match(&self, metadata: &Metadata) -> bool {...}
}

impl ContentPredicate {
    pub fn is_match(&self, utf8_contents: &str) -> bool {...}
}
```

The specific implementation of each `is_match` function isn't important, for this post we just need to know that each predicate type requires a specific input for evaluation.



## Expression language implementation

Here's our expression language written as a Rust enum. Note that it's written in a simple and idiomatic style, without resorting to galaxy-brained haskell idioms. You can include this in the API surface of a library, write instances of it in test code, and pattern match on it. It does not lock you into any nonstandard idiom.


```rust
pub enum Expr<Name, Metadata, Content> {
    // literal boolean values
    KnownResult(bool),
    // boolean operators
    Not(Box<Self>),
    And(Vec<Self>),
    Or(Vec<Self>),
    // predicates
    Name(Name),
    Metadata(Metadata),
    Contents(Content),
}
```

This expression language is parameterized over each predicate type, so (as we evaluate each predicate) we can mark it as done using `Done` - an uninhabited enum used to signify that some predicate type has already been evaluated:

```rust
pub enum Done{}
```

It's impossible to construct an instance of `Done`, so it provides a type-level assertion that we've already evaluated some part of an expression.

## sidebar: ownership model

- at the top level, we'll be working with an expression that _owns_ its predicates
- as we step through the multiple phases of evaluating expressions, we'll be creating and collapsing a series of short-lived expressions that _borrow_ the predicates. This lets us minimize cloning regexes.

 ```rust
 /// A filesystem entity matcher expression that owns its predicates
pub type OwnedExpr<Name = NamePredicate, Metadata = MetadataPredicate, Content = ContentPredicate> =
    Expr<Name, Metadata, Content>;

/// A filesystem entity matcher expression with borrowed predicates
pub type BorrowedExpr<
    'a,
    Name = &'a NamePredicate,
    Metadata = &'a MetadataPredicate,
    Content = &'a ContentPredicate,
> = Expr<Name, Metadata, Content>;
```

# Evaluating Expressions

For evaluation, we're going to use `ExprLayer` which mirrors the structure of `Expr` . You can think of it as a purpose-specific stack frame for our expression language. 

If you haven't read the earlier blog posts in this series, this pattern is probably a bit unfamiliar. Don't worry too much, you should be able to get the general idea of how things work by skimming the code and viewing the animated visualizations of each stage.

```rust
pub enum ExprLayer<
    'a,
    Recurse,
    Name = NamePredicate,
    Metadata = MetadataPredicate,
    Contents = ContentPredicate,
> {
    // boolean literals
    KnownResult(bool),
    // boolean operators
    Operator(Operator<Recurse>),
    // borrowed predicates
    Name(&'a Name),
    Metadata(&'a Metadata),
    Contents(&'a Contents),
}

pub enum Operator<Recurse> {
    And(Vec<Recurse>),
    Or(Vec<Recurse>),
}

```

Defining `Operator` as its own enum lets us factor out boolean short circuiting logic as the `attempt_short_circuit` function. The rules are:
- an And operator
  - returns true if all elements are true
  - returns false if _any_ element is false
- an Or operator
  - returns true if any element is true
  - returns false if _all_ elements are false
- a Not operator
  - returns false if its single element is true, and true if its single element is false


This is especially desirable because the actual implementation is a bit verbose. It's sufficient to think of it as implementing the short circuiting rules for And and Or, but the implementation is been provided below if you're curious:.

(minimized)
<details>
      <summary>Click to expand operator short circuiting</summary>

```rust
impl<A, B, C> Operator<Expr<A, B, C>> {
    pub fn attempt_short_circuit(self) -> Expr<A, B, C> {
        use Expr::*;
        match self {
            Operator::And(ands) => {
                if ands.iter().any(|b| matches!(b, KnownResult(false))) {
                    KnownResult(false)
                } else if ands.iter().all(|b| matches!(b, KnownResult(true))) {
                    KnownResult(true)
                } else {
                    And(ands)
                }
            }
            Operator::Or(ors) => {
                if ors.iter().any(|b| matches!(b, KnownResult(true))) {
                    KnownResult(true)
                } else if ors.iter().all(|b| matches!(b, KnownResult(false))) {
                    KnownResult(false)
                } else {
                    Or(ors)
                }
            }
            Operator::Not(x) => match x {
                KnownResult(b) => KnownResult(!b),
                _ => Not(Box::new(x)),
            },
        }
    }
}
```
</details>

## machinery of recursion sidebar: boilerplate

We're going to _project_ our boxed expression type into `ExprLayer` layers - think of it as a purpose-specific stack frame, with the `project` function handling traversing the boxed expression type `Expr` and generating our stack frames.
- this lets us write _collapse_layer_ directly over `Expr` instances - as mentioned before, there's no need to use any idosyncratic idioms.
- These concepts (except for `project`, which just factors out the boilerplate) are introduced in the previous posts in this series

```rust
impl<'a, S1: 'a, S2: 'a, S3: 'a> Project for &'a Expr<S1, S2, S3> {
    type To = ExprLayer<'a, Self, S1, S2, S3>;

    // project into ExprLayer
    fn project(self) -> Self::To {
        match self {
            Expr::Not(x) => ExprLayer::Operator(Operator::Not(x)),
            Expr::And(xs) => ExprLayer::Operator(Operator::And(xs.iter().collect())),
            Expr::Or(xs) => ExprLayer::Operator(Operator::Or(xs.iter().collect())),
            Expr::KnownResult(b) => ExprLayer::KnownResult(*b),
            Expr::Name(n) => ExprLayer::Name(n),
            Expr::Metadata(m) => ExprLayer::Metadata(m),
            Expr::Contents(c) => ExprLayer::Contents(c),
        }
    }
}
```

TODO: link to previous/next posts in each post of said series (modify previous posts to include project, I think - mb, would also require dropping fixed point bullshit)


## The eval function

Now that we've defined our expression type, `Expr`, and our short-lived `ExprLayer` type for expressing recursive algorithms, we can move on to the `eval` function

We start with an `OwnedExpr` and a `Path`:

```rust
pub fn eval(e: &OwnedExpr, path: &Path) -> std::io::Result<bool> {
```

## Matching on File Names

Then, we evaluate the first phase: `NamePredicate`, substituting each for a `KnownResult`. Since we remove all `Name` branches from the expression, we can use `Done` to mark that stage as being completely evaluated.

```rust
    let e: BorrowedExpr<Done> = e.collapse_layers(|layer| {
        match layer {
            // evaluate all NamePredicate predicates
            ExprLayer::Name(p) => Expr::KnownResult(p.is_match(path)),
            // boilerplate
            ExprLayer::Operator(op) => op.attempt_short_circuit(),
            ExprLayer::KnownResult(k) => Expr::KnownResult(k),
            ExprLayer::Metadata(p) => Expr::Metadata(p),
            ExprLayer::Contents(p) => Expr::Contents(p),
        }
    });
```

This pattern might be a bit unfamiliar, but I quite like it: each layer is collapsed, one at a time. Here's some GIFs showing what evaluating this phase looks like for our `extension(.rs) && contents(eval) || filesize(0..1024) && executable() ` expression.

For the `my_executable` file (size 64 bytes, executable):


For the `eval.rs` (contents `fn() eval { todo!() }`):



After evaluating the `NamePredicate` predicates and short circuiting the operators, we attempt to short circuit:

```rust
    if let Expr::KnownResult(b) = e {
        return Ok(b);
    }
```

Since we've constructed an `Expr`, we can just pattern match - if we have a `KnownResult`, evaluation is complete. If not, as with both of the above examples, we proceed to the next phase: reading the file metadata.

## Matching on File Metadata

```rust
    let metadata = fs::metadata(path)?;

    let e: BorrowedExpr<Done, Done> = e.collapse_layers(|layer| {
        match layer {
            // evaluate all MetadataPredicate predicates
            ExprLayer::Metadata(p) => Expr::KnownResult(p.is_match(&metadata)),
            // boilerplate
            ExprLayer::Operator(op) => op.attempt_short_circuit(),
            ExprLayer::KnownResult(k) => Expr::KnownResult(k),
            ExprLayer::Contents(p) => Expr::Contents(*p),
            // unreachable: predicate already evaluated
            ExprLayer::Name(_) =>
                unreachable!("name predicate has already been evaluated"),
        }
    });
```

We know we _cannot_ have `Name` branches for our `Expr` because it's marked as `Done`, so it's marked as unreachable using the `unreachable!()` macro. Since `collapse_layers` evaluates all the `Metadata` branches, that branch is also marked as `Done` in the resulting `BorrowedExpr`.

Here's some GIFs showing what evaluating this phase looks like for our `extension(.rs) && contents(eval) || filesize(0..1024) && executable() ` expression.

For the `my_executable` file (size 64 bytes, executable):

For , the`eval.rs` (contents `fn() eval { todo!() }`):

Once again, we attempt to short circuit:

```rust
    if let Expr::KnownResult(b) = e {
        return Ok(b);
    }
```

For the `my_executable` file, we have a single boolean value - a known result - `true`. Short circuiting succeeds and the `eval` function returns `true`. For the `eval.rs` file, we don't yet have a known result, so we continue to the next phase: reading the file contents.

## Matching on File Contents

```rust
    let utf8_contents = if metadata.is_file() {
        // read file contents via multiple syscalls
        let contents = fs::read(path)?;
        String::from_utf8(contents).ok()
    } else {
        None
    };

    let e: BorrowedExpr<Done, Done, Done> = e.collapse_layers(|layer| {
        match layer {
            // evaluate all ContentPredicate predicates
            ExprLayer::Contents(p) => {
                // only examine contents if we have a valid utf8 contents string
                let is_match = utf8_contents.as_ref()
                             .map_or(false, |s| p.is_match(s));
                Expr::KnownResult(is_match)
            }
            // boilerplate
            ExprLayer::Operator(op) => op.attempt_short_circuit(),
            ExprLayer::KnownResult(k) => Expr::KnownResult(k),
            // unreachable: predicates already evaluated
            ExprLayer::Name(_) =>
                unreachable!("name predicate has already been evaluated"),
            ExprLayer::Metadata(_) =>
                unreachable!("metadata predicate has already been evaluated"),
        }
    });
```

In this phase, we _cannot_ have `Name` _or_ `Metadata` branches for  our `Expr` because both branches are marked as `Done`. All that's left to do is to evaluate the `Contents` branch, and to short circuit any remaining operators. After `collapse_layers` completes, that too is marked as `Done` in the resulting `BorrowedExpr`.

Here's a GIF showing what evaluating this phase looks like for our `extension(.rs) && contents(eval) || filesize(0..1024) && executable() ` expression, for our `eval.rs` 
 file (contents `fn() eval { todo!() }`):

Here, finally, we 've evaluated all three predicate types and short circuited all of the boolean operators. At this point we _know_ we have a known result, and the function completes

```rust
    if let Expr::KnownResult(b) = e {
        Ok(b)
    } else {
        panic!("programmer error, unknown result but all predicates evaluated")
    }
```

That's it! This may have been a bit conceptually dense, but the function implementing this three-phase evaluation uses  _only 70 lines of code_ (and less than 50 lines if you don't count whitespace and comments). The operator short circuiting code has been factored out, and can thus be tested separately. The `is_match` predicate evaluation functions have also been factored out and can also be tested in isolation.

I think this is genuinely pretty neat - not just the fact of having performant stack safe recursion in rust, but also this idiom of expressing recursive logic. Complex recursive algorithms can be expressed concisely, in Rust. Just as concisely as Haskell, but with full control over memory management.

# A Review

SOME NOTES HERE


This kind of recursive algorithm is a special interest of mine, and I will happily volunteer time to help you implement expression languages in your project. Please reach out and say hi if you're interested! (todo: contact info on blog!)