# Type Matching

In this chapter, we are going to construct a list, which contains arbitrary number of distinct types.
Similar to `Vec<T>`, but in type level. We will learn the concept of advanced matching and recursive trait operators.

## Type-level List

We construct the typed list with two components: an intermediate node and an end-of-list marker.

- `Cons<Item, Next>` is an intermediate node. The `Item` holds the type stored in this node, while `Next` is the remaining list.
- `Nil` is the end-of-list marker.

We can have a list of types like this.

```rust
type L = Cons<u8, Cons<u16, Cons<u32, Nil>>>;
```

We define `Cons` and `Nil` as zero-sized structs shown below.
A small detail is that because `Cons<Item, Next>` has two generics, Rust requires at least one field has the generic type.
We use `PhantomData` to avoid storing actual data.

```rust
trait List {}
impl<Item, Next: List> List for Cons<Item, Next> {}
impl List for Nil {}

struct Cons<Item, Next: List> {
    _phantom: PhantomData<(Item, Next)>
}
struct Nil;
```

## Access the First Element: Argument Type Matching

How to obtain the first item of the list?
If we write the Rust as usual, we need to check if the list is empty before we access the first item.
Here we go on a different mindset. Instead of checking the list explicitly, we only accept non-empty lists by matching the type.

To elborate, non-empty lists can be `Cons<X, Nil>` or `Cons<X, Cons<Y, Nil>>` but cannot be `Nil`.
We can ensure the input list has the pattern `Cons<item, next>` in the argument.

```rust
typ! {
    pub fn First<item, next: List>(Cons::<item, next>: List) {
        item
    }
}
```

To verify our implementation, we need a utility trait `Same` that compares if two types are equal.

```rust
pub trait Same<Lhs, Rhs> {
    type Output;
}
impl<T> Same<T, T> for () {
    type Output = ();
}

type SameOp<Lhs, Rhs> = <() as Same<Lhs, Rhs>>::Output;
```

Now we make use of `Same` to see if `FirstOp` actually works.

```rust
type Input = Cons<u8, Cons<u16, Cons<u32, Nil>>>;
type Outpu = FirstOp<List>;
type Void = SameOp<Output, u8>;
```

## Access the Last Element: In-place Generics and Recursion

How about accessing the last item of the list? Apart from empty lists, we check not only if the list is empty,
but also check if last node of the list is reached.
We can consider the cases that the list has exactly one or more than one items. That is, we expect the list is one of the following two types.

- `Cons::<first, Nil>`: The list contains exactly one item, which is also the last node.
- `Cons::<first, Cons<second, next>>`: The list contains at least two items.

We use `match` for type matching. If it has one item, return the type immediately. Otherwise, start a recursive call on remaining list to find the last item.

```rust
typ! {
    fn Last<input>(input: List) {
        match input {
            #[generics(first)]
            Cons::<first, Nil> => first,
            #[generics(first, second, next: List)]
            Cons::<first, Cons::<second, next>> => {
                let remaining: List = Cons::<second, next>;
                Last(remaining)
            }
        }
    }
}
```

You can see the extra attribute like this.

```rust
#[generics(first, second, next: List)]
Cons::<first, Cons::<second, next>> => { /* .. */ }
```

That's because when we try to match `Cons::<first, Cons::<second, next>>`, the types of the `first`, `second` and `next` are not known yet.
The attribute tells the compiler that the names are type generics rather than exact types.


## Type Captureing

Let's move on the next example. Given two lists, can we assert that their first items are the same type?
It's true for `Cons<u8, Nil>` and `Cons<u8, Cons<u16, Nil>>` for example, but not for `Cons<u8, Nil>` and `Cons<u16, Cons<u32, Nil>>`.

To say the first items are the same , we respectively decompose the first and second lists into the forms `Cons::<item, lhs_next>` and `Cons::<item, rhs_next>`.
The same name `item` on both sides marks they are the same type.

Our implementation goes in two steps.

1. Match if the first list has the type `Cons::<item, lhs_next>`.
2. Match if the second list has the type `Cons::<item, rhs_next>`, where `item` comes from the first list.

Let's see how it can be done in code.

```rust
typ! {
    fn Last<lhs, rhs>(lhs: List, rhs: List) {
        match lhs {
            #[generics(item, lhs_next: List)]
            Cons::<item, lhs_next> => {
                match rhs {
                    #[generics(rhs_next: List)]
                    #[capture(item)]
                    Cons::<item, rhs_next> => ()
                }
            }
        }
    }
}
```

The attribute `#[capture(item)]` tells that the `item` in the pattern `Cons::<item, rhs_next>` comes from the first list.
In this way, the code compiles only when both lists are non-empty and have first items are the same type.
