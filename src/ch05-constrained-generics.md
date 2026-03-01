# Constrained Generics (Level 2)

Level 1 gave us universal quantification — statements about all types, unconditionally. The price was severe: knowing nothing about `T`, we could do almost nothing with it. Move, store, return. The free theorems were powerful precisely because the implementation space was narrow.

Level 2 changes the terms of the bargain. By adding a single trait bound — `T: Ord`, `T: Display`, `T: Clone` — we restrict the domain of the universal quantifier to those types satisfying a predicate. In exchange, the implementation gains access to the operations that predicate guarantees. The quantification is no longer "for all types" but "for all types such that."

In the lambda cube, this does not correspond to a new vertex — bounded quantification is orthogonal to the cube's three axes. Rather, it extends System F with **qualified types**: universals whose type variable is restricted by a predicate. In Curry-Howard terms, we move from unconditional universals to conditional ones — universal statements with hypotheses. The function signature becomes not just a theorem but a theorem with premises, and the bounds are those premises.

Most Rust generics live at Level 2. The moment you write `T: Ord`, `T: Iterator`, or `T: Into<String>`, you have entered this level. Understanding its structure — the lattice of constraints, the interplay of bounds, the role of associated types — is understanding the layer where Rust programmers spend most of their generic programming time.

## Bounds as Predicates

At Level 1, the type parameter `T` ranges over the entire type universe — every type in the language is a valid instantiation. At Level 2, a bound restricts this range. The bound is a **predicate**: a proposition that must hold for each type in the parameter domain.

```rust
fn maximum<T: Ord>(a: T, b: T) -> T {
    if a >= b { a } else { b }
}
```

The bound `T: Ord` is the predicate Ord(T): "T possesses a total ordering." The parameter domain is no longer "all types" but "all types for which Ord(T) holds" — the set { T | Ord(T) }. Every type in this set has a guaranteed comparison operation, and the function body uses it.

Under Curry-Howard, the signature reads:

> ∀T. Ord(T) → (T → T → T)

For all types T, *if* T is ordered, *then* given two T values, a T value can be produced. The bound is the hypothesis; the function body is the proof; the `impl Ord for ConcreteType` that the caller must provide is the evidence discharging the hypothesis at the call site.

This is the fundamental structure of Level 2: **conditional universal quantification**. Every bounded generic function is an if-then statement, universally quantified over the types satisfying its conditions.

### What a Bound Grants

A bound does two things simultaneously:

**It restricts the domain.** Fewer types satisfy `T: Ord` than satisfy no bound at all. The set of valid instantiations shrinks.

**It enables operations.** Within the function body, the bound's methods become available. `T: Ord` grants access to `cmp`, `max`, `min`, and the comparison operators. `T: Display` grants `fmt` and, by extension, `to_string()`. `T: Clone` grants `clone()`.

These two effects are inseparable — they are the same mechanism viewed from different angles. The bound restricts the domain to types that have an ordering, and because every type in the restricted domain has an ordering, the ordering operations are safe to use. The restriction *is* the enablement.

```rust
fn print_max<T: Ord + std::fmt::Display>(a: T, b: T) {
    let winner = if a >= b { a } else { b };
    println!("maximum: {winner}");
}
```

Here, `T: Ord + Display` grants both comparison (`>=`) and formatting (`println!`). Remove either bound and the corresponding operation becomes unavailable — not as a "restriction" imposed on the programmer, but because the predicate no longer guarantees the operation's existence.

## The Constraint Lattice

Chapter 1 introduced the type lattice: the partial order on trait bounds where adding constraints moves down (narrower domain, more operations) and removing constraints moves up (wider domain, fewer operations). At Level 2, we can examine this lattice in detail.

### The Partial Order

Given two bounds A and B, we say A ≤ B (A is *weaker* than B) when every type satisfying B also satisfies A. Equivalently: the predicate region of B is contained within the predicate region of A. The weaker bound admits more types but grants fewer operations.

