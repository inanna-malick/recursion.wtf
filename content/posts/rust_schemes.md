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


We're going to start with a simple expression language: addition, subtraction, multiplication, just enough to illustrate some concepts. This is a naive representation of a recursive expression language that uses boxed pointers to handle the recursive case. If you're not familiar to boxed pointers, a `Box<A>` is just the Rust way of storing a pointer to some value of type `A` - think of it as a box with a value of type `A` inside it.

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

It has some issues: if we try to evaluate a sufficiently large expression it will fail with a stack overflow - we're not likely to hit that case here, but this is a real problem when working with larger recursive data structures. Also, each recursive `eval` call requires us to traverse a boxed pointer. This means we can't take advantage of cache locality - there's no guarantee that all these boxed pointers live in the same region of memory.


## Optimizing for performance

We can fix that by writing an expression language using a Vec of values (guaranteeing memory locality), with boxed pointers replaced with usize indices pointing into our vector.

```rust
pub enum Expr<A> {
    Add { a: A, b: A },
    Sub { a: A, b: A },
    Mul { a: A, b: A },
    LiteralInt { literal: i64 },
}
pub struct RecursiveExpr {
    /// nonempty, in topological-sorted order. for every node `n`, all of `n`'s child nodes have vec indices greater than that of n
    elems: Vec<Expr<usize>>,
}
```

We're also going to define a function to map from one type of Expr to another, for convenience - nothing complex, it just applies a function to each 'A', sort of like mapping over an Option or an Iterator. If you're familiar with functional languages, this is basically just `fmap`.

```rust
impl<A> Expr<A> {
    fn map<B, F: FnMut(A) -> B>(self, mut f: F) -> Expr<B> {
        match self {
            Expr::Add { a, b } => Expr::Add { a: f(a), b: f(b) },
            Expr::Sub { a, b } => Expr::Sub { a: f(a), b: f(b) },
            Expr::Mul { a, b } => Expr::Mul { a: f(a), b: f(b) },
            Expr::LiteralInt { literal } => Expr::LiteralInt { literal },
        }
    }
}
```



The problem here is that this is harder to read - we don't want to construct these by hand, because it would be tedious and error prone. Instead, we'll just create them from boxed expressions. Here's how: start with a boxed expression, unfold a single layer of structure from it, then just repeatedly unfold layers until there aren't any more layers to unfold. Feel free to skim this function, it's  only here is to demonstrate what non-elegant recursive code looks like.

```rust
impl RecursiveExpr {
    fn from_boxed(a: &ExprBoxed) -> Self {
        let mut frontier: VecDeque<&ExprBoxed> = VecDeque::from([a]);
        let mut elems = vec![];

        // unfold to build a vec of elems while preserving topo order
        while let Some(seed) = frontier.pop_front() {
            let node = match seed {
                ExprBoxed::Add { a, b } => Expr::Add { a, b },
                ExprBoxed::Sub { a, b } => Expr::Sub { a, b },
                ExprBoxed::Mul { a, b } => Expr::Mul { a, b },
                ExprBoxed::LiteralInt { literal } => Expr::LiteralInt { literal: *literal },
            };

            let node = node.map(|aa| {
                frontier.push_back(aa);
                // idx of pointed-to element determined from frontier + elems size
                elems.len() + frontier.len()
            });

            elems.push(node);
        }

        Self { elems }
    }
}
```

It's not exactly elegant, right? The `let node = match {...}` block defines a single recursive step, which builds a single layer of `Expr<&ExprBoxed>` structure from an `&ExprBoxed` seed value, but it's surrounded with a bunch of bookkeeping boilerplate that handles combining the layers to build a vec of `Expr<usize>`.


Don't worry, we have a fix for this. Before we can get to that, let's look at what evaluating a a `RecursiveExpr` looks like:

```rust
impl RecursiveExpr {
    fn eval(self) -> i64 {
        let mut results: HashMap<usize, i64> = HashMap::with_capacity(self.elems.len());

        for (idx, node) in self.elems.into_iter().enumerate().rev() {
            // each node is only referenced once so just remove it
            let node = node.map(|x| results.remove(&x).expect("node not in result map"));
            let alg_res = match node {
                Expr::Add { a, b } => a + b,
                Expr::Sub { a, b } => a - b,
                Expr::Mul { a, b } => a * b,
                Expr::LiteralInt { literal } => literal,
            };
            results.insert(idx, alg_res);
        }

        results.remove(&0).unwrap()
    }
}
```


Here we fold up the expression tree from the leaves to the root, evaluating it one layer at a time and storing the results in a hash map until they are used. Since everything we're folding over is stored in a vec, in one contiguous region of memory, we don't need to worry about the overhead of traversing a bunch of pointers (I have benchmarks over a more optimized version that shows a consistient 20-30% improvement over evaluating boxed `ExprBoxed` nodes).

Unfortunately, it's not elegant. Once again, the logic of _how_ we fold layers of recursive structure (`Expr<i64>`) into a single value (`i64`) is combined with a bunch of boilerplate that handles the actual mechanics of recursion. 

Let's fix that.

# RECURSION SCHEMES

The key idea here, which is taken entirely from recursion schemes, is to _separate_ the mechanism of recursion from the logic of recursion. Let's see what that looks like:

```rust
impl RecursiveExpr {
    fn fold<A, F: FnMut(Expr<A>) -> A>(self, mut fold_layer: F) -> A {
        let mut results: HashMap<usize, A> = HashMap::with_capacity(self.elems.len());

        for (idx, node) in self.elems.into_iter().enumerate().rev() {
            // each node is only referenced once so just remove it
            let node = node.map(|x| results.remove(&x).expect("node not in result map"));
            let alg_res = fold_layer(node);
            results.insert(idx, alg_res);
        }

        results.remove(&0).unwrap()
    }
}
```

