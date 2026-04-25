# A Type-Theoretical Guide to Rust

**[Read the book online](https://pukeko37.github.io/type-theoretical_guide_to_rust/)**

## About

Rust's type system is not just a safety net -- it is a design tool. This book
develops a formal model that reveals the structure behind traits, bounds,
lifetimes, and generics, showing why they were designed the way they were and
how they fit together as a coherent whole. Understanding that structure changes
the way you write Rust: from working around the compiler to designing with it,
and from reaching for familiar patterns to exploring the full reach of what the
type system makes possible.

The book introduces a two-category model -- a compile-time *type category*
(𝒯) where abstract reasoning lives, and a runtime *value category* (𝒱) where
concrete computation happens -- connected by a *forgetful functor* that erases
type-level structure without adding runtime overhead. This boundary is the
formal content of Rust's founding principle of zero-cost abstraction:
arbitrarily sophisticated structure above the boundary, zero weight below it.

Within the type category, the book identifies five levels of increasing
abstraction. Level 0 covers concrete types with no parameters. Level 1
introduces unconstrained generics and parametricity. Level 2 adds trait bounds
as logical predicates, forming a constraint lattice. Level 3 treats `impl`
blocks as proof objects and blanket implementations as proof constructors.
Level 4 reaches type constructors and GATs -- functions on types themselves.

The later chapters put the model to work: Chapter 8 examines the boundary
between compile-time and runtime in full detail. Chapters 9 and 10 apply the
framework to domain modelling and software architecture, showing how the type
system shapes practical design decisions. Chapter 11 places Rust in the broader
landscape of typed languages -- effect types, session types, dependent types,
universe polymorphism -- to show precisely where Rust stands and why its
particular trade-offs are principled.

## Audience

This book is written for experienced Rust programmers who want a deeper
understanding of *why* Rust's type system works the way it does. You should be
comfortable with generics, trait bounds, lifetimes, and `impl` blocks. Prior
exposure to formal logic or type theory is not required -- the necessary
background is developed in the opening chapters -- but curiosity about the
mathematical structures underlying programming languages will help you get the
most from the material. Researchers and students in programming language theory
may also find the Rust-specific perspective useful as a case study of
Curry-Howard ideas applied in a systems language.

## Chapters

0. **Introduction** -- Zero-cost abstraction, the two-category model, the five levels
1. **Background and Foundations** -- Lambda cube, Reynolds, type lattice, fibrations, phase distinction
2. **Propositions as Types** -- Curry-Howard correspondence, connectives, quantification, proof erasure
3. **The Value Domain (Level 0)** -- Concrete types, structs as conjunctions, enums as disjunctions
4. **Parametric Polymorphism (Level 1)** -- System F, parametricity, theorems for free
5. **Constrained Generics (Level 2)** -- Bounds as predicates, constraint lattice, associated types
6. **The Proof Domain (Level 3)** -- Impl category, blanket impls, orphan rule, proof erasure
7. **Type Constructors and GATs (Level 4)** -- Type-level functions, typestate, HKT gap
8. **The Boundary** -- Forgetful functor, monomorphisation, dyn Trait, permeability spectrum
9. **Domain Modelling as Proof Construction** -- Smart constructors, typestate lifecycle, algebraic data flows
10. **Architecture as Category** -- Crate topology, composition root, testing as alternative proof
11. **Beyond Rust** -- Effect types, session types, dependent types, universe polymorphism

## Building

This book is built with [mdBook](https://rust-lang.github.io/mdBook/). To build locally:

```sh
cargo install mdbook
mdbook build
mdbook serve  # optional: serves at http://localhost:3000
```

## Licence

The **prose content** of this book is licensed under the
[Creative Commons Attribution 4.0 International License (CC BY 4.0)](LICENSE).

The **code examples** contained in this book are dual-licensed under:

- [MIT License](LICENSE-MIT)
- [Apache License 2.0](LICENSE-APACHE)

at your option.

## Author

Andrew Bowers
