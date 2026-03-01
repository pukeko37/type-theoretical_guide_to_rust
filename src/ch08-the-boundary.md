# The Boundary

The previous five chapters ascended through the type category 𝒯 level by level — from concrete types at Level 0, through parametric polymorphism at Level 1, constrained generics at Level 2, proof witnesses at Level 3, to type constructors and GATs at Level 4. At each level, the boundary made a brief appearance: monomorphisation collapsed generic families, proof witnesses were consumed and inlined, phantom types were erased to zero bytes. But the boundary itself was never the subject. It was background — a transformation that happened between the chapter's type-level analysis and the generated machine code, acknowledged but not examined.

This chapter makes the boundary the subject — and in doing so, makes it a lens through which the entire type system becomes visible from a single vantage point.

The boundary is the forgetful functor F: 𝒯 → 𝒱 — the mapping from the type category to the value category. It is the mechanism by which Rust's compile-time structure is transformed into runtime behaviour. Every feature of Rust's type system, from newtypes to lifetimes, from blanket impls to typestate machines, passes through this mapping. What the functor preserves determines what Rust can do at runtime. What it erases determines what Rust gets for free.

The boundary is also where structures that were introduced separately — in different chapters, at different levels — reveal their common shape. Inherent impls, trait impls, and proof-carrying phantom types were each presented as distinct mechanisms. Viewed from the boundary, they are three expressions of a single principle: compile-time proof that the forgetful functor erases for free. The boundary chapter is therefore both an analysis of the functor's mechanics and a synthesis of the proof system as a whole.

Understanding the boundary precisely — what crosses it, what does not, what partly crosses it, and at what cost — is the key to understanding Rust's design as a whole. The boundary is not a line. It is a membrane with specific channels, each with its own direction, cost, and information capacity.

## The Forgetful Functor

A functor, in the categorical sense, is a structure-preserving map between categories. It maps objects to objects, morphisms to morphisms, and respects composition: if two morphisms compose in the source category, their images compose in the target category.

The boundary F: 𝒯 → 𝒱 is a **forgetful** functor — it preserves some structure while deliberately discarding the rest. Every functor preserves something; what makes F forgetful is that many distinct objects in 𝒯 map to the same object in 𝒱. The functor "forgets" the distinctions between them.

The familiar newtype example makes this concrete:

```rust
struct Metres(f64);
struct Seconds(f64);
```

Chapter 1 established that F maps both to the same `f64` — the type distinction that prevents dimensional confusion is exactly what the functor forgets. But in this chapter we can add a crucial framing: the forgetting is not information loss. The information has already done its work. The compiler verified that no `Metres` was used where a `Seconds` was expected. The proof is complete; the evidence is no longer needed. F discards it because retaining it would impose a runtime cost — and the zero-cost abstraction guarantee means that type-level structure must not survive as runtime overhead.

The forgetful functor has a systematic action at each level:

| Level | 𝒯 (compile time) | F | 𝒱 (runtime) |
|---|---|---|---|
| 0 | Type identity (`Metres ≠ Seconds`) | → | Erased (same bytes) |
| 1 | Generic family (`fn f<T>`) | → | Set of monomorphised functions |
| 2 | Trait bounds (`T: Ord`) | → | Erased (verified and discarded) |
| 3 | Proof witnesses (`impl Ord for T`) | → | Inlined code or vtable entries |
| 4 | Phantom parameters, typestate | → | Zero bytes, zero instructions |

The functor is **faithful to computation** — the runtime code does exactly what the type-level specification demanded — but **forgetful of logic** — the reasons, relationships, and proof structures that justified the computation vanish completely.

## Monomorphisation: Collapsing the Family

Monomorphisation is the boundary's most visible operation. It takes a generic function — a family of functions indexed by type parameters — and collapses it into a finite set of concrete functions, one for each type that the program actually uses.

Consider a generic function at Level 1:

```rust
fn wrap<T>(value: T) -> Option<T> {
    Some(value)
}

fn main() {
    let a = wrap(42_i32);
    let b = wrap("hello");
    let c = wrap(3.14_f64);
}
```

In 𝒯, `wrap` is a single object — a universally quantified function ∀T. T → Option\<T\>. It is one entity with a parametric structure. It does not exist as three separate functions; it exists as a *family*, a fibration over the base space of all types.

When F is applied, the family collapses:

```text
F(wrap<T>) = { wrap_i32: i32 → Option<i32>,
               wrap_str: &str → Option<&str>,
               wrap_f64: f64 → Option<f64> }
```

Each member of the family becomes a separate function in 𝒱, with its own machine code, its own memory layout assumptions, its own calling convention. The parametric structure — the fact that these three functions are instances of a single universal specification — is gone. In 𝒱, they are unrelated functions that happen to have similar shapes.

### What Monomorphisation Preserves and Discards

Monomorphisation preserves **computational behaviour** — each monomorphised function does exactly what the generic specification says it should do for its particular type. It preserves **composition** — if the generic code calls other generic functions, the monomorphised version calls the corresponding monomorphised versions. This is what makes F a functor, not merely a function: it respects the compositional structure of the source category.

