# Type Constructors and GATs (Level 4)

The previous levels dealt with types as parameters to functions. A generic function takes a type and produces a value. A bounded generic function takes a type satisfying a predicate and produces a value. But in all of these, the *function* is a value-level entity — something that exists (after monomorphisation) in 𝒱. The type parameter is an input to that value-level function, resolved at compile time and erased at the boundary.

Level 4 changes the subject. Here, the functions are not value-level functions parameterised by types. They are **type-level functions** — functions that take types and produce types, operating entirely within 𝒯. `Vec` is not a type. It is a function that takes a type `T` and returns the type `Vec<T>`. `Option` takes a type and returns a type. `Result` takes two types and returns a type. These are **type constructors**: the second axis of the lambda cube, types depending on types.

This is the highest level that Rust reaches. At Level 4, the programmer works not with values or with propositions about values, but with the type-level machinery itself — constructing types from types, parameterising associated types by lifetimes, encoding state machines in phantom parameters that exist purely above the boundary with zero runtime cost. The boundary erases everything at this level. The proofs, the types, the state transitions — all verified, all discarded. What remains in 𝒱 is bare computation, with the guarantee that the type-level reasoning was sound.

## Type Constructors as Functions in 𝒯

Chapter 1 introduced the second axis of the lambda cube: types depending on types. In Rust, this axis is inhabited by every generic type definition.

```rust
struct Wrapper<T> {
    inner: T,
}
```

`Wrapper` is not a type. It is a **type-level function** — a mapping from the space of types to the space of types:

```text
Wrapper: Type → Type

Wrapper(i32)    = Wrapper<i32>
Wrapper(String) = Wrapper<String>
Wrapper(bool)   = Wrapper<bool>
```

The input is a type. The output is a type. The function operates entirely within 𝒯 — no values are involved in the mapping itself. Only when a specific fibre like `Wrapper<i32>` is inhabited by a value does the mapping touch 𝒱.

Multi-parameter type constructors are functions of multiple arguments:

```text
Result: Type × Type → Type

Result(i32, String)    = Result<i32, String>
Result((), ParseError) = Result<(), ParseError>
```

And type constructors compose. `Vec<Option<T>>` is the composition of two type-level functions: first apply `Option` to `T`, then apply `Vec` to the result. In functional notation: `Vec ∘ Option`, evaluated at `T`.

```text
(Vec ∘ Option)(i32)    = Vec<Option<i32>>
(Vec ∘ Option)(String) = Vec<Option<String>>
```

This composition happens constantly in Rust — every nested generic type is a composition of type-level functions. But Rust does not let you *name* this composition abstractly. You can write `Vec<Option<T>>` for a specific `T`, but you cannot write a function that takes `Vec ∘ Option` as an argument — that would require higher-kinded types, which Rust does not have. We will return to this gap at the end of the chapter.

## The Distinction from Lower Levels

At Level 1, `fn identity<T>(x: T) -> T` is a value-level function parameterised by a type. The type `T` is an input that determines *which* function you get, but the output is a value.

At Level 4, `Vec<T>` is a type-level function. The type `T` is an input, and the output is another type. No values are involved in the mapping itself.

The distinction is:

| Level | Domain | Codomain | Rust syntax |
|---|---|---|---|
| 1–2 | Types | Values | `fn f<T>(x: T) -> T` |
| 4 | Types | Types | `struct Vec<T> { ... }` |

Level 4 operates one stratum higher in the lambda cube. At Levels 1–2, the programmer writes functions that *use* types. At Level 4, the programmer writes definitions that *produce* types. The type constructor is a tool for building the type landscape itself.

## Generic Associated Types

Associated types, introduced in Chapter 5, define a type-level function from the implementing type to another type: for each type `C: Container`, the associated type `C::Element` is determined by `C`. But ordinary associated types are fixed — they cannot themselves be parameterised.

**Generic associated types** (GATs) lift this restriction. A GAT is an associated type that takes its own parameters — type parameters or lifetime parameters — making it a *type-level function within a type-level function*.

### Lending Iterators

The motivating example for GATs in Rust is the **lending iterator**: an iterator whose items borrow from the iterator itself, rather than from an external source.

The standard `Iterator` trait cannot express this:

