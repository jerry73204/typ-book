# Type Operators and Trait Bounds

_Type operators_ are type-level functions that transform the types.
You can define variables with appropriate trait bounds, and construct the control flow inside the operator.
They are eventually translated to traits. The chapter will cover the basic components and explain how the translation works.

## The Function Signature

Let's continue to the revised version traffic light state transition function.
It takes an input state and a boolean value that determines whether to change the state,
and produces output accordingly.

```rust
use typenum::Bit;

typ! {
    fn Next<input, change>(input: State, change: Bit) -> State {
        let output: State = if change {
            match input {
                Green => Yellow,
                Yellow => Red,
                Red => Green,
            }
        } else {
            input
        };
        output
    }
}
```

The function signature is comprised of these parts:

- **Name** `Next`.
- **Type generic list** `<input, change>`. They are placeholders to be filled in types.
- **Type argument list** `(input: State, chagne: Bit)`. The colon in `input: State` means `input` has a `State` trait bound.
- **Output trait bound** `-> State` restricts the output type must implement the `State` trait. It is optional.

You can see the type arguments `(input: State, change: Bit)` repeat the generics `<input, change>`.
That's because the arguments are happened to be generics in this example. Type arguments can be more than generics.
We'll discuss it in later chapters.

## Invoke the Type Operator

The `typ!` macro translate type operators into traits. It becomes the form:

```rust
trait Next<arguments, ..> {
    type Output;
}

impl<generics, ..> Next<arguments, ..> for () {
    type Output = /* computation.. */;
}
```

There are two ways to invoke the computation: Using the trait directly or using the type alias.

```rust
use typenum::{B0, B1};

type Outcome1 = <() as Next<Green, B1>>::Output; // Outcome1 == Yellow
type Outcome2 = NextOp<Yellow, B0>; // Outcome2 == Red
```


## Trait Bounds

The first places to put trait bounds is inside the function signature.
You can add trait bounds to generics, type arguments, return type and extra where clauses.
We can extend the example like the following. Many trait bounds are repeated but here is just for example.

```rust
fn Next<input: State, change: Bit>(input: State, change: Bit) -> State
where
    input: State,
    change: Bit,
```

You can also omit the trait bounds. In argument places, put the underscore `_` to omit trait bounds.

```rust
fn Next<input, change>(input: _, change: _)
```

Another place to put trait bounds is on the local variables.
In our example, you can find:

```rust
let output: State = if change { .. } else { .. };
```

## Translation of Type Operators

Type oeprators are translated to a series of traits and impl blocks with appropriate trait bounds.
It covers the following items:

- The main trait `trait Next` for the entry point of the computation.
- The type alias `type NextOp` for invoking the main trait.
- Other traits and impl items that are necessary for `if` and `match` branching.

Only the main trait and its alias are public, while others are wrapped inside a private module.
So after the `typ!` macro expansion, our example will look like the following.


```rust
type NextOp<input, change> = <() as Next<input, change>>::Output; // the type alias
use __TYP_mod_Next::Next; // the trait

// actual type computations are wrapped inside a mod
mod __TYP_mod_Next {
    pub trait Next<input, change>
    where
        input: State,
        change: Bit,
        Self::Output: State
    {
        type Output;
    }

    impl<input, change> Next<input, change> for ()
    where
        input: State,
        change: Bit,
        (): NextIf<input, change, change>,
        <() as NextIf<input, change, change>>::Output: State,
    {
        type Output = <() as NextIf<input, change, change>>::Output;
    }

    pub trait NextIf<input, change, if_condition>
    where
        input: State,
        change: Bit,
        if_condition: Bit,
        Self::Output: State
    {
        type Output;
    }

    impl<input, change> NextIf<input, change, B1> for ()
    where
        input: State,
        change: Bit,
        ... // omit
    {
        type Output = /* omit */;
    }

    impl<input, change> NextIf<input, change, B0> for ()
    where
        input: State,
        change: Bit,
        ... // omit
    {
        type Output = /* omit */;
    }
}
```
