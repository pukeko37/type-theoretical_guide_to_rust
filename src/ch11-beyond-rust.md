# Beyond Rust

Rust occupies a specific position in the space of possible type systems. The preceding chapters have mapped that position from within — ascending through the five levels of 𝒯, examining the boundary's behaviour, tracing the forgetful functor's action on each kind of type-level structure. The previous two chapters showed what those five levels can build in practice: domain models whose invariants are proved by construction and architectures whose ring boundaries are enforced by the type system. This chapter maps Rust's position from without, placing it in the broader landscape of type systems and exploring the territories it does not enter.

The question is not "what is Rust missing?" Languages are not ranked on a linear scale of type-system sophistication. Rust's position is a *design choice* — a principled selection of features determined by the boundary's requirements. The question is: what does each region of the landscape offer, what does it cost, and why does Rust's boundary rule certain regions out?

## The Extended Level Hierarchy

The five levels introduced in this book (0 through 4) cover Rust's type-level expressiveness. But the space of type systems extends in both directions. Below Level 0 lie mechanisms that Rust does not have. Above Level 4 lie mechanisms that Rust cannot reach. The full hierarchy, with representative languages, places Rust in context.

### Levels Below Zero: Effects and Linearity

**Level −1: Effect Types.** Some type systems track not just what a function takes and returns, but what *effects* it may perform: mutation, I/O, exceptions, non-determinism. An effect type is a proposition about a function's side effects, checked at compile time.

Rust does not have effect types. A function signature `fn read_file(path: &str) -> String` does not indicate that it performs I/O. A function `fn sort(data: &mut [i32])` does not indicate that it mutates through a mutable reference in any way the effect-type literature would recognise — the mutation is encoded in the borrow system, not in an effect annotation.

Languages with effect systems include:

**Koka** (Microsoft Research) tracks effects algebraically. Every function's type includes an effect row — a list of effects the function may perform:

```text
// Koka: effect-annotated function type
fun read-file(path : string) : <io,exn> string
```

The effects `<io,exn>` are part of the type. A pure function has the empty effect row `<>`. The type checker verifies that effects are handled — you cannot call an effectful function from a pure context without an effect handler. Effect handlers are first-class: they can be defined, composed, and passed as arguments.

**Effekt** (University of Tübingen) takes a similar approach with second-class capabilities. Effects are tied to capability objects that must be in scope for the effect to be performed. This prevents effects from leaking through closures or being captured in data structures.

**Haskell** approximates effect types through the `IO` monad. A function of type `IO a` may perform I/O; a function of type `a` may not. The `IO` monad is not a true effect system — it is an encoding of effects within ordinary types — but it achieves a similar separation:

```text
-- Haskell: IO as an effect marker
readFile :: FilePath -> IO String
pure     :: a -> IO a        -- lift a pure value into IO
```

The `IO` type acts as a boundary within Haskell's type system: pure code cannot call `IO` functions without being in the `IO` monad. This is a coarse effect annotation — `IO` lumps all effects together — but it captures the essential idea: the type tells you whether a function is effectful.

Rust's alternative to effect types is pragmatic: effects are encoded in return types (`Result` for failure, `Option` for absence) or in the type system's structure (`&mut` for mutation). The borrow checker is, in a sense, a *linearity-based* effect system for memory effects. But general effects — I/O, non-determinism, exceptions — are not tracked in the type system. This is a boundary decision: tracking effects would require the boundary to translate effect annotations into runtime effect-handling code, adding complexity to both 𝒯 and the boundary.

**Level −2: Uniqueness and Session Types.** At a deeper level, some type systems track not just effects but the *communication protocols* between concurrent processes.

**Session types** describe the protocol of a communication channel as a type. A channel of type `Send<i32, Recv<String, End>>` must first send an `i32`, then receive a `String`, then close. The type checker verifies that both endpoints follow the protocol. Violating the protocol — sending when the type says to receive — is a type error.

Session types are related to linear types: a channel must be used exactly once at each protocol step, ensuring that messages are not duplicated or lost. The Rust crate ecosystem has experimental implementations, but session types are not part of Rust's core type system.

