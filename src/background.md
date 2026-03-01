# Book Background

Rust is frequently taught through the lens of memory safety — a pragmatic inheritance from its C and C++ lineage, where types are primarily understood as descriptions of machine memory layout and tools for catching implementation errors. This framing, while useful for systems programmers crossing from those languages, obscures a deeper and more powerful truth: Rust's type system is one of the most sophisticated realisations of type theory available in a production programming language.

This book reframes Rust entirely from a type-theoretical perspective. Its central thesis is that **types are propositions**, **implementations are proofs**, and **writing well-typed Rust is an act of formal reasoning about a problem domain** — not an exercise in satisfying a compiler. This is not a metaphor. It is the precise content of the Curry-Howard correspondence, and Rust's trait system, generic bounds, associated types, and GATs are all direct expressions of it.

## The Structural Model

The book is structured around a stratified model of Rust's type system derived from Henk Barendregt's lambda cube and John Reynolds' work on parametric polymorphism and relational parametricity. Types exist on a continuum from unconstrained parametric abstractions through progressively constrained families down to concrete runtime values.

### The Two Categories

The model begins with a categorical distinction between two worlds:

**𝒯 — The Type Category (compile-time).** Objects are types, type constructors, trait bounds, generic families, and impl blocks. Morphisms are functions between types, subtyping relationships, and trait implementations as proof morphisms. The internal structure of 𝒯 is a lattice — trait bounds form a partial order where more constrained types sit lower (fewer inhabitants, more guaranteed capabilities).

**𝒱 — The Value Category (runtime).** Objects are runtime values and their memory representations. Morphisms are executable functions, ownership transfers, and borrows. This category has affine/linear structure — morphisms carry resource constraints enforced by the borrow checker.

### The Boundary

Separating 𝒯 from 𝒱 is a **forgetful functor F: 𝒯 → 𝒱**. This functor:

- Maps types to their runtime memory representations
- Maps generic functions (type-level) to monomorphised machine code (value-level)
- Maps trait implementations to vtables (in `dyn` cases) or inlined code (in monomorphic cases)
- **Erases** lifetimes, phantom types, proof obligations, and type identity entirely

The functor is forgetful: many distinct objects in 𝒯 map to the same object in 𝒱. Lifetimes, for instance, form a complete sub-logic in 𝒯 — a proof system for pointer validity — and vanish entirely below the boundary with zero runtime cost.

The boundary has **selective permeability** — several mechanisms allow partial information transfer:

- **`const` generics** — scalar values cross upward from 𝒱 into 𝒯 (the only dependent type mechanism in Rust)
- **`dyn Trait`** — a vtable is a partial downward projection of type behaviour; identity is erased, interface survives
- **`PhantomData<T>`** — a one-way valve; type information exists in 𝒯 only, zero runtime presence
- **`TypeId`** — a narrow read-only channel; runtime can query type identity but not manipulate it

### The Five Levels

The type category 𝒯 has internal structure — a hierarchy of abstraction levels:

| Level | Name | Core Idea | Lambda Calculus | Rust Examples |
|-------|------|-----------|-----------------|---------------|
| 0 | Value Domain | Concrete types only | λ→ | `struct Point { x: f64, y: f64 }` |
| 1 | Unconstrained Parametric | ∀T with no bounds | System F (λ2) | `fn identity<T>(x: T) -> T` |
| 2 | Constrained Parametric | ∀T: Bound | System F + predicates | `fn sorted<T: Ord>(data: Vec<T>)` |
| 3 | Proof Domain | impl blocks as witnesses | Curry-Howard proofs | `impl Display for Metres` |
| 4 | Type Constructor | Type → Type, GATs, typestate | Higher-kinded | `PhantomData<State>` typestate |

Beyond these, the book discusses levels that Rust does not reach — full dependent types, proof terms as runtime values, universe polymorphism, and effect systems — situating Rust precisely within the broader landscape.

## Intended Audience

The intended audience is the working Rust programmer who suspects there is more to types than the compiler's error messages reveal. The book assumes fluency with Rust's syntax and standard patterns but does not require prior knowledge of type theory, category theory, or formal logic. All theoretical concepts are introduced from first principles and immediately grounded in concrete Rust.

## Theoretical Heritage

The book draws on three major traditions:

- **Barendregt's lambda cube** — systematising typed lambda calculi along four axes, used to situate Rust precisely and explain what lies beyond it
- **Reynolds' work on polymorphism** — System F, parametricity, representation independence, and defunctionalisation, providing the formal foundations for the level structure
- **The Curry-Howard correspondence** — the book's philosophical centre, mapping propositions to types, proofs to implementations, and logical connectives to type constructors

## How to Read This Book

The chapters are ordered by ascending level in the model. Each chapter introduces the type-theoretical ideas at that level, shows their concrete expression in Rust, and articulates what this perspective reveals that the standard framing misses. The final two chapters zoom out: Chapter 8 examines the boundary between 𝒯 and 𝒱 in full detail and uses it as a lens to unify the proof mechanisms introduced across the preceding chapters, and Chapter 9 places Rust in the broader landscape of type systems.

Readers comfortable with Rust's generics and trait system can begin at Chapter 0 and read straight through. Those wanting theoretical foundations first may prefer to begin with Chapter 1 (Background and Foundations) and Chapter 2 (Propositions as Types) before proceeding.