Monomorphisation discards:
- **Universality**: the fact that the function works for all T, not just the types used in this program
- **Parametricity constraints**: the guarantee that the function treats all types uniformly
- **Family structure**: the relationship between `wrap_i32` and `wrap_f64` as members of the same family
- **Proof obligations**: any trait bounds that were checked and satisfied

The discarded information is substantial. In 𝒯, `wrap` and a hypothetical `wrap_with_logging` (which logs the type name before wrapping) have different types — the second cannot have type ∀T. T → Option\<T\> because it inspects T. After monomorphisation, both produce functions with the same signatures. The parametricity guarantee — the theorem that `wrap` must behave uniformly — exists only in 𝒯. F forgets it.

### The Cost of Monomorphisation

Monomorphisation trades compile time and binary size for runtime performance. Each instantiation generates new code. A function used with 50 different types produces 50 copies. The compiler performs optimisations on each copy independently — inlining trait methods, specialising comparisons, eliminating dead branches — producing code that is as fast as hand-written Level 0 code for each specific type.

This trade-off is itself a property of the boundary. A different boundary — one that preserves type abstraction at runtime, like Haskell's dictionary-passing or Java's type erasure with boxing — would produce a single function body that works for all types at the cost of indirection. Rust's boundary chooses the other end of the spectrum: maximum code generation, zero abstraction overhead.

The choice is not neutral. It determines what Rust can and cannot express efficiently. A data structure that is generic in 100 different types (rare in practice, but possible) generates 100 copies of every method. The boundary does not offer a middle ground — there is no "monomorphise these but dictionary-pass those" option at the language level. The boundary's strategy is uniform.

## Dynamic Dispatch: Projecting Through the Boundary

Monomorphisation is F's primary mechanism, but not its only one. Dynamic dispatch via `dyn Trait` is a second mechanism with fundamentally different properties.

When you write `Box<dyn Display>`, you are constructing an object that carries type information *across* the boundary — but in a degraded, partial form.

```rust
use std::fmt;

fn print_it(value: &dyn fmt::Display) {
    println!("{value}");
}

fn main() {
    let n: i32 = 42;
    let s: String = "hello".to_string();

    print_it(&n);
    print_it(&s);
}
```