```rust,ignore
// Standard Iterator — Item has no connection to the iterator's lifetime
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

The associated type `Item` is fixed for each implementing type. It cannot depend on the lifetime of the borrow in `&mut self`. A lending iterator needs `Item` to be parameterised by that lifetime — which requires a GAT:

```rust
trait LendingIterator {
    type Item<'a> where Self: 'a;
    fn next(&mut self) -> Option<Self::Item<'_>>;
}
```

The associated type `Item<'a>` is a type-level function from lifetimes to types. For each lifetime `'a`, there is a corresponding `Item<'a>` type. The `where Self: 'a` bound ensures the iterator outlives the borrow.

```rust
# trait LendingIterator {
#     type Item<'a> where Self: 'a;
#     fn next(&mut self) -> Option<Self::Item<'_>>;
# }
struct Windows<'data, T> {
    data: &'data [T],
    pos: usize,
    size: usize,
}

impl<'data, T> LendingIterator for Windows<'data, T> {
    type Item<'a> = &'a [T] where Self: 'a;

    fn next(&mut self) -> Option<Self::Item<'_>> {
        if self.pos + self.size > self.data.len() {
            return None;
        }
        let window = &self.data[self.pos..self.pos + self.size];
        self.pos += 1;
        Some(window)
    }
}
```

The GAT `Item<'a> = &'a [T]` says: for each lifetime `'a`, the item type is a slice reference with that lifetime. The lifetime of each yielded item is tied to the borrow of the iterator, not to some external data structure. This is a type-level function — lifetime → type — defined within the associated type.

### Collection Families

GATs also allow expressing families of collections parameterised by their element type:

```rust
trait Collect {
    type Output<T>;
    fn collect_from<I: Iterator>(iter: I) -> Self::Output<I::Item>;
}

struct VecCollector;

impl Collect for VecCollector {
    type Output<T> = Vec<T>;
    fn collect_from<I: Iterator>(iter: I) -> Vec<I::Item> {
        iter.collect()
    }
}

struct OptionCollector;

impl Collect for OptionCollector {
    type Output<T> = Option<T>;
    fn collect_from<I: Iterator>(iter: I) -> Option<I::Item> {
        iter.last()
    }
}
```

The associated type `Output<T>` is a type-level function from `T` to a concrete collection type. For `VecCollector`, it maps every `T` to `Vec<T>`. For `OptionCollector`, it maps every `T` to `Option<T>`. The GAT captures the *pattern* of the type constructor — the shape of the container — as an abstract type-level function.

This is Level 4 in its purest form: a trait whose associated type is itself a type constructor.

### GATs in the Lambda Cube

GATs are Rust's closest approach to the second axis of the lambda cube in its full generality. An ordinary associated type is a fixed type — a point in 𝒯 determined by the implementing type. A GAT is a *function* in 𝒯 — a mapping from types (or lifetimes) to types, determined by the implementing type.

In categorical terms, a GAT is an **indexed type-level function**: for each implementing type, you get not a type but a type constructor. The `Collect` trait above maps each implementing type to a type-level function Type → Type. It is a function from types to type-level functions — a second-order construct.

## Typestate: State Machines Above the Boundary

The typestate pattern is Level 4's most distinctive practical application. It uses phantom type parameters to encode a state machine entirely within 𝒯, making invalid state transitions *unrepresentable* — not merely unchecked, but impossible to express in the type system.

### The Pattern

Consider a network connection that must go through states: created → connected → authenticated → closed. At Level 0, you might model this with an enum:

```rust
# #[allow(dead_code)]
enum ConnectionState {
    Created,
    Connected,
    Authenticated,
    Closed,
}
```

But an enum encodes *which* state the connection is in — it does not prevent invalid transitions. Nothing stops the programmer from going directly from `Created` to `Authenticated`, skipping `Connected`.

The typestate pattern encodes each state as a *type*, and the transitions as functions that consume one state and produce another:

```rust
use std::marker::PhantomData;

struct Created;
struct Connected;
struct Authenticated;

struct Connection<State> {
    address: String,
    _state: PhantomData<State>,
}

impl Connection<Created> {
    fn new(address: &str) -> Self {
        Connection {
            address: address.to_string(),
            _state: PhantomData,
        }
    }

    fn connect(self) -> Connection<Connected> {
        // ... perform connection ...
        Connection {
            address: self.address,
            _state: PhantomData,
        }
    }
}

impl Connection<Connected> {
    fn authenticate(self, _token: &str) -> Connection<Authenticated> {
        // ... perform authentication ...
        Connection {
            address: self.address,
            _state: PhantomData,
        }
    }
}

impl Connection<Authenticated> {
    fn send(&self, _data: &[u8]) {
        // ... send data ...
    }

    fn close(self) {
        // ... close connection ...
        // Connection<Authenticated> is consumed; no further use possible.
    }
}
```

