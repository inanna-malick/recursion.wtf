+++
title = "Elegant and performant recursion in Rust"
date = "2022-07-18"
author = "Inanna Malick"
authorTwitter = "inanna_malick"
tags = ["recursion schemes", "rust", "code"]
keywords = ["recursion schemes", "rust", "code"]
showFullContent = false
images = ["/img/rust_schemes/criterion_screenshot_preview.png"]
feature = "/img/rust_schemes/criterion_screenshot_preview.png"
thumbnail = "/img/rust_schemes/criterion_screenshot_preview.png"
+++


This is a post about writing elegant and performant recursive algorithms in Rust. It makes heavy use of a pattern from Haskell called recursion schemes, but you don't need to know anything about that; it's just an implementation detail. Instead, as motivation, I have benchmarks showing a 14-34% improvement over the typical boxed pointer representation of recursive data structures in Rust.


<!--more--> 

# Performance test results


These test results show a performance improvement of 34% for evaluating a very large expression tree (131072 elements, recursive depth 17). They were run on a 6th generation X1 carbon laptop with an Intel i7-8550U with 8MB CPU cache:

<pre><font color="#A6CC70">Evaluate expression tree of depth 17 with standard method</font>                                                                            
                        time:   [722.18 µs <font color="#77A8D9"><b>733.00 µs</b></font> 746.43 µs]

<font color="#A6CC70">Evaluate expression tree of depth 17 with my new fold method</font>                                                                            
                        time:   [477.87 µs <font color="#77A8D9"><b>482.54 µs</b></font> 488.58 µs]
</pre>

The same tests, when run on an AMD Ryzen 9 3900X CPU with more than 64MB total cache (L1/L2/L3), still show a 14% speed improvement over the usual method.

<pre><font color="#A6CC70">Evaluate expression tree of depth 17 with standard method</font>                                                                            
                        time:   [295.76 µs <font color="#77A8D9"><b>295.89 µs</b></font> 296.03 µs]

<font color="#A6CC70">Evaluate expression tree of depth 17 with my new fold method</font>                                                                            
                        time:   [250.96 µs <font color="#77A8D9"><b>251.12 µs</b></font> 251.31 µs]
</pre>



# Evaluating an expression language


We're going to start with a simple expression language: addition, subtraction, multiplication – just enough to illustrate some concepts. You've probably seen something like this before, but if not, it's just a way to represent simple arithmetic as a tree of expressions. For example, an expression like `1 * (2 - 3)` would be written as (pseudocode) `Mul(1, Sub(2, 3))`.

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

This is a recursive expression language that uses boxed pointers to handle the recursive case. If you're not familiar with boxed pointers, a `Box<A>` is just the Rust way of storing a pointer to some value of type `A` - think of it as a box with a value of type `A` inside it. (If you're curious, there's [more documentation here](https://doc.rust-lang.org/std/boxed/index.html))

Using this data structure, we can write `Mul(1, Sub(2, 3))` as:


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

This algorithm has some issues: 
- If we try to evaluate a sufficiently large expression it will fail with a stack overflow - we're not likely to hit that case here, but this is a real problem when working with larger recursive data structures.
- Each recursive `eval` call requires us to traverse a boxed pointer. This means we can't take advantage of cache locality - there's no guarantee that all these boxed pointers live in the same region of memory. [^bignote_cache]