In 𝒯, `print_it` takes a reference to any type that satisfies `Display`. In 𝒱, the function takes a **fat pointer**: a pair consisting of a data pointer (pointing to the value's bytes) and a vtable pointer (pointing to a table of function pointers implementing the trait methods).

The vtable is the interesting structure. It is what remains of the proof after F is applied:

| Proof component | In 𝒯 | After F (dyn) |
|---|---|---|
| Type identity | Known (e.g. `i32`) | Erased |
| Method implementations | Abstract trait methods | Function pointers in vtable |
| Supertrait relationships | Verified chain | Erased (vtable includes only this trait) |
| Associated types | Resolved to concrete | Erased (not accessible through `dyn`) |
| Other trait implementations | Full lattice position | Inaccessible |

The vtable is a **partial projection** of the proof into 𝒱. It preserves the *computational content* — you can call the trait methods — while erasing the *logical content* — you cannot ask which type is behind the pointer, what other traits it satisfies, or how the proof was derived.

### Monomorphisation versus Dynamic Dispatch

The two mechanisms are complementary projections of the same type-level structure:

| Property | Monomorphisation | Dynamic dispatch |
|---|---|---|
| Type identity | Resolved to concrete | Erased |
| Code generated | One copy per type | One copy, shared |
| Dispatch cost | Zero (inlined) | Indirect call through vtable |
| Binary size | Grows with instantiations | Constant |
| Optimisation | Full (specialised per type) | Limited (no type-specific optimisation) |
| Type information at runtime | None (erased) | Partial (vtable) |

Monomorphisation is F applied fully: everything in 𝒯 is resolved, specialised, and discarded. The result in 𝒱 is as if the generic code never existed — pure concrete computation.

Dynamic dispatch is F applied partially: the type identity is erased, but the trait interface survives as a vtable. The result in 𝒱 retains a trace of the type-level structure — enough to call methods, not enough to recover the original type.

In Reynolds' terminology, monomorphisation is **full defunctionalisation** — every abstract operation is replaced by a concrete implementation chosen at compile time. Dynamic dispatch is **partial defunctionalisation** — abstract operations are replaced by a dispatch table, with the concrete selection deferred to runtime.

## The Permeability Spectrum

The boundary is not a wall. It is a membrane with specific channels — mechanisms by which information can cross between 𝒯 and 𝒱. These channels vary in their direction, bandwidth, and cost.

### PhantomData: 𝒯-Only Information

`PhantomData<T>` is a zero-sized type that exists purely in 𝒯. The forgetful functor maps it to nothing — zero bytes, zero instructions. It is a one-way valve: type information enters 𝒯 through the phantom parameter, influences compilation (which methods are available, which trait bounds are checked, which type equalities hold), and then vanishes at the boundary.

```rust
use std::marker::PhantomData;

struct Token<Permission> {
    id: u64,
    _permission: PhantomData<Permission>,
}

struct ReadOnly;
struct ReadWrite;

impl Token<ReadOnly> {
    fn read(&self) -> u64 { self.id }
}

impl Token<ReadWrite> {
    fn read(&self) -> u64 { self.id }
    fn write(&mut self, new_id: u64) { self.id = new_id; }
}
```

In 𝒱, `Token<ReadOnly>` and `Token<ReadWrite>` are identical: a single `u64`. The phantom parameter occupies zero bytes. The distinction between read-only and read-write — the entire permission model — exists only in 𝒯. The boundary erases it completely, leaving behind only the guarantee that the permission rules were respected.

PhantomData is the boundary's most extreme channel: maximum information in 𝒯, zero presence in 𝒱. It is the purest expression of zero-cost abstraction.

### Const Generics: Values Crossing Upward

Const generics are the boundary's only channel for information flowing *from* 𝒱 *into* 𝒯. A const generic parameter is a value — a scalar from the value domain — that appears in a type.

```rust
fn dot_product<const N: usize>(a: &[f64; N], b: &[f64; N]) -> f64 {
    let mut sum = 0.0;
    let mut i = 0;
    while i < N {
        sum += a[i] * b[i];
        i += 1;
    }
    sum
}
```

The parameter `N` is a `usize` — a value — but it appears in the type `[f64; N]`. This is the third axis of the lambda cube: types depending on terms. The value `N` crosses upward from 𝒱 into 𝒯, influencing type-level computation (the array size is part of the type).

When F is applied, the const parameter collapses to a literal: `dot_product::<3>` becomes a function operating on `[f64; 3]`, with `N = 3` baked into the code as a constant. The parameter's journey is: born as a value, lifted into a type, and then lowered back into a concrete literal by monomorphisation.

This is Rust's only dependent-type mechanism. It is heavily restricted: only scalar types can serve as const generic parameters, and type-level arithmetic on them requires nightly features. The restriction exists because the boundary demands that all type-level computation be resolvable at compile time. Arbitrary value-to-type dependencies would require evaluating runtime expressions during type checking — which would dissolve the boundary entirely.

### TypeId: A Narrow Read-Only Channel

`TypeId` provides a minimal downward channel: runtime code can query the identity of a type, but cannot act on it in a type-safe way.

```rust
use std::any::TypeId;

fn is_string<T: 'static>() -> bool {
    TypeId::of::<T>() == TypeId::of::<String>()
}
```

`TypeId` is a hash of the type's identity, available at runtime. It allows equality comparisons — "is this the same type as that?" — but nothing more. You cannot recover the type's methods, its trait implementations, or its structure from a `TypeId`. It is a fingerprint, not a photograph.

In the categorical model, `TypeId` is a **section** of the forgetful functor — a partial inverse that maps a runtime value back to a type-level identity. But it is an extremely weak section: it recovers only the identity, not any of the structure that F erased. The functor forgets the type lattice position, the proof witnesses, the parametric structure. `TypeId` remembers only the name.

### dyn Trait: Partial Projection Downward

As discussed above, `dyn Trait` carries a partial projection of the proof domain into 𝒱. The vtable preserves method implementations as function pointers while erasing type identity and lattice position. The cost is an indirect function call per method invocation and a pointer-width overhead for the vtable reference.

### The Spectrum

These four mechanisms, together with monomorphisation itself, form a spectrum of boundary permeability — extending the per-level view from the opening table to show each crossing's direction and cost:

| Mechanism | Direction | Information preserved | Runtime cost |
|---|---|---|---|
| `PhantomData<T>` | 𝒯 only | Full type-level structure | Zero |
| Const generics | 𝒱 → 𝒯 | Scalar values in types | Zero (monomorphised) |
| `TypeId` | 𝒯 → 𝒱 (read-only) | Type identity only | Minimal (hash comparison) |
| `dyn Trait` | 𝒯 → 𝒱 (partial) | Method implementations | Vtable pointer + indirect call |
| Monomorphisation | 𝒯 → 𝒱 (full erasure) | Computational content only | Zero overhead, code size cost |

Each mechanism represents a different trade-off between information preservation and runtime cost. PhantomData pays nothing because it carries nothing into 𝒱. Const generics pay nothing because monomorphisation resolves them to literals. TypeId pays a minimal cost for minimal information. `dyn Trait` pays a real cost — indirection, lost optimisation opportunities — for the ability to work with heterogeneous types at runtime.

The spectrum is not an accident. It reflects a design principle: the boundary permits exactly as much information to cross as the programmer explicitly requests, and charges exactly the runtime cost that the crossing requires. There is no hidden overhead, no implicit boxing, no surprise allocations. The cost of each crossing is visible in the syntax: `dyn` in the type signature, `PhantomData` in the struct definition, `const N` in the generic parameter list.

## Lifetimes: The Most Dramatic Erasure

Lifetimes are the most striking example of the boundary's forgetful nature. An entire sub-logic — a proof system for reference validity — operates in 𝒯, is verified by the borrow checker, and vanishes without a trace when F is applied.

```rust
fn first_word(s: &str) -> &str {
    match s.find(' ') {
        Some(i) => &s[..i],
        None => s,
    }
}
```

The lifetime annotations (elided here, inferred as `fn first_word<'a>(s: &'a str) -> &'a str`) assert: the returned reference is valid for the same region as the input reference. The borrow checker verifies this by tracking the reference's provenance through the function body — the output is always a sub-slice of the input, so it necessarily lives as long as the input.

In 𝒱, after F is applied, the function takes a pointer and a length (the `&str` fat pointer) and returns a pointer and a length. There are no lifetimes. There are no validity proofs. The pointers are bare memory addresses, indistinguishable from C pointers — except that the lifetime proof, verified and erased, guarantees they are safe.

### The Lifetime Sub-Logic

Chapter 2 described lifetimes as a sub-logic within 𝒯. Let us be precise about what this means.

The lifetime system has:
- **Propositions**: lifetime bounds like `'a: 'b` ("reference `'a` outlives reference `'b`") and `T: 'a` ("type `T`'s data is valid for at least lifetime `'a`")
- **Inference rules**: the borrow checker's analysis, which derives lifetime relationships from the control flow of the program
- **A partial order**: `'static` is the top element (outlives everything), and the subtyping relationship `'long: 'short` means a reference valid for `'long` can be used where `'short` is expected
- **Proof checking**: the borrow checker verifies that all lifetime constraints are satisfiable — that no reference outlives its referent

This is a complete deductive system. It has hypotheses (lifetime annotations), inference rules (the borrowing rules), and theorems (the verified constraints). The borrow checker is the proof checker for this system.

And all of it is erased by F. Every lifetime annotation, every borrow check, every proof of reference validity — discarded at the boundary. The resulting machine code uses raw pointers. The safety guarantee is real, but the mechanism that established it has no runtime presence.

This is zero-cost abstraction at its most extreme. The lifetime system is arguably the most sophisticated part of Rust's type system — the part that most distinguishes Rust from other languages — and it contributes exactly zero bytes and zero instructions to the compiled program. The entire proof system exists to justify the use of bare pointers instead of garbage collection or reference counting. The boundary erases the proofs and keeps the bare pointers.

### Comparison with Other Memory Management Strategies

The lifetime erasure is best understood by contrast:

| Strategy | Proof of validity | Runtime presence | Cost |
|---|---|---|---|
| Rust (lifetimes) | Static proof, borrow checker | Erased completely | Zero |
| Garbage collection (Java, Go) | Dynamic tracing at runtime | GC roots, pause times | Throughput + latency |
| Reference counting (Swift, Python) | Dynamic count at runtime | Counter per allocation | Increment/decrement per copy |
| Manual (C, C++) | Programmer's informal reasoning | None | Zero, but unsound |

Rust and C have the same runtime cost for memory management: zero. The difference is that Rust's zero cost is backed by a verified proof, while C's zero cost is backed by the programmer's fallible judgement. The boundary enables this: by hosting the proof entirely in 𝒯, Rust gets the safety of garbage collection with the cost profile of manual management. The boundary is the mechanism that makes this possible.

## The World Stack: Davies and Pfenning Revisited

Chapter 1 introduced Davies and Pfenning's modal analysis, which frames compile time and runtime as possible worlds connected by an accessibility relation. At the boundary chapter, this framing earns its full weight.

In the modal logic interpretation:
- **Compile time** is the current world — the world of reasoning, proof, and type-level computation.
- **Runtime** is an accessible future world — the world where computation actually occurs.
- The **necessity operator** (□) marks information available in the current world that will persist into the future world. Constants, type-level decisions baked into monomorphised code, and vtable layouts are all □-typed: they are determined now and available later.
- The **possibility operator** (◇) marks information that exists only in the future world. User input, I/O results, and dynamically computed values are ◇-typed: they cannot be known at compile time.

The boundary F is the transition between worlds. Crossing the boundary means moving from the current world (where type-level reasoning is possible) into the future world (where only computation occurs). What survives the transition is precisely the □-information: the necessary truths that the current world has established.

### What the Modal Framing Reveals

The modal framing clarifies several features of the boundary:

**Lifetimes are current-world proofs about the future world.** When you annotate a function with lifetime `'a`, you are asserting (in the current world, at compile time) that a reference will be valid during a region of execution in the future world (at runtime). The borrow checker verifies this assertion in the current world. The assertion is then erased — it was about the future world, but the future world does not need to check it again. The proof has already been established.

**Const generics are future-world values pulled into the current world.** A const generic `N: usize` takes a value that would normally be a future-world entity (a runtime integer) and lifts it into the current world (a type-level parameter). This is the possibility operator's limited inverse: ◇-information (a runtime value) is made into □-information (a compile-time constant) by restricting it to values that can be determined statically.

**`dyn Trait` is □-information about the future world.** A vtable is a compile-time decision (which methods to include, how to lay them out) that persists into the future world as a data structure. It is necessarily true (□) that the vtable contains the correct function pointers — this was verified at compile time and cannot change at runtime.

## Moggi's Computational Monads and the Boundary

Eugenio Moggi's 1991 work on computational monads provides another lens on the boundary. Moggi showed that computational effects — state, exceptions, I/O, non-determinism — can be modelled by distinguishing between *values* (pure data) and *computations* (processes that may have effects). A monad is the mathematical structure that mediates between the two.

In Rust's two-category model, the distinction between values and computations maps onto the distinction between 𝒱's pure objects (data in memory) and 𝒱's effectful morphisms (functions that may allocate, panic, perform I/O, or diverge). The boundary itself is not a monad, but it shares a structural similarity with monadic mediation: just as a monad separates "what the value is" from "how it is computed," the boundary separates "what the type says" from "how the computation runs."

The connection becomes concrete in Rust's treatment of `Result` and `Option`. These types are not monads in the Haskell sense — Rust has no `Monad` trait, as Chapter 7 explained — but they encode computational effects (failure, absence) as values. The `?` operator performs monadic bind: it sequences computations that may fail, propagating errors through the return type. This is effect handling through the type system — an effect encoded in 𝒯 (the `Result` type) and checked at compile time (the `?` operator's type constraints), with the effect's runtime mechanism (early return on error) present in 𝒱.