Each state is a separate type (`Created`, `Connected`, `Authenticated`). The `Connection<State>` struct uses `PhantomData<State>` to carry the state in 𝒯 without any runtime representation. Each impl block applies only to connections in a specific state. The transitions — `connect`, `authenticate`, `close` — consume `self` (taking ownership) and return a connection in the new state.

The critical property: **invalid transitions do not type-check.**

```rust,ignore
let conn = Connection::new("example.com");
// conn is Connection<Created>

conn.send(&[1, 2, 3]);
// ERROR: no method named `send` found for `Connection<Created>`
// send is only available on Connection<Authenticated>

conn.authenticate("token");
// ERROR: no method named `authenticate` found for `Connection<Created>`
// authenticate is only available on Connection<Connected>
```

The programmer cannot send data before authenticating, because `send` exists only on `Connection<Authenticated>`. The programmer cannot authenticate before connecting, because `authenticate` exists only on `Connection<Connected>`. The state machine is enforced at compile time.

### Zero Cost

The typestate pattern has **zero runtime cost**. Let us trace what the forgetful functor does to this code.

`PhantomData<State>` is a zero-sized type. The compiler allocates no memory for it. In 𝒱, `Connection<Created>`, `Connection<Connected>`, and `Connection<Authenticated>` all have identical memory layouts: a `String` (the address). The state parameter exists only in 𝒯.

The transition function `connect` takes a `Connection<Created>` and returns a `Connection<Connected>`. In 𝒱, after monomorphisation, this is a function that takes a struct containing a `String` and returns a struct containing a `String` — the "state transition" is invisible. What was a type-level state change in 𝒯 is a no-op in 𝒱.

The verification — the guarantee that `send` is called only after `authenticate`, which is called only after `connect` — is performed entirely above the boundary and erased completely. This is zero-cost abstraction at its most dramatic: an entire state machine, with all its transition rules, verified at compile time and contributing exactly zero bytes and zero instructions to the runtime program.

### Typestate versus Enum State

Compare the two approaches:

| Property | Enum state (Level 0) | Typestate (Level 4) |
|---|---|---|
| Invalid transitions | Possible, caught at runtime (or not) | Impossible, rejected at compile time |
| Runtime overhead | Tag byte, branch on state | Zero — phantom types are ZSTs |
| State in 𝒯 | Partially (the enum type exists) | Fully (each state is a distinct type) |
| State in 𝒱 | Yes (the tag discriminant) | No (erased by F) |
| Flexibility | Can change state dynamically | State path fixed at compile time |

The typestate approach trades runtime flexibility for compile-time guarantee. An enum-based connection can transition dynamically (useful if the next state depends on runtime input). A typestate connection follows a fixed path — the sequence of states is determined by the code's structure, verified by the compiler, and erased.

The choice between them is a design decision about where the state logic should live: in 𝒱 (enum, with runtime dispatch and dynamic transitions) or in 𝒯 (typestate, with static verification and zero cost).

## The Boundary at Level 4

Level 4 has the most dramatic relationship with the boundary of any level. Everything at Level 4 — type constructors, GATs, phantom parameters, typestate machines — exists purely in 𝒯. The forgetful functor erases all of it.

Consider a function that exercises multiple levels:

```rust
use std::fmt;
use std::iter::FromIterator;

fn demonstrate<C, T>(items: &[T]) -> String
where
    C: FromIterator<T>,
    C: fmt::Debug,
    T: Clone,
{
    let collected: C = items.iter().cloned().collect();
    format!("{collected:?}")
}
```

This function lives simultaneously at multiple levels:

- **Level 1**: It is parameterised by `C` and `T` (universally quantified).
- **Level 2**: The bounds `C: FromIterator<T> + Debug` and `T: Clone` are predicates.
- **Level 3**: Implicit proof witnesses (`impl FromIterator<T> for C`, `impl Debug for C`, `impl Clone for T`) must exist.
- **Level 4**: `C` is being used as a type constructor — the caller specifies a container shape, and the function fills it.

When the boundary is crossed:

```rust
# use std::fmt;
# use std::iter::FromIterator;
# fn demonstrate<C, T>(items: &[T]) -> String
# where
#     C: FromIterator<T>,
#     C: fmt::Debug,
#     T: Clone,
# {
#     let collected: C = items.iter().cloned().collect();
#     format!("{collected:?}")
# }
fn main() {
    let data = [1, 2, 3];
    let as_vec = demonstrate::<Vec<i32>, i32>(&data);
    let as_set = demonstrate::<std::collections::BTreeSet<i32>, i32>(&data);
    println!("{as_vec}");
    println!("{as_set}");
}
```

