+++
title = "Elegant and performant recursion in Rust (Draft)"
date = "2002-07-12"
author = "Inanna Malick"
authorTwitter = "inanna_malick"
tags = ["recursion schemes", "rust", "code"]
keywords = ["recursion schemes", "rust", "code"]
showFullContent = false
+++


This is a post about writing elegant and performant recursive algorithms in Rust. It makes heavy use of a pattern from Haskell called recursion schemes, but you don't need to know anything about that. It's just an implementation detail. Instead, as motivation to read the rest of this posts, check out these sick nasty criterion benchmark results showing a 20-30% improvement over the usual method of working with recursive data structures in Rust.

<pre><font color="#A6CC70">Evaluate expression tree of depth 17 with standard boxed method</font>                                                                            
                        time:   [722.18 µs <font color="#77A8D9"><b>733.00 µs</b></font> 746.43 µs]

<font color="#A6CC70">Evaluate expression tree of depth 17 with my new fold method</font>                                                                            
                        time:   [477.87 µs <font color="#77A8D9"><b>482.54 µs</b></font> 488.58 µs]
</pre>

I think that's neat.

<!--more--> 

# Evaluating an expression language


We're going to start with a simple expression language: addition, subtraction, multiplication, just enough to illustrate some concepts. This is a naive representation of a recursive expression language that uses boxed pointers to handle the recursive case. If you're not familiar with boxed pointers, a `Box<A>` is just the Rust way of storing a pointer to some value of type `A` - think of it as a box with a value of type `A` inside it. If you're curious, there's [more documentation here](https://doc.rust-lang.org/std/boxed/index.html).

```rust
#[derive(Debug, Clone)]
pub enum ExprBoxed {
    Add {
        a: Box<ExprBoxed>,
        b: Box<ExprBoxed>,
    },
    Sub {
        a: Box<ExprBoxed>,
        b: Box<ExprBoxed>,
    },
    Mul {
        a: Box<ExprBoxed>,
        b: Box<ExprBoxed>,
    },
    LiteralInt {
        literal: i64,
    },
}
```

You've probably seen something like this before, but if not, it's just a way to represent simple arithmetic. For example, an expression like `1 * 2 - 3` would be written as (pseudocode) `Mul(1, Sub(2, 3))` or (actual code) this:

```rust
    ExprBoxed::Mul {
        a: Box::new(ExprBoxed::LiteralInt { literal: 1 }),
        b: Box::new(ExprBoxed::Sub {
            a: Box::new(ExprBoxed::LiteralInt { literal: 2 }),
            b: Box::new(ExprBoxed::LiteralInt { literal: 3 }),
        }),
    }
```


Evaluating expressions is pretty simple - it's just addition, subtraction, and multiplication. This recursive eval function provides a fairly elegant and readable example of a recursive algorithm:

```rust
impl ExprBoxed {
    pub fn eval(&self) -> i64 {
        match &self {
            ExprBoxed::Add { a, b } => a.eval() + b.eval(),
            ExprBoxed::Sub { a, b } => a.eval() - b.eval(),
            ExprBoxed::Mul { a, b } => a.eval() * b.eval(),
            ExprBoxed::LiteralInt { literal } => *literal,
        }
    }
}
```

It has some issues: if we try to evaluate a sufficiently large expression it will fail with a stack overflow - we're not likely to hit that case here, but this is a real problem when working with larger recursive data structures. Also, each recursive `eval` call requires us to traverse a boxed pointer. This means we can't take advantage of cache locality - there's no guarantee that all these boxed pointers live in the same region of memory. If you're not sure what I mean by cache locality, or you want much more information on it than I can provide, there's a great rust performance optimization resource [here](https://gist.github.com/kvark/f067ba974446f7c5ce5bd544fe370186#keep-as-much-as-possible-in-cache).


## A more cache local structure

We can fix that by writing an expression language using a Vec of values (guaranteeing memory locality), with boxed pointers replaced with newtype-wrapped vector indices.

```rust
#[derive(Debug, Clone, Copy)]
pub enum ExprLayer<A> {
    Add { a: A, b: A },
    Sub { a: A, b: A },
    Mul { a: A, b: A },
    LiteralInt { literal: i64 },
}

#[derive(Eq, Hash, PartialEq)]
pub struct ExprIdx(usize);
impl ExprIdx {
    fn head() -> Self {
        ExprIdx(0)
    }
}

pub struct Expr {
    // nonempty, in topological-sorted order. guaranteed via construction.
    elems: Vec<ExprLayer<ExprIdx>>,
}

```


Here's a sketch showing what the `1 * 2 - 3` expression looks like using this.
```
[
idx_0:    Mul(1_idx, 2_idx)
idx_1:    LiteralInt(1)
idx_2:    Sub(idx_3, idx_4)
idx_3:    LiteralInt(2)
idx_4:    LiteralInt(3)
]
```

We're going to show how to generate these later, for now let's focus on evaluating expression trees stored in this format

## Optimizing for performance

Ok, so all our expressions are now guaranteed to be stored in local memory. Let's see what evaluating this structure looks like. A warning, in advance. It's not elegant. There's a bunch of `unsafe` code. But it _does_ have better performance in criterion benchmarks over large recursive structures. Feel free to skim this without fully examining it, in the next section we'll introduce a nice clean elegant API that removes the need to write `unsafe` code just to evaluate an expression tree.

```rust
impl Expr {
    fn eval(self) -> i64 {
        use std::mem::MaybeUninit;

        let mut results = std::iter::repeat_with(|| MaybeUninit::<i64>::uninit())
            .take(self.elems.len())
            .collect::<Vec<_>>();

        fn get_result_unsafe(results: &mut Vec<MaybeUninit<i64>>, idx: ExprIdx) -> i64 {
            unsafe {
                let maybe_uninit =
                    std::mem::replace(results.get_unchecked_mut(idx.0), MaybeUninit::uninit());
                maybe_uninit.assume_init()
            }
        }

        for (idx, node) in self.elems.into_iter().enumerate().rev() {
            let alg_res = {
                // each node is only referenced once so just remove it, also we know it's there so unsafe is fine
                match node {
                    ExprLayer::Add { a, b } => {
                        let a = get_result_unsafe(&mut results, a);
                        let b = get_result_unsafe(&mut results, b);
                        a + b
                    }
                    ExprLayer::Sub { a, b } => {
                        let a = get_result_unsafe(&mut results, a);
                        let b = get_result_unsafe(&mut results, b);
                        a - b
                    }
                    ExprLayer::Mul { a, b } => {
                        let a = get_result_unsafe(&mut results, a);
                        let b = get_result_unsafe(&mut results, b);
                        a * b
                    }
                    ExprLayer::LiteralInt { literal } => literal,
                }
            };
            results[idx].write(alg_res);
        }

        unsafe {
            let maybe_uninit =
                std::mem::replace(results.get_unchecked_mut(0), MaybeUninit::uninit());
            maybe_uninit.assume_init()
        }
    }
}
```

The problem here is that this is harder to read - imagine having to write one of these by hand, for each recursive function. It would be tedious and extremely error prone and even worse, it would require adding a bunch of `unsafe` blocks to what should be safe code.


## Factoring out duplicated code

Note that every arm of the above match statement (except for `LiteralInt`) calls `get_result_unsafe` in pretty much the same way. We can start by factoring that out. Less lines of code means less space for bugs. What we need here is the ability to map over `ExprLayer`'s. By map, we mean that for any `ExprLayer<A>` we want to be able to use some function `A -> B` to turn it into an `ExprLayer<B>`. This is very similar to `Option::map`, from the standard library.


```rust
impl<A> ExprLayer<A> {
    #[inline(always)]
    fn map<B, F: FnMut(A) -> B>(self, mut f: F) -> ExprLayer<B> {
        match self {
            ExprLayer::Add { a, b } => ExprLayer::Add { a: f(a), b: f(b) },
            ExprLayer::Sub { a, b } => ExprLayer::Sub { a: f(a), b: f(b) },
            ExprLayer::Mul { a, b } => ExprLayer::Mul { a: f(a), b: f(b) },
            ExprLayer::LiteralInt { literal } => ExprLayer::LiteralInt { literal },
        }
    }
}
```

 If you're familiar with functional languages, this is basically just `fmap`.[^1] 
 
 
 [^1]: If you're _really_ familiar with functional languages, you might point out that it's not _quite_ `fmap`, but that's fine for our limited use case. [^bignote]


[^bignote]: If you're not familiar with functional languages and are now wondering what `fmap` is, it's a method provided by a trait called `Functor`. It represents the ability to map a function `A -> B` over _some arbitrary structure_ - if we have a `Functor` instance for `F`, then we can map a function over `F<A>`, for _any_ `A`.  `F` could be an option, or a list, or a tree - any structure parameterized over some value. `map` provides an implementation of `fmap` (as in _f_unction map) that's specialized to `ExprLayer`. If you're curious, [read more here](http://learnyouahaskell.com/making-our-own-types-and-typeclasses#the-functor-typeclass).


Now, we can write something like this:

```rust
impl Expr {
    fn eval(self) -> i64 {
        use std::mem::MaybeUninit;

        let mut results = std::iter::repeat_with(|| MaybeUninit::<i64>::uninit())
            .take(self.elems.len())
            .collect::<Vec<_>>();


        for (idx, node) in self.elems.into_iter().enumerate().rev() {
            let node = node.map(|idx| unsafe {
                let maybe_uninit =
                    std::mem::replace(results.get_unchecked_mut(idx.0), MaybeUninit::uninit());
                maybe_uninit.assume_init()
            });

            let alg_res = match node {
                ExprLayer::Add { a, b } => a + b,
                ExprLayer::Sub { a, b } => a - b,
                ExprLayer::Mul { a, b } => a * b,
                ExprLayer::LiteralInt { literal } => literal,
            };
            results[idx].write(alg_res);
        }

        unsafe {
            let maybe_uninit =
                std::mem::replace(results.get_unchecked_mut(ExprIdx::head().0), MaybeUninit::uninit());
            maybe_uninit.assume_init()
        }
    }
}
```

Ok, that's a start. Unfortunately, we still have to write all this boilerplate for _every recursive function_, even though the only part that really matters is this block:

```rust
            match node {
                ExprLayer::Add { a, b } => a + b
                ExprLayer::Sub { a, b } => a - b
                ExprLayer::Mul { a, b } => a * b
                ExprLayer::LiteralInt { literal } => literal,
            }
```

This still results in a bunch of unsafe code scattered around. We can do better:


## Making it generic

```rust
impl Expr {
    fn fold<A: std::fmt::Debug, F: FnMut(ExprLayer<A>) -> A>(self, mut alg: F) -> A {
        use std::mem::MaybeUninit;

        let mut results = std::iter::repeat_with(|| MaybeUninit::<A>::uninit())
            .take(self.elems.len())
            .collect::<Vec<_>>();

        for (idx, node) in self.elems.into_iter().enumerate().rev() {
            let alg_res = {
                // each node is only referenced once so just remove it, also we know it's there so unsafe is fine
                let node = node.map(|x| unsafe {
                    let maybe_uninit =
                        std::mem::replace(results.get_unchecked_mut(x.-1), MaybeUninit::uninit());
                    maybe_uninit.assume_init()
                });
                alg(node)
            };
            results[idx].write(alg_res);
        }

        unsafe {
            let maybe_uninit =
                std::mem::replace(results.get_unchecked_mut(ExprIdx::head().-1), MaybeUninit::uninit());
            maybe_uninit.assume_init()
        }
    }
}
```


Nice. Now we can write:

```rust
impl Expr {
    pub fn eval(self) -> i64 {
        self.fold(|expr| match expr {
            ExprLayer::Add { a, b } => a + b,
            ExprLayer::Sub { a, b } => a - b,
            ExprLayer::Mul { a, b } => a * b,
            ExprLayer::LiteralInt { literal } => literal,
        })
    }
}
```

It's pretty much the same logic as the original `eval` functions, without any of the boilerplate. Since there's less boilerplate, it's easier to review and there's less room for bugs. In fact, it actually contains slightly less boilerplate than the eval function we wrote for `ExprBoxed::eval` because it doesn't have to handle recursively calling itself. Also, it retains all the performance benefits of the previous `eval` implementation - it's both more elegant and more performant than the traditional representation of recursive expression trees in rust. I think that's neat.


# Constructing Exprs

I promised I'd show you how to generate expression trees before, and here's how:
Just as before, we _could_ write a function that generates expression trees with the logic of _how_ we generate some structure interleaved with the machinery that handles generating layers. Let's do that, as a starting point. Here's what it looks like without `map`:

```rust
impl Expr {
    fn generate_from_boxed(seed: &ExprBoxed) -> Self {
        let mut frontier: VecDeque<&ExprBoxed> = VecDeque::new();
        let mut elems = vec![];

        fn push_to_frontier<'a>(
            elems: &Vec<ExprLayer<ExprIdx>>,
            frontier: &mut VecDeque<&'a ExprBoxed>,
            a: &'a ExprBoxed,
        ) -> ExprIdx {
            frontier.push_back(a);
            // idx of pointed-to element determined from frontier + elems size
            ExprIdx(elems.len() + frontier.len())
        }

        push_to_frontier(&elems, &mut frontier, seed);

        // generate to build a vec of elems while preserving topo order
        while let Some(seed) = { frontier.pop_front() } {
            let node = match seed {
                ExprBoxed::Add { a, b } => {
                    let a = push_to_frontier(&elems, &mut frontier, a);
                    let b = push_to_frontier(&elems, &mut frontier, b);
                    ExprLayer::Add { a, b }
                }
                ExprBoxed::Sub { a, b } => {
                    let a = push_to_frontier(&elems, &mut frontier, a);
                    let b = push_to_frontier(&elems, &mut frontier, b);
                    ExprLayer::Sub { a, b }
                }
                ExprBoxed::Mul { a, b } => {
                    let a = push_to_frontier(&elems, &mut frontier, a);
                    let b = push_to_frontier(&elems, &mut frontier, b);
                    ExprLayer::Mul { a, b }
                }
                ExprBoxed::LiteralInt { literal } => {
                    // no more nodes to explore
                    ExprLayer::LiteralInt { literal: *literal }
                }
            };

            elems.push(node);
        }

        Self { elems }
    }
}
```

## Factoring out duplicated code

Just as before, we can clean it up a bit if we use `map`:

```rust
impl Expr {
    fn generate_from_boxed(seed: &ExprBoxed) -> Self {
        let mut frontier: VecDeque<&ExprBoxed> = VecDeque::from([seed]);
        let mut elems = vec![];

        // generate to build a vec of elems while preserving topo order
        while let Some(seed) = { frontier.pop_front() } {
            let node = match seed {
                ExprBoxed::Add { a, b } => ExprLayer::Add { a, b },
                ExprBoxed::Sub { a, b } => ExprLayer::Sub { a, b },
                ExprBoxed::Mul { a, b } => ExprLayer::Mul { a, b },
                ExprBoxed::LiteralInt { literal } => ExprLayer::LiteralInt { literal: *literal },
            };
            let node = node.map(|seed| {
                frontier.push_back(seed);
                // idx of pointed-to element determined from frontier + elems size
                ExprIdx(elems.len() + frontier.len())
            });

            elems.push(node);
        }

        Self { elems }
    }
}
```

## Making it generic

That's better, but just as with `fold`, we only really care about the `match` expression here:

```rust
            let node = match seed {
                ExprBoxed::Add { a, b } => ExprLayer::Add { a, b },
                ExprBoxed::Sub { a, b } => ExprLayer::Sub { a, b },
                ExprBoxed::Mul { a, b } => ExprLayer::Mul { a, b },
                ExprBoxed::LiteralInt { literal } => ExprLayer::LiteralInt { literal: *literal },
            };
```

Fortunately, just as with `fold`, we can separate the machinery of recursion from the actual recursive (or, in this case, co-recursive) logic.


```rust
impl RecursiveExpr {
    fn generate<A, F: Fn(A) -> Expr<A>>(seed: A, generate_layer: F) -> Self {
        let mut frontier = VecDeque::from([seed]);
        let mut elems = vec![];

        // repeatedly generate nodes to build a vec of elems while preserving topo order
        while let Some(seed) = frontier.pop_front() {
            let node = generate_layer(seed);

            let node = node.map(|seed| {
                frontier.push_back(seed);
                // idx of pointed-to element determined from frontier + elems size
                ExprIdx(elems.len() + frontier.len())
            });

            elems.push(node);
        }

        Self { elems }
    }
}
```

This lets us write `generate_from_boxed` as:

```rust
impl Expr {
    pub fn generate_from_boxed(ast: &ExprBoxed) -> Self {
        Self::generate(ast, |seed| match seed {
            ExprBoxed::Add { a, b } => ExprLayer::Add { a, b },
            ExprBoxed::Sub { a, b } => ExprLayer::Sub { a, b },
            ExprBoxed::Mul { a, b } => ExprLayer::Mul { a, b },
            ExprBoxed::LiteralInt { literal } => ExprLayer::LiteralInt { literal: *literal },
        })
    }
}
```

Nice and, as promised, elegant.

# Testing for Correctness

I used proptest to test this code for correctness. It generates many expression trees, each of which is evaluated via both `eval` methods. I then assert that they have the same result. (todo: describe this technique in more detail, mention that I learned it from Rain)

This actually helped me find a bug! In my first implementation of `unfold`, I used a stack instead of a queue for the frontier, which ended up mangling the order of the expression tree. Since proptest is awesome, it not only found this bug but reduced the failing test case to `Add (0, Sub(0, 1))`.

```rust
// generate a bunch of expression trees and evaluate them via each method (TODO: added new methods, test those too)
#[cfg(test)]
proptest! {
    #[test]
    fn expr_eval(boxed_expr in arb_expr()) {
        let eval_boxed = boxed_expr.eval();
        let eval_via_fold = RecursiveExpr::generate_from_boxed(&boxed_expr).eval();

        assert_eq!(eval_boxed, eval_via_fold);
    }
}

#[cfg(test)]
pub fn arb_expr() -> impl Strategy<Value = ExprBoxed> {
    let leaf = any::<i8>().prop_map(|x| ExprBoxed::LiteralInt { literal: x as i64 });
    leaf.prop_recursive(
        8,   // 8 levels deep
        256, // Shoot for maximum size of 256 nodes
        10,  // We put up to 10 items per collection
        |inner| {
            prop_oneof![
                (inner.clone(), inner.clone()).prop_map(|(a, b)| ExprBoxed::Add {
                    a: Box::new(a),
                    b: Box::new(b)
                }),
                (inner.clone(), inner.clone()).prop_map(|(a, b)| ExprBoxed::Sub {
                    a: Box::new(a),
                    b: Box::new(b)
                }),
                (inner.clone(), inner).prop_map(|(a, b)| ExprBoxed::Mul {
                    a: Box::new(a),
                    b: Box::new(b)
                }),
            ]
        },
    )
}
```


# Testing for performance

TODO: get numbers from rain's box I guess, laptop is a bit shite for this

For performance testing, we used criterion to benchmark the simple `ExprBoxed::eval` vs our `RecursiveExpr::eval`. This code basically just builds up a really big (as in, 131072 nodes) recursive structure (using unfold/fold, because they're honestly really convenient) and evaluates it a bunch of times. I also ran this test on recusive structures of other sizes, because graphs are cool. You can find the benchmarks [defined here](https://github.com/inanna-malick/rust-schemes/blob/99620b4f9a0bb742996c0dece342c50c4ab31071/benches/expr.rs).



<pre><font color="#A6CC70">Evaluate expression tree of depth 17 with standard boxed method</font>                                                                            
                        time:   [722.18 µs <font color="#77A8D9"><b>733.00 µs</b></font> 746.43 µs]

<font color="#A6CC70">Evaluate expression tree of depth 17 with my new fold method</font>                                                                            
                        time:   [477.87 µs <font color="#77A8D9"><b>482.54 µs</b></font> 488.58 µs]
</pre>


Evaluating a boxed expression of size  takes an average 785.41 µs. Evaluating an expression stored in our `RecursiveExpr` takes an average of 559.22 µs. That's a 28% improvement. Running these tests with expression trees of different depths generated via the above method yields similar results. The standard boxed method is slightly more efficient for expression trees of size 256 or less. That said, this test provides pretty much optimal conditions with regard to pointer locality, because there are no other heap allocations to fragment things and force the boxed pointers to use different regions of memory.

# To be continued

We started with a simplified non-generic version of this algorithm to build understanding. In future blog posts, I plan on showing how I made it generic, going into more detail on how I optimized it for performance (`MaybeUninit` absolutely slaps, as do stack machines), and how I used it to implement an async file tree search tool using `tokio::fs`.

# Thank you

Thank you to everyone that helped me write this blog post and the code that it describes (todo: ask if ppl want to be credited, and by what name/handle)