Moggi's framework helps explain why Rust encodes effects in types rather than in a separate effect system: Rust's boundary demands that all compile-time structure be resolvable to concrete runtime code. An effect system would require the boundary to translate abstract effect annotations into concrete effect-handling code — which is precisely what Rust does with `Result` and `?`, but without a separate effect language. The effect *is* the type. The handling *is* the code. The boundary has nothing extra to erase.

## Programming Style as Altitude

Chapter 0 introduced the idea that programming style corresponds to a preferred altitude in the level hierarchy. The boundary chapter is the right place to make this precise, because the boundary is what connects altitude to runtime cost.

### Level 0: Below the Boundary

The C-style programmer works primarily in 𝒱. Types are data layout specifications. Functions are concrete transformations. There is minimal type-level structure to erase, so the boundary is nearly invisible. The code you write is close to what the compiler emits.

```rust
fn clamp(value: f64, min: f64, max: f64) -> f64 {
    if value < min { min } else if value > max { max } else { value }
}
```

No generics, no bounds, no proofs to erase. The boundary crossing is trivial: the 𝒯 representation (the type `f64`) maps directly to the 𝒱 representation (eight bytes). The programmer pays no abstraction cost because there is no abstraction.

### Levels 1–2: The Boundary Works Automatically

The generic programmer works in 𝒯, writing families of functions parameterised by types and bounds. The boundary does the heavy lifting: monomorphisation generates concrete code, trait bounds are verified and erased, proof witnesses are consumed and inlined.

