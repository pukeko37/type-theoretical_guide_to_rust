# The Proof Domain (Level 3)

The previous chapters examined types as propositions and trait bounds as predicates. Level 2 showed how a function signature states a conditional universal claim — "for all types satisfying these bounds, this property holds." But every conditional claim demands evidence. The bound `T: Ord` in a function signature is a hypothesis, and hypotheses must be discharged. Something must supply the proof that a particular type actually satisfies a particular trait.

That something is the `impl` block. And the space of all impl blocks — their relationships, their derivation rules, their constraints — constitutes a domain in its own right. This is Level 3: the **proof domain**.

At Levels 1 and 2, the programmer works with types and the propositions they satisfy. At Level 3, the focus shifts to the proofs themselves — the evidence that connects concrete types to abstract propositions. This shift in perspective reveals structure that is invisible from the lower levels: impl blocks form a category with its own objects, morphisms, and coherence conditions. Blanket impls are not mere convenience — they are proof constructors, systematic derivations that produce new proofs from existing ones. The orphan rule is not an arbitrary restriction — it is the condition that keeps the proof system consistent.

Chapter 2 established that impl blocks are proof terms. This chapter examines the *structure* of the proof domain as a mathematical object.

## Impl Blocks as Proof Objects

An impl block is a piece of evidence. When you write:

```rust
use std::fmt;

struct Metres(f64);

impl fmt::Display for Metres {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{:.2}m", self.0)
    }
}
```

you are constructing a proof of the proposition Display(Metres). The proof consists of a concrete implementation of every method the trait requires — here, just `fmt`. The compiler verifies the proof by checking that the method signatures match and the body type-checks. If the proof is valid, the proposition is established: `Metres` satisfies `Display`.

This is a **ground proof** — it concerns one specific type and one specific trait. It is a fact, not a derivation. It does not depend on other proofs or on any conditions. The proposition Display(Metres) is simply true, established by the evidence in the impl block.

Let us now consider the collection of all such proof objects for a given trait.

## The Category Impl(Trait)

For any trait `Trait`, we can define a category **Impl(Trait)** whose objects are the impl blocks witnessing that various types satisfy `Trait`.

Take `Display` as our example. The objects of Impl(Display) include:

- `impl Display for i32` (provided by the standard library)
- `impl Display for String` (provided by the standard library)
- `impl Display for bool` (provided by the standard library)
- `impl Display for Metres` (our proof above)
- Every other `impl Display for T` in the program

Each object in this category is a proof — a concrete piece of evidence that a specific type satisfies the `Display` proposition. The category Impl(Display) is the collection of all known proofs of Display-ness.

But a category needs more than objects. It needs **morphisms** — arrows between objects that express relationships. What are the morphisms in Impl(Trait)?

## Blanket Impls as Morphisms

The morphisms in Impl(Trait) are **blanket impls**: impl blocks that systematically derive new proofs from existing ones.

Consider the standard library's blanket impl:

```rust,ignore
impl<T: Display> ToString for T {
    fn to_string(&self) -> String {
        // uses Display::fmt internally
        format!("{self}")
    }
}
```

This is not a ground proof. It is a **proof constructor**: given any proof that `T: Display`, it produces a proof that `T: ToString`. It is a morphism in the proof domain — an arrow from Impl(Display) to Impl(ToString).

In categorical terms, this blanket impl is a **functor** between proof categories: it maps objects of Impl(Display) to objects of Impl(ToString), systematically and uniformly. For every type `T` that has a Display proof, the blanket impl automatically generates a ToString proof. No manual construction is needed — the derivation is mechanical.

```text
    Impl(Display)                    Impl(ToString)
    ┌────────────────┐               ┌────────────────┐
    │ Display for i32 │──────────────→│ ToString for i32│
    │ Display for String│────────────→│ ToString for String│
    │ Display for bool │─────────────→│ ToString for bool│
    │ Display for Metres│────────────→│ ToString for Metres│
    └────────────────┘               └────────────────┘
              blanket impl: ∀T: Display. ToString for T
```

Each arrow is an instance of the blanket impl, applied to a specific type. The blanket impl is the rule; the arrows are its applications. The functor maps the entire category at once.

### Multi-Premise Derivations

Some blanket impls derive a proof from multiple existing proofs:

```rust
trait Summarise {
    fn summarise(&self) -> String;
}

impl<T: std::fmt::Display + Clone> Summarise for Vec<T> {
    fn summarise(&self) -> String {
        let items: Vec<String> = self.iter().map(|x| x.to_string()).collect();
        format!("[{}]", items.join(", "))
    }
}
```