```text
         (no bounds)           ← Level 1: all types, no operations on T
          /    |    \
       Clone  Ord  Display     ← single bounds: large predicate regions
          \    |    /
       Clone + Ord             ← compound: intersection of regions
            |
     Clone + Ord + Display
            |
          ...
            |
     (concrete type)           ← Level 0: one type, all its operations
```

The top of the lattice is Level 1 — the unconstrained parameter, satisfying no predicates, admitting all types. The bottom consists of concrete types — Level 0 — where the type is fully determined and all its specific operations are available. Level 2 occupies the vast middle ground between these extremes.

### Intersection of Predicate Regions

Each trait bound defines a **predicate region** in the type lattice — the set of all types satisfying that bound. When you write a compound bound `T: A + B`, you are taking the **intersection** of the predicate regions for A and B.

Consider the standard library traits `Clone`, `Ord`, and `Hash`:

```text
    Clone region: { i32, String, Vec<T>, bool, ... }     ← types implementing Clone
    Ord region:   { i32, String, bool, char, ... }        ← types implementing Ord
    Hash region:  { i32, String, bool, char, ... }        ← types implementing Hash

    Clone + Ord:  { i32, String, bool, char, ... }        ← intersection
    Clone + Ord + Hash: { i32, String, bool, char, ... }  ← smaller intersection
```

Each `+` narrows the set. A type must satisfy *all* the predicates simultaneously to lie within the compound region. Under Curry-Howard, the `+` operator is **logical conjunction**: `T: Clone + Ord` is the proposition Clone(T) ∧ Ord(T).

This has a practical consequence for API design: every bound you add to a function signature narrows the set of types that can call it. A function bounded by `T: Display` accepts more types than one bounded by `T: Display + Debug + Clone`. The right number of bounds is the *minimum* needed to implement the function — any additional bound is an unnecessary hypothesis, a premise that the proof does not use.

```rust
// Too many bounds — Hash is never used in the body.
fn print_sorted_bad<T: Ord + std::fmt::Display + std::hash::Hash>(
    items: &mut Vec<T>,
) {
    items.sort();
    for item in items.iter() {
        println!("{item}");
    }
}

// Correct — only the bounds actually used.
fn print_sorted<T: Ord + std::fmt::Display>(items: &mut Vec<T>) {
    items.sort();
    for item in items.iter() {
        println!("{item}");
    }
}
```

The `Hash` bound in the first version narrows the predicate region unnecessarily. No operation in the body requires hashing. The bound is a vacuous hypothesis — logically valid but wasteful. The second version states the minimal premises, admitting the broadest useful set of types.

### Supertraits as Implication

Some traits carry implicit additional bounds through **supertrait** relationships. A trait declaration `trait A: B` asserts that every type satisfying A also satisfies B — the proposition ∀T. A(T) → B(T).

```rust
# use std::fmt;
trait Report: fmt::Display {
    fn header(&self) -> &str;
}
```

The supertrait bound `Report: Display` means that any type implementing `Report` must also implement `Display`. A function bounded by `T: Report` gets access to both `Report`'s methods and `Display`'s methods, even though only `Report` appears in the bound:

```rust
# use std::fmt;
# trait Report: fmt::Display { fn header(&self) -> &str; }
fn print_report<T: Report>(item: &T) {
    println!("=== {} ===", item.header());
    println!("{item}");  // Display is available via the supertrait
}
```

In lattice terms, the `Report` region is contained within the `Display` region. Bounding by `Report` implicitly bounds by `Display`. The supertrait establishes a *refinement* relationship: `Report` is a stronger proposition that implies `Display`.

The standard library uses this extensively. `Eq` requires `PartialEq`. `PartialOrd` also requires `PartialEq`. `Ord` requires both `Eq` and `PartialOrd`. The result is a diamond of supertrait implications:

```text
    PartialEq
     ↑       ↑
    Eq    PartialOrd
      ↖     ↗
       Ord
```

Bounding by `Ord` grants access to the entire diamond. A function bounded by `T: Ord` can use `==`, `!=`, `<`, `>`, `<=`, `>=`, `cmp`, `max`, and `min` — all the operations from `Ord` and every trait it implies. The single bound `Ord` is, in reality, a compound proposition: Ord(T) implies PartialOrd(T) ∧ Eq(T) ∧ PartialEq(T).

## Bounds and the Impl Domain

At Level 2, we begin to see clearly the relationship between bounds and the proof domain that Chapter 6 will examine in full.

A bound is a **demand**. An impl block is a **supply**. The bound `T: Ord` in a function signature demands that whoever calls this function must supply evidence that their concrete type satisfies `Ord`. That evidence is the `impl Ord for ConcreteType` block.

```rust
struct Metres(f64);

impl PartialEq for Metres {
    fn eq(&self, other: &Self) -> bool {
        self.0 == other.0
    }
}

impl Eq for Metres {}

impl PartialOrd for Metres {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        Some(self.cmp(other))
    }
}

impl Ord for Metres {
    fn cmp(&self, other: &Self) -> std::cmp::Ordering {
        self.0.total_cmp(&other.0)
    }
}
```

These four impl blocks are **proof terms** — evidence that `Metres` satisfies `Ord` (and its supertraits). Together, they discharge the obligation that a function like `maximum::<Metres>` demands. Without them, the call site is a gap in the proof: the hypothesis is stated but unsubstantiated.

The relationship is:
- **Level 2** states the propositions (trait bounds in function signatures).
- **Level 3** supplies the proofs (impl blocks).

Level 2 cannot function without Level 3. A bounded generic function is a theorem with hypotheses, and hypotheses require evidence. The programmer working at Level 2 is *stating* what they need; the programmer working at Level 3 is *providing* it. In practice, the same programmer often does both — writing bounded functions and the impl blocks that satisfy those bounds — but the two activities belong to different levels of the model.

## Where Clauses

The inline bound syntax `T: Ord + Display` handles simple cases. For more complex propositions — those involving multiple type parameters, associated types, or relationships between types — Rust provides **where clauses**.

### Basic Where Clauses

A where clause moves the bounds out of the angle brackets into a separate block:

```rust
fn print_all<T>(items: &[T])
where
    T: std::fmt::Display,
{
    for item in items {
        println!("{item}");
    }
}
```

This is syntactically equivalent to `fn print_all<T: Display>(items: &[T])`. The where clause adds no expressive power in this simple case — it is a formatting choice.

### Relational Bounds

Where clauses become essential when the proposition involves relationships *between* type parameters:

```rust
fn convert_all<I, T, U>(iter: I) -> Vec<U>
where
    I: Iterator<Item = T>,
    T: Into<U>,
{
    iter.map(|item| item.into()).collect()
}
```

The where clause states two hypotheses:
1. `I` is an iterator yielding values of type `T`.
2. `T` is convertible into `U`.

These hypotheses are interdependent — the associated type `Item` connects `I` to `T`, and the `Into` bound connects `T` to `U`. The where clause is the natural place to express such multi-parameter propositions, because the relationships span across parameters in a way that inline syntax cannot cleanly capture.

Under Curry-Howard, the where clause is a **conjunction of hypotheses**:

> ∀I, T, U. (Iterator(I, Item=T) ∧ Into(T, U)) → (I → Vec\<U\>)

Each line of the where clause is a separate conjunct. The function body must use all of them — `map` relies on `Into`, and `Iterator` provides the `map` method itself.

### Bounds on Associated Types

Where clauses can constrain not just type parameters but their associated types:

```rust
fn sum_iter<I>(iter: I) -> i64
where
    I: Iterator<Item = i64>,
{
    iter.sum()
}
```

The bound `I: Iterator<Item = i64>` is a compound proposition: "I is an iterator, *and* its associated type `Item` equals `i64`." The equality constraint on the associated type is a proposition about a type-level functional dependency — a topic we will examine shortly.

A more general version constrains the associated type with its own bound rather than fixing it to a specific type:

```rust
fn sum_generic<I>(iter: I) -> I::Item
where
    I: Iterator,
    I::Item: std::iter::Sum,
{
    iter.sum()
}
```

Here, `I::Item` is not fixed — it can be any type that implements `Sum`. The where clause expresses two propositions: I is an iterator, and I's item type supports summation. The return type `I::Item` depends on the type parameter `I`, making this an example of the codomain depending on the parameter — a topic we take up at the end of this chapter.

## Associated Types as Functional Dependencies

An associated type defines a **type-level function** from the implementing type to another type. When a trait declares `type Item;`, it establishes that for each type implementing the trait, there is exactly one corresponding `Item` type.

```rust
trait Container {
    type Element;

    fn first(&self) -> Option<&Self::Element>;
    fn len(&self) -> usize;
}
```

For any type `C` implementing `Container`, the associated type `C::Element` is *determined by* `C`. It is a function in 𝒯: given the input type `C`, the output type `C::Element` is fixed. This is a **functional dependency** — knowing `C` tells you `Element`.

```rust
# trait Container {
#     type Element;
#     fn first(&self) -> Option<&Self::Element>;
#     fn len(&self) -> usize;
# }
struct IntBuffer {
    data: Vec<i32>,
}

impl Container for IntBuffer {
    type Element = i32;

    fn first(&self) -> Option<&i32> {
        self.data.first()
    }

    fn len(&self) -> usize {
        self.data.len()
    }
}
```

The impl block establishes: Container(IntBuffer) with Element = i32. The associated type is not a free parameter — it is *determined* by the impl. For `IntBuffer`, the element type is always `i32`. There is no choice; the functional dependency is total.

### Associated Types versus Type Parameters

A trait could use a type parameter instead of an associated type:

```rust
trait ContainerOf<E> {
    fn first(&self) -> Option<&E>;
    fn len(&self) -> usize;
}
```

The difference is logical. With an associated type, the relationship is *functional*: each implementing type determines exactly one element type. With a type parameter, the relationship is *relational*: a single type might implement `ContainerOf<i32>` *and* `ContainerOf<String>`, being a container of both simultaneously.

In lattice terms:
- An **associated type** creates a function in 𝒯: one input, one output, no ambiguity.
- A **type parameter on the trait** creates a relation in 𝒯: one input, potentially many outputs.

Rust's `Iterator` uses an associated type because an iterator yields exactly one kind of item. `From<T>` uses a type parameter because a single type can be constructed from many different source types — `String` implements both `From<&str>` and `From<Vec<u8>>`.

The choice between associated types and type parameters is a logical design decision: is the relationship between the implementing type and the dependent type a *function* (one-to-one) or a *relation* (one-to-many)?

### Chains of Functional Dependencies

Associated types can form chains of dependencies:

```rust
trait Process {
    type Input;
    type Output;
    type Error;

    fn run(&self, input: Self::Input) -> Result<Self::Output, Self::Error>;
}
```

For any type `P: Process`, the triple (P::Input, P::Output, P::Error) is determined by `P`. This is a type-level function from one type to three types — or equivalently, a function into a product type in 𝒯.

When you write a generic function over `Process`:

```rust
# trait Process {
#     type Input;
#     type Output;
#     type Error;
#     fn run(&self, input: Self::Input) -> Result<Self::Output, Self::Error>;
# }
fn run_with_default<P>(proc: &P) -> Result<P::Output, P::Error>
where
    P: Process,
    P::Input: Default,
{
    proc.run(P::Input::default())
}
```

The where clause states: P is a process, *and* P's input type has a default value. The second hypothesis is a bound on the *output* of a type-level function — a proposition about a derived type, not about P directly. This is a second-order constraint: a predicate not on the parameter itself but on a type computed from the parameter.

## The Codomain Depending on the Parameter

One of Level 2's most distinctive features is that the **return type** of a generic function can depend on the same type parameter as the input. At Level 1, this happens trivially — `fn identity<T>(x: T) -> T` returns the same type it receives. At Level 2, the dependency becomes substantive through associated types.

```rust
use std::str::FromStr;

fn parse_or_default<T>(input: &str) -> T
where
    T: FromStr + Default,
{
    input.parse().unwrap_or_default()
}
```

The return type `T` depends on the type parameter, which the *caller* chooses. But the caller's choice is constrained: `T` must satisfy both `FromStr` and `Default`. The function's behaviour — what it parses, what default it falls back to — is entirely determined by the caller's choice of `T`.

```rust
# use std::str::FromStr;
# fn parse_or_default<T>(input: &str) -> T
# where T: FromStr + Default {
#     input.parse().unwrap_or_default()
# }
fn main() {
    let n: i32 = parse_or_default("42");        // parses as i32, default 0
    let s: String = parse_or_default("hello");   // parses as String, default ""
    let b: bool = parse_or_default("invalid");   // parse fails, default false
}
```

Each call instantiates the same function with a different type, producing different behaviour — not because the function branches on `T`, but because the *proofs* (the impl blocks for `FromStr` and `Default`) differ for each type. The function's logic is uniform; the variation comes from the proof witnesses, which are resolved at compile time and erased at the boundary.

### Associated Types in Return Position

When the return type involves an associated type, the dependency becomes indirect:

```rust
trait Produce {
    type Output;
    fn produce(&self) -> Self::Output;
}

struct Counter(u32);

impl Produce for Counter {
    type Output = u32;
    fn produce(&self) -> u32 {
        self.0
    }
}

struct Greeter;

impl Produce for Greeter {
    type Output = String;
    fn produce(&self) -> String {
        "hello".to_string()
    }
}

fn produce_twice<P: Produce>(p: &P) -> (P::Output, P::Output)
where
    P::Output: Clone,
{
    let first = p.produce();
    let second = first.clone();
    (first, second)
}
```

The return type `(P::Output, P::Output)` depends on `P` through the functional dependency of the associated type. For `Counter`, the return type is `(u32, u32)`. For `Greeter`, it is `(String, String)`. The compiler resolves this at monomorphisation time — in 𝒯, the function has a family of return types; in 𝒱, after the boundary, each instance has a fixed concrete return type.

This is where Level 2 begins to approach the expressiveness of dependent types — not fully, because the dependency is on a *type* parameter, not a *value* parameter, but enough to allow the return type of a function to vary with its input type in a structured, type-safe way.

## Negative Reasoning: What Bounds Exclude

Bounds at Level 2 enable operations by restricting the domain. But they also carry implicit *negative* information — things the function cannot do because a bound is absent.

Consider a function bounded only by `T: Clone`:

```rust
fn duplicate<T: Clone>(x: &T) -> (T, T) {
    (x.clone(), x.clone())
}
```

This function can clone, but it *cannot*:
- Compare the two copies (requires `PartialEq`)
- Print them (requires `Display` or `Debug`)
- Sort a collection of them (requires `Ord`)
- Hash them (requires `Hash`)

The absence of each bound is the absence of a hypothesis. Without the hypothesis, the corresponding operations are unavailable, and the corresponding free theorems hold. For example, parametricity guarantees that `duplicate` treats the cloned values uniformly — it cannot distinguish them, because it has no equality test.

This connects to a design principle: **state only the hypotheses you use.** Each unused bound is a missed free theorem — a property of your function that holds but is obscured by the unnecessary constraint. The minimal bound set maximises both the function's applicability (more types can call it) and its guarantees (more free theorems hold).

```rust
// This version promises too little — the Debug bound is unused.
fn duplicate_verbose<T: Clone + std::fmt::Debug>(x: &T) -> (T, T) {
    (x.clone(), x.clone())
}

// This version is strictly better — same behaviour, broader applicability.
fn duplicate_clean<T: Clone>(x: &T) -> (T, T) {
    (x.clone(), x.clone())
}
```