```rust
fn clamp<T: PartialOrd>(value: T, min: T, max: T) -> T {
    if value < min { min } else if value > max { max } else { value }
}
```

The same logic, now universal. The boundary generates a separate `clamp` for each type used. Each generated function is identical in quality to the Level 0 version — the abstraction cost is zero at runtime, paid only at compile time (type checking, monomorphisation) and in binary size.

This is the altitude where most Rust programmers work most of the time, and it is the altitude where the boundary's zero-cost guarantee is most visibly at work.

### The dyn Boundary: Explicit Crossing

The `dyn Trait` programmer explicitly engages the boundary, choosing to preserve trait-level structure at runtime:

```rust
fn print_all(items: &[&dyn std::fmt::Display]) {
    for item in items {
        println!("{item}");
    }
}
```

Here the boundary is visible in the syntax: `dyn Display` marks the point where type identity is erased and replaced by a vtable. The programmer accepts the cost (indirection, lost monomorphisation) in exchange for the ability to store heterogeneous types in a single collection.

This is the boundary used as a tool — a deliberate decision about how much type information to preserve at runtime.

### Level 4: Maximum Altitude

The typestate programmer works at the highest altitude in 𝒯, encoding state machines and invariants in phantom types. The boundary erases everything:

```rust
use std::marker::PhantomData;

struct Locked;
struct Unlocked;

struct Door<State> {
    name: String,
    _state: PhantomData<State>,
}

impl Door<Locked> {
    fn unlock(self) -> Door<Unlocked> {
        Door { name: self.name, _state: PhantomData }
    }
}

impl Door<Unlocked> {
    fn open(&self) {
        println!("Opening {}", self.name);
    }
    fn lock(self) -> Door<Locked> {
        Door { name: self.name, _state: PhantomData }
    }
}

fn main() {
    let door: Door<Locked> = Door {
        name: "Front".to_string(),
        _state: PhantomData,
    };
    let door = door.unlock();
    door.open();
    let _door = door.lock();
}
```

In 𝒱, every `Door<State>` is the same: a `String`. The lock/unlock transitions are zero-cost state changes — the state exists only in 𝒯, and the boundary erases it completely. The maximum altitude produces the maximum erasure: an entire state machine verified at compile time, contributing nothing to the runtime.