First, we have a generic representation of folding some structure into a single value - instead of folding an `Expr<i64>` into a single `i64`, we fold some `Expr<A>` into an `A`. The code looks pretty much the same as `eval`, but it lets us factor out the mechanism of recursion.

Here's what `eval` looks like using this idiom:

```rust
impl RecursiveExpr {
    pub fn eval(self) -> i64 {
        self.fold(|expr| match expr {
            Expr::Add { a, b } => a + b,
            Expr::Sub { a, b } => a - b,
            Expr::Mul { a, b } => a * b,
            Expr::LiteralInt { literal } => literal,
        })
    }
}
```

It's pretty much the same logic as the previous `eval` functions, without any of the boilerplate. Since there's less boilerplate, it's easier to review and there's less room for bugs. In fact, it actually contains slightly less boilerplate than the eval function we wrote for `ExprBoxed::eval` because it doesn't have to handle recursively calling itself. Also, it retains all the performance benefits of the previous `eval` implementation - it's both more elegant and more performant than the traditional representation of recursive expression trees in rust. I think that's neat.



```rust
impl RecursiveExpr {
    fn unfold<A, F: Fn(A) -> Expr<A>>(a: A, unfold_layer: F) -> Self {
        let mut frontier = VecDeque::from([a]);
        let mut elems = vec![];

        // unfold to build a vec of elems while preserving topo order
        while let Some(seed) = frontier.pop_front() {
            let node = unfold_layer(seed);

            let node = node.map(|aa| {
                frontier.push_back(aa);
                // idx of pointed-to element determined from frontier + elems size
                elems.len() + frontier.len()
            });

            elems.push(node);
        }

        Self { elems }
    }
}
```

Here we have a generic representation of unfolding some structure from a single value - instead of unfolding a single layer of `Expr<&ExprBoxed>` structure from an `&ExprBoxed` seed value, we unfold some `A` into an `Expr<A>`. As with `fold`, the code looks pretty much the same as `from_boxed`, just with the specific unfold logic factored out.


```rust
impl RecursiveExpr {
    pub fn from_boxed(e: &ExprBoxed) -> Self {
        Self::unfold(e, |seed| match seed {
            ExprBoxed::Add { a, b } => Expr::Add { a, b },
            ExprBoxed::Sub { a, b } => Expr::Sub { a, b },
            ExprBoxed::Mul { a, b } => Expr::Mul { a, b },
            ExprBoxed::LiteralInt { literal } => Expr::LiteralInt { literal: *literal },
        })
    }
}
```

Here's what `RecursiveExpr::from_boxed` looks like as written using `unfold`. Just as before, there's almost no boilerplate, the body of the function is almost entirely taken up by the logic of unfolding.


# Testing for Correctness

I used proptest to test this code for correctness. It generates many expression trees, each of which is evaluated via both `eval` methods. I then assert that they have the same result. (todo: describe this technique in more detail, credit Rain)

This actually helped me find a bug! In my first implementation of `unfold`, I used a stack instead of a queue for the frontier, which ended up mangling the order of the expression tree. Since proptest is awesome, it not only found this bug but reduced the failing test case to `Add (0, Sub(0, 1))`.

```rust
// generate a bunch of expression trees and evaluate them via each method
#[cfg(test)]
proptest! {
    #[test]
    fn expr_eval(boxed_expr in arb_expr()) {
        let eval_boxed = boxed_expr.eval();
        let eval_via_fold = RecursiveExpr::from_boxed(&boxed_expr).eval();

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

For performance testing, we used criterion to benchmark the simple `ExprBoxed::eval` vs our `RecursiveExpr::eval`. This code basically just builds up a really big (as in, 131072 nodes) recursive structure (using unfold/fold, because they're honestly really convenient) and evaluates it a bunch of times.

```rust
fn bench_eval(criterion: &mut Criterion) {
    let big_expr = RecursiveExpr::unfold(17, |x| {
        if x > 0 {
            Expr::Add(x - 1, x - 1)
        } else {
            Expr::LiteralInt(0)
        }
    });

    let boxed_big_expr = big_expr.as_ref().fold(|n| match n {
        Expr::Add(a, b) => Box::new(ExprBoxed::Add(a, b)),
        Expr::LiteralInt(x) => Box::new(ExprBoxed::LiteralInt(x)),
        _ => unreachable!(),
    });

    let h = HashMap::new();

    criterion.bench_function("eval boxed", |b| {
        b.iter(|| naive_eval(&h, black_box(&boxed_big_expr)))
    });
    criterion.bench_function("eval fold", |b| b.iter(|| eval(&h, black_box(&big_expr))));
}

criterion_group!(benches, bench_eval);
criterion_main!(benches);
```

Here's the result, after a few optimization passes:


<pre><font color="#A6CC70">Evaluate expression tree of depth 17 with standard boxed method</font>                                                                            
                        time:   [722.18 µs <font color="#77A8D9"><b>733.00 µs</b></font> 746.43 µs]

<font color="#A6CC70">Evaluate expression tree of depth 17 with my new fold method</font>                                                                            
                        time:   [477.87 µs <font color="#77A8D9"><b>482.54 µs</b></font> 488.58 µs]
</pre>

Evaluating a boxed expression of size  takes an average 785.41 µs. Evaluating an expression stored in our `RecursiveExpr` takes an average of 559.22 µs. That's a 28% improvement. Running these tests with expression trees of different depths generated via the above method yields similar results.


# To be continued

We started with a simplified non-generic version of this algorithm to build understanding. In future blog posts, I plan on showing how I made it generic, how I optimized it for performance (`MaybeUninit` absolutely slaps), and how I used it to implement an async file tree search tool using `tokio::fs`.