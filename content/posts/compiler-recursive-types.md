---
title: "Writing Recursively Defined Types in Rust"
date: 2019-04-22T19:54:41-05:00
showDate: true
draft: false
tags: ["blog","compilers","rust"]
---

Let's say we're writing a theoretical compiler, and we want to teach it how to parse some math expressions.
Consider this snippet of source code:
```
(+ (- 20 10) 7)
```
In this example, we see a math expression. The operator in this case is `+`, and the operands are another math expression and `7`. Internally,
the compiler sees these operands as just two more distinct expressions (literals are a type of expression). Since a math expression is an operator 
and two more expressions, the grammar matches and this parses successfully. To get a better understanding of how this is defined, 
consider a possible set of grammars:
```
OPERATOR <-- + | - | / | *
EXPR <-- MATHEXPR | NUMLIT
MATHEXPR <-- OPERATOR EXPR EXPR
```
As you can see, each math expression could contain two more math expressions. This could theoretically repeat forever,
with each new math expression containing two more new math expressions, and so on. Furthermore, as `EXPR` increases in scope; that is,
it gains more possible expression types, `MATHEXPR` gains more functionality. If you added a function call expression, then a math
expression could contain a function call as an operand, and so on<sup>1</sup>.

To effectively represent the recursive nature of our theoretical grammar, we might try this:
```rust
enum Expr {
	Literal(LiteralExpr),
	FunctionCall(FunctionCallExpr),
}

struct MathExpression {
	op: MathOperator,
	lhs: Expr,
	rhs: Expr,
}

enum LiteralExpr {
	Num(usize), // or your favorite numeric type.
}

enum MathOperator {
	Plus,
	Minus,
	Div,
	Mul,
}
```
However, the compiler takes issue with this definition:
```
-- snip --
error[E0072]: recursive type `MathExpression` has infinite size
  --> test.rs:7:1
   |
7  | struct MathExpression {
   | ^^^^^^^^^^^^^^^^^^^^^ recursive type has infinite size
8  |     op: MathOperator,
9  |     lhs: Expr,
   |     --------- recursive without indirection
10 |     rhs: Expr,
   |     --------- recursive without indirection
   |
   = help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `MathExpression` representable
```
The issue here is that Rust needs to know how much space on the stack an object will take up. Since we defined `MathExpression` as two more expressions,
and each of those sub-expressions could contain more `MathExpression`s, we end up with an unbounded object size.
The solution to this issue is to add heap indirection of some kind to stamp down the size of an `Expr` so that Rust can reserve the right
amount of memory. In this example, the best solution is to allocate  the `Expr`'s within `MathExpression` on the heap. This will define the
size of a `MathExpression`, since a pointer to some location on the heap has a concrete size - and as the Rust compiler so graciously tells us,
we have a few different options for introducing this indirection.

The first option is `Box`. The Rust documentation defines `Box` as "a pointer type for heap allocation", and that's exactly
what it is: wrapping a value in a `Box` will heap allocate the value. Note that `Box` can only have one
owner, if you call `.clone()` it will copy the data within the `Box` as well. The memory within a `Box` will be freed
when its owner is dropped.

The second listed choice is `Rc`. `Rc` stands for "reference-counted"<sup>2</sup>, and it's the same as a `Box`
as far as the allocation part is concerned. However, `Rc` is not limited to a single owner like `Box` - it permits **shared ownership**. 
This means that multiple variables can own the data an `Rc` points to, with the caveat that it is not mutable<sup>3</sup>. 
So, if you have some code like the following:
```rust
let x = Rc::new(22);
let y = Rc::clone(&x); // you can also use x.clone()
drop(x);
println!("{}", y); // y is still valid even though x was dropped!
```
In this example, both `x` and `y` point to the same memory. However, even though we tell Rust to explicitly drop
`x` before the end of the scope, `y` is still valid. This can be attributed to the reference-counting `Rc`
employs: the memory that stores the value `22` will not be freed until the reference-count is zero. Dropping `x`
only decreased the reference-count by one, leaving it at one reference remaining. When `y` is dropped,
the memory will be freed.

As an aside, note that the presented `Rc` example would work with `Box` too - you can `.clone()` a `Box` with
no problems. However, as mentioned, cloning a `Box` will also clone the value within it, effectively
doubling your allocations. Cloning an `Rc` will simply return a pointer to the same data and increase the reference-count.
Because of this ambiguous nature, when cloning `Rc`s the better syntax is `Rc::clone(&from)`, as it is more clear
about what's actually happening.

There is also `Arc`, which is the thread-safe atomic version of `Rc`. This isn't really relevant to the example here,
so I won't bother going over it - as far as I understand, it works the same as `Rc` anyway, sans the atomic part.

So, which is the best option for this case? In this situation we don't really need multiple owners - `Box` is the right
choice. So, our `MathExpression` definition changes to this:
```rust
struct MathExpression {
	op: MathOperator,
	lhs: Box<Expr>,
	rhs: Box<Expr>,
}
```
And it compiles just fine. Now our theoretical grammar is prefectly represented in Rust, recursion and all.

### Footnotes
<sup>1</sup>: Obviously, there are limits to this - you wouldn't want an assignment expression as an argument to a math expression.
While that kind of use would parse, it would not pass further analysis.

<sup>2</sup>: Note that reference-counted types have runtime overhead, because they need to keep track of references. `Arc` even more
so, because it is atomic.

<sup>3</sup>: You actually *can* mutate the contents of a `Rc`, if they're wrapped in a `Cell` or `RefCell`.
This type of mutability is known as **interior mutability**, and allows for types like `Rc` to have mutable
data. `Arc` has the same capability, but you must use `Mutex` or `RwLock` instead, as they are thread-safe.