### The Altitude Trade-Off

| Altitude | Type-level richness | Boundary work | Runtime cost | Flexibility |
|---|---|---|---|---|
| Level 0 | Minimal | Trivial | None | Maximum runtime flexibility |
| Level 1–2 | Parametric | Monomorphisation | Zero (code size cost) | Static polymorphism |
| `dyn Trait` | Partial | Vtable projection | Indirection cost | Runtime polymorphism |
| Level 4 | Maximum | Full erasure | Zero | Compile-time guarantees only |

The trade-off is not linear. Higher altitude does not simply cost more. Levels 1–2 and Level 4 are *both* zero-cost — the boundary erases them completely. The cost appears only when the programmer requests runtime polymorphism via `dyn Trait`, which is the boundary's partial-projection mode. The choice of altitude is a choice about *where* the polymorphism lives: in 𝒯 (resolved at compile time, zero cost) or partly in 𝒱 (resolved at runtime, indirection cost).

## Three Proof Mechanisms and the Boundary

The preceding chapters introduced three distinct mechanisms by which Rust programs carry proof. Each mechanism operates at a different level, encodes a different class of proposition, and interacts with the boundary in a different way. But the chapters presented them separately — inherent impls in Chapter 3, trait impls in Chapter 6, phantom types in Chapter 7 — and never placed them side by side. The boundary is the right lens for unifying them, because the boundary is where their differences become concrete: each proof mechanism is defined, in part, by what the forgetful functor does to it.

### Mechanism 1: Inherent Impl Blocks — Unnamed Implications

Chapter 3 presented the inherent `impl` block as a ground proposition. Each method is a proof of an implication: `fn area(&self) -> f64` proves "given a `Circle`, an area exists." The impl block as a whole is a collection of such implications — a bundle of claims about what the type can do.

These propositions are **unnamed**. There is no predicate, no trait, no label that other code can reference. You cannot write a bound `T: HasArea` based on the existence of an inherent `area` method. The propositions exist, and the proofs are valid, but they are invisible to the constraint system. They cannot participate in quantification.

At the boundary, inherent methods survive as concrete functions in 𝒱 — specific sequences of machine instructions with specific calling conventions. The functor preserves their computational content fully. There is nothing to erase: no generics, no bounds, no proof witnesses. The methods were concrete in 𝒯 and remain concrete in 𝒱. Of the three mechanisms, this is the one the boundary touches least.

### Mechanism 2: Trait Impl Blocks — Named Propositions

Chapter 6 presented `impl Trait for Type` as a proof of a named proposition. The trait is the predicate; the impl is the proof term; the orphan rule ensures coherence. Unlike inherent methods, trait propositions are **named** — they can be referenced in bounds, enabling quantification. When a function signature says `T: Display`, it references the `Display` predicate by name, and the caller must supply a proof (an impl) that the predicate holds for their concrete type.

The naming is what connects Level 2 (bounds as predicates) to Level 3 (impls as proofs). Without named propositions, there would be no way for a generic function to demand specific capabilities — it could only accept any `T` and do nothing with it (Level 1) or work with a specific concrete type (Level 0).

At the boundary, trait impls are erased — but their computational content is preserved. Under monomorphisation, the functor resolves each trait method call to a concrete function call, inlines it, and discards the proof. Under dynamic dispatch, the proof survives partially as a vtable — a table of function pointers extracted from the impl. In either case, the logical structure (the fact that a predicate was satisfied) vanishes. Only the operational content (the code that runs) remains.

### Mechanism 3: Proof-Carrying Types — Existence as Evidence

The third mechanism is the most subtle: a type whose very existence constitutes proof that some property holds. This is not about what methods a type has, or what traits it satisfies, but about the fact that a value of that type could only have been constructed if certain conditions were met.

Chapter 7 developed one version of this: typestate with `PhantomData`, where the phantom parameter encodes a verified property. `Connection<Authenticated>` carries no runtime evidence of authentication — the proof is the type itself. The only way to obtain a `Connection<Authenticated>` is through the `authenticate` method, which checks credentials and transitions the state. The type system ensures that no code path can produce an `Authenticated` connection without passing through the check.

But the pattern is broader than typestate. Consider a newtype wrapper that certifies a runtime property:

```rust
/// A non-empty collection. The only way to construct this
/// value is through `try_new`, which verifies non-emptiness.
struct NonEmpty<T> {
    items: Vec<T>,
}

impl<T> NonEmpty<T> {
    fn try_new(items: Vec<T>) -> Option<Self> {
        if items.is_empty() {
            None
        } else {
            Some(NonEmpty { items })
        }
    }

    /// Safe: the constructor guarantees at least one element.
    fn first(&self) -> &T {
        &self.items[0]
    }
}
```

Under Curry-Howard, `NonEmpty<T>` is not merely a conjunction (a `Vec<T>` and nothing else). It is a **proof-carrying type**: its existence is evidence that the contained vector is non-empty. The proof was established at construction time (the `try_new` check), and the type carries that proof forward through all subsequent code. Any function that receives a `NonEmpty<T>` can rely on non-emptiness without re-checking it — the type *is* the proof.