Both functions behave identically, but `duplicate_clean` makes a stronger promise: it works for *any* cloneable type, not just debuggable cloneable types. The additional `Debug` bound in `duplicate_verbose` restricts the domain without justification.

## Worked Example: A Bounded Pipeline

Let us trace a realistic example through the Level 2 lens, reading each component as a proposition and proof.

```rust
use std::fmt;

trait Metric: fmt::Display + PartialOrd {
    fn zero() -> Self;
    fn combine(&self, other: &Self) -> Self;
}

fn best_of<M: Metric>(readings: &[M]) -> Option<String> {
    let mut best = readings.first()?;
    for reading in &readings[1..] {
        if reading > best {
            best = reading;
        }
    }
    Some(format!("best: {best}"))
}
```

Read this under Curry-Howard:

**The trait `Metric`** is a compound proposition with supertraits: Metric(T) implies Display(T) ∧ PartialOrd(T). It also asserts two additional capabilities: the existence of a zero value and a combination operation.

**The function signature** `fn best_of<M: Metric>(readings: &[M]) -> Option<String>` reads:

> ∀M. Metric(M) → (&\[M\] → Option\<String\>)

For all types M, if M is a metric, then given a slice of M values, an optional string can be produced.

**The function body** is the proof:
1. `readings.first()?` — uses slice operations (always available) and the `?` operator on `Option` (handling the empty-slice case by returning `None`).
2. `reading > best` — uses the comparison operation granted by `PartialOrd`, which is available because `Metric: PartialOrd`.
3. `format!("best: {best}")` — uses the formatting operation granted by `Display`, which is available because `Metric: Display`.

Every operation in the body is justified by a bound. The proof is valid because every step follows from the stated hypotheses. If you removed `PartialOrd` from the supertrait list, the comparison `reading > best` would fail — the proof step would reference an unavailable hypothesis. If you removed `Display`, the format macro would fail.

## The Lattice in Practice

The constraint lattice is not merely a theoretical construct. It has direct practical implications for API design.

### Finding the Right Altitude

When designing a generic function, the question is: where in the lattice should this function live? Too high (too few bounds) and the function cannot do its job. Too low (too many bounds) and the function excludes types unnecessarily.

```rust
use std::collections::HashMap;
use std::hash::Hash;

// Too high — cannot compile, missing bounds for HashMap.
// fn count_unique_bad<T>(items: Vec<T>) -> usize {
//     let set: HashMap<T, ()> = items.into_iter().map(|x| (x, ())).collect();
//     set.len()
// }

// Right altitude — exactly the bounds HashMap requires.
fn count_unique<T: Eq + Hash>(items: Vec<T>) -> usize {
    let set: HashMap<T, ()> = items.into_iter().map(|x| (x, ())).collect();
    set.len()
}
```

The bounds `Eq + Hash` are the minimum for `HashMap` keys. Adding `Ord` or `Display` would narrow the domain without benefit. The function sits at precisely the right altitude in the lattice.

### Bound Propagation

When a function calls another bounded function, the bounds propagate upward:

```rust
fn max_of_three<T: Ord>(a: T, b: T, c: T) -> T {
    std::cmp::max(std::cmp::max(a, b), c)
}
```

The function calls `std::cmp::max`, which requires `T: Ord`. Therefore `max_of_three` must also require `T: Ord` — it inherits the bounds of the functions it calls. This is bound propagation: the hypotheses required by sub-proofs propagate to the enclosing proof.

In larger codebases, bound propagation can lead to **bound accumulation** — functions deep in the call stack requiring many bounds because they transitively call many bounded functions. This is a signal that the function operates at a low altitude in the lattice, with a narrow predicate region. Sometimes this is necessary; sometimes it indicates that the function is doing too much, and should be decomposed into smaller, less-constrained pieces.

### The Standard Library's Altitude Choices

