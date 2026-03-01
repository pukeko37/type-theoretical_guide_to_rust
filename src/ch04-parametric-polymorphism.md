# Parametric Polymorphism (Level 1)

At Level 0, every type is fixed and every function names its types explicitly. The price of this concreteness is repetition: the same logic, restated for each type it applies to. We ended the previous chapter staring at that ceiling — identical function bodies differing only in the types they mention, identical impl blocks differing only in the types they prove propositions about.

Level 1 lifts this ceiling. It introduces a single new mechanism — the type parameter with no bounds — and in doing so, it moves from the simply typed lambda calculus (λ→) to **System F** (λ2), Girard and Reynolds' polymorphic lambda calculus. A function at Level 1 does not operate on a specific type. It operates on *all* types simultaneously, uniformly, without knowing anything about which type it has been given.

This sounds like freedom. It is, in fact, the opposite. The absence of any constraint on `T` is itself the most powerful constraint of all: it means the implementation can do almost nothing. And it is precisely this inability — this enforced ignorance of what `T` is — that gives Level 1 its remarkable properties. At this level, the type signature alone determines the implementation. The type *is* the theorem, and the theorem admits essentially one proof.

## System F: The Formal Foundation

Chapter 1 introduced System F as the second vertex of the lambda cube — the system obtained by adding the axis "terms depending on types" to the simply typed lambda calculus. Let us now examine what this system actually permits.

In System F, a value can be parameterised by a type. The type of such a value is written with a universal quantifier:

```text
∀T. T → T
```

This reads: "for all types T, a function from T to T." The ∀ binds the type variable T across the entire expression. A value of this type must be a function that, given *any* type whatsoever, takes a value of that type and returns a value of the same type.

In Rust:

```rust
fn identity<T>(x: T) -> T {
    x
}
```

The angle-bracket syntax `<T>` is Rust's notation for the universal quantifier. The function `identity` does not have a single type like `fn(i32) -> i32`. It has a *universally quantified* type: for every type `T`, it is a function from `T` to `T`. The compiler instantiates this universal at each call site — `identity::<i32>`, `identity::<String>`, `identity::<Vec<bool>>` — but the definition is written once, for all types.

This is the first axis of the lambda cube made concrete: a value (the function) depending on a type (the parameter `T`). Level 0 had no such mechanism. Level 1 has nothing else.

## The Meaning of "No Bounds"

The defining characteristic of Level 1 is the absence of trait bounds on the type parameter. When you write `<T>` with no `: Bound` clause, you are making a universal claim with no hypotheses. Under Curry-Howard, this is the proposition ∀T. P(T) — for *all* types, unconditionally.

What can you do with a value of type `T` when you know nothing about `T`?

You cannot compare it. Comparison requires `Ord` or `PartialOrd` — that is a bound.
You cannot print it. Printing requires `Display` or `Debug` — those are bounds.
You cannot clone it. Cloning requires `Clone` — that is a bound.
You cannot drop it and construct a new one. Construction requires knowing the type's structure — but `T` is opaque.
You cannot hash it, serialise it, add it to another value, or test it for equality.

You can do exactly four things with a value of unknown type `T`:

1. **Move it.** Transfer ownership from one binding to another.
2. **Store it.** Place it into a generic container — a struct field, a tuple, a `Vec`.
3. **Return it.** Give it back to the caller.
4. **Pass it to another function** that also accepts `T` with no bounds.

These four operations — move, store, return, delegate — are the complete vocabulary of Level 1. They are the operations that require no knowledge of the value whatsoever, only that it exists and has some type.

```rust
fn first_of_two<T>(a: T, _b: T) -> T {
    a
}

fn second_of_two<T>(_a: T, b: T) -> T {
    b
}

fn wrap_in_option<T>(x: T) -> Option<T> {
    Some(x)
}

fn pair<A, B>(a: A, b: B) -> (A, B) {
    (a, b)
}
```

Each of these functions operates at Level 1: fully generic, no bounds, nothing known about the type parameters. Each one can only rearrange, package, or select among its arguments. None can inspect, transform, or create values of the unknown type.