This reading applies wherever construction is guarded by validation: `NonZero<u32>` in the standard library, domain types like `EmailAddress` or `PortNumber` that parse and validate on construction, the `Validated<T>` conjunction from Chapter 2 (which is simultaneously a product type *and* a proof witness — its existence certifies that validation passed). In each case, the proposition is: "this property was verified." The proof is: "a value of this type exists."

At the boundary, proof-carrying types behave like ordinary values — the functor maps them to their runtime representation (a `Vec<T>`, a `u32`, a struct). The proof structure is erased entirely. In 𝒱, a `NonEmpty<Vec<i32>>` is indistinguishable from a `Vec<i32>`. The guarantee is real, but the evidence for it has vanished — consumed by the type checker, verified once, and discarded.

### The Three Mechanisms Compared

| | Inherent impl | Trait impl | Proof-carrying type |
|---|---|---|---|
| **Proposition** | 1. Unnamed implications | 2. Named predicate | 3. Existence of a type |
| **Proof** | Method body | `impl` block | Guarded constructor |
| **Level** | 0 (ground) | 2–3 (bounds + impls) | 0–4 (any level) |
| **Referenceable in bounds?** | No | Yes | Indirectly (via trait bounds on the wrapper) |
| **Boundary action** | Methods survive as functions | Erased (monomorphised) or partial (vtable) | Type identity erased; value survives |
| **Runtime cost of proof** | Zero | Zero (monomorphised) or indirection (dyn) | Zero (checked at construction) |

The three mechanisms are complementary, and idiomatic Rust often combines them. The typestate pattern from Chapter 7 is a clear example: a phantom parameter carries the state proposition (Mechanism 3), state-specific inherent impls make methods conditionally available (Mechanism 1), and trait bounds on the state parameter can constrain which states support which operations (Mechanism 2). All three proof mechanisms operate simultaneously, at different levels of the type category, and the boundary erases all of them to the same concrete representation in 𝒱.

The unifying principle is the boundary itself: Rust permits multiple forms of compile-time proof because the forgetful functor erases all of them equally. The richness of the proof system is possible precisely because it imposes no runtime cost. Whether the proof is a method body, an impl block, or the guarded construction of a value, the boundary discards the logical structure and preserves only the computation. The proofs are free, and so the language can afford to have many kinds.

## What the Boundary Cannot Cross

Some information is structurally unable to cross the boundary in either direction.

**Types cannot cross downward as values** (except via TypeId). You cannot pattern-match on a type at runtime, cannot branch on whether `T` is `i32` or `String`, cannot dispatch based on type identity. Types are 𝒯-only entities. (The `Any` trait provides a narrow exception via downcasting, but this is a controlled escape hatch, not a general mechanism.)

**Runtime values cannot cross upward into types** (except via const generics). You cannot use a runtime integer as an array size, a runtime string as a type name, or a runtime boolean as a type-level condition. The type system is closed before the program runs.

**Proofs cannot cross downward as data** (except via vtables). You cannot store an `impl Ord for T` in a variable, pass it as a function argument, or inspect it at runtime. Proofs are 𝒯-only entities that are consumed during compilation.

**Effects cannot cross upward into types** (Rust has no effect system). A function's type does not indicate whether it allocates memory, performs I/O, panics, or diverges. The `Result` and `Option` types encode *specific* failure modes, but there is no general mechanism for reflecting effects in the type system. This is a deliberate boundary constraint: encoding arbitrary effects in types would require the boundary to translate them, adding complexity to both 𝒯 and the boundary itself.

These restrictions define the boundary's shape. They are not limitations to be worked around — they are the structural properties that enable zero-cost abstraction. If types could cross downward, they would need runtime representation. If runtime values could cross upward freely, type checking would require program execution. If proofs could cross downward, they would have a runtime cost. Each restriction removes a class of runtime overhead by preventing a class of boundary crossing.

## The Boundary in Other Languages

Rust's boundary has a specific character: aggressive erasure, zero-cost abstraction, full monomorphisation. Other languages draw the boundary differently. Examining these alternatives clarifies what Rust's boundary costs and what it buys.

### Haskell: Dictionary-Passing Boundary

Haskell's boundary preserves more information than Rust's. Type class constraints are compiled to **dictionaries** — records of method implementations passed as implicit arguments at runtime. Where Rust's boundary erases proofs, Haskell's boundary preserves them as data structures.

```text
-- Haskell: the constraint survives as a runtime dictionary
sort :: Ord a => [a] -> [a]
-- compiles to approximately:
sort :: OrdDict a -> [a] -> [a]
```

The dictionary is a proof of `Ord a` that exists at runtime — it carries the comparison function as a field. This means Haskell's polymorphic functions are truly polymorphic at runtime: a single compiled function works for all types, consulting the dictionary for type-specific operations.