This derivation says: for all types T, if T satisfies Display *and* Clone, then `Vec<T>` satisfies Summarise. The proof draws from two premises (Display and Clone) and produces a conclusion for a different type (`Vec<T>` rather than `T` itself). In categorical terms, this is a functor from the product category Impl(Display) × Impl(Clone) into Impl(Summarise), composed with the `Vec` type constructor.

### Iterated Derivation

Blanket impls can chain. Consider how the standard library derives proofs for compound types:

```rust,ignore
// If T: Clone, then Vec<T>: Clone
impl<T: Clone> Clone for Vec<T> { ... }

// If T: Clone, then Option<T>: Clone
impl<T: Clone> Clone for Option<T> { ... }
```

Starting from a ground proof `impl Clone for i32`, the first blanket impl derives `impl Clone for Vec<i32>`. Applying the second derives `impl Clone for Option<Vec<i32>>`. Applying the first again derives `impl Clone for Vec<Option<Vec<i32>>>`. The derivation can iterate indefinitely, producing proofs for arbitrarily nested types from a single ground proof.

This iterated derivation has the structure of a **free construction**: given a set of ground proofs (the generators) and a set of derivation rules (the blanket impls), the proof domain is the closure of the generators under the rules. Every proof in the domain is either a ground proof or the result of finitely many rule applications.

The standard library's trait implementations are designed with this structure in mind. The ground proofs cover the primitive types (`i32`, `bool`, `char`, etc.), and the blanket impls propagate traits through the standard type constructors (`Vec`, `Option`, `Box`, `Result`, tuples). The result is that most compound types — `Vec<Option<String>>`, `(i32, bool, Vec<u8>)` — automatically satisfy the common traits, without any manual impl blocks. The proof domain generates them.

## The Orphan Rule as Coherence

The orphan rule is Rust's most distinctive constraint on the proof domain. It states: you may write `impl Trait for Type` only if you define either `Trait` or `Type` (or both) in the current crate. You cannot implement a foreign trait for a foreign type.

```rust,ignore
// In your crate — allowed, because you own Metres
impl Display for Metres { ... }

// In your crate — FORBIDDEN, because you own neither Display nor Vec
impl Display for Vec<i32> { ... }
```

Under the standard framing, this is presented as an arbitrary restriction or a practical necessity to avoid conflicts. Under the proof-domain framing, it is something more precise: the orphan rule is a **coherence condition** on the proof category.

### What Coherence Means

Coherence requires that for any type `T` and any trait `Trait`, there is **at most one** proof of Trait(T) in the entire program. No ambiguity. No conflicting evidence. If you ask "does `i32` satisfy `Display`?", there is exactly one answer, backed by exactly one proof.

Why does this matter? Consider what would happen without coherence. Suppose two crates could each provide `impl Ord for SomeType`, with different comparison functions. A generic function bounded by `T: Ord` would behave differently depending on which impl was selected. The same call to `sort` could produce different orderings. The program's behaviour would depend on the *choice* of proof — and in a system with proof erasure, that choice is invisible at runtime.

This is precisely the situation that Reynolds' **representation independence** theorem addresses. Reynolds showed in 1978 that if two implementations of an abstract type are related by a suitable simulation, no program can distinguish them. The formal justification for coherence is the *converse* requirement: since Rust erases proofs at the boundary, and erased proofs cannot be distinguished at runtime, the language must ensure that no observable behaviour depends on which proof is selected. The simplest way to ensure this is to require that there is only one proof — which is exactly what the orphan rule enforces.

Coherence is not about preventing bugs. It is about maintaining the soundness of proof erasure. If proofs are going to be erased, they must be unique — otherwise the erasure would discard information that matters.

### The Orphan Rule in Detail

The rule is more nuanced than "own the trait or the type." The precise condition involves the notion of a **local type** — a type defined in the current crate — and applies recursively through type parameters. The key cases:

```rust
# use std::fmt;
# struct Metres(f64);
// You own Metres → you can implement foreign traits for it
impl fmt::Display for Metres {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{:.2}m", self.0)
    }
}
```

```rust
// You own the trait → you can implement it for foreign types
trait Measurable {
    fn magnitude(&self) -> f64;
}

impl Measurable for f64 {
    fn magnitude(&self) -> f64 {
        self.abs()
    }
}
```

```rust,ignore
// You own neither Display nor Vec<i32> → FORBIDDEN
impl Display for Vec<i32> { ... }
```