## Parametricity: The Constraint of Ignorance

In 1983, John Reynolds formalised the consequences of this enforced ignorance in his paper "Types, Abstraction, and Parametric Polymorphism." The result — **relational parametricity** — is the theoretical backbone of Level 1.

The key insight is this: because a Level 1 function knows nothing about its type parameter, it must behave *uniformly* across all instantiations. It cannot branch on what `T` is. It cannot special-case `T = i32` to do one thing and `T = String` to do another. The function's behaviour with respect to `T` is fixed by its type signature — not by its implementation.

Reynolds stated this precisely using relations. Consider a function `f` of type `∀T. T → T`. Parametricity says: for *any* two types `A` and `B`, and *any* relation `R` between values of `A` and values of `B`, if `a: A` and `b: B` are related by `R`, then `f(a)` and `f(b)` are also related by `R`.

In other words, `f` preserves all possible relationships between types. The only function that preserves *every* possible relationship between its input and output, for every possible type, is the function that does nothing: the identity.

This is why `fn identity<T>(x: T) -> T` has exactly one sensible implementation. The type ∀T. T → T admits precisely one inhabitant (up to observational equivalence): the function that returns its argument unchanged. Any other behaviour would violate parametricity — it would require distinguishing between types, which the absence of bounds forbids.

### A Semi-Formal Argument

Let us see why this is the case without the full relational machinery. Suppose you attempted to write a different implementation:

```rust
fn not_identity<T>(x: T) -> T {
    // What could we write here that differs from `x`?
    // We need to produce a value of type T.
    // We cannot construct a new T — we do not know its structure.
    // We cannot modify x — we have no operations on T.
    // The only T-typed value available is x itself.
    x
}
```

The function must return a `T`. The only value of type `T` in scope is `x`. There is no other way to obtain a `T` — you cannot conjure one from nothing, because you do not know what `T` is. So you must return `x`.

This argument generalises. For any Level 1 function, the implementation is determined (up to choice among available values) by the type signature alone. The types constrain the plumbing so tightly that there is often only one way to connect the inputs to the output.

## Theorems for Free

Philip Wadler, building on Reynolds' parametricity, published "Theorems for Free!" in 1989, demonstrating that the type signature of a polymorphic function implies non-trivial properties of its behaviour — properties that hold without examining the implementation.

Let us derive several such theorems for Rust functions.

### Theorem 1: The Identity

**Type:** `fn<T>(T) -> T`

**Free theorem:** The function must return its argument unchanged.

We have already seen why. There is exactly one value of type `T` in scope, and it must be the return value. The identity function is the *only* function of this type.

### Theorem 2: Constant Selection

**Type:** `fn<T>(T, T) -> T`

**Free theorem:** The function must return one of its arguments — either always the first, or always the second.

```rust
fn choose_first<T>(a: T, _b: T) -> T { a }
fn choose_second<T>(_a: T, b: T) -> T { b }
```

These are the only two implementations. The function cannot construct a new `T`, cannot combine its arguments (no operations on `T`), and cannot decide at runtime which to return (that would require inspecting `T`, which is forbidden). The choice is baked into the code at definition time.


[!Note]
A caveat on totality. In Rust, which allows divergence, the precise statement is: if the function returns, it returns one of its arguments, and the choice does not depend on the type T. Parametricity theorems hold modulo termination and effects — they constrain the function's behaviour under the assumption that it terminates normally. The function may also loop or panic, which are escape routes that parametricity does not rule out.

### Theorem 3: List Rearrangement

**Type:** `fn<T>(Vec<T>) -> Vec<T>`

**Free theorem:** The function can only rearrange, duplicate, or drop elements. It cannot create new elements or inspect existing ones.

This is a powerful result. Without reading a single line of implementation, you know from the type alone that a function `Vec<T> -> Vec<T>` (with no bounds on `T`) can:
- Reverse the vector
- Shuffle it (if it uses external randomness)
- Take the first N elements
- Repeat elements
- Return an empty vector
- Return the vector unchanged