The standard library's API design is a study in lattice navigation. Consider `Vec<T>`:

```rust,ignore
// Level 1 — no bounds
impl<T> Vec<T> {
    fn new() -> Vec<T> { ... }
    fn push(&mut self, value: T) { ... }
    fn pop(&mut self) -> Option<T> { ... }
    fn len(&self) -> usize { ... }
}

// Level 2 — Clone bound
impl<T: Clone> Vec<T> {
    fn extend_from_slice(&mut self, other: &[T]) { ... }
}

// Level 2 — PartialEq bound
impl<T: PartialEq> Vec<T> {
    fn contains(&self, x: &T) -> bool { ... }
    fn dedup(&mut self) { ... }
}
```

Each impl block sits at a different altitude. The structural operations (push, pop, len) require no bounds — they are Level 1. Operations involving duplication require `Clone`. Operations involving comparison require `PartialEq`. The separation ensures that `Vec<T>` is maximally usable: you get the structural operations for free, and each additional capability becomes available only when the type parameter supports it.

This is the lattice made into API design: each impl block declares the minimum predicate region for its methods, and the overall API is a *stack* of capabilities, each layer requiring slightly more from the type parameter and offering slightly more in return.

## Existential Quantification at Level 2

Chapter 2 introduced `impl Trait` in return position as existential quantification. At Level 2, existential return types interact with bounds to create powerful abstractions.

```rust
fn make_repeater(n: u32) -> impl Iterator<Item = u32> {
    std::iter::repeat(n)
}
```

The return type says: "there exists a type satisfying `Iterator<Item = u32>`, and this function returns it." The caller knows the bound but not the concrete type. The concrete type (`std::iter::Repeat<u32>`) is hidden — existentially quantified away.

The combination of universal input parameters and existential return types creates functions that are *specific in what they demand and abstract in what they provide*:

```rust
fn evens_from<I>(iter: I) -> impl Iterator<Item = I::Item>
where
    I: Iterator,
    I::Item: Copy + std::ops::Rem<Output = I::Item> + PartialEq + From<u8>,
{
    iter.filter(|x| *x % I::Item::from(2u8) == I::Item::from(0u8))
}
```

The function universally quantifies over the input iterator type (the caller chooses) and existentially quantifies over the output iterator type (the callee chooses). The bounds on `I::Item` are the hypotheses enabling the filtering operation. The caller sees only the interface; the implementation details of the returned iterator are hidden above the boundary.

This interplay between ∀ (input) and ∃ (output) is characteristic of Level 2. At Level 1, there was no meaningful use of existential return types — without bounds, the returned type could not be used for anything. At Level 2, the bound on the existential gives the caller enough information to work with the opaque value.

## Higher-Ranked Trait Bounds

Ordinarily, lifetime parameters in a function signature are chosen by the *caller*. When you write `fn apply<'a, F: Fn(&'a str) -> &'a str>(f: F)`, the caller picks a specific lifetime `'a`, and the closure `F` must work for that one lifetime. But sometimes we need a stronger guarantee: the closure must work for *all* lifetimes, not just one chosen in advance.

This is the role of **higher-ranked trait bounds** (HRTBs). The syntax `for<'a>` introduces universal quantification over a lifetime *inside* a bound:

```rust
fn apply<F>(f: F) -> String
where
    F: for<'a> Fn(&'a str) -> &'a str,
{
    let owned = String::from("hello");
    f(&owned).to_string()
}
```

The bound `F: for<'a> Fn(&'a str) -> &'a str` says: for *every* lifetime `'a`, `F` can take a `&'a str` and return a `&'a str`. The closure does not commit to one lifetime — it promises to work for all of them. This is essential here because the reference `&owned` has a lifetime local to `apply`'s body, which the caller cannot name.

Contrast the two forms:

```rust,ignore
// Caller chooses 'a — the closure works for one specific lifetime.
fn apply_one<'a, F: Fn(&'a str) -> &'a str>(f: F, s: &'a str) -> &'a str {
    f(s)
}

// Closure works for ALL lifetimes — universally quantified inside the bound.
fn apply_all<F>(f: F) -> String
where
    F: for<'a> Fn(&'a str) -> &'a str,
{
    let local = String::from("world");
    f(&local).to_string()
}
```

In the first version, `'a` is a parameter of the function — the caller picks it. In the second, `'a` is quantified within the bound itself — the closure must handle whatever lifetime it encounters.

Under Curry-Howard, this is **second-order quantification nested inside a first-order constraint**. The function's outer quantification is over the type `F`; the inner quantification, `for<'a>`, is over lifetimes within the bound on `F`. This nesting — ∀F. (∀'a. Fn(&'a str) → &'a str)(F) → String — gives where clauses expressive power beyond what simple lifetime parameters can achieve.

In practice, HRTBs are more common than they appear. When you write a closure bound involving references with elided lifetimes — `F: Fn(&str) -> &str` — the compiler desugars it to `F: for<'a> Fn(&'a str) -> &'a str`. Lifetime elision in closure bounds is syntactic sugar for higher-ranked quantification. Most Rust programmers use HRTBs regularly without writing `for<'a>` explicitly.

HRTBs are essential when a function must *create* references with lifetimes that are not known to the caller — internal borrows, temporary allocations, or iterator adaptors that borrow from their environment. They extend Level 2's expressiveness by allowing lifetime universals to appear in positions that ordinary lifetime parameters cannot reach.

## Parametricity at Level 2

Chapter 4 established that Level 1 functions obey full parametricity — they preserve all relations on their type parameters. At Level 2, parametricity is **weakened but not destroyed**.

A Level 2 function must preserve all relations *consistent with its bounds*. The bound restricts the class of relations, and the implementation may exploit operations provided by the bound. But it still cannot distinguish between specific types within the bounded region.

Consider:

```rust
fn largest<T: Ord>(items: &[T]) -> Option<&T> {
    items.iter().max()
}
```

Parametricity at Level 2 says: for any two types `A: Ord` and `B: Ord`, and any *order-preserving* relation R between A and B, applying `largest` commutes with R. The function treats all orderable types uniformly — it cannot distinguish `i32` from `String` from `Metres`, as long as each is ordered.

The free theorem is weaker than at Level 1. At Level 1, a function `fn<T>(&[T]) -> Option<&T>` can only select an element by position (first, last, arbitrary fixed position). At Level 2, with `T: Ord`, it can select the *largest* or *smallest* — the bound grants access to the ordering, and the parametricity theorem must account for it. The implementation space is wider, so the type tells you less about the behaviour. But it still tells you something: the function selects an element based on the ordering, and nothing else.

## The View from Level 2

Level 2 is where most Rust programmers live when they write generic code. It is the sweet spot of the model — enough abstraction to avoid repetition, enough constraint to perform useful operations, and enough structure to reason about.

From Level 2, the landscape becomes visible:

**Downward** lies Level 0 — the concrete ground where every type is fixed and every operation is specific. Level 0 is where the generic function ultimately arrives, after monomorphisation collapses the family into its fibres.

**Upward** lies Level 1 — the rarefied domain of pure structure, where nothing is known about the type and the free theorems are absolute. Level 1 is the limiting case of Level 2: what remains when you remove all bounds.

**Laterally** lies Level 3 — the proof domain. Level 2 *states* propositions; Level 3 *supplies* proofs. The bounds in a Level 2 function signature are demands that Level 3's impl blocks must satisfy. The two levels are complementary: predicates and witnesses, hypotheses and evidence.

The programmer working at Level 2 is making conditional universal claims — "for all types satisfying these predicates, this property holds" — and relying on the proof domain to ensure that the predicates are substantiated for each concrete type. The next chapter examines that proof domain in its own right: the space of impl blocks, blanket impls, coherence, and the structure that makes Rust's proof system consistent.