Monomorphisation generates two versions: one for `Vec<i32>` and one for `BTreeSet<i32>`. In each version:
- The type parameters `C` and `T` are resolved to concrete types (Level 1 → Level 0).
- The bounds are verified and erased (Level 2 → gone).
- The proof witnesses are consumed and inlined (Level 3 → gone).
- The type constructor `C` is resolved to a specific container (Level 4 → Level 0).

What remains in 𝒱 is two concrete functions — each collecting elements into a specific container and formatting the result. The entire generic machinery, from type constructors to proof witnesses, has been verified and discarded. The functions in 𝒱 are indistinguishable from hand-written Level 0 code.

## Higher-Kinded Types: What Rust Cannot Express

Level 4 is where Rust reaches the edge of its expressive power. The limitation becomes visible when you try to abstract over *type constructors themselves* — when you want to write code that is generic not just in a type, but in a type-level function.

### The Problem

Consider the `map` operation. Many types support it:

```rust
fn map_option<A, B>(opt: Option<A>, f: impl FnOnce(A) -> B) -> Option<B> {
    opt.map(f)
}

fn map_vec<A, B>(v: Vec<A>, f: impl FnMut(A) -> B) -> Vec<B> {
    v.into_iter().map(f).collect()
}

fn map_result<A, B, E>(
    res: Result<A, E>,
    f: impl FnOnce(A) -> B,
) -> Result<B, E> {
    res.map(f)
}
```

Three functions, all expressing the same idea: apply a function inside a container. The pattern is identical; only the container differs. At Level 2, we would abstract over the element type with a bound. But here we need to abstract over the *container* — and the container is a type constructor, not a type.

In Haskell, this abstraction is straightforward:

```text
-- Haskell
class Functor f where
    fmap :: (a -> b) -> f a -> f b
```