It *cannot*:
- Sort the vector (requires `Ord`)
- Deduplicate (requires `Eq`)
- Insert a default value (requires `Default`)
- Filter by a predicate on elements (requires inspecting `T`)

Every operation that examines the *content* of elements requires a bound. Level 1 can only manipulate the *structure* of the container.

```rust
fn reverse<T>(mut items: Vec<T>) -> Vec<T> {
    items.reverse();
    items
}

fn take_first_two<T>(items: Vec<T>) -> Vec<T> {
    items.into_iter().take(2).collect()
}

fn duplicate_all<T: Clone>(items: Vec<T>) -> Vec<T> {
    // Wait — this requires Clone!
    // This function is Level 2, not Level 1.
    items.iter().cloned().cycle().take(items.len() * 2).collect()
}
```

Notice the trap: `duplicate_all` appears to be a structural operation, but it requires `Clone` — a bound — because duplicating a value means producing a second copy, and copying requires knowledge of how to copy. At Level 1, each value of type `T` is unique and unreproducible. You can move it, but you cannot multiply it. This is a direct consequence of Rust's ownership semantics interacting with parametricity: without `Clone`, every `T` value exists exactly once.

### Theorem 4: Pair Manipulation

**Type:** `fn<A, B>((A, B)) -> (B, A)`

**Free theorem:** The function must swap the pair components.

```rust
fn swap<A, B>(pair: (A, B)) -> (B, A) {
    (pair.1, pair.0)
}
```

The output must contain one value of type `B` and one of type `A`. The only `B` available is `pair.1`; the only `A` is `pair.0`. There is exactly one way to construct the output.

### Theorem 5: Function Application

**Type:** `fn<A, B>(A, fn(A) -> B) -> B`

**Free theorem:** The function applies its second argument to its first.

```rust
fn apply<A, B>(value: A, f: fn(A) -> B) -> B {
    f(value)
}
```

We must produce a `B`. We have no `B` value directly, but we have a function from `A` to `B` and a value of type `A`. The only way to obtain a `B` is to apply `f` to `value`. The implementation is forced.

## The Parametricity Boundary

Theorems for free rest on a critical assumption: the function cannot inspect or discriminate on the type parameter. In a language with runtime type inspection — Java's `instanceof`, Go's type switches, C's `void*` casting — parametricity breaks down. A function of type `∀T. T → T` in Java can check at runtime whether `T` is `String` and return a different string.

Rust largely preserves parametricity. A generic function *cannot* branch on the concrete type of its parameter. There is no `instanceof`, no runtime type switch, no way to downcast a generic `T` to a specific type.

But there are exceptions — narrow channels where type identity leaks through:

**`TypeId::of::<T>()`** returns a runtime identifier for a type. A function bounded by `T: 'static` can call `TypeId::of::<T>()` and compare it against known type IDs:

```rust
use std::any::TypeId;

fn is_string<T: 'static>(_x: &T) -> bool {
    TypeId::of::<T>() == TypeId::of::<String>()
}
```

This function violates parametricity: it distinguishes `String` from other types at runtime. Note, however, that it requires the `'static` bound — it is not a Level 1 function. Pure Level 1 (no bounds at all) preserves parametricity fully. The leak occurs only when the `'static` bound opens the `TypeId` channel.

**Specialisation** (unstable) would allow different implementations for different type parameters, breaking parametricity by design. It remains a nightly-only feature precisely because of the tension between specialisation and the parametric guarantees that stable Rust preserves.

For the purposes of this chapter, the rule is: **at Level 1, with no bounds whatsoever, parametricity holds.** The type signature determines the implementation. The theorems are free.

## Generic Data Structures at Level 1

Level 1 is not only about functions. It is equally about data structures — generic types that hold values of an unknown type.

```rust
struct Slot<T> {
    value: T,
}

impl<T> Slot<T> {
    fn new(value: T) -> Self {
        Slot { value }
    }

    fn into_inner(self) -> T {
        self.value
    }

    fn replace(&mut self, new_value: T) -> T {
        std::mem::replace(&mut self.value, new_value)
    }
}
```

