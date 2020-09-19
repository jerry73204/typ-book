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

<!-- TODO: insert example -->
