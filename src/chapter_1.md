# Getting Started

TYP unleashes the true power of type-level programming in Rust. It enables you to write functions that comsume and produce types.
It is written in Rust-like syntax, but processes and stores types instead of values.
It enables you to realize complex logics in types and gain the advantage of type safety and zero-cost abstraction.

## Type-level Programming Basics

Rust already has the capability of type level programming to some degree, but is done indirectly.
We will take a glance at the old school way to write _type operators_ to understand the concepts.
In later chapter, we will learn TYP to write in more compact syntax.
Let's start by building a traffic light machine.

To represent the states of the traffic light, we usualy encode the `Green`, `Yellow` and `Red` states into an enum.
But now we're doing type programming. We manipulate types instead of enum values.
We define each state as a type instead of an enum variant to leverage it to type level.

```rust
trait State {}
impl State for Red {}
impl State for Green {}
impl State for Blue {}

struct Red;
struct Green;
struct Yellow;
```

To compute transition of states, we need a function-like something `fn next(State) -> State`.
Unlike writing a function as usual, it should be able to take `Green`, `Yellow` and `Red` input types and produce a new type.

Here we abuse a Rust feature called _associated types_. Whenever you implement a trait on a type, you can bind a associated type on the implementation.
It is common in Rust standard library, for example, the `Add ` trait has an associated `Output` type.

```rust
pub trait Add<Rhs = Self> {
    type Output;
    // omit..
}


#[derive(Debug, Copy, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Output = Self;
    // omit..
}
```

As you might have guessed, we realize the transition function as a trait. Let name it `Next` here.
We write `impl Next on Green` block and bind associate the type `type Output = Yellow` within it.
Such that the output type `Yellow` is associated to the input type `Green`.

```rust
trait Next
where
    Self: State,
    Self::Output: State
{
    type Output;
}

impl Next for Green {
     type Output = Yellow;
}

impl Next for Yellow {
     type Output = Red;
}
impl Next for Red {
     type Output = Green;
}
```

You may come up with a question: how to _call_ this kind of function?
We use the qualified syntax to do this job.

```rust
type Outcome = <Green as Next>::Output; // Outcome == Yellow
```

The code reads "Get the associated `Output` type from `Next` trait implemented on `Red` type".
It is a bit verbose, but actually works. The compiler will find the appropriate implementation and compute the output type for you.


We can make the _trait call_ less verbose by by type aliasing.

```rust
// define the type alias
type NextOp<Input> = <Input as Next>::Output;

// so we can write this
type Outcome = NextOp<Green>;
```

That's the way the type-level programming is done in Rust. As you can see, the tiny state machine takes many lines of code.
Imagine you're working on a more complex types, such as type level integers. It could be tedious. That's TYP comes to our rescue!

## Using TYP

TYP helps you write type-level functions, or _type operators_, as though you're writing Rust functions as usual.
Let's rewrite the state transition type operator.

```rust
use typ::typ; // import the macro

typ! {
    fn Next<input>(input: State) -> State {
        match input {
            Green => Yellow,
            Yellow => Red,
            Red => Green,
        }
    }
}
```

The `typ!` macro is a transcompiler that translates the code into traits and impl blocks.
Just the same thing we've done in old school. The example produces the trait `Next` and a corresponding type alias `NextOp`.
We invoke the type operator in the same way.

```rust
type Outcome1 = <Green as Next>::Output; // Outcome1 == Yellow
type Outcome2 = NextOp<Green>; // Outcome2 == Yellow
```

As you can see, TYP adopts the Rust syntax, but the context is totally different. Look at the signature.

```rust
fn Next<input>(input: State) -> State
```

The `input` argument is a type instead of a value, while `State` is a trait bound instead of a type.
Also, the `match` compares the types instead of values. In the next chapter, we will explain in details.
