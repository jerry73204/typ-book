# Getting Started

TYP unleashes the true power of type-level programming in Rust. It enables you to write functions that comsume and produce types.
It is written in Rust-like syntax, but processes and stores types instead of values.
It enables you to realize complex logics in types and gain the advantage of type safety and zero-cost abstraction.

## Type-level Programming Basics

Rust already provides the capability of type level programming to some degree, but in an indirect way.
Let's start by a traffic light state machine to see how it could be done in Rust.

When we build a state machine, we usually encode the `Green`, `Yellow` and `Red` states into an enum,
and define a transition function `next()` to compute change of states.

To leverage it to type level, we encode the states by types instead of enums.
In this way, `Green`, `Yellow` and `Red` become three distinct types.

```rust
trait State {}
impl State for Red {}
impl State for Green {}
impl State for Blue {}

struct Red;
struct Green;
struct Yellow;
```

How could be leverage the `next` transition function to type level?
It means We have to define a function-like something that take `Green`, `Yellow` and `Red` types and produce a new type.

Here we abuse a Rust feature called _associated types_. Whenever you implement a trait on a type, you can bind a associated type on the implementation.
As you might have guessed, we realize the transition function as a trait `Next`.
To bind the output type `Yellow` to an input type `Green`, we write the implementation `impl Next on Green` and associate the `Yellow` within it.

```rust
trait Next
where
    Self: State,
    Self::Output: State
{
    type Output;
}

impl Next for Red {
     type Output = Green;
}

impl Next for Green {
     type Output = Yellow;
}

impl Next for Yellow {
     type Output = Red;
}
```

You may come up with a question: how to _call_ this kind of function?
We use the qualified syntax to achieve it.

```rust
type Outcome = <Green as Next>::Output; // Outcome == Yellow
```

The code reads "Get the associated `Output` type from `Next` trait implemented on `Red` type".
It is a bit verbose, but actually works. We can save some ink by type aliasing.

```rust
// define the alias
type NextOp<Input> = <Input as Next>::Output;

// So we can write this
type Outcome = NextOp<Green>; // Outcome == Yellow
```

That's the way the type-level programming is done in Rust. As you can see, the tiny state machine takes many lines of code.
Imagine you're going to process more complex types, such as addition of two type level numbers. Writing it could be tedious.
That's TYP comes to our rescue!

## Using TYP

With TYP, the implementation becomes more compact. It helps you write _type operators_ as though you are writing Rust as usual.

```rust
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

TYP is in fact a transcompiler. It translates the code into an actual a trait and a series of impl blocks.
You can _call_ the operator by the trait `Next` or an alias `NextOp`.

```rust
type Outcome1 = <Green as Next>::Output; // Outcome == Yellow
type Outcome2 = NextOp<Green>; // Outcome == Yellow
```

Let's see the difference from Rust. Look at the signature first.

```rust
fn Next<input>(input: State) -> State
```

Here the `input` argument is a type instead of a value, while `State` is a trait bound instead of a type.
The signature is composed into several parts.

- `Next`: The name of the type operator.
- `input`: Type generics. They are placeholders that will be filled into actual types.
- `(input: State)`: Type arguments. The `input` is the input type happens to be a generic. The bound `input: State` ensures the argument must implements the `State` trait.
- `-> State`: It restricts the output type must implement the `State` trait.


Next, look at the `match` block. It is similar to vanilla Rust, but we have to understand it differently.
It compares if the input type can be derived to one of the enumerated types.
It  does not require you to exhause all possible types, nor check if the same type is matched twice.

Our example is straightforward. We simply check `input` is one of `Green`, `Yellow`, `Red`.

```rust
match input {
    Green => Yellow,
    Yellow => Red,
    Red => Green,
}
```

You can choose to match only `Green` instead. In this case, the code only compiles only when `input` is `Green`.

```rust
match input {
    Green => Yellow,
}
```