The `impl<T> Slot<T>` block is a Level 1 impl: it applies to `Slot<T>` for *all* `T`, with no bounds. Every method in this block can only move, store, or return values of type `T`. The `replace` method uses `std::mem::replace` — itself a Level 1 function in the standard library — to swap the stored value with a new one, returning the old value without ever inspecting it.

This is the essence of a Level 1 container: it is structure without content. A `Slot<T>` holds a `T`-shaped hole into which any type fits. The container's behaviour — storing, retrieving, replacing — is entirely independent of what fills the hole.

The standard library's most fundamental generic types are Level 1 at their core:

- `Option<T>` — a slot that may be empty
- `Vec<T>` — a growable sequence of slots
- `Box<T>` — a heap-allocated slot
- `(A, B)` — a pair of slots

Each of these defines its basic structure — the `new`, `map`, `unwrap`, `push`, `pop` operations — at Level 1, with no bounds on the type parameter. Additional operations that require bounds (sorting a `Vec`, displaying an `Option`) are added in separate, bounded impl blocks at Level 2. The structural core remains parametric.

### Map: The Canonical Level 1 Operation

There is one operation that transcends simple storage and retrieval while remaining at Level 1: **mapping**. Given a function from `T` to `U`, transform a container of `T` into a container of `U`:

```rust
# struct Slot<T> { value: T }
impl<T> Slot<T> {
    fn map<U>(self, f: impl FnOnce(T) -> U) -> Slot<U> {
        Slot { value: f(self.value) }
    }
}
```

The `map` operation does not require any bounds on `T` or `U`. It takes a function — provided by the caller — and applies it to the contained value. The `Slot` does not need to know what `T` is or what `U` is; it only needs to know that a transformation exists between them.

This is why `map` appears on `Option`, `Result`, `Vec`, iterators, and virtually every generic container in Rust. It is the fundamental Level 1 transformation: apply a caller-supplied function to a contained value, without the container needing to understand either type.

The parametricity theorem for `map` says: mapping `f` then `g` is the same as mapping their composition `g ∘ f`. This is the functor law, and it holds *for free* from the type signature — no implementation inspection needed.

## PhantomData: Type-Level Presence, Value-Level Absence

Not every generic parameter needs to appear in a value. Sometimes a type needs to *mention* a type parameter — to establish a relationship in 𝒯 — without actually containing a value of that type in 𝒱. This is the role of `PhantomData<T>`.

```rust
use std::marker::PhantomData;

struct Tagged<T, Tag> {
    value: T,
    _tag: PhantomData<Tag>,
}
```

The field `_tag` has type `PhantomData<Tag>`. At runtime, `PhantomData` is a zero-sized type — the forgetful functor maps it to nothing. In 𝒱, `Tagged<i32, Metres>` and `Tagged<i32, Seconds>` have identical memory layouts: a single `i32`. But in 𝒯, they are distinct types. The `Tag` parameter exists purely above the boundary.

`PhantomData` is a Level 1 mechanism because it requires no bounds on the phantom parameter. The parameter `Tag` is unconstrained — it could be any type — and the struct imposes no requirements on it. The phantom parameter's purpose is *identity*, not capability.

```rust
use std::marker::PhantomData;

struct Metres;
struct Seconds;

struct Quantity<Unit> {
    value: f64,
    _unit: PhantomData<Unit>,
}

impl<Unit> Quantity<Unit> {
    fn new(value: f64) -> Self {
        Quantity { value, _unit: PhantomData }
    }

    fn scale(self, factor: f64) -> Self {
        Quantity::new(self.value * factor)
    }
}
```

The `impl<Unit> Quantity<Unit>` block is Level 1: it works for all `Unit` types, with no bounds. The `scale` method multiplies the value without knowing or caring what the unit is. The unit parameter prevents type confusion — `Quantity<Metres>` and `Quantity<Seconds>` cannot be mixed — but contributes nothing at runtime.

This illustrates a general principle: **Level 1 can carry type-level information without runtime cost.** The phantom parameter is structure in 𝒯 that the forgetful functor erases completely. It is the simplest form of a pattern that reaches its full expression at Level 4 (typestate), where phantom parameters encode entire state machines above the boundary.