The orphan rule draws a boundary around each crate's proof authority. Each crate can produce proofs involving its own types and traits, but cannot produce proofs that are entirely about foreign entities. This partitioning ensures that proofs cannot conflict: two crates cannot both provide `impl Trait for Type` unless at least one of them owns `Trait` or `Type`, in which case the other's attempt would be rejected by the rule.

### Coherence and the Category

In categorical terms, coherence means that Impl(Trait) is a **thin category**: between any two objects, there is at most one morphism. More precisely, for any type `T`, either there exists exactly one proof of Trait(T) (via a ground impl or a unique blanket derivation chain), or there is no proof. There is never a choice between proofs.

This thinness is what makes proof erasure safe. If the category were thick — if multiple proofs could coexist for the same type and trait — then erasing the proof would lose information about which proof was intended. In a thin category, there is nothing to lose. The proof's existence matters; its identity does not.

### Overlap and Specialisation

The thin category property depends on the compiler rejecting **overlapping impls** — two blanket impls that could both apply to the same type. Consider:

```rust,ignore
trait Describe {
    fn describe(&self) -> String;
}

// These two impls overlap: a type implementing both Debug and Display
// would have two candidate proofs of Describe.
impl<T: std::fmt::Debug> Describe for T {
    fn describe(&self) -> String { format!("{:?}", self) }
}

impl<T: std::fmt::Display> Describe for T {   // ERROR: conflicting impl
    fn describe(&self) -> String { format!("{}", self) }
}
```

Stable Rust forbids this entirely. If two blanket impls could apply to any overlapping set of types, the proof category would no longer be thin — some types would have two proofs of the same proposition, and the compiler would have no principled way to choose between them. The overlap ban is the constructive mechanism that enforces coherence.

**Specialisation** (available only on nightly, via `#![feature(specialization)]` or `#![feature(min_specialization)]`) relaxes this rule in a controlled way. It permits overlapping impls when one is strictly *more specific* than the other — the more specific impl "wins" for the types it covers:

```rust,ignore
#![feature(min_specialization)]

trait Describe {
    fn describe(&self) -> String;
}

// General blanket impl — applies to all types with Debug.
impl<T: std::fmt::Debug> Describe for T {
    default fn describe(&self) -> String { format!("{:?}", self) }
}

// Specialised impl — applies to types with Display (a stricter condition
// in practice, since Display implies a more curated representation).
// Where both apply, this one wins.
impl<T: std::fmt::Display> Describe for T {
    fn describe(&self) -> String { format!("{}", self) }
}
```

From the categorical perspective, specialisation relaxes the thin category to a **preorder-enriched** category. Multiple candidate proofs may exist for a single type, but a deterministic selection rule — "choose the most specific" — ensures a unique winner. The category is no longer thin (multiple morphisms exist), but a priority ordering on morphisms recovers uniqueness of selection.

The tension is real. Specialisation enables useful patterns — the standard library would benefit from `impl<T: Clone> ToOwned for T` with a special case for `str` that avoids unnecessary allocation. But it weakens the absolute coherence guarantee that makes proof erasure straightforward. Specialisation remains on nightly precisely because the proof-theoretic consequences — particularly around soundness when default associated types interact with lifetime bounds — are still being resolved. The decision to stabilise it is, in effect, a question about how much thinness the proof category can afford to lose.

## Newtype Wrappers: Working Within the Orphan Rule

The orphan rule occasionally prevents you from providing a proof you need. You want `impl SomeForeignTrait for SomeForeignType`, but you own neither. The standard solution is the **newtype wrapper**: a new type that you *do* own, wrapping the foreign type.

```rust
use std::fmt;

struct FormattedList(Vec<String>);

impl fmt::Display for FormattedList {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}
```

`FormattedList` is a new type — you own it — that wraps `Vec<String>`. You can implement any trait for `FormattedList` because you own it. The newtype is a one-line definition: `struct FormattedList(Vec<String>)`.

In the proof domain, the newtype is a **relabelling of the proof target**. You cannot prove Display(Vec\<String\>) directly (orphan rule), so you create a new node in the type lattice — FormattedList — that is representationally identical to `Vec<String>` but logically distinct. Then you prove Display(FormattedList), which the orphan rule permits.

The cost is that you must convert between `Vec<String>` and `FormattedList` at the point of use. The forgetful functor erases the distinction — both are identical in 𝒱 — so the conversion is zero-cost. The newtype exists purely in 𝒯 as a device for navigating the orphan rule.

