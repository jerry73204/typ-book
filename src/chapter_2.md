# Advanced Matching

In this chapter, we are going to construct a list, which contains arbitrary number of distinct types.
Just like `Vec<T>` but in type level. We will learn the concept of advanced matching and recursive trait operators.

We construct the typed list with two components: an intermediate node and an end-of-list marker.

- `Cons<Item, Next>` is an intermediate node. The `Item` holds the type stored in this node, while `Next` is the remaining list.
- `Nil` is the end-of-list marker.

So that we can have a list of types like this. You can chain up the list as long as you can.

```rust
type L = Cons<u8, Cons<u16, Cons<u32, Nil>>>;
```

Here we realize `Cons` and `Nil` as zero-sized structs.
A small detail is that since `Cons<Item, Next>` has two generics, Rust requires the generics must appear in the fields of the struct.
We use `PhantomData` to avoid storing actual data of generic types.

```rust
trait List {}
impl<Item, Next: List> List for Cons<Item, Next> {}
impl List for Nil {}

struct Cons<Item, Next: List> {
    _phantom: PhantomData<(Item, Next)>
}
struct Nil;
```

Let's start simple. How to obtain the first item of the list?
If we write the Rust as usual, we need to check if the list is empty before we access the first item.
Here we go on a different mindset. Instead of checking the list explicitly, we only accept non-empty lists by matching the type.

To be precise, non-empty lists cannot be `Nil`, but can be `Cons<X, Nil>` or `Cons<X, Cons<Y, Nil>>` and so on.
We can ensure the input list has the pattern `Cons<item, next>`, so it can be written as the following.

```rust
typ! {
    pub fn First<item, next: List>(Cons::<item, next>: List) {
        item
    }
}
```

To verify our implementation actually works, we need a utility trait called `Same` here.

```rust
pub trait Same<Lhs, Rhs> {
    type Output;
}
impl<T> Same<T, T> for () {
    type Output = ();
}

type SameOp<Lhs, Rhs> = <() as Same<Lhs, Rhs>>::Output;
```

Now you can check if the code compiles.

```rust
type Input = Cons<u8, Cons<u16, Cons<u32, Nil>>>;
type Outpu = FirstOp<List>;
type Void = SameOp<Output, u8>;
```

How about accessing the last item of the list? Apart from empty lists, we check if we reach the last node of the list.
We can distinguish the cases that the list has exactly one or more items. If it has one, return the item immediately.
If it has more than one items, rip off the first item and recursively find the last item in the remaining list.

To sum up, we match the list to following two cases. The `Nil` case is not counted here because the list cannot be empty.

- `Cons::<first, Nil>`: The list contains exactly one item, which is also the last item.
- `Cons::<first, Cons<second, next>>`: The list contains at least two items.


We match the types by `match`.

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

You may notice the extra per-arm attributes like this.

```rust
#[generics(first, second, next: List)]
Cons::<first, Cons::<second, next>> => { /* .. */ }
```

That's because when we try to match `Cons::<first, Cons::<second, next>>`, the types of the places `first`, `second` and `next` are not known yet.
We have to tell the compiler that the names are generic types than can filled in yet.




Let's move on the next example. Given two lists, can we assert that their first items are the same type?
For example, it's true for `Cons<u8, Nil>` and `Cons<u8, Cons<u16, Nil>>`, but not for `Cons<u8, Nil>` and `Cons<u16, Cons<u32, Nil>>`.

To say two lists have the same first item, they can be respectively decomposed into the form `Cons::<item, lhs_next>` and `Cons::<item, rhs_next>`.
The same name `item` on both sides marks they are the same type. It can be done in two steps.

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

You can see when we're matching the second list, the attribute `#[capture(item)]` tells that the `item` in the pattern `Cons::<item, rhs_next>` comes from the first list.
In this way, the code compiles only when both lists are non-empty and have the same first item.