[^bignote_cache]: If you're not sure what I mean by cache locality, or you want much more information on it than I can provide, there's a great rust performance optimization resource [here](https://gist.github.com/kvark/f067ba974446f7c5ce5bd544fe370186#keep-as-much-as-possible-in-cache).


## A more cache-local structure

We can fix that by writing an expression language using a Vec of individual expression nodes (guaranteeing memory locality), with boxed pointers replaced with newtype-wrapped vector indices.

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

pub struct ExprTopo {
    // nonempty, in topological-sorted order. guaranteed via construction.
    elems: Vec<ExprLayer<ExprIdx>>,
}
```

You might have noticed that we have used a generic parameter `A` rather than simply writing `ExprLayer<ExprIdx>`. Put a pin in that for now, we'll come back to that soon.

All our expressions are now guaranteed to be stored in local memory. Here's a sketch showing what the `Mul(1, Sub(2, 3))` expression would look like using this data structure.

```
[
idx_0:    Mul(1_idx, 2_idx)
idx_1:    LiteralInt(1)
idx_2:    Sub(idx_3, idx_4)
idx_3:    LiteralInt(2)
idx_4:    LiteralInt(3)
]
```

The nodes are stored in [topological order](https://en.wikipedia.org/wiki/Topological_sorting), which means that for each node, all of its child nodes are stored at larger indices. To evaluate an `ExprTopo`, we can perform bottom up recursion: fold leaf values into their parents, one `ExprLayer` at a time, until the entire `ExprTopo` structure is folded into a single value. Since it's topologically sorted, we can do this by iterating over the element vector in reverse order.


 Let's see what evaluating this structure looks like in practice. It's not elegant. There's a bunch of `unsafe` code, but it _does_ have better performance in benchmarks. Feel free to skim; in the next section we'll introduce an elegant API that removes the need to write `unsafe` code.

```rust
impl ExprTopo {
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
            let result = {
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
            results[idx].write(result);
        }

        unsafe {
            let maybe_uninit =
                std::mem::replace(results.get_unchecked_mut(0), MaybeUninit::uninit());
            maybe_uninit.assume_init()
        }
    }
}
```

The problem here is that this is very difficult to read and write. Imagine having to write all of this by hand, for each recursive function. It would be tedious at best and error prone at worst.


## Factoring out duplicated code

Every arm of the above match statement (except for `LiteralInt`) calls `get_result_unsafe` in pretty much the same way. We can start by factoring that out.


Now you can see why we made `ExprLayer<A>` parameterized over some `A`. Since it is parameterized over some `A`, we can apply a function to each `A` inside it, turning it into an `ExprLayer<B>`. We're going to write some code that's very similar to `Option::map` in the standard library.


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
impl ExprTopo {
    fn eval(self) -> i64 {
        use std::mem::MaybeUninit;

        let mut results = std::iter::repeat_with(|| MaybeUninit::<i64>::uninit())
            .take(self.elems.len())
            .collect::<Vec<_>>();


        for (idx, layer) in self.elems.into_iter().enumerate().rev() {
            let layer: ExprLayer<i64> = layer.map(|idx| unsafe {
                let maybe_uninit =
                    std::mem::replace(results.get_unchecked_mut(idx.0), MaybeUninit::uninit());
                maybe_uninit.assume_init()
            });

            let result = match layer {
                ExprLayer::Add { a, b } => a + b,
                ExprLayer::Sub { a, b } => a - b,
                ExprLayer::Mul { a, b } => a * b,
                ExprLayer::LiteralInt { literal } => literal,
            };
            results[idx].write(result);
        }

        unsafe {
            let maybe_uninit =
                std::mem::replace(results.get_unchecked_mut(ExprIdx::head().0), MaybeUninit::uninit());
            maybe_uninit.assume_init()
        }
    }
}
```


## Making it generic

Ok, that's a start. Unfortunately, we still have to write all this boilerplate for _every recursive function_, even though the only part that really matters is this block:

```rust
let result = match layer {
    ExprLayer::Add { a, b } => a + b
    ExprLayer::Sub { a, b } => a - b
    ExprLayer::Mul { a, b } => a * b
    ExprLayer::LiteralInt { literal } => literal,
}
```

This code takes  `layer`, a value of type `ExprLayer<i64>`, and consumes it to create `result`, a value of type `i64`. What if, instead of `ExprLayer<i64> -> i64`, we use a function of type `ExprLayer<A> -> A`?


This function lets us provide an arbitrary function of type `ExprLayer<A> -> A` and uses it to fold a recursive `ExprTopo` structure into a single value:

```rust
impl ExprTopo {
    fn fold<A: std::fmt::Debug, F: FnMut(ExprLayer<A>) -> A>(self, mut fold_layer: F) -> A {
        use std::mem::MaybeUninit;

        let mut results = std::iter::repeat_with(|| MaybeUninit::<A>::uninit())
            .take(self.elems.len())
            .collect::<Vec<_>>();

        for (idx, layer) in self.elems.into_iter().enumerate().rev() {
            let result = {
                let layer = layer.map(|x| unsafe {
                    let maybe_uninit =
                        std::mem::replace(results.get_unchecked_mut(x.0), MaybeUninit::uninit());
                    maybe_uninit.assume_init()
                });
                fold_layer(layer)
            };
            results[idx].write(result);
        }

        unsafe {
            let maybe_uninit =
                std::mem::replace(results.get_unchecked_mut(ExprIdx::head().0), MaybeUninit::uninit());
            maybe_uninit.assume_init()
        }
    }
}
```

Nice. Now we can write:

```rust
impl ExprTopo {
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

It's pretty much the same logic as the original `eval` functions, without any of the boilerplate. Since there's less boilerplate, it's easier to review and there's less room for bugs. Also, it retains all the performance benefits of the previous `eval` implementation - it's both more elegant and more performant than the traditional representation of recursive expression trees in rust.


# Constructing Exprs

Let's write a function to generate `ExprTopo` values from the `ExprBoxed` representation. Just as before, `map` helps us keep it concise. Feel free to skim this one too, we'll be abstracting over the specifics just like we did with `fold`:

```rust
impl ExprTopo {
    fn generate_from_boxed(seed: &ExprBoxed) -> Self {
        let mut frontier: VecDeque<&ExprBoxed> = VecDeque::from([seed]);
        let mut elems = vec![];

        // generate to build a vec of elems while preserving topo order
        while let Some(seed) = { frontier.pop_front() } {
            let layer = match seed {
                ExprBoxed::Add { a, b } => ExprLayer::Add { a, b },
                ExprBoxed::Sub { a, b } => ExprLayer::Sub { a, b },
                ExprBoxed::Mul { a, b } => ExprLayer::Mul { a, b },
                ExprBoxed::LiteralInt { literal } => ExprLayer::LiteralInt { literal: *literal },
            };
            let layer = layer.map(|seed| {
                frontier.push_back(seed);
                // idx of pointed-to element determined from frontier + elems size
                ExprIdx(elems.len() + frontier.len())
            });

            elems.push(layer);
        }

        Self { elems }
    }
}
```

## Making it generic

Just as with `fold`, we only really care about the `match` expression here:

```rust
let layer = match seed {
    ExprBoxed::Add { a, b } => ExprLayer::Add { a, b },
    ExprBoxed::Sub { a, b } => ExprLayer::Sub { a, b },
    ExprBoxed::Mul { a, b } => ExprLayer::Mul { a, b },
    ExprBoxed::LiteralInt { literal } => ExprLayer::LiteralInt { literal: *literal },
};
```

This matches on `seed`, a value of type `&ExprBoxed`, and consumes it to create `layer`, a value of type `ExprLayer<i64ExprBoxed>`. What if, instead of `i64ExprBoxed -> ExprLayer<i64ExprBoxed>`, we use a function of type `A -> ExprLayer<A>`? 


Fortunately, just as with `fold`, we can separate the machinery of recursion from the actual recursive (or, in this case, co-recursive) logic.


```rust
impl ExprTopo {
    fn generate<A, F: Fn(A) -> ExprLayer<A>>(seed: A, generate_layer: F) -> Self {
        let mut frontier = VecDeque::from([seed]);
        let mut elems = vec![];

        // repeatedly generate layers to build a vec of elems while preserving topo order
        while let Some(seed) = frontier.pop_front() {
            let layer = generate_layer(seed);

            let layer = layer.map(|seed| {
                frontier.push_back(seed);
                // idx of pointed-to element determined from frontier + elems size
                ExprIdx(elems.len() + frontier.len())
            });

            elems.push(layer);
        }

        Self { elems }
    }
}
```

This lets us write `generate_from_boxed` as:

```rust
impl ExprTopo {
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

I used [proptest](https://lib.rs/crates/proptest) to test this code for correctness. It generates many expression trees, each of which is evaluated via both `eval` methods. I then assert that they have the same result. [^rain_is_awesome]


[^rain_is_awesome]: I learned this technique from my partner [Rain](https://sunshowers.io/)

This actually helped me find a bug! In my first implementation of `unfold`, I used a stack instead of a queue for the frontier, which ended up mangling the order of the expression tree. Since proptest is awesome, it not only found this bug but reduced the failing test case to `Add (0, Sub(0, 1))`.

```rust
// generate a bunch of expression trees and evaluate them via each method (TODO: added new methods, test those too)
#[cfg(test)]
proptest! {
    #[test]
    fn expr_eval(boxed_expr in arb_expr()) {
        let eval_boxed = boxed_expr.eval();
        let eval_via_fold = ExprTopo::generate_from_boxed(&boxed_expr).eval();

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

For performance testing, we used [criterion](https://github.com/bheisler/criterion.rs) to benchmark the simple `ExprBoxed::eval` vs `ExprTopo::eval`. This code basically just builds up a really big (as in, 131072 nodes) recursive structure (using unfold/fold, because they're honestly really convenient) and evaluates it a bunch of times. I also ran this test on recursive structures of other sizes, because graphs are cool. You can find the benchmarks [defined here](https://github.com/inanna-malick/rust-schemes/blob/99620b4f9a0bb742996c0dece342c50c4ab31071/benches/expr.rs).



<pre><font color="#A6CC70">Evaluate expression tree of depth 17 with standard boxed method</font>                                                                            
                        time:   [722.18 µs <font color="#77A8D9"><b>733.00 µs</b></font> 746.43 µs]

<font color="#A6CC70">Evaluate expression tree of depth 17 with my new fold method</font>                                                                            
                        time:   [477.87 µs <font color="#77A8D9"><b>482.54 µs</b></font> 488.58 µs]
</pre>


Evaluating a boxed expression of depth 17 takes an average 733 µs. Evaluating an expression stored in our `ExprTopo` takes an average of 482 µs. That's a 34% improvement. Running these tests with expression trees of different depths generated via the above method yields similar results. The standard boxed method is slightly faster for expression trees of size 256 or less. That said, this test provides pretty much optimal conditions with regard to pointer locality, because there are no other heap allocations to fragment things and force the boxed pointers to use different regions of memory.

# To be continued

We started with a simplified non-generic version of this algorithm to build understanding. In future blog posts, I plan on showing how I made it generic, going into more detail on how I optimized it for performance (`MaybeUninit` absolutely slaps, as do stack machines), and how I used it to implement an async file tree search tool using `tokio::fs`.

# Thank you

Thank you to [Fiona](https://twitter.com/munin), [Rain](https://twitter.com/sunshowers6), [Eliza](https://twitter.com/mycoliza) and [Gankra](https://gist.github.com/Gankra), among others, for reviewing drafts of this post.