The trade-off: Haskell's boundary preserves abstraction at the cost of indirection. Every polymorphic call requires a dictionary lookup. Rust's boundary erases abstraction at the cost of code duplication. Neither is universally superior — they are different boundary designs optimised for different priorities.

### Java: Type-Erased Boundary

Java's generics use **type erasure** — but a different kind from Rust's. Java erases type parameters to their bounds (or to `Object` if unbounded) and inserts runtime casts at usage sites. The result is a single compiled method for all type parameter instantiations, working on boxed objects.

```text
// Java: erased to Object, cast at usage
<T> T identity(T x) { return x; }
// compiles to approximately:
Object identity(Object x) { return x; }
```

Java's boundary is simpler than Rust's — no monomorphisation, no code duplication — but it forces boxing (heap allocation for primitives) and runtime casts (checked type conversions). The abstraction has a cost in both performance and safety (unchecked casts in pre-generics code). Java's boundary was designed for backward compatibility, not zero-cost abstraction.

### Idris: Programmable Boundary

Idris offers what Rust does not: programmer control over the boundary. In Idris, the programmer can annotate individual types and proofs with **erasure annotations**, specifying which compile-time structures should be retained at runtime and which should be erased.

```text
-- Idris: explicit erasure annotation
myFunc : {0 prf : Eq a} -> a -> a -> Bool
```

The `0` before `prf` indicates that the proof argument is used zero times at runtime — it is compile-time only. Idris' boundary is *programmable*: the programmer decides, for each piece of type-level structure, whether it crosses the boundary.

This is possible because Idris is dependently typed: proofs and types can appear in runtime positions, so the question of what to erase is a genuine choice, not a foregone conclusion. In Rust, the boundary's behaviour is determined by the language: all proofs are erased, all generics are monomorphised. In Idris, the boundary is a dial that the programmer can turn.

### Coq: No Boundary (Almost)

In Coq, the boundary between types and values is nearly dissolved. Types, proofs, and programs inhabit the same universe. A proof is a value; a type is a value; a function can take a proof as an argument and return a type as a result.

When Coq code is *extracted* to a conventional language (OCaml, Haskell), an extraction boundary is imposed — proofs marked as `Prop` (logical) are erased, while proofs marked as `Set` (computational) are retained. But within Coq itself, there is no boundary. The type category and the value category are the same category.

This is the Calculus of Constructions at work — the top vertex of the lambda cube, where all three axes are active and all four dependencies (terms on terms, terms on types, types on types, types on terms) are available. The boundary's dissolution is the logical consequence of full dependent typing: if types can depend on runtime values, and values can depend on types, the distinction between compile time and runtime becomes untenable.

### The Spectrum of Boundaries

| Language | Boundary style | What survives | Cost |
|---|---|---|---|
| C | No type erasure (no generics) | N/A | N/A |
| Java | Type erasure + boxing | Casts, boxed representations | Boxing overhead |
| Haskell | Dictionary passing | Method dictionaries | Indirection per call |
| Rust | Full monomorphisation | Nothing (all erased) | Code size |
| Idris | Programmer-controlled erasure | Whatever is annotated | Varies |
| Coq | Near-dissolution (extraction) | Whatever is computational | Full representation |

Rust occupies the most aggressive erasure position among production languages. This is a principled choice: Rust is a systems language, and systems languages are defined by their commitment to predictable, minimal-overhead execution. The boundary is the mechanism that fulfils this commitment while permitting sophisticated type-level reasoning.

## The Boundary as Design Centre

The preceding chapters have presented the boundary as a transformation applied after the type-level work is done — a final step that collapses 𝒯 into 𝒱. But from a design perspective, the relationship is reversed: the boundary came first.

Rust was designed around the zero-cost abstraction principle: type-level structure must not impose runtime overhead. This principle *is* the boundary. The type system was built to be compatible with full erasure. The choice of monomorphisation over dictionary-passing, the orphan rule's enforcement of coherence, the restriction to compile-time-resolvable generics — all of these are consequences of a boundary that demands total erasure.

The boundary is not a feature of Rust's type system. It is the architectural constraint that *shapes* the type system. The type system is the way it is — with its particular strengths (zero-cost generics, lifetime safety, typestate encoding) and its particular limitations (no higher-kinded types, no runtime proof manipulation, no effect system) — because the boundary requires it.

This is the deepest insight of the two-category model. 𝒯 and 𝒱 are not independent categories that happen to be connected by a functor. 𝒯 is designed *for* the functor — designed to contain exactly the structure that can be usefully verified at compile time and then erased without trace. The boundary is not the last step in the pipeline. It is the first decision in the design.

## Looking Beyond

The boundary defines what Rust can express. Everything within the boundary — the five levels, the type lattice, the proof domain, the typestate machines — exists because it is compatible with zero-cost erasure. Everything beyond the boundary — dependent types, first-class proofs, effect systems, universe polymorphism — is excluded because it would require the boundary to change.

The final chapter examines what lies beyond. It maps the full space of type systems, places Rust precisely within it, and explores what becomes possible when the boundary is moved, weakened, or dissolved entirely.
