# Propositions as Types

This chapter is the heart of the book. Everything before it was preparation; everything after it is elaboration. The claim that types are propositions and implementations are proofs is not a pedagogical device or a loose analogy. It is a theorem — the Curry-Howard correspondence — and this chapter develops it in full, shows how it maps onto Rust, identifies where Rust truncates it, and demonstrates what it means to read a Rust program as a logical argument.

## The Correspondence

In 1934, Haskell Curry observed a structural similarity between the axioms of combinatory logic and the types of certain primitive functions. In 1969, William Alvin Howard extended this observation into a full isomorphism between natural deduction (a system of formal proof) and the simply typed lambda calculus (a system of typed computation). The result, now known as the **Curry-Howard correspondence**, establishes that:

- Every **proposition** in logic corresponds to a **type** in a type system.
- Every **proof** of a proposition corresponds to a **value** (an inhabitant) of the corresponding type.
- Every **rule of logical deduction** corresponds to a **rule of type construction**.

The correspondence is not a metaphor. It is a precise structural isomorphism: the same mathematical object can be read as a logical statement *or* as a type specification, and the two readings are formally equivalent. A proof that A implies B is *the same thing* as a function from A to B. A proof that A and B both hold is *the same thing* as a pair containing a value of type A and a value of type B.

This chapter develops the correspondence connective by connective, showing how each logical operation maps onto a Rust type construction. We begin with the full table, then examine each row in detail.

## The Full Correspondence

| Logic | Type Theory | Rust |
|---|---|---|
| Proposition | Type | Type or trait bound |
| Proof | Inhabitant of a type | Value, `impl` block |
| True (⊤) | Unit type | `()` |
| False (⊥) | Uninhabited type | `!` (never type) |
| Conjunction (A ∧ B) | Product type (A × B) | `(A, B)`, structs |
| Disjunction (A ∨ B) | Sum type (A + B) | `enum { A(A), B(B) }` |
| Implication (A → B) | Function type (A → B) | `fn(A) -> B` |
| Negation (¬A) | A → ⊥ | `fn(A) -> !` |
| Universal (∀x. P(x)) | Dependent product (Π) | `fn f<T: Bound>(...)` |
| Existential (∃x. P(x)) | Dependent sum (Σ) | `-> impl Trait` |

Each row of this table is a theorem. Let us examine them.

## Truth: The Unit Type

The simplest proposition is **truth** — the proposition that is always provable, that carries no information, that asserts nothing beyond its own validity.

In logic, this is written ⊤ (top, or *verum*). In Rust, it is the unit type `()`:

```rust
fn trivial() -> () {
    ()
}
```

The unit type has exactly one inhabitant: `()`. A proof of truth requires no evidence — you simply produce the trivial witness. Every function that "returns nothing" in Rust is actually returning a proof of truth. When a block ends with a semicolon, its type becomes `()` — under Curry-Howard, the block's result is a trivial proof, constructed implicitly by discarding the preceding expression's value.

This may seem pedantic, but it matters. The unit type is the identity element for conjunction (product types), just as truth is the identity element for logical AND. A struct with one field of type `()` is isomorphic to a struct without that field — the trivial proposition contributes nothing.

## Falsity: The Never Type

The opposite of truth is **falsity** — the proposition that has no proof. It is the claim that cannot be substantiated.

In logic, this is written ⊥ (bottom, or *falsum*). In Rust, it is the never type `!`:

```rust
fn diverge() -> ! {
    loop {}
}
```

The never type has **no inhabitants**. There is no value of type `!`. A function that claims to return `!` can never actually return — it must diverge (loop forever), panic, or exit the process. The type `!` is a promise that "this code path is unreachable."

Under Curry-Howard, the uninhabited type is the false proposition. Since there is no proof of falsity, there is no value of the uninhabited type. And just as anything follows from a false premise in logic (*ex falso quodlibet*), any type can be produced from an uninhabited type in Rust. We can see this clearly by defining our own uninhabited type:

```rust
enum Void {}

fn absurd(x: Void) -> String {
    // This function is vacuously valid — it can never be called,
    // because no one can construct a value of type Void to pass in.
    match x {}
}
```