The type variable `f` ranges over type constructors — things of kind `* -> *` (functions from types to types). `f` can be `Maybe` (Haskell's `Option`), `[]` (Haskell's `Vec`), `Either e` (Haskell's `Result<_, E>`), and so on. The class abstracts over the container shape itself.

Rust cannot express this directly. The generic parameter in a trait bound must be a *type*, not a type constructor. You cannot write:

```text
// Not valid Rust — illustrative pseudocode
trait Functor<F: * -> *> {
    fn fmap<A, B>(fa: F<A>, f: impl FnOnce(A) -> B) -> F<B>;
}
```

The parameter `F` would need to be a type-level function — a higher-kinded type. Rust's type system does not have a way to quantify over type constructors. You can quantify over types (`<T>`), but not over type-level functions (`<F: * -> *>`).

### GATs as a Partial Workaround

GATs provide a limited workaround. Instead of parameterising the trait by a type constructor, you can define a trait with a GAT that *acts* as a type constructor:

```rust
trait Mappable {
    type Item;
    type Output<U>;

    fn map_into<U>(self, f: impl FnMut(Self::Item) -> U) -> Self::Output<U>;
}

impl<T> Mappable for Option<T> {
    type Item = T;
    type Output<U> = Option<U>;

    fn map_into<U>(self, f: impl FnMut(T) -> U) -> Option<U> {
        self.map(f)
    }
}

impl<T> Mappable for Vec<T> {
    type Item = T;
    type Output<U> = Vec<U>;

    fn map_into<U>(self, mut f: impl FnMut(T) -> U) -> Vec<U> {
        self.into_iter().map(&mut f).collect()
    }
}
```

The GAT `Output<U>` acts as a type constructor: for `Option<T>`, it maps `U` to `Option<U>`. For `Vec<T>`, it maps `U` to `Vec<U>`. A generic function can now abstract over the container:

```rust
# trait Mappable {
#     type Item;
#     type Output<U>;
#     fn map_into<U>(self, f: impl FnMut(Self::Item) -> U) -> Self::Output<U>;
# }
# impl<T> Mappable for Option<T> {
#     type Item = T;
#     type Output<U> = Option<U>;
#     fn map_into<U>(self, f: impl FnMut(T) -> U) -> Option<U> { self.map(f) }
# }
# impl<T> Mappable for Vec<T> {
#     type Item = T;
#     type Output<U> = Vec<U>;
#     fn map_into<U>(self, mut f: impl FnMut(T) -> U) -> Vec<U> {
#         self.into_iter().map(&mut f).collect()
#     }
# }
fn stringify<M: Mappable>(container: M) -> M::Output<String>
where
    M::Item: std::fmt::Display,
{
    container.map_into(|x| x.to_string())
}

fn main() {
    let nums = vec![1, 2, 3];
    let strs: Vec<String> = stringify(nums);
    assert_eq!(strs, vec!["1", "2", "3"]);

    let maybe = Some(42);
    let s: Option<String> = stringify(maybe);
    assert_eq!(s, Some("42".to_string()));
}
```

This works — `stringify` operates generically over any mappable container. But there are limitations. The GAT approach ties the type constructor to the *implementing type*, not to a standalone parameter. You cannot easily express "apply any type constructor `F` to any type `T`" — you can only express "this specific implementing type has an associated constructor." The abstraction is one level less general than full higher-kinded types.

### The Functor/Monad Gap

The inability to abstract over type constructors has a well-known consequence: Rust cannot define `Functor`, `Applicative`, or `Monad` as traits in the way Haskell does.

In Haskell, the monad abstraction is:

```text
-- Haskell
class Monad m where
    return :: a -> m a
    (>>=)  :: m a -> (a -> m b) -> m b
```

The variable `m` ranges over type constructors. `m` can be `Maybe`, `IO`, `Either e`, `[]`, or any other type constructor of kind `* -> *`. This single abstraction powers Haskell's `do` notation, its effect handling, its entire approach to sequencing computations.

Rust has no equivalent. The `?` operator works on `Option` and `Result` via compiler desugaring to pattern matching, not via a general monad abstraction. Iterator chains use a concrete `Iterator` trait, not a functor/applicative tower. Each "monad-like" type in Rust — `Option`, `Result`, `Vec`, `Future` — provides its own `map`, `and_then`, and `flatten` methods, with no shared trait.

This is not a deficiency in Rust's design. It is a consequence of Rust's position in the lambda cube. Full higher-kinded types live on the second axis in its full generality — the ability to quantify over type-level functions of arbitrary kind. Rust occupies this axis partially: it has type constructors (you can *define* `Vec<T>`), but it cannot quantify over them (you cannot write `<F: * -> *>`). GATs extend the reach, but the gap remains.

The practical impact is that Rust favours *concrete abstraction* over *abstract abstraction*. Rather than a single `Monad` trait that unifies all sequential computation, Rust has `Option::and_then`, `Result::and_then`, `Future::then`, each concrete, each zero-cost, each optimised for its specific use case. The type-level generality is sacrificed; the runtime efficiency is preserved.

## What Level 4 Reveals

The Level 2 programmer thinks about types and their bounds. The Level 4 programmer thinks about **type-level computation** — functions that produce types, parameters that carry no runtime data, state machines that exist only in 𝒯.

This perspective reveals the final layer of Rust's type-theoretical structure:

**Type constructors are not types — they are functions.** When you write `Vec`, you are not naming a type. You are naming a type-level function that, given a type, produces a type. The distinction matters because it determines what you can and cannot abstract over. You can abstract over `T` (the input); you cannot abstract over `Vec` (the function) without GATs or full HKT.

**PhantomData is a zero-cost proof carrier.** A phantom parameter adds structure in 𝒯 without cost in 𝒱. Typestate machines exploit this to encode invariants that are verified at compile time and erased at runtime. The boundary between 𝒯 and 𝒱 is what makes this possible: the phantom parameter is real structure in 𝒯, and the forgetful functor's guarantee of erasure is what makes it zero-cost.

**GATs extend the expressiveness of traits by one level.** An ordinary associated type is a type (a point in 𝒯). A GAT is a type constructor (a function in 𝒯). This extra level of abstraction is enough for lending iterators, collection families, and other patterns that require the associated type to vary with an additional parameter. It is not enough for full higher-kinded types.

**Rust's boundary determines its position in the lambda cube.** The zero-cost abstraction guarantee requires that all type-level structure be resolvable at compile time. Higher-kinded types, in their full generality, would require the compiler to reason about type-level functions as abstract entities — not just apply them to specific arguments. Rust's monomorphisation strategy resolves every type-level function to a concrete result before crossing the boundary. This works for type constructors applied to specific types; it does not generalise to quantification over type constructors.

Level 4 is the ceiling. Above it lie the territories that Rust does not enter — dependent types, universe polymorphism, the full Calculus of Constructions. Chapter 8 examines the boundary itself: the forgetful functor F: 𝒯 → 𝒱 in its full depth, the mechanisms of erasure, and the selective permeability that allows certain information to cross between the two categories. Chapter 9 then maps the landscape beyond the ceiling, placing Rust precisely within the space of possible type systems.
