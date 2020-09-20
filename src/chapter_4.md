# Type Composition

Type composition is a useful feature to glue up types together.
The chapter will present tiny examples to show several ways to compose types.

## Type Substitution

TYP always substitutes variables in generic places to actual types.
For example, we can write a operator that build complex types.

```rust
typ! {
    fn Wrap<input>(input: _) {
        let first = Option::<Box<input>>;
        let second = Box::<Option<input>>;
        (first, second)  // produces (Option<Box<input>>, Box<Option<input>>)
    }
}
```

Note that there is no move semantics and no borrow checker in TYP. Whenever a substitution occurs, it simply copies the type.
In our example, it copies the `input` value to `first` and `second.`

```
let first = Option::<Box<input>>;
let second = Box::<Option<input>>;
```

## Work with Traits

TYP also supports type generics on traits. For example, let's add up two types using standard library's `Add`.

```rust
use core::ops::Add;

typ! {
    fn ComputeAdd<input>(lhs: _, rhs: _) {
        <lhs as Add<rhs>>::Output
    }
}
```

The type `<lhs as Add<rhs>>::Output` reads "Get the associated type `Output` of `Add<rhs>` trait implemented on `lhs` type". It is a bit verbose.
It can be abbreviated to the code below. Here `lhs.Add(rhs)` is a syntactic sugar of `<lhs as Add<rhs>>::Output`. You can write it as though you're calling a trait.

```rust
typ! {
    fn ComputeAdd<input>(lhs: _, rhs: _) {
        lhs.Add(rhs)
    }
}
```

Furthermore, we can compose several several type operators together by _calling_ them.

```rust
typ! {
    fn MakeBox<input>(input: _) {
        Box::<input>
    }

    fn MakeOption<input>(input: _) {
        Option::<input>
    }

    fn MakeOptionBox<input>(input: _) {
        MakeOption(MakeBox(input))
    }
}
```

The call `MakeBox(input)` is in fact an abbreviation of `<() as MakeBox<input>>::Output`.
Hence, the following writing is basically the same.

```rust
fn MakeOptionBox<input>(input: _) {
    <() as MakeOption<<() as MakeBox<input>>::Output>>::Output
}
```