The enum `Void` has no variants, so it has no inhabitants — it is a user-defined ⊥. The empty match `match x {}` on a value of `Void` is logically sound: since `x` has no possible values, there are no cases to handle, and the expression can be assigned any type. This is *ex falso* made executable.

Uninhabited types appear naturally in Rust wherever impossibility needs to be expressed. The standard library's `Infallible` type (used as the error type of infallible conversions) plays the same role as our `Void`: `Result<T, Infallible>` is a result that can never be an error, which is logically T ∨ ⊥, which simplifies to T. The `!` (never) type serves as Rust's built-in ⊥ in return position — a function returning `!` can never return, and the compiler uses this to reason about control flow and dead code.

## Conjunction: Product Types

Logical **conjunction** — A ∧ B, "A and B" — corresponds to the **product type**: a type whose values contain *both* a value of type A *and* a value of type B.

In Rust, the canonical product type is the tuple:

```rust
fn prove_conjunction() -> (u32, bool) {
    (42, true)
}
```

To produce a value of type `(u32, bool)`, you must supply both a `u32` and a `bool`. This is exactly what a proof of A ∧ B requires: evidence for A *and* evidence for B. The tuple is the proof; its components are the sub-proofs.

Structs are named product types:

```rust
struct Measurement {
    value: f64,
    unit: String,
    timestamp: u64,
}
```

A `Measurement` is a conjunction of three propositions: there exists a value (f64), a unit description (String), and a timestamp (u64). To construct a `Measurement`, you must supply all three. The struct fields are the conjuncts.

**Elimination** (using a conjunction) mirrors logical AND-elimination. From a proof of A ∧ B, you may extract a proof of A or a proof of B. In Rust, this is field access:

```rust
# struct Measurement { value: f64, unit: String, timestamp: u64 }
fn extract_value(m: &Measurement) -> f64 {
    m.value  // AND-elimination: from (A ∧ B ∧ C), extract A
}
```

The correspondence extends to nested products. A struct with five fields is a five-fold conjunction. A tuple `(A, B, C)` is A ∧ B ∧ C. The unit type `()` is the empty conjunction — truth.

## Disjunction: Sum Types

Logical **disjunction** — A ∨ B, "A or B" — corresponds to the **sum type**: a type whose values contain *either* a value of type A *or* a value of type B, but not both.

In Rust, the canonical sum type is the enum:

```rust
enum StringOrInt {
    Text(String),
    Number(i64),
}
```

A value of type `StringOrInt` is either `Text(s)` for some `String s`, or `Number(n)` for some `i64 n`. It is a proof of "either String or i64," but you do not know which until you examine it.

**Introduction** (constructing a disjunction) mirrors OR-introduction. From a proof of A, you may conclude A ∨ B:

```rust
# enum StringOrInt { Text(String), Number(i64) }
fn from_number(n: i64) -> StringOrInt {
    StringOrInt::Number(n)  // OR-introduction: from B, conclude A ∨ B
}
```

**Elimination** (using a disjunction) mirrors OR-elimination: if you know A ∨ B, and you can show that A implies C and B implies C, then C holds. In Rust, this is pattern matching:

```rust
# enum StringOrInt { Text(String), Number(i64) }
fn describe(value: StringOrInt) -> String {
    match value {
        StringOrInt::Text(s) => format!("text: {s}"),      // A → C
        StringOrInt::Number(n) => format!("number: {n}"),   // B → C
    }
}
```

The `match` expression requires you to handle every variant — every disjunct. If you add a new variant to the enum and forget to handle it, the compiler rejects the match. Under Curry-Howard, this is sound: an incomplete case analysis is an invalid proof.

Rust's `Option<T>` is the disjunction T ∨ ⊤ (either a value of type T, or nothing — where `None` carries the unit type). `Result<T, E>` is the disjunction T ∨ E (either success or error). These are among the most used types in Rust, and they are, logically, disjunctions.

## Implication: Function Types

Logical **implication** — A → B, "if A then B" — corresponds to the **function type**: the type of values that, given a value of type A, produce a value of type B.