Note, however, that `PhantomData` has an important role in variance and drop-checking. The compiler treats `PhantomData<T>` as if the struct *logically owns* a `T`, which affects lifetime analysis. This is a Level 1 mechanism with consequences for the lifetime sub-logic — a reminder that the levels interact, even when a concept belongs primarily to one.

## The Implicit Bound: `Sized`

Every type parameter in Rust carries an invisible axiom. When you write `<T>`, the compiler reads `<T: Sized>` — an implicit bound asserting that `T` has a statically known size. This is the only bound that Rust adds without being asked, and it is the only bound that can be *removed* rather than added.

`Sized` is a Level 1 proposition: "this type occupies a fixed, known number of bytes." It is the prerequisite for the forgetful functor — F needs a concrete memory layout to map a type from 𝒯 to 𝒱. A value must have a known size to be placed on the stack, stored in a struct field, or passed by value. Without `Sized`, the boundary cannot do its work.

Most types satisfy `Sized` trivially: `i32` is 4 bytes, `(u8, u64)` is 16 bytes (with padding), `Option<bool>` is 1 byte. But some types are **dynamically sized types** (DSTs) — types that exist in 𝒯 as valid types but whose size is not known at compile time:

- `[T]` — a slice of unknown length
- `str` — a string of unknown length
- `dyn Trait` — a trait object whose concrete type is erased

These types cannot cross the boundary directly. You cannot write `let x: [i32]` — the compiler does not know how much stack space to allocate. DSTs must always appear behind indirection: `&[T]`, `&str`, `Box<dyn Trait>`. The indirection provides the size information that the DST itself lacks — a pointer (fixed size) plus, for slices, a length, or for trait objects, a vtable pointer.

The syntax `?Sized` relaxes the implicit bound. It is unique in Rust: every other bound is something you *add* to a type parameter, but `?Sized` is something you *remove*. Writing `T: ?Sized` says "T may or may not be sized" — the function accepts both sized and unsized types.

```rust
fn print_ref<T: ?Sized + std::fmt::Display>(value: &T) {
    println!("{value}");
}

fn main() {
    print_ref(&42);                          // T = i32 (Sized)
    print_ref("hello");                       // T = str (!Sized)
    let d: &dyn std::fmt::Display = &42;
    print_ref(d);                             // T = dyn Display (!Sized)
}
```

The `?Sized` bound widens the function's domain: it ranges over all displayable types, including those that cannot be passed by value. The trade-off is that the function can only accept `T` behind a reference or pointer — it cannot move, store, or return a `T` by value, because doing so requires knowing the size.

In the fibration view, `Sized` partitions the base space. The `Sized` types form the sub-fibration where F can operate directly. The `?Sized` extension includes fibres that F can only reach through indirection. The implicit `Sized` bound keeps functions in the directly-accessible region by default; `?Sized` explicitly opts into the wider space.

## Subtyping and Variance

In general type theory, **subtyping** is a preorder on types: if `A <: B`, then a value of type `A` can be used wherever a value of type `B` is expected. In object-oriented languages, subtyping is pervasive — a `Dog` can be used where an `Animal` is expected, because `Dog <: Animal`. Width subtyping (records with more fields are subtypes of records with fewer) and depth subtyping (records with more specific field types are subtypes) give these systems a rich subtyping lattice.

Rust's subtyping is far narrower. There is no structural subtyping on structs, no inheritance hierarchy, no implicit widening of types. Rust's subtyping operates almost exclusively through **lifetimes**.

### Lifetime Subtyping

The subtyping relation in Rust is: if `'long: 'short` (the lifetime `'long` outlives `'short`), then:

```text
&'long T  <:  &'short T
```

A reference that lives longer can be used where a shorter-lived reference is expected. This is sound because a reference valid for a longer duration is *at least as valid* as one needed for a shorter duration. The relation is a preorder: reflexive (`'a: 'a`) and transitive (if `'a: 'b` and `'b: 'c`, then `'a: 'c`).