Rust's ownership system is a form of **affine typing**: values can be used at most once (moved). Full linear types would require that values be used *exactly* once — you cannot simply drop a value; you must explicitly consume it. Linear Haskell (`-XLinearTypes`) adds linear function arrows — functions that must use their argument exactly once — while the rest of the language remains unrestricted. Granule goes further, making linearity pervasive. Both explore the territory of requiring that resources be consumed rather than abandoned.


### Levels Above Four: Dependent Types and Beyond

**Level 5: Full Dependent Types.** In a fully dependently typed language, arbitrary values can appear in types. The boundary between 𝒯 and 𝒱 dissolves: types can compute with runtime values, and runtime code can manipulate types as data.

**Idris** and **Agda** are the primary examples. In Idris:

```text
-- Idris: a vector type indexed by its length
data Vect : Nat -> Type -> Type where
    Nil  : Vect 0 a
    (::) : a -> Vect n a -> Vect (n + 1) a
```

The type `Vect 3 Int` is different from `Vect 5 Int` — the length, a natural number, is part of the type. The type system tracks lengths through every operation:

```text
-- Idris: append preserves length information
append : Vect n a -> Vect m a -> Vect (n + m) a
append Nil       ys = ys
append (x :: xs) ys = x :: append xs ys
```

The compiler verifies at compile time that appending a vector of length `n` and a vector of length `m` produces a vector of length `n + m`. The arithmetic happens at the type level. This is the third axis of the lambda cube fully activated: types depending on terms without restriction.

What does this mean for the boundary? In Idris, the boundary becomes *programmable* — the programmer can annotate which proofs are erased and which are retained at runtime (as described in Chapter 8). In Agda, which targets verification rather than execution, the boundary is even less defined: programs are primarily objects of proof, and extraction to executable code is a secondary concern.

The cost of full dependent types is complexity. Type checking becomes undecidable in general — the type checker may need to evaluate arbitrary computations to determine whether two types are equal. Idris and Agda manage this through *totality checking* (requiring that all functions terminate) and *universe levels* (stratifying the type hierarchy to prevent paradoxes). These mechanisms add significant cognitive overhead for the programmer.

Rust's const generics are a pinhole into Level 5 — they allow scalar values in types, but not arbitrary computations. The restriction exists because Rust's boundary demands that all type-level computation complete at compile time. Full dependent types would require the boundary to reason about runtime values during type checking, which would either make type checking undecidable or require a totality checker that Rust does not have.

**Level 6: Proof Terms as Runtime Values.** In Coq and Lean, proof terms are first-class values. An `impl` block equivalent is not erased — it exists at runtime as a data structure that can be inspected, stored, and computed with.

```text
-- Coq: a proof term is a value
Definition proof_comm : forall n m : nat, n + m = m + n.
Proof. intros. omega. Qed.
```

The proof `proof_comm` is a value of type `forall n m : nat, n + m = m + n`. It can be passed to functions, stored in data structures, and used to justify subsequent proofs. When Coq code is extracted to OCaml or Haskell, proofs in the `Prop` universe are erased (they are logically irrelevant) and proofs in the `Set` universe are retained (they carry computational content). But within Coq's type system, all proofs are values.

This is the full Curry-Howard correspondence without truncation. Where Rust's boundary erases proofs after verification, Coq's proofs persist as runtime entities. The cost is the overhead of carrying proof data at runtime. The benefit is that proofs can be computed with — a function can examine its proof obligations, construct proofs dynamically, and select different implementations based on the structure of the proof.

Rust's proof erasure, by contrast, means that all proof selection happens at compile time. The choice of which `impl Ord` to use is resolved before the program runs. This is less expressive — you cannot select proofs dynamically — but it is zero-cost, which is the boundary's fundamental requirement.

**Level 7: Universe Polymorphism.** In Agda and Coq, types themselves have types (called *universes* or *sorts*). `Type` has type `Type₁`, which has type `Type₂`, and so on. Universe polymorphism allows functions and data types to be parameterised by their universe level — quantifying not just over types, but over the *level* of the type hierarchy.

```text
-- Agda: universe-polymorphic identity
id : {l : Level} → {A : Set l} → A → A
id x = x
```