```rust
fn length_is_even(s: &str) -> bool {
    s.len() % 2 == 0
}
```

This function is a proof of the proposition "if there is a string, then there is a boolean." More precisely, it is a proof of `&str → bool`. Given evidence of the hypothesis (a string reference), it produces evidence of the conclusion (a boolean).

Implication is the most fundamental connective in the correspondence. It is the bridge between propositions: if you know A, and you know A → B, then you know B. In type theory, this is function application: if you have a value of type A and a function of type A → B, applying the function gives you a value of type B.

**Chained implication** is function composition:

```rust
fn is_short_even_string(s: &str) -> bool {
    s.len() < 10 && s.len() % 2 == 0
}
```

**Multi-argument functions** are iterated implications. A function `fn f(a: A, b: B) -> C` corresponds to A → B → C, which is equivalent (under currying) to A → (B → C): given A, produce a function that, given B, produces C.

## Negation

Logical **negation** — ¬A, "not A" — is defined in classical logic as A → ⊥: if A were true, we could derive falsehood. Equivalently: A is false if assuming A leads to a contradiction.

In Rust, ¬A corresponds to a function from A to the never type:

```rust
# #[allow(dead_code)]
fn not_possible(x: std::convert::Infallible) -> ! {
    match x {}
}
```

This function compiles because it can never be called — no value of type `Infallible` exists to pass as the argument. The empty match `match x {}` is valid for the same reason as the `Void` example below: there are no cases to handle. Under Curry-Howard, the function is a proof of `¬Infallible` (i.e. `Infallible → ⊥`): it demonstrates that assuming Infallible is inhabited leads to an uninhabited type, which is the constructive definition of negation.

In practice, negation in Rust is more commonly expressed through the *absence* of an impl block. The proposition "`i32` implements `Iterator`" is false in Rust — not because there exists a proof of its negation, but because no proof of the proposition exists. This is a difference between Rust's type system and full constructive logic. Rust can express "this type does not implement this trait" only through absence, not through a first-class negation type. We will return to this limitation when we discuss proof erasure at the boundary.

## Universal Quantification: Generics

Logical **universal quantification** — ∀x. P(x), "for all x, P(x) holds" — corresponds to the **dependent product type** (Π-type): a type whose values are functions that work uniformly for all possible inputs from some domain.

In Rust, this is a generic function:

```rust
fn wrap_in_vec<T>(value: T) -> Vec<T> {
    vec![value]
}
```

Read this signature under Curry-Howard: *for all types T, given a value of type T, produce a value of type `Vec<T>`*. The generic parameter `<T>` is the universal quantifier. The function body is the proof that the proposition holds for all T. The compiler verifies this proof by checking that the body makes no assumptions about T that are not warranted by its bounds (which here are empty — no assumptions at all).

When bounds are added, the quantification becomes conditional:

```rust
fn to_string_vec<T: std::fmt::Display>(items: &[T]) -> Vec<String> {
    items.iter().map(|item| item.to_string()).collect()
}
```

This reads: *for all types T, **if** T is displayable, **then** given a slice of T values, produce a vector of strings*. The bound `T: Display` is the hypothesis of a conditional universal statement. In logical notation: ∀T. (Display(T) → (`&[T]` → `Vec<String>`)). The universal quantifier and the implication are both present, nested.

The Rust programmer who writes `<T: Display>` is not "adding a constraint." They are stating the hypothesis of a universal theorem. The function body is the proof. The `impl Display for ConcreteType` that a caller must provide is the discharge of the hypothesis for a specific T.

### Multiple Quantifiers

Multiple generic parameters are iterated universal quantification:

```rust
fn zip_pair<A, B>(a: A, b: B) -> (A, B) {
    (a, b)
}
```

This reads: ∀A. ∀B. (A → B → A ∧ B). For all types A and B, given evidence of A and evidence of B, produce evidence of their conjunction. This is a proof of AND-introduction, universally quantified over the types.

## Existential Quantification: `impl Trait` in Return Position

