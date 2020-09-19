# Work with Typenum

TYP has first-class support to [typenum](https://docs.rs/typenum/).

## Integer Literals

TYP understands two kinds of integer literals: signed and unsigned.

- `3` or `3i` is a signed type-level integer.
- `3u` is an unsigned type-level integer.

The literals are translated into typenum types. For example, `3i` becomes `PInt<...>`, and `3u` becomes `UInt<...>`.
The maximum value can be up to the maximum of `u128` regardless of signed or unsigned type.

```rust
use typenum::{Integer, Unsigned};

typ! {
    fn GetSignedThree() -> Integer {
        3i
    }

    fn GetUnsignedThree() -> Unsigned {
        3u
    }
}
```

## Operators

Common binary operators `+`, `-`, `*`, `/`, `%` are understood by TYP.

```rust
use typenum::{Integer, Unsigned};

typ! {
    fn IntegerAdd<lhs, rhs>(lhs: Integer, rhs: Integer) -> Integer {
        lhs + rhs
    }
}
```

The comparison operators `<`, `<=`, `>`, `>=`, `==` compare the types and emit `typenum::B0` or `typenum::B1` boolean values that implement `typenum::Bit`.
The outcome can be used in conjunction with `if/else`.

```rust
use typenum::Bit;

typ! {
    fn Abs(input: Integer) -> Integer {
        let is_non_negative: Bit = input >= 0;
        if is_non_negative {
            input
        } else {
            -input
        }
    }
}
```

The logic operators `!`, `&`, `|` applies on types that implement `typenum::Bit` trait. TYP treats `&&` and `||` the same as `&`and `|`.
For example, we can define an xor function.

```rust
use typenum::Bit;

typ! {
    fn Xor(lhs: Bit, rhs: Bit) -> Bit {
        (!lhs && rhs) || (lhs && !rhs)
    }
}
```