The identity function works for values of types at any universe level. This prevents the need to duplicate definitions across universe levels (an analogue of Rust's Level 0 repetition problem, but one level of abstraction higher).

Universe polymorphism is necessary to avoid logical paradoxes (Russell's paradox) in dependently typed systems. It is irrelevant to Rust because Rust does not have types-of-types in the relevant sense: `Vec` is a type constructor (Type → Type), but there is no "type of all type constructors" that would require universe stratification.

## Languages in the Lambda Cube

The lambda cube provides a map of type system capabilities. Here is where major languages sit:

| Language | λ→ | Axis 1 (terms←types) | Axis 2 (types←types) | Axis 3 (types←terms) | Position |
|---|---|---|---|---|---|
| C | Yes | No (pre-C11) | No | No | λ→ |
| Java | Yes | Yes (generics) | Limited | No | Near λ2 |
| Haskell | Yes | Yes | Yes (type families) | No | λω + extensions |
| Rust | Yes | Yes | Yes | Pinhole (const generics) | λω + pinhole |
| Idris | Yes | Yes | Yes | Yes | CC (extended) |
| Agda | Yes | Yes | Yes | Yes | CC (extended) |
| Coq | Yes | Yes | Yes | Yes | CC (extended) |

The bottom row of the cube (no dependent types) is where all mainstream production languages live. The top row (full dependent types) is occupied by proof assistants and research languages. Rust sits at the most expressive position on the bottom row — λω, with polymorphism and type operators — and reaches upward through const generics into the dependent-type dimension without committing to it.

### Haskell's Extensions

Haskell is an instructive comparison because it occupies a similar position to Rust in the cube (λω) but extends in different directions:

**Type families** give Haskell type-level computation. A type family is a function from types to types, evaluated at compile time:

```text
-- Haskell: type-level computation
type family Element (container :: Type) :: Type where
    Element [a]       = a
    Element (Set a)   = a
    Element (Map k v) = (k, v)
```

This is similar to Rust's associated types, but more general — type families can be open (extensible by later modules) or closed (fixed at definition), and they can perform pattern matching on types.

**GADTs** (Generalised Algebraic Data Types) allow constructors to refine the type they produce:

```text
-- Haskell: GADT with refined return types
data Expr a where
    Lit  :: Int -> Expr Int
    Add  :: Expr Int -> Expr Int -> Expr Int
    IsZero :: Expr Int -> Expr Bool
```

The constructor `Lit` produces `Expr Int` (not `Expr a`). Pattern matching on `Lit` refines the type variable: inside the pattern, the compiler knows `a = Int`. This is a form of dependent typing — the constructor's value influences the type — and it is available in Haskell but not in Rust's standard enum system.

**DataKinds** promote data types to the kind level — values become types, types become kinds. This allows type-level natural numbers, type-level booleans, and other type-level data without full dependent types.

These extensions push Haskell further up the lambda cube than Rust, approaching dependent types from below. The cost is complexity: Haskell's type system with all extensions enabled is significantly harder to learn and reason about than Rust's. The benefit is expressiveness: Haskell can express invariants (like well-typed expression trees) that Rust cannot.

### The Diagonal

There is a notable diagonal in the cube: languages tend to advance along all three axes together or not at all. C has none of the three. Rust has two fully and one partially. Idris, Agda, and Coq have all three. Few languages occupy the off-diagonal positions (dependent types without polymorphism, or polymorphism without type operators). This is not coincidental — the axes interact, and the features that make one axis useful tend to require the others.

## What Dependent Types Make Possible

The most significant territory beyond Rust is full dependent types. Understanding what they enable — and what they cost — clarifies Rust's design position.

### Verified Data Structures

With dependent types, data structure invariants become part of the type:

```text
-- Idris: a sorted list type
data SortedList : (a : Type) -> Ord a => Type where
    SNil  : SortedList a
    SCons : (x : a) -> (rest : SortedList a) ->
            {auto prf : So (maybe True (\y => x <= y) (head' rest))} ->
            SortedList a
```

The type `SortedList a` is not just "a list of `a` values" — it is "a list of `a` values that is sorted." The sortedness invariant is in the type. You cannot construct a `SortedList` that is not sorted, because the constructor requires a proof that the new element is less than or equal to the existing head.

In Rust, sortedness is a runtime property — you call `sort()` and trust that the implementation is correct. The type system does not track whether a `Vec` is sorted. You can wrap it in a newtype and provide only methods that preserve sortedness, but the compiler does not verify the invariant — the programmer must ensure it manually within the `impl` block.

### Correct-by-Construction Programs

Dependent types enable programs that are proven correct as a consequence of type checking:

```text
-- Idris: matrix multiplication with dimension checking
multiply : Matrix m n -> Matrix n p -> Matrix m p
```

The types guarantee that the inner dimensions match (`n` in both matrices). A call with mismatched dimensions is a type error, not a runtime panic. The compiler verifies dimensional correctness as part of type checking.

In Rust, you could encode matrix dimensions with const generics:

```rust
struct Matrix<const M: usize, const N: usize> {
    data: [[f64; N]; M],
}

fn multiply<const M: usize, const N: usize, const P: usize>(
    a: &Matrix<M, N>,
    b: &Matrix<N, P>,
) -> Matrix<M, P> {
    let mut result = Matrix { data: [[0.0; P]; M] };
    for i in 0..M {
        for j in 0..P {
            for k in 0..N {
                result.data[i][j] += a.data[i][k] * b.data[k][j];
            }
        }
    }
    result
}
```

This works — the const generic `N` appears in both matrix types, ensuring dimensional agreement. Rust's pinhole into dependent types is enough for this case. But it breaks down for dimensions that are not known at compile time — if `M`, `N`, and `P` come from user input, const generics cannot help. Full dependent types can track dimensions through runtime computation; Rust's const generics require the values to be compile-time constants.

### The Cost of Dependent Types

Full dependent types are powerful, but they impose costs that explain why Rust does not have them:

**Undecidable type checking.** Determining whether two dependent types are equal may require evaluating arbitrary computations. If those computations do not terminate, type checking does not terminate. Idris and Agda manage this by requiring totality (all functions must terminate), which is itself a significant restriction.

**Cognitive overhead.** Writing dependently typed programs requires the programmer to think simultaneously about code and proofs. A function that sorts a list must not only sort it but *prove* it is sorted. This doubles the specification burden for every operation.

**Boundary complexity.** If types depend on runtime values, the boundary between compile time and runtime becomes harder to draw. Idris addresses this with explicit erasure annotations. Agda sidesteps it by targeting verification rather than execution. Rust's clean boundary — all type-level computation completes at compile time — would be impossible with unrestricted dependent types.

**Compilation speed.** Type checking in dependently typed languages can be slow, because it involves evaluating type-level programs. Rust's type checking is already sometimes criticised for compilation speed; adding type-level evaluation would compound the problem.

These costs are not theoretical. They are practical obstacles that limit the adoption of dependently typed languages for production systems programming. Rust's design avoids them by staying on the bottom row of the lambda cube, accepting the expressiveness limitation in exchange for a predictable, fast, zero-cost boundary.

## Effect Systems: The Other Axis

Effect types represent an axis of type system design orthogonal to the lambda cube. While the cube tracks dependencies between terms and types, effect systems track *what a computation does* beyond producing a value.

### Algebraic Effects

Algebraic effects, as implemented in Koka and Effekt, decompose effects into operations and handlers:

```text
// Koka: declaring and handling an effect
effect ask<a>
  ctl ask() : a

fun greeting() : ask<string> string
  val name = ask()
  "Hello, " ++ name

fun main()
  with handler
    ctl ask() -> resume("World")
  greeting().println
```

The function `greeting` has the effect `ask<string>` — it may perform the `ask` operation, which returns a string. The handler provides the implementation of `ask`: in this case, always returning `"World"`. The effect type is checked: you cannot call `greeting` without providing a handler for `ask`.

This is a genuine axis of type system design that the lambda cube does not capture. Effects are about what *happens* during computation (side effects, control flow), not about what types *are* (the cube's concern). A language can have algebraic effects without dependent types (Koka), or dependent types without algebraic effects (Agda), or both (some experimental systems).

### Rust's Implicit Effect Encoding

Rust handles effects without an effect system by encoding them in types and language features:

| Effect | Effect system approach | Rust's approach |
|---|---|---|
| Failure | `effect exn { ctl throw(e) }` | `Result<T, E>`, `?` operator |
| Absence | `effect maybe { ctl none() }` | `Option<T>`, `?` operator |
| Asynchrony | `effect async { ctl yield() }` | `async`/`await`, `Future` trait |
| Mutation | `effect state { ctl get(); ctl put(s) }` | `&mut T` references |
| I/O | `effect io { ctl read(); ctl write(s) }` | Not tracked in types |

Rust's approach is pragmatic: the most important effects are encoded in types (`Result`, `Option`, `Future`), one effect is tracked by a specialised sub-system (mutation via the borrow checker), and the rest (I/O, allocation, panicking) are untracked. This avoids the complexity of a general effect system while capturing the most common failure and absence patterns in the type system.

The trade-off is visible in `unsafe` code. Within an `unsafe` block, Rust's effect guarantees (memory safety, data race freedom) are suspended — the programmer takes responsibility. An effect system would track exactly which guarantees are suspended and verify that they are restored. Rust's `unsafe` is a coarse escape hatch where an effect system would provide fine-grained tracking.

## Linear and Affine Type Systems

Rust's ownership system is an affine type system: values can be used at most once (they can be moved). The borrow checker extends this with shared and exclusive references, creating a practical system for resource management. But affine types are not the only option in this space.

**Linear types** require that values be used *exactly* once — not zero times, not twice, but once. A linear value cannot be dropped without being consumed by some operation. Linear Haskell (`-XLinearTypes`) introduces linear function arrows:

```text
-- Linear Haskell: the function must use its argument exactly once
dup :: a %1 -> (a, a)  -- TYPE ERROR: uses 'a' twice
```

The `%1` annotation means "this argument must be used exactly once." A function that duplicates its argument violates linearity and is rejected.

In Rust, dropping a value is always permitted (affine, not linear). The `Drop` trait provides a hook for cleanup, but the type system does not *require* that a value be consumed. A `File` handle can be dropped without being closed (the `Drop` implementation closes it, but the programmer could `mem::forget` it). With linear types, the type system would require explicit closing — the file handle must be consumed by a `close` operation, not silently dropped.

**Granule** (University of Kent) is a research language with graded modal types — a generalisation that subsumes linear, affine, and other resource-tracking disciplines under a single framework. In Granule, every variable carries a *grade* indicating how many times it may be used, and these grades can be combined algebraically.

Rust's affine system is a pragmatic point in this design space. Full linearity would be more expressive (guaranteeing resource cleanup) but less ergonomic (requiring explicit consumption of every value). Rust's `Drop` trait provides a practical compromise: resources are cleaned up automatically on drop, and the type system prevents use-after-move, without requiring that every value be explicitly consumed.

## Session Types and Protocol Verification

Session types represent communication protocols as types. A session type describes the sequence of messages that must be sent and received on a channel, and the type checker verifies that both endpoints follow the protocol.

```text
// Pseudocode: session-typed channel
type LoginProtocol = Send<Username, Recv<Challenge, Send<Response,
                     Branch<Recv<Token, End>, Recv<Error, End>>>>>;
```

This type describes a login protocol: send a username, receive a challenge, send a response, then branch — either receive a token (success) or an error (failure). The type checker verifies that the client and server implementations follow this protocol at each step.

Rust's typestate pattern (Chapter 7) encodes a related but weaker property: it ensures that a single object goes through states in the correct order. Session types generalise this to *two-party* protocols, ensuring that both sides of a communication channel agree on the message sequence. The difference is concurrency: typestate is sequential (one object, one owner), while session types are concurrent (two endpoints, potentially in different threads or processes).

Several Rust crates (like `session-types`) implement session types using Rust's ownership system. The ownership transfer (move semantics) naturally models the "use each protocol step exactly once" requirement. But these are library-level encodings, not first-class type system features, and they lack the compiler's full verification power.

## Rust's Position as a Principled Choice

Rust does not occupy its position in the type system landscape by default or by accident. It is a principled choice, driven by the boundary.

The boundary requires:
1. **All type-level computation must complete at compile time.** This rules out full dependent types (which may require runtime evaluation for type checking).
2. **All proofs must be erasable without runtime cost.** This rules out first-class proof terms (which would need runtime representation).
3. **Monomorphisation must produce code equivalent to hand-written concrete code.** This rules out higher-kinded type quantification (which would require runtime type-constructor dispatch).
4. **The type system must be decidable.** This rules out unrestricted type-level computation (which could diverge during type checking).

These four requirements determine Rust's position: λω (polymorphism + type operators), with a pinhole into dependent types (const generics), proof erasure (monomorphised trait dispatch), and decidable type checking (no type-level Turing completeness). Every feature of Rust's type system is compatible with these requirements. Every feature beyond Rust violates at least one.

This is not a limitation to be regretted. It is a design that achieves something no other language achieves: a type system expressive enough for parametric polymorphism, trait-bounded generics, typestate machines, and lifetime-based memory safety — all at zero runtime cost, with a clean boundary between the type-level reasoning and the generated machine code.

The alternative designs are not wrong — they are different trade-offs:
- Haskell trades zero-cost abstraction for higher-kinded types and lazy evaluation
- Idris trades compilation speed and simplicity for full dependent types
- Koka trades ecosystem size for algebraic effects
- Go trades expressiveness for simplicity and fast compilation

Each language answers the same question differently: how much type-level structure should survive the boundary, and what is the programmer willing to pay for it? Rust's answer is: as much structure as can be verified and erased at zero cost, and not one byte more.

## The Curry-Howard Correspondence as Research Programme

This book has used the Curry-Howard correspondence as its organising principle: types are propositions, implementations are proofs, and Rust's type system is a practical realisation of this correspondence with proof erasure at the boundary.

The correspondence itself is not a closed result. It is an ongoing research programme — a programme that has deepened and generalised steadily since Curry's original observation in 1934.

**The original correspondence** (Curry 1934, Howard 1969) connected propositional logic with the simply typed lambda calculus. Implication corresponds to function types. This is Level 0.

**The polymorphic extension** (Girard 1972, Reynolds 1974) connected second-order logic with System F. Universal quantification corresponds to parametric polymorphism. This is Levels 1 and 2.

**The dependent extension** (Martin-Lof 1971, 1984) connected constructive predicate logic with dependent type theory. Dependent products (Π-types) correspond to universal quantification over values. This is Level 5.

**The categorical extension** (Lambek 1980, Seely 1984) connected the correspondence to category theory. Types are objects. Functions are morphisms. The type system is a category. This is the framework this book uses.

**The homotopy extension** (Voevodsky 2006, Univalent Foundations Program 2013) connected type theory to homotopy theory. Types are spaces. Equalities are paths. Higher equalities are homotopies. This is the frontier: Homotopy Type Theory (HoTT) uses the Curry-Howard correspondence to connect logic, computation, and geometry in a single framework.

Each extension deepens the correspondence. Each reveals new structure in the relationship between logic and computation. And each is, at its core, the same insight that animates this book: the structure of a type system is not an engineering convenience. It is a mathematical reality — a reflection of the deep identity between formal reasoning and computation.

Rust participates in this programme concretely. When a Rust programmer writes a trait bound, they are stating a proposition. When they write an impl block, they are constructing a proof. When the compiler erases the proof at the boundary, it is performing the type-theoretic operation of proof irrelevance — discarding logical structure that has served its purpose. Every day, in every Rust program, the Curry-Howard correspondence is at work.

## Conclusion

The space of type systems is vast. Rust occupies a specific, carefully chosen position within it: expressive enough for systems programming, zero-cost enough for performance-critical applications, and principled enough that its type system forms a coherent logical framework.

The boundary — the forgetful functor F: 𝒯 → 𝒱 — is the architectural decision that determines this position. It is the mechanism that transforms type-level reasoning into runtime computation, the membrane that separates propositions from execution, and the guarantor of zero-cost abstraction. Understanding the boundary is understanding Rust.

Beyond the boundary lie dependent types, effect systems, universe polymorphism, session types, and the full Calculus of Constructions. These are not improvements Rust is missing — they are alternative boundary designs, each with its own trade-offs. Rust's boundary is aggressive: it erases everything, preserves nothing, and charges zero. This is the right boundary for a systems language. It is not the only possible boundary, and the study of alternative boundaries is one of the most active areas of programming language research.

The type-theoretical perspective on Rust does not change what the language can do. It reveals *why* the language is the way it is. Types are propositions. Implementations are proofs. The boundary erases the proofs and keeps the computation. Zero-cost abstraction is not a feature — it is a property of the boundary. And the boundary is where everything begins.