Logical **existential quantification** — ∃x. P(x), "there exists an x such that P(x)" — corresponds to the **dependent sum type** (Σ-type): a type that packages together a witness and a proof that the witness satisfies a predicate.

In Rust, this is `impl Trait` in return position:

```rust
fn make_counter() -> impl Iterator<Item = u32> {
    0u32..
}
```

Read this under Curry-Howard: *there exists a type T such that T implements `Iterator<Item = u32>`, and this function returns a value of that type*. The caller knows the returned value satisfies `Iterator<Item = u32>`, but does not know *which* specific type it is. The concrete type (here, `std::ops::Range<u32>`) is hidden — existentially quantified away.

The key property of existential quantification is that the witness is opaque. The caller can use the returned value through the trait interface (`next()`, `map()`, `filter()`, etc.) but cannot name its type, match on its structure, or use any capability not guaranteed by the bound. This is the logical dual of universal quantification: where ∀ says "this works for any type you choose," ∃ says "I have a specific type, but I am not telling you which one."

The asymmetry between argument position and return position in Rust reflects this duality precisely:

- `fn f<T: Trait>(x: T)` — **universal**: the *caller* chooses T. The function must work for all T satisfying Trait.
- `fn f() -> impl Trait` — **existential**: the *callee* chooses T. The caller must work with any T satisfying Trait.

This is the ∀/∃ duality made concrete in function signatures.

### `dyn Trait` as Existential Quantification

There is a second form of existential quantification in Rust: `dyn Trait`. A `Box<dyn Iterator<Item = u32>>` also says "there exists some type implementing `Iterator<Item = u32>`" — but unlike `impl Trait`, it is a runtime existential. The concrete type is erased not just from the caller's view but from the compiled code: the value is accessed through a vtable, and the original type identity is lost.

Under Curry-Howard, `dyn Trait` is an existential that survives the boundary. The forgetful functor F does not fully erase it — it transforms it into a vtable, which is a partial projection of the proof. The trait methods survive as function pointers; the type identity does not. This is a *runtime* existential, and it has a cost: dynamic dispatch, heap allocation (when boxed), and loss of monomorphisation opportunities.

By contrast, `impl Trait` in return position is a *compile-time* existential. The concrete type is known to the compiler and monomorphised away — the existential is erased at the boundary, leaving no runtime trace. Both are existential quantification, but they live on different sides of the boundary.

## Trait Bounds as Propositions

With the individual connectives established, we can now read trait bounds as what they are: propositions in a formal logic.

A single bound states a single proposition:

```rust
fn minimum<T: Ord>(a: T, b: T) -> T {
    if a <= b { a } else { b }
}
```

The bound `T: Ord` asserts: "T possesses a total ordering." This is a proposition about T, and the existence of `impl Ord for T` is its proof.

Compound bounds are conjunctions:

```rust
fn display_sorted<T: Ord + std::fmt::Display>(items: &mut Vec<T>) {
    items.sort();
    for item in items.iter() {
        println!("{item}");
    }
}
```

The bound `T: Ord + Display` asserts: "T possesses a total ordering **and** T can be formatted for display." The `+` operator in bounds is logical conjunction (∧). To call this function for some type `MyType`, you must provide two proofs: `impl Ord for MyType` and `impl Display for MyType`.

**Where clauses** allow more complex propositions:

```rust
fn convert_and_collect<I, T, U>(iter: I) -> Vec<U>
where
    I: Iterator<Item = T>,
    T: Into<U>,
{
    iter.map(|item| item.into()).collect()
}
```

The where clause is a set of hypotheses: "if I is an iterator over T values, **and** T is convertible into U, **then** the iterator can be converted into a vector of U values." Each line of the where clause is a separate hypothesis in the argument.

**Supertraits** encode implication between propositions:

```rust
# use std::fmt;
trait Summary: fmt::Display {
    fn summarise(&self) -> String;
}
```

The supertrait bound `Summary: Display` asserts: "if a type satisfies Summary, then it satisfies Display." In logical notation: ∀T. (Summary(T) → Display(T)). This is a universally quantified implication — a theorem about the relationship between two propositions.

## Impl Blocks as Proof Terms