This is the primary — and nearly the only — subtyping relation in Rust. Unlike Java or C#, where subtyping pervades the type system through class hierarchies, Rust confines subtyping to the lifetime sub-logic. The result is a simpler, more predictable system where subtyping interactions are limited to reference validity.

### Variance

Given a type constructor `F<T>`, **variance** describes how subtyping on `T` propagates to subtyping on `F<T>`. There are three cases:

**Covariant:** `T <: U` implies `F<T> <: F<U>` — subtyping propagates in the same direction. Shared references are covariant: if `'long: 'short`, then `&'long T <: &'short T`. Similarly, `Vec<&'long T> <: Vec<&'short T>`. The container preserves the subtyping relationship of its contents.

**Contravariant:** `T <: U` implies `F<U> <: F<T>` — subtyping propagates in the *reverse* direction. Function parameters are contravariant: `fn(&'short T)` can be used where `fn(&'long T)` is expected, because a function that handles short-lived references certainly handles long-lived ones too. The direction reverses because the function *receives* rather than *produces* the value.

**Invariant:** no subtyping relationship propagates. `&mut T` is invariant in `T` — if `&mut T` were covariant, you could write a `&'short T` into a location that promises `&'long T`, creating a dangling reference. `Cell<T>` and `RefCell<T>` are also invariant, for the same reason: interior mutability requires that the type parameter be fixed exactly.

### Why `&mut T` Is Invariant

The invariance of `&mut T` is not arbitrary — it is the soundness condition for mutable references. Consider what would happen if `&mut T` were covariant:

```rust,ignore
fn unsound<'long, 'short>(x: &mut &'long str, y: &'short str)
where
    'long: 'short,
{
    // If &mut T were covariant, this would be allowed:
    // &mut &'long str  <:  &mut &'short str
    // Then we could write:
    *x = y;  // Stores a 'short reference where a 'long reference is expected!
}
// After this function returns, *x contains a reference that may dangle.
```

Invariance prevents this: `&mut &'long str` has no subtyping relation with `&mut &'short str`, so the coercion is rejected. The mutable reference's invariance preserves the contract that whatever you write through it must be valid for the original lifetime.

### PhantomData and Variance Control

When a struct has a type parameter that does not appear in any field — a phantom parameter — `PhantomData` determines the struct's variance with respect to that parameter. The choice of `PhantomData` wrapper signals the intended variance to the compiler:

| `PhantomData` form | Variance in `T` | Use case |
|---|---|---|
| `PhantomData<T>` | Covariant | The struct logically owns a `T` |
| `PhantomData<fn(T)>` | Contravariant | The struct logically consumes `T` |
| `PhantomData<fn(T) -> T>` | Invariant | The struct both consumes and produces `T` |

This is how the programmer communicates variance for phantom parameters. The compiler infers variance for parameters that appear in real fields based on their position (owned values: covariant; behind `&mut`: invariant; in function argument position: contravariant). For phantom parameters, `PhantomData` is the explicit declaration.

### The Categorical View

Variance is a functor property. A covariant type constructor is a **covariant functor** on the subtyping preorder: it maps the ordering `A <: B` to `F<A> <: F<B>`, preserving direction. A contravariant type constructor is a **contravariant functor**: it reverses the ordering. An invariant type constructor is neither — it does not induce any relationship on the subtyping preorder.

This connects to the functor laws from Chapter 1. The `map` operation on `Option<T>`, `Vec<T>`, and other covariant containers is not merely a convenience — it is the functorial action that witnesses the covariance. The existence of a natural `map` is precisely what it means for a type constructor to be covariant.

## The Fibration View

Chapter 1 introduced the concept of a fibration: a generic type as a family of types indexed by the parameter. At Level 1, this structure is at its simplest.

Consider `Vec<T>`. The type constructor `Vec` defines a total space — a family of types parameterised by `T`:

```text
    Vec<_>
     |
     ├── Vec<i32>       (fibre over i32)
     ├── Vec<String>    (fibre over String)
     ├── Vec<bool>      (fibre over bool)
     ├── Vec<Vec<u8>>   (fibre over Vec<u8>)
     └── ...            (one fibre for every type)
```