```rust
# use std::fmt;
# struct FormattedList(Vec<String>);
# impl fmt::Display for FormattedList {
#     fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
#         write!(f, "[{}]", self.0.join(", "))
#     }
# }
fn display_items(items: Vec<String>) {
    let formatted = FormattedList(items);
    println!("{formatted}");
}
```

The newtype pattern is the primary tool for extending the proof domain when the orphan rule blocks direct proof construction. It is used so frequently in Rust that it has become idiomatic — a standard technique that every experienced Rust programmer recognises.

### Newtypes and Proof Variation

Newtypes also serve a second purpose in the proof domain: providing **alternative proofs** for the same proposition, in a system that allows only one proof per type.

Consider ordering. `f64` has a partial ordering (some values like `NaN` are incomparable), but sometimes you want a total ordering that handles `NaN` in a specific way. You cannot write a second `impl Ord for f64` — coherence forbids it. But you can define a newtype:

```rust
#[derive(Clone, Copy, PartialEq)]
struct TotalF64(f64);

impl Eq for TotalF64 {}

impl PartialOrd for TotalF64 {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        Some(self.cmp(other))
    }
}

impl Ord for TotalF64 {
    fn cmp(&self, other: &Self) -> std::cmp::Ordering {
        self.0.total_cmp(&other.0)
    }
}
```

`TotalF64` is a new type with a new proof of `Ord` — a proof that uses `total_cmp` to impose a total ordering on all `f64` values including `NaN`. The original `f64` retains its partial ordering; `TotalF64` has a total ordering. Two different proofs of ordering-ness, coexisting without conflict, because they apply to different types.

In the proof domain, the newtype is a mechanism for *proliferating proof targets*. If you need multiple proofs of the same proposition but coherence requires uniqueness, you create distinct types (distinct nodes in the lattice) and attach distinct proofs to each. The representational identity is preserved by the forgetful functor — in 𝒱, they are the same bytes — but the logical identity in 𝒯 is distinct, and distinct identities can carry distinct proofs.

## Representation Independence

John Reynolds' representation independence theorem, introduced in Chapter 1, provides the formal foundation for coherence. The theorem states: if two implementations of an abstract type are related by a suitable simulation, then no program can distinguish them.

In the proof domain, this means: if two impl blocks for the same trait and type are "equivalent" (they produce observationally identical behaviour), then replacing one with the other does not change any program's behaviour. Representation independence tells us that the *choice* of proof witness is unobservable — as long as the proofs agree on their observable behaviour.

Rust takes this one step further. Rather than requiring proofs to agree (which would be difficult to verify), Rust simply prohibits the situation from arising: the orphan rule ensures there is only one proof, so the question of agreement never arises. This is coherence by *uniqueness* rather than coherence by *agreement*.

The distinction matters when comparing with other languages:

**Haskell** has a similar coherence requirement for type class instances, but it is weaker in practice: orphan instances (instances defined in a module that owns neither the class nor the type) are permitted with a warning. This can lead to incoherence — different modules seeing different instances of the same class for the same type. The result is subtle bugs where the behaviour of a function depends on which module's instance is in scope.

**Scala** with its `given` (formerly `implicit`) mechanism permits multiple proof witnesses in scope, with an explicit resolution order. This is coherence by *priority* rather than by uniqueness — the language chooses the "best" proof according to a set of rules, but the programmer must understand the rules to predict which proof is selected.

**Rust** is the most conservative: one proof, no exceptions, no resolution needed. The cost is the orphan rule's restrictions. The benefit is absolute predictability — the proof selected for `T: Trait` is never ambiguous.

## Proof Erasure Revisited

Chapter 2 established that Rust erases proofs at the boundary. At Level 3, we can examine precisely what this erasure means for the proof domain.

Consider a function that uses a trait bound:

```rust
fn sort_and_first<T: Ord>(mut items: Vec<T>) -> Option<T> {
    items.sort();
    items.into_iter().next()
}
```

In 𝒯, this function requires a proof of Ord(T) — an impl block. The proof provides the comparison function that `sort` uses. At the call site, the compiler resolves the proof:

```rust
# fn sort_and_first<T: Ord>(mut items: Vec<T>) -> Option<T> {
#     items.sort();
#     items.into_iter().next()
# }
fn main() {
    let result = sort_and_first(vec![3, 1, 2]);
    // The compiler resolves: impl Ord for i32
    // and monomorphises sort_and_first::<i32>
    assert_eq!(result, Some(1));
}
```

When the boundary is crossed (monomorphisation), the proof is consumed:

1. The compiler finds `impl Ord for i32`.
2. It extracts the comparison function from the impl block.
3. It generates `sort_and_first::<i32>` with the comparison function inlined.
4. The impl block itself — the proof object — is discarded. In the generated code, there is no trace of the proof. There is only the comparison function, baked into the sorted function's machine code.

The proof's *content* (the comparison function) survives the boundary. The proof's *identity* (the fact that it was an impl of Ord for i32) does not. This is the forgetful functor applied to the proof domain: F preserves computational content while erasing logical structure.

### What Survives and What Does Not

| Proof component | Above the boundary (𝒯) | Below the boundary (𝒱) |
|---|---|---|
| Method implementations | Abstract trait methods | Inlined concrete code |
| Proof identity (which impl) | Known, used for coherence checking | Erased |
| Supertrait chain | Verified, used for bound checking | Erased |
| Associated type bindings | Known, used for type resolution | Resolved to concrete types |
| The impl block as an object | Exists as a proof term | Does not exist |

The erasure is thorough. In the generated machine code, there is no record that `sort_and_first::<i32>` was ever generic, that it ever required `Ord`, or that a proof was ever consulted. The code is identical to what you would write by hand at Level 0.

## First-Class Proofs: What Other Languages Do

Rust erases proofs. Not all languages make this choice. Understanding the alternatives illuminates what Rust gives up and what it gains.

### Haskell: Dictionary Passing

Haskell's type classes — the closest analogue to Rust's traits — are implemented via **dictionary passing**. When a Haskell function has a type class constraint, the compiler translates it into a function that takes an extra argument: the **dictionary**, a record of the class's methods for the specific type.

In pseudocode, the Haskell equivalent of:

```text
-- Haskell
sort :: Ord a => [a] -> [a]
```

is compiled to something like:

```text
-- After dictionary translation
sort :: OrdDict a -> [a] -> [a]
```

where `OrdDict a` is a record containing the comparison function (and the superclass dictionaries for `Eq`). The dictionary is passed at runtime, as an ordinary function argument. The proof *survives* the boundary as a data structure.

This gives Haskell capabilities that Rust lacks. A Haskell program can:
- Store a dictionary in a data structure (store a proof for later use)
- Pass a dictionary explicitly to override the default instance
- Compute with dictionaries at runtime

The cost is runtime overhead: every polymorphic function call carries an extra pointer (or several, for compound constraints). Dictionary passing is why Haskell's polymorphic code is typically slower than monomorphised code — the proof has a runtime representation, and that representation has a cost.

### Scala: Given Instances

Scala's `given` mechanism (formerly `implicit`) takes a middle path. Proof witnesses (called *given instances*) are values that the compiler passes automatically, but the programmer can also pass them explicitly:

```text
// Scala
given Ordering[Metres] with
  def compare(a: Metres, b: Metres): Int = ...

// Explicit passing
def sortWith[T](items: List[T])(using ord: Ordering[T]): List[T] = ...
```

The `using` keyword makes the proof argument visible in the signature. A caller can supply an alternative proof explicitly, overriding the default resolution. Proofs are first-class values — they can be stored, passed, and computed with — but the compiler provides them automatically when an unambiguous resolution exists.

This is more flexible than Rust's approach. It allows multiple orderings for the same type, selected by the caller:

```text
// Scala
sortWith(items)(using Ordering.reverse)  // sort in reverse order
sortWith(items)(using myCustomOrdering)   // sort with a custom ordering
```

Rust cannot express this directly. The proof of `Ord for T` is always the unique one selected by coherence. To get alternative orderings, you must use the newtype pattern — creating a new type with a different proof, rather than selecting among proofs for the same type.

### The Trade-Off

The three approaches occupy a spectrum:

| Approach | Proofs at runtime | Multiple proofs | Overhead |
|---|---|---|---|
| Rust | Erased | Forbidden (coherence) | Zero |
| Haskell | Dictionaries | Forbidden (coherence, in principle) | Pointer per constraint |
| Scala | Given values | Permitted (explicit selection) | Depends on usage |
| Coq/Idris | First-class values | Permitted (fully dependent) | Full value cost |

Rust sits at the far end of the spectrum: maximum erasure, maximum coherence, zero overhead. The price is inflexibility — you cannot select among proofs or manipulate proofs as values. The benefit is the zero-cost guarantee and the absolute predictability of proof resolution.

## Unsafe: Axioms Without Proof

In the proof domain so far, every proof is verified by the compiler. The impl block is checked: method signatures must match, bodies must type-check, lifetime constraints must be satisfied. The proof system is *sound* — only true propositions can be proved.