If trait bounds are propositions, then `impl` blocks are their proofs. An `impl` block is a **proof term** — a piece of evidence that a particular type satisfies a particular proposition.

```rust
struct Celsius(f64);

impl std::fmt::Display for Celsius {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{:.1}°C", self.0)
    }
}
```

This `impl` block is a proof of the proposition "Celsius can be formatted for display." The proof consists of a concrete implementation of the `fmt` method — evidence that the required operation exists and produces the required result.

The compiler verifies this proof by checking that every method required by the trait is provided, with the correct signature and return type. If a method is missing, the proof is incomplete. If a return type is wrong, the proof is invalid. The compiler's error messages are, under this reading, reports of proof failures.

**Blanket impls** are universally quantified proofs — proof schemas that apply to all types satisfying certain conditions:

```rust
# trait Printable {
#     fn print(&self);
# }
impl<T: std::fmt::Display> Printable for T {
    fn print(&self) {
        println!("{self}");
    }
}
```

This reads: *for all types T, if T is displayable, then T is printable*. It is a universal theorem in the proof domain. The `impl` block provides the proof construction — given the hypothesis (T: Display), it derives the conclusion (T: Printable) by using `println!`, which requires `Display`.

The **orphan rule** ensures that proofs are unique: for any given type and trait, at most one `impl` block exists in the entire program. Under Curry-Howard, this is a **coherence condition** on the proof system. It ensures that the logical system is consistent — that there is no type for which two conflicting proofs of the same proposition could cause ambiguity. We will examine this in depth in Chapter 6.

## Reading a Program as a Logical Argument

Now let us apply the full correspondence to a realistic Rust program and read it as a logical argument from start to finish.

Consider a small system for validating and processing configuration data:

```rust
use std::fmt;
use std::str::FromStr;

/// A validated configuration value that has been parsed
/// from a string and confirmed to be within acceptable bounds.
struct Validated<T> {
    value: T,
    source: String,
}

trait Bounded {
    fn is_in_bounds(&self) -> bool;
}

fn parse_and_validate<T>(input: &str) -> Result<Validated<T>, String>
where
    T: FromStr + Bounded,
    T::Err: fmt::Display,
{
    let value: T = input
        .parse()
        .map_err(|e: T::Err| format!("parse error: {e}"))?;

    if !value.is_in_bounds() {
        return Err(format!("value out of bounds: {input}"));
    }

    Ok(Validated {
        value,
        source: input.to_string(),
    })
}
```

Read this under Curry-Howard:

**The type `Validated<T>`** is a conjunction: a value of type T ∧ a source string. It is a product type asserting that both a parsed value and its origin exist together.

**The trait `Bounded`** is a proposition: "this type has a notion of acceptable bounds." Any type that implements `Bounded` has proven it can answer the question "is this value acceptable?"

**The function signature** is a universally quantified conditional statement:

> ∀T. (FromStr(T) ∧ Bounded(T) ∧ Display(T::Err)) → (&str → (Validated(T) ∨ String))

In prose: *For all types T, if T can be parsed from a string, T has bounds checking, and T's parse error is displayable, then given a string reference, we can produce either a validated value or an error message.*

**The where clause** lists the hypotheses:
- `T: FromStr` — hypothesis 1: T can be parsed from strings
- `T: Bounded` — hypothesis 2: T has bounds checking
- `T::Err: Display` — hypothesis 3: parse errors can be rendered as text

**The return type `Result<Validated<T>, String>`** is a disjunction: either a validated value (success) or an error message (failure). This is honest: the function acknowledges that parsing may fail, and encodes both outcomes in the type.

**The function body** is the proof. It proceeds step by step:

1. **Parse**: using hypothesis 1 (FromStr), attempt to parse the input. This may fail — the `?` operator handles the `Err` case, mapping the parse error to a string using hypothesis 3 (Display).

2. **Validate**: using hypothesis 2 (Bounded), check whether the parsed value is acceptable. If not, return `Err`.

3. **Construct**: if both steps succeed, construct the `Validated` conjunction — pairing the value with its source.