At Level 1, the base space is *unrestricted* — every type in the language is a valid index. The generic function `fn first<T>(items: &[T]) -> Option<&T>` operates uniformly across the entire total space. It does not distinguish one fibre from another.

At Level 2, bounds restrict the base space. `fn sort<T: Ord>(items: &mut [T])` operates only over the sub-fibration indexed by orderable types. The base space narrows; the available operations widen.

The fibration perspective makes visible what "adding a bound" actually does. It does not add a capability to the function in isolation — it *restricts the family of types* the function ranges over, and it is this restriction that enables the additional operations. The bound is a predicate on the index set, not a tool given to the implementation.

## Level 1 versus Level 2: The Boundary Between Them

The distinction between Level 1 and Level 2 is sharp: Level 1 has no bounds; Level 2 has at least one. This binary difference produces qualitative changes in what can be expressed.

| Property | Level 1 (no bounds) | Level 2 (bounded) |
|---|---|---|
| Type parameter knowledge | Nothing | The bound's interface |
| Available operations on `T` | Move, store, return | Bound's methods |
| Parametricity | Full | Restricted to bound-consistent relations |
| Free theorems | Maximal | Weaker (more implementations possible) |
| Implementation freedom | Minimal | Wider |
| Base space of fibration | All types | Types satisfying the bound |

The relationship is an inverse: the more you know about `T`, the less constrained your implementation, and the fewer theorems follow for free.

Consider these three functions:

```rust,ignore
fn first_unbounded<T>(items: &[T]) -> Option<&T> {
    items.first()
}

fn first_cloneable<T: Clone>(items: &[T]) -> Option<T> {
    items.first().cloned()
}

fn first_default<T: Default>(items: &[T]) -> T {
    items.first().cloned_or_default()  // does not compile!
}
```

That last function does not compile — `cloned_or_default` is not a real method. Let us write it properly:

```rust
fn first_or_default<T: Clone + Default>(items: &[T]) -> T {
    match items.first() {
        Some(v) => v.clone(),
        None => T::default(),
    }
}
```

Each step adds a bound and widens the implementation space:
- `first_unbounded` can only return a reference to an existing element, or `None`. It cannot create values.
- `first_cloneable` can return a *copy* of an element (via `Clone`), but still cannot create values from nothing.
- `first_or_default` can return a copy of an element *or* construct a default value (via `Default`), eliminating the `Option`.

Each additional bound is an additional hypothesis in the Curry-Howard reading. More hypotheses mean a weaker universal statement (it applies to fewer types) but a stronger conclusion (it can do more). This is the lattice structure from Chapter 1: descending in the lattice adds constraints, narrows the domain, and expands the available operations.

Level 1 is the top of this lattice — the point of maximum generality and minimum capability. It is where the free theorems are strongest, where the implementation is most constrained, and where the type signature carries the most information about what the function does.

### Type Inference and the Turbofish

When a Level 1 function is called, the compiler must determine the concrete type for each type parameter — it must recover 𝒯 information from 𝒱 context. This is **type inference**: a partial inverse of the forgetful functor, reconstructing erased type structure from the constraints imposed by how a value is used.

In most cases, inference succeeds without intervention. The compiler examines the argument types, the expected return type, and the operations performed on the result, and deduces the type parameter uniquely:

```rust
let items = vec![1, 2, 3]; // inferred: Vec<i32>
let first = items.first();  // inferred: Option<&i32>
```

But inference sometimes faces ambiguity — multiple types could satisfy the constraints. The **turbofish** syntax `::<>` lets the programmer supply the missing type information explicitly:

```rust
let x: Vec<i32> = vec![1, 2, 3].into_iter().collect();
// equivalently:
let y = vec![1, 2, 3].into_iter().collect::<Vec<i32>>();
```

Both forms provide the same 𝒯 information through different channels: a type annotation on the binding, or a turbofish on the call. The turbofish is needed when the 𝒱 context does not uniquely determine the type parameter — when the partial inverse of F is not uniquely defined. Where the usage context resolves the ambiguity, the compiler fills in the type silently; where it does not, the programmer must intervene.