`unsafe` introduces a different proof mechanism: the **axiom**. An axiom is a proposition the programmer asserts without the compiler verifying it.

### Unsafe Traits and Unsafe Impls

An `unsafe trait` declares a proposition with proof obligations that go beyond what the compiler can check:

```rust
/// # Safety
/// Implementors must ensure that the type can be safely
/// transferred across thread boundaries.
unsafe trait ThreadSafe {
    fn check(&self) -> bool;
}
```

An `unsafe impl` provides an *unverified* proof — the programmer claims the obligations are met, but the compiler trusts without checking:

```rust
# unsafe trait ThreadSafe { fn check(&self) -> bool; }
struct MyBuffer {
    data: Vec<u8>,
}

// The programmer asserts — without compiler verification — that
// MyBuffer is safe to transfer across threads.
unsafe impl ThreadSafe for MyBuffer {
    fn check(&self) -> bool { true }
}
```

Under Curry-Howard, an `unsafe impl` is an axiom added to the proof system. It extends the system's reach — propositions that the compiler cannot verify can now be asserted — but it introduces a soundness risk. If the axiom is false (the type is *not* actually safe to transfer across threads), the entire proof system becomes unsound. False axioms are the source of undefined behaviour.

The standard library's most important unsafe traits are `Send` and `Sync`. Their proof obligations — "this type can be sent to another thread" and "this type can be shared between threads" — are semantic properties that depend on runtime behaviour the compiler cannot fully analyse. The `unsafe` marker signals that the proof obligation exists, and the `unsafe impl` signals that the programmer takes responsibility for discharging it.

### Unsafe Blocks: Suspended Verification

`unsafe` blocks serve a different but related purpose. An unsafe block is not an unverified proof — it is a region where the compiler **suspends certain verification**. Inside an unsafe block, the programmer can perform operations that the compiler cannot prove safe: dereferencing raw pointers, calling unsafe functions, accessing mutable statics.

```rust
fn read_at(data: &[u8], index: usize) -> u8 {
    assert!(index < data.len());
    // SAFETY: bounds check above guarantees index is valid
    unsafe { *data.as_ptr().add(index) }
}
```

The proof obligations — pointer validity, no aliasing violations, proper initialisation — are not discharged by the compiler. They are discharged by the programmer's reasoning, expressed in the `// SAFETY` comment. The compiler trusts the programmer within the block's scope.

### The Boundary Perspective

Unsafe code does not change F — the forgetful functor erases unsafe proofs exactly as it erases safe ones. Below the boundary, there is no distinction between a function whose `Send` impl was compiler-verified and one whose `Send` impl was asserted by the programmer. The difference is entirely in 𝒯: safe proofs are verified, unsafe proofs are trusted. The generated machine code is identical.

This means that an incorrect `unsafe impl` is undetectable below the boundary. The unsoundness manifests not as a type error but as undefined behaviour — a violation of the proof system's assumptions that the runtime cannot catch. This is why unsafe is Rust's sharpest tool: it extends the proof system's power at the cost of transferring the burden of correctness from the compiler to the programmer.

## The Proof Domain in Practice

Let us trace a realistic example through the proof domain, identifying each proof and its role.

```rust
use std::fmt;

trait Unit: fmt::Display + Copy {
    fn abbreviation() -> &'static str;
}

#[derive(Clone, Copy)]
struct Kg;

impl fmt::Display for Kg {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "kg")
    }
}

impl Unit for Kg {
    fn abbreviation() -> &'static str { "kg" }
}

#[derive(Clone, Copy)]
struct Lb;

impl fmt::Display for Lb {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "lb")
    }
}

impl Unit for Lb {
    fn abbreviation() -> &'static str { "lb" }
}

struct Measurement<U: Unit> {
    value: f64,
    _unit: std::marker::PhantomData<U>,
}

impl<U: Unit> Measurement<U> {
    fn new(value: f64) -> Self {
        Measurement { value, _unit: std::marker::PhantomData }
    }
}

impl<U: Unit> fmt::Display for Measurement<U> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{:.2} {}", self.value, U::abbreviation())
    }
}
```

The proof domain for this code contains:

**Ground proofs:**
- `impl Display for Kg` — Kg can be displayed
- `impl Unit for Kg` — Kg is a unit (which implies Display + Copy)
- `impl Display for Lb` — Lb can be displayed
- `impl Unit for Lb` — Lb is a unit
- `impl Copy for Kg` — Kg can be copied (via `derive`)
- `impl Copy for Lb` — Lb can be copied (via `derive`)