Each step uses exactly the hypotheses it needs. The proof is valid because every operation is justified by a stated hypothesis. If you remove any bound from the where clause, the corresponding step in the body becomes unjustified, and the compiler rejects the proof.

This is what it means to read a Rust program as a logical argument: the signature states the theorem, the bounds state the hypotheses, and the body constructs the proof.

## Lifetimes as a Sub-Logic

Lifetimes are the most dramatic example of the Curry-Howard correspondence in Rust — and the most dramatic example of proof erasure at the boundary.

A lifetime annotation is a **proposition about temporal validity**: the claim that a reference will remain valid for a certain region of execution. The borrow checker is a **proof checker** for these propositions. And every lifetime proof is **completely erased** at the boundary — no runtime trace, no cost, no representation.

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() >= y.len() { x } else { y }
}
```

Read the lifetime annotations under Curry-Howard:

The signature asserts: *given two string references that are both valid for region 'a, the returned reference is also valid for region 'a*. This is a proposition about the temporal relationships between three references. The function body is the proof: since the return value is always one of the two inputs, and both inputs are valid for `'a`, the output is necessarily valid for `'a`.

The borrow checker verifies this proof by tracking the lifetimes of all references through the function body. If the proof is invalid — if the function could return a reference that outlives its data — the borrow checker rejects the program.

**Lifetime subtyping** establishes an ordering on lifetime propositions. If `'a: 'b` (read: `'a` outlives `'b`), then a reference valid for `'a` is also valid for `'b`. This is a form of logical implication: the proposition "valid for the longer region" implies the proposition "valid for the shorter region."

**The `'static` lifetime** is the strongest lifetime proposition: "this reference is valid for the entire duration of the program." It is the top element in the lifetime sub-lattice. Every other lifetime is implied by `'static`: if something is valid forever, it is valid for any region.

The entire lifetime system — the annotations, the variance rules, the subtyping relationships, the borrow checker's analysis — constitutes a **complete sub-logic** within 𝒯. It is a logic of resource ownership and temporal validity, with its own propositions (lifetime bounds), its own proof rules (borrowing rules), and its own proof checker (the borrow checker).

And all of it is erased at the boundary. When your program runs, there are no lifetimes. There are no borrow checks. The proofs have been verified and discarded. The references are bare pointers, indistinguishable from what you would write in C — except that the proofs, verified above the boundary, guarantee they are safe.

This is zero-cost abstraction in its purest form: an entire proof system, operating in 𝒯, verified at compile time, erased by the forgetful functor, contributing nothing to the runtime cost. The safety guarantees are real; the mechanism that establishes them is invisible in 𝒱.

## Proof Erasure: Where Curry-Howard Stops

In full type-theoretic systems like Coq, Agda, or Idris, proof terms are values. You can pass a proof as an argument, store it in a data structure, pattern-match on it, and compute with it at runtime. The Curry-Howard correspondence extends all the way down: proofs are first-class citizens of the computation.

Rust does not do this. In Rust, proof terms (`impl` blocks) are not values. You cannot write:

```text
// Not valid Rust — illustrative pseudocode
fn apply_proof<T>(proof: impl Ord for T, a: T, b: T) -> T {
    proof.cmp(a, b)
}
```

You cannot store an impl block in a variable, pass it as a function argument, or return it from a function. The impl block exists in 𝒯 — it is verified at compile time — and then it is erased at the boundary. In 𝒱, there is no proof. There is only the code that the proof licensed.

This is where Rust truncates the Curry-Howard correspondence. The correspondence maps:

| Full Curry-Howard | Rust |
|---|---|
| Propositions → Types | Trait bounds → Types |
| Proofs → Values | Impl blocks → **erased** |
| Proof checking → Type checking | Borrow checking → Type checking |
| Proof terms at runtime | **Not available** |

The truncation is not a limitation — it is a design choice, and it is the same design choice that gives Rust zero-cost abstraction. If proof terms survived as runtime values, they would have a runtime cost. Haskell makes the opposite choice: its trait-like mechanism (type classes) is implemented via **dictionary passing**, where the impl block (the "instance dictionary") is passed as an implicit runtime argument. This makes proofs first-class but adds runtime overhead: every polymorphic function call carries an extra pointer to the dictionary.

The comparison clarifies what the boundary costs and what it buys:

| Feature | Rust | Haskell | Coq/Idris |
|---|---|---|---|
| Proof terms | Erased | Dictionaries (runtime) | First-class values |
| Polymorphic dispatch | Monomorphised (static) | Dictionary-passed (dynamic) | Depends on extraction |
| Runtime cost of proofs | Zero | Pointer per constraint | Full value |
| Can inspect proofs at runtime | No | Partially (via dictionaries) | Yes |
| Can compute with proofs | No | Limited | Yes |

Rust's position is: proofs exist to verify correctness; once verified, they are discarded. This is the forgetful functor applied to the proof domain: F maps every proof in 𝒯 to its computational residue in 𝒱, which may be zero (for lifetime proofs), inlined code (for monomorphised trait methods), or a vtable (for dynamic dispatch). The proof itself — the *reason* the code is correct — never exists at runtime.

### What Proof Erasure Precludes

Because proofs are erased, certain things are impossible in Rust that would be straightforward in a dependently typed language:

**Proof-dependent branching.** You cannot branch at runtime on *which* impl is being used:

```text
// Not valid Rust — illustrative pseudocode
fn process<T: Serialize>(value: T) {
    if T::serialization_is_json() {
        // fast path for JSON
    } else {
        // general path
    }
}
```

The impl block's internal structure is not available at runtime. The compiler may inline different code for different types (monomorphisation), but the programmer cannot write conditional logic that inspects the proof.

**Proof transport.** You cannot take a proof from one context and use it in another:

```text
// Not valid Rust — illustrative pseudocode
fn store_proof<T: Ord>() -> ProofOf<T, Ord> {
    // capture the proof that T: Ord and return it
}
```

Since proofs are not values, they cannot be stored, transported, or deferred. The proof must be available at the call site where it is needed — it cannot be fetched from elsewhere at runtime.

These limitations are real, and they define the boundary of what Rust can express compared to dependently typed systems. But within that boundary, the correspondence holds fully: types are propositions, bounds are hypotheses, functions are proofs, and the compiler is the proof checker.

## The Logical Structure of Common Patterns

To solidify the correspondence, let us read several common Rust patterns as logical statements.

### `Option<T>` as T ∨ ⊤

`Option<T>` is `Some(T)` or `None`. Under Curry-Howard, `None` carries the unit type (no data), so `Option<T>` is the disjunction T ∨ ⊤ — "either T holds, or trivially true (no information)." This reads naturally: an optional value either exists or does not.

### `Result<T, E>` as T ∨ E

`Result<T, E>` is `Ok(T)` or `Err(E)`. This is the disjunction T ∨ E — "either success (with evidence T) or failure (with evidence E)." The `?` operator is an elimination form for this disjunction: if the result is `Err`, propagate the error (take the E branch); if `Ok`, continue with the T value.

### The `From`/`Into` traits as Implication

`impl From<A> for B` is a proof that A → B: given a value of type A, you can produce a value of type B. The `Into` trait is the same implication in reverse notation. When you write `let b: B = a.into()`, you are applying the proof of A → B to a value of A.

### Iterators as Repeated Disjunction

An iterator of type `Iterator<Item = T>` produces a sequence of `Option<T>` values — repeated applications of the disjunction T ∨ ⊤, where `Some(t)` is the left branch ("here is a value") and `None` is the right branch ("no more values"). The iterator protocol is a sequence of logical queries.

### `PhantomData<T>` as a Vacuous Proposition

`PhantomData<T>` is a type that mentions `T` but carries no value of type `T` at runtime. Under Curry-Howard, it is a proposition about T that requires no evidence — a **vacuous truth** that exists purely to create a type-level relationship. It exists in 𝒯 and is erased to zero bytes by F. We will examine this fully in Chapter 7.

## Thinking in Propositions

The Curry-Howard correspondence is not merely a theoretical observation. It is a practical tool for thinking about program design.

When you design a function signature, you are writing a theorem statement. Ask yourself:

- **What am I claiming?** The return type is the conclusion. The argument types are the hypotheses. Is the claim true? Is it as strong as it could be?

- **Are my hypotheses minimal?** Each trait bound is a hypothesis. If you can prove the conclusion with fewer hypotheses, you should — a more general theorem is a more reusable function.

- **Is my conclusion honest?** A function returning `Option<T>` admits that it might fail. A function returning `T` claims it always succeeds. An `unwrap()` is a claim that the `None` case is impossible — an assertion that requires its own (informal) proof.

- **Does the structure of my proof match the structure of the problem?** If the problem is naturally a disjunction (multiple cases), the proof should use an enum and pattern matching. If it is naturally a conjunction (multiple simultaneous requirements), the proof should use a struct or tuple.

When you encounter a compiler error, read it as logical feedback:

- "The trait bound `T: Ord` is not satisfied" means: "your argument lacks a necessary hypothesis."
- "Expected `Result<T, E>`, found `T`" means: "your conclusion claims certainty, but your proof goes through a fallible step."
- "Cannot move out of borrowed content" means: "your proof of ownership is insufficient — you have a proof of borrowing, which is weaker."

The compiler is not obstructing you. It is telling you where your reasoning is incomplete.

## The Correspondence at Each Level

The Curry-Howard correspondence manifests differently at each level of our model:

**Level 0** — Ground propositions. Concrete types are specific propositions about data layout. A plain `impl` block is a proof that holds for one type, with no quantification. The logic is simple: fixed premises, fixed conclusions.

**Level 1** — Universal quantification without hypotheses. `fn f<T>(x: T) -> T` is ∀T. T → T. The proposition holds for all types, unconditionally. The proof must work without inspecting T. Reynolds' parametricity governs what proofs are possible.

**Level 2** — Universal quantification with hypotheses. `fn f<T: Bound>(x: T) -> T` is ∀T. Bound(T) → T → T. The proposition holds for all types satisfying the bound. The hypotheses enable operations; the proof uses them.

**Level 3** — The proof terms themselves. Impl blocks are the objects of this level. Blanket impls are universally quantified proofs. The orphan rule is the coherence condition. At this level, you reason not about types but about the structure of proofs — which proofs exist, how they are derived, and why they are unique.

**Level 4** — Propositions about type constructors. GATs assert propositions parameterised by lifetimes or types. Typestate machines encode temporal propositions ("the connection must be opened before data is sent") in the type structure. The proofs at this level exist entirely in 𝒯 — the boundary erases them completely.

**The boundary** — Where proof meets computation. The forgetful functor F preserves the computational content of proofs (the code that runs) while erasing their logical content (the reason the code is correct). This is the truncation point of Curry-Howard in Rust: below the boundary, there are values and functions. Above the boundary, there are propositions and proofs. The correspondence lives entirely in 𝒯.

## What Lies Beyond

In Coq, when you write a proof that a list is sorted, the proof is a data structure. You can examine it, transform it, optimise it. In Agda, when you write a proof that two paths in a type are equal, that proof can influence the computation. In Idris, you can choose on a case-by-case basis which proofs to erase and which to retain at runtime.

These languages occupy the full Curry-Howard correspondence. They live at the top of the lambda cube, where types depend on terms, terms depend on types, and the boundary between propositions and computation dissolves. The cost is complexity: dependent type checking is undecidable in general, and these languages require programmer-supplied hints to guide the proof checker. The benefit is expressiveness: anything you can state, you can prove; and anything you can prove, you can compute with.

Rust makes a different choice. It occupies a lower vertex of the lambda cube, maintains a strict boundary between 𝒯 and 𝒱, and erases proofs completely. The cost is expressiveness: there are propositions that Rust cannot state and proofs that Rust cannot construct. The benefit is the guarantee that the propositions it *can* state carry zero runtime cost.

The remaining chapters of this book explore what Rust *can* express within these constraints — level by level, from the ground types of Level 0 to the type constructors of Level 4. Each chapter builds on the correspondence established here: types are propositions, implementations are proofs, and the compiler is the proof checker that stands between your reasoning and the machine.