## What Level 1 Reveals

The Level 0 programmer sees types as data descriptions. The Level 1 programmer sees something different: **types as specifications**.

At Level 1, the type signature is not merely a contract about data layout. It is a near-complete specification of behaviour. When you read `fn<T>(T, T) -> T`, you know the function selects one of its arguments — not because you have read the implementation, but because the type leaves no other possibility. The type is the specification; the implementation is determined.

This is the practical content of Reynolds' parametricity for the working Rust programmer:

**Generic code is constrained code.** The more generic your function, the fewer things it can do, and the more you can reason about it from its signature alone. This is the opposite of the naïve expectation that generics provide freedom — in fact, they provide *discipline*.

**Bounds are not restrictions on the caller; they are permissions for the implementation.** When you add `T: Ord` to a function signature, you are not "restricting which types can be passed." You are granting the implementation permission to compare values. Without that permission, comparison is impossible. The bound is a grant, not a constraint.

**The type signature is documentation.** At Level 1, the signature tells you nearly everything. A function `fn<T>(Vec<T>) -> Vec<T>` rearranges elements. A function `fn<A, B>(A, fn(A) -> B) -> B` applies a function. A function `fn<T>(T) -> (T, T)` — wait, that requires `Clone`. The type reveals what is needed.

This perspective also clarifies why Rust's approach to generics produces better-documented APIs than, say, C++ templates. In C++, a template function's requirements on its type parameters are implicit — discovered through compilation errors when requirements are not met. In Rust, the bounds are stated explicitly in the type signature. The signature *is* the specification, stated upfront, verified by the compiler, visible to every reader.

## The Boundary at Level 1

When a Level 1 function crosses the boundary, the forgetful functor F performs **monomorphisation**: the generic family is collapsed into a set of concrete functions, one for each type the program actually uses.

```rust
fn wrap<T>(x: T) -> Option<T> {
    Some(x)
}

fn main() {
    let a = wrap(42_i32);      // generates wrap::<i32>
    let b = wrap("hello");     // generates wrap::<&str>
    let c = wrap(vec![1, 2]);  // generates wrap::<Vec<i32>>
}
```

In 𝒯, `wrap` is a single universally quantified function — one object in the type category. In 𝒱, after F is applied, it becomes three separate concrete functions, each with a fixed type. The universal quantification has been resolved; the polymorphism has been eliminated. Each monomorphised instance is a Level 0 function — a ground proof with no generics.

This is the boundary's role at Level 1: it transforms universal statements into collections of ground instances. The universal proposition ∀T. T → Option\<T\> becomes the set of ground propositions { i32 → Option\<i32\>, &str → Option\<&str\>, Vec\<i32\> → Option\<Vec\<i32\>>, ... }. The universal is verified once, above the boundary. Below the boundary, only its instances exist.

The cost model is clear: each monomorphisation produces a separate copy of the function's machine code. This is the runtime cost of Level 1 — not in execution speed (each instance is as fast as a hand-written Level 0 function) but in code size. A heavily generic program can produce many monomorphisations. The boundary does not add overhead *per call*, but it may add overhead *in binary size*. This is the trade-off Rust accepts: zero-cost dispatch in exchange for potential code duplication.

## Ascending Further

Level 1 is the domain of pure structure. Functions at this level are plumbing — they route, package, and rearrange values without examining them. The parametricity theorem ensures that this structural manipulation is all they can do, and the free theorems let you reason about them from their types alone.

But most interesting programs need to do more than rearrange. They need to compare, transform, combine, and display values — operations that require knowing *something* about the type. The moment you write `T: Ord`, or `T: Display`, or even `T: Clone`, you have left Level 1 and entered Level 2.

The transition is not a leap but a single step: the addition of one bound. Yet that single bound changes everything. The fibration narrows. The free theorems weaken. The implementation gains capabilities. And the function, instead of stating "for all types, unconditionally," begins to state "for all types satisfying this proposition" — a conditional universal, with hypotheses that must be discharged by proof.

In the next chapter, we examine what those hypotheses mean, how they interact, and what structure they form.