**Derived proofs (blanket impls):**
- `impl<U: Unit> Display for Measurement<U>` — a blanket impl that derives Display for *any* measurement whose unit satisfies Unit. This is a morphism in the proof domain: it maps each proof in Impl(Unit) to a proof in Impl(Display) for the corresponding Measurement type.

**Derivation chain for `Measurement<Kg>`:**
1. `impl Unit for Kg` (ground proof)
2. `impl Display for Measurement<Kg>` (derived from step 1 via the blanket impl)

The blanket impl is the engine of the proof domain. It says: *I do not need to prove Display for every concrete measurement type. I prove it once, generically, and the proof domain generates the specific instances.* This is the power of Level 3: proof construction, not just proof assertion.

## The Structure of Derivation

The proof domain's derivation structure has several properties worth noting.

### Proof Derivations Are Unique

Because of coherence, the derivation chain that produces a proof is unique. There is exactly one path from the ground proofs to `impl Display for Measurement<Kg>`: the blanket impl applied to `impl Unit for Kg`. No alternative derivation exists — if it did, it would create a second proof of Display(Measurement\<Kg\>), violating coherence.

This uniqueness is what makes the proof domain a **thin category** — at most one morphism between any two objects. In richer proof systems (Coq, for instance), multiple derivations of the same proposition can coexist, and the choice between them can affect the program. In Rust, the choice never arises.

### Derive Macros as Proof Generators

Rust's `derive` attribute is a mechanism for automatically generating ground proofs:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct Point {
    x: i32,
    y: i32,
}
```

Each trait in the `derive` list generates an impl block — a ground proof. `derive(Debug)` produces `impl Debug for Point` by inspecting the struct's fields and generating formatting code. `derive(Clone)` produces `impl Clone for Point` by cloning each field.

In the proof domain, `derive` is a **proof generator**: given a type definition, it mechanically constructs proofs of standard propositions. The proofs it generates are ground proofs (not blanket impls), but they follow a uniform pattern determined by the struct's structure.

The derive mechanism is Level 3 automation. Rather than writing each proof by hand, the programmer declares which propositions should hold, and the compiler generates the evidence. This is analogous to proof tactics in a theorem prover — automated strategies for constructing proofs of standard forms.

### Conditional Derivation: Where Blanket Impls and Derive Interact

The derived impls often carry implicit conditions. `derive(Clone)` on a generic struct generates:

```rust,ignore
// What derive(Clone) generates for a generic struct:
impl<T: Clone> Clone for Wrapper<T> {
    fn clone(&self) -> Self {
        Wrapper(self.0.clone())
    }
}
```

This is a *conditional* proof: Clone(Wrapper\<T\>) holds *if* Clone(T) holds. It is a blanket impl generated by the derive macro. The condition `T: Clone` propagates the requirement downward — to clone the wrapper, you must be able to clone the contents.

This interaction between derive and blanket impls is the proof domain's everyday mechanism. When you write `#[derive(Clone)]` on a generic struct, you are not just requesting a proof — you are requesting a *proof schema* that derives Clone for each instantiation of the struct, conditional on the type parameter satisfying Clone.

## Auto Traits: Compiler-Generated Proofs

`derive` requires the programmer to opt in — you must write `#[derive(Clone)]` to request the proof. **Auto traits** go further: the compiler generates proofs *automatically*, without any annotation.

`Send` and `Sync` are the primary auto traits. The compiler proves `Send` for a type by **structural induction**: if every field of a struct is `Send`, the struct itself is `Send`. No `derive` annotation is needed; no impl block is written. The proof is generated silently, based on the type's structure.

```rust
struct Pair {
    name: String,    // String: Send ✓
    count: u64,      // u64: Send ✓
}
// Pair: Send — proved automatically by the compiler.
// No annotation required.
```

This is **proof by structural recursion** — a well-known technique in type theory, here implemented by the compiler as an automatic derivation rule. The base cases are the primitive types (`i32`, `bool`, `u8`, etc., are all `Send`). The inductive step is: if all fields are `Send`, the composite is `Send`. The compiler applies this rule to every type in the program, building proofs bottom-up from the primitives.

### Negative Impls: Refutation

Auto traits are *negative by default for certain types*. `Rc<T>` is not `Send` because its reference count is not atomic — sharing an `Rc` across threads would cause data races. `Cell<T>` is not `Sync` because interior mutability without synchronisation is unsound.

Rust expresses this through **negative impls** — the only place in the language where a proposition is explicitly *denied*:

```rust,ignore
// In the standard library (using an unstable feature):
impl<T: ?Sized> !Send for Rc<T> {}
impl<T: ?Sized> !Sync for Cell<T> {}
```

Under Curry-Howard, a negative impl is a **refutation**: an assertion that the proposition is false. Where a positive impl says "here is evidence that `T` satisfies `Trait`," a negative impl says "no such evidence can exist." The proposition Send(Rc\<T\>) is not merely unproved — it is actively denied.

This denial propagates structurally. If a struct contains an `Rc<T>` field, the compiler's structural induction encounters a field that is not `Send`, and the automatic proof fails. The struct inherits the negative property: it is not `Send` either, unless the programmer provides an `unsafe impl Send` — an axiom overriding the compiler's analysis.

### The Default-With-Override System

Together, auto traits and negative impls form a **default-with-override** proof system:

1. **Auto-derivation** generates `Send`/`Sync` proofs for types whose structure permits it — the default.
2. **Negative impls** deny the proof for types whose semantics forbid it — the override downward.
3. **`unsafe impl`** asserts the proof for types whose safety the compiler cannot verify — the override upward, at the programmer's risk.

This system extends the proof domain beyond what `derive` can express. `derive` generates proofs the programmer *requests*; auto traits generate proofs the programmer *did not request*, based on structural analysis. The connection to unsafe is direct: `Send` and `Sync` are `unsafe trait` — implementing them manually is an axiom (as discussed in the previous section), while the automatic derivation is safe because it follows structural rules the compiler can verify.

## The Boundary at Level 3

What happens to the proof domain when the forgetful functor F is applied?

The entire domain — every impl block, every derivation chain, every coherence relationship — is erased. In 𝒱, there are no proofs. There are only the concrete functions that the proofs licensed.

For a monomorphised function:

```text
sort_and_first::<i32>
```

The proof `impl Ord for i32` has been consumed. The comparison function has been extracted and inlined. The impl block itself — the proof that `i32` is orderable — has no runtime representation. The question "is `i32` orderable?" cannot even be asked at runtime, because the proof machinery does not exist in 𝒱.

For a `dyn Trait` object, the story is different. A vtable is a *partial projection* of a proof into 𝒱. The proof's methods survive as function pointers, but the proof's identity (which type, which impl) is erased. The vtable preserves *computational content* — you can call the methods — but not *logical content* — you cannot ask which type the proof was about.

This dual treatment — full erasure for monomorphic dispatch, partial projection for dynamic dispatch — is characteristic of Rust's boundary. The proof domain exists entirely in 𝒯. The boundary either erases it completely (static dispatch, zero cost) or projects it partially (dynamic dispatch, vtable cost). Either way, the proof as a logical object does not survive.

## What Level 3 Reveals

The Level 2 programmer sees traits as interfaces and bounds as requirements. The Level 3 programmer sees something different: **a deductive system**.

The impl blocks are axioms and derivation rules. The blanket impls are inference rules — "if these propositions hold, then this proposition also holds." The orphan rule is the consistency constraint. The derive macro is an automated prover. And the entire system is verified at compile time and erased at the boundary.

This perspective changes how you think about API design:

**Trait definitions are not just interfaces — they are propositions you are introducing into the logic.** When you define a trait, you create a new predicate that types can satisfy. Every impl block for that trait is a proof. Every blanket impl involving that trait is an inference rule. Choose your propositions carefully: each one extends the deductive system, and the coherence constraint means the extension is permanent within a crate's scope.

**The orphan rule is not a restriction — it is a partition of proof authority.** Each crate has a jurisdiction: the types and traits it defines. Within that jurisdiction, it has full authority to construct proofs. The orphan rule prevents jurisdictions from conflicting. The newtype pattern is the standard way to extend your jurisdiction when needed.

**Blanket impls are the most powerful tool at Level 3.** A single blanket impl can generate proofs for an unbounded number of types. `impl<T: Display> ToString for T` covers every displayable type, present and future. This is proof by universal derivation — a single rule that applies wherever its premises hold.

The proof domain is where Rust's type system achieves its most distinctively logical character. At Level 2, types express propositions. At Level 3, the compiler manages a deductive system — constructing, verifying, and erasing proofs in a formally coherent framework. The programmer participates in this system every time they write an impl block, whether they think of it in these terms or not.

The next chapter ascends to Level 4, where types themselves become the objects of computation — type constructors, GATs, and the typestate pattern. The proofs of Level 3 will remain essential, but the propositions they prove will grow more abstract.
