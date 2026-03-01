# Introduction

## The Story You Have Been Told

If you learned Rust the way most people learn Rust, the story went something like this.

Rust is a systems programming language. It gives you control over memory layout and allocation, like C and C++, but it prevents you from making the mistakes that plague those languages — use-after-free, data races, dangling pointers, buffer overflows. It achieves this through a system of ownership, borrowing, and lifetimes, enforced at compile time by the borrow checker. Types tell the compiler how values are laid out in memory so it can verify that your program is safe.

This story is true. It is also, in a precise and important sense, backwards.

It frames types as servants of a runtime concern. The type system exists, in this telling, because memory is dangerous and programmers are fallible. Types are a safety net. Trait bounds are guardrails. Lifetime annotations are bookkeeping that the compiler insists on so it can check your work. You learn to satisfy the type checker the way you learn to satisfy a strict but well-meaning bureaucrat: comply with its demands and it will let your program run.

This framing has a cost. It trains you to think of type errors as obstacles rather than information. It teaches you to reach for `clone()` or `Box<dyn Trait>` when the types resist, rather than asking what the resistance is telling you. It obscures the fact that when you write a generic function with trait bounds, you are not merely describing what operations you need — you are making a formal statement about the structure of your problem, and the compiler is checking whether that statement is logically consistent.

This book tells a different story.

## The Story This Book Tells

Types are propositions. Implementations are proofs. Writing well-typed Rust is an act of formal reasoning about a problem domain.

This is not a metaphor. It is the precise content of a result in mathematical logic called the Curry-Howard correspondence, discovered independently by Haskell Curry in 1934 and William Alvin Howard in 1969. The correspondence establishes a structural identity — not merely an analogy — between systems of formal logic and systems of typed computation. Propositions correspond to types. Proofs correspond to programs that inhabit those types. The rules of logical deduction correspond to the rules of type construction.

Rust's type system is one of the most sophisticated realisations of this correspondence available in a production programming language. When you write:

```rust
fn sort<T: Ord>(data: &mut [T]) {
    data.sort();
}
```

you are not writing a function that "happens to require" the `Ord` trait. You are asserting a universally quantified proposition: *for all types T, if T satisfies the ordering relation, then a slice of T values can be sorted*. The bound `T: Ord` is the hypothesis. The function body is the proof. The `impl Ord for MyType` block that a caller must provide is the evidence that the hypothesis holds for their particular type.

When the compiler rejects your program, it is not catching a bug. It is identifying an invalid proof — a place where your reasoning about the problem does not follow from your premises.

This shift in perspective changes how you use the language. Types become a medium for thought, not a constraint to be worked around. The question changes from "how do I make the compiler accept this?" to "what am I actually claiming about this code, and is that claim true?"

## Two Worlds

To make this precise, we need a structural model. This book develops one from first principles, but the core idea can be stated simply: Rust programs exist simultaneously in two worlds.

The first world is the **type category**, which we will write as 𝒯. This is the world of compile time — of types, traits, generic parameters, bounds, lifetime annotations, and `impl` blocks. Everything in 𝒯 is abstract. A generic function in 𝒯 is not a single function but a *family* of functions, one for each type that satisfies its bounds. A trait bound in 𝒯 is not a runtime check but a logical predicate. A lifetime annotation in 𝒯 is a proposition about how long a reference remains valid — a proposition that is verified by the borrow checker and then discarded entirely.

The second world is the **value category**, which we will write as 𝒱. This is the world of runtime — of concrete values, machine instructions, heap allocations, and pointer arithmetic. Everything in 𝒱 is concrete. A function in 𝒱 takes specific bytes and produces specific bytes. There are no type parameters, no trait bounds, no lifetimes. There is only computation.

Between these two worlds lies a **boundary** — a mapping from 𝒯 to 𝒱 that transforms abstract type-level structure into concrete runtime behaviour. This mapping is a *forgetful functor*: it preserves computational behaviour but erases logical structure. When your generic function is compiled, the boundary collapses each member of the family into a separate concrete function (monomorphisation). When your program runs, every lifetime annotation has vanished. Every trait bound has been verified and forgotten. Every `PhantomData` field occupies zero bytes.

This erasure is not merely an observation about Rust's compilation strategy. It is the formal content of one of the language's founding principles: **zero-cost abstraction**. The slogan — *what you don't use, you don't pay for; what you do use, you couldn't hand-code any better* — is precisely the claim that the forgetful functor erases type-level structure without adding runtime overhead. Rust is a systems language, and systems languages are defined by their commitment to predictable, minimal-overhead execution. The two-category model reveals how Rust honours that commitment: it permits arbitrarily sophisticated structure in 𝒯 — generics, trait bounds, lifetimes, phantom types, typestate machines — and guarantees that the functor F carries none of that structure's weight into 𝒱. The abstraction machinery exists to support formal reasoning about programs. The boundary ensures that this reasoning is free. Zero-cost abstraction is not a feature of Rust's type system. It is a property of the boundary.

The boundary is also where Rust's particular character emerges. Many languages have type systems. What makes Rust distinctive is *where it draws the line* between what exists above the boundary and what survives below it — and the fact that the boundary is not a wall but a selectively permeable membrane with specific, well-defined channels of communication.

## The Levels

The type category 𝒯 is not flat. It has internal structure — a hierarchy of increasing abstraction that this book organises into five levels. Each level introduces a new kind of type-level expression, a new class of propositions that can be stated, and a new relationship to the boundary.

**Level 0: The Value Domain.** Concrete types with no parameters. `struct Point { x: f64, y: f64 }` lives here. So does `enum Direction { North, South, East, West }`. There are no type variables, no bounds, no generics. Types at Level 0 are fixed descriptions of data — they correspond to the simply typed lambda calculus, the most basic system of typed computation. A plain `impl` block at this level is a ground proposition: a statement that holds for one specific type, with no quantification.

**Level 1: Unconstrained Parametric Polymorphism.** Functions and types parameterised over a type variable `T` with no constraints whatsoever. `fn identity<T>(x: T) -> T` lives here. Because you know nothing about `T`, you can do almost nothing with it — you can move it, store it, return it, and that is all. This severe restriction is the source of Level 1's power: the type signature alone determines the implementation, up to isomorphism. There is essentially one function with the signature `T -> T`, and it is the identity. This is the content of Reynolds' parametricity theorem, and it operates at Level 1 because it depends on the *absence* of constraints.

**Level 2: Constrained Parametric Polymorphism.** Functions and types parameterised over `T` where `T` is bound by one or more traits. `fn maximum<T: Ord>(a: T, b: T) -> T` lives here. The bound `T: Ord` is a predicate — it restricts the domain of the universal quantifier to those types for which an ordering relation exists. Each bound you add narrows the domain and widens the set of operations available. Compound bounds like `T: Ord + Display + Clone` are intersections in a lattice of propositions.

**Level 3: The Proof Domain.** The world of `impl` blocks — the evidence that propositions hold. `impl Ord for Metres` is a proof object: it witnesses the proposition that `Metres` satisfies the ordering relation. Blanket implementations like `impl<T: Ord> Ord for Vec<T>` are something more: they are *proof constructors*, systematic derivations that produce new proofs from existing ones. The orphan rule — Rust's requirement that you can only write an `impl` if you own either the trait or the type — is a coherence condition on this proof system, ensuring that no proposition has two conflicting proofs.

**Level 4: Type Constructors and GATs.** The highest level Rust reaches. Here, types are not values or parameters but *functions on types*. `Vec` is not a type — it is a function that takes a type `T` and produces the type `Vec<T>`. Generic associated types (GATs) extend this further, allowing associated types to be parameterised by lifetimes or other type variables. Typestate patterns use `PhantomData` to encode state machines entirely within 𝒯, with zero runtime representation — the boundary erases them completely, leaving behind only the guarantee that the state transitions were valid.

Beyond Level 4 lie territories that Rust does not enter: full dependent types (where arbitrary runtime values can appear in types), proof terms as first-class runtime values (where `impl` blocks can be passed as function arguments), and universe polymorphism (where you can quantify over the levels themselves). Chapter 9 maps this terrain and explains precisely where Rust's boundary lies and why.

## A Worked Example

To see all five levels operating simultaneously, consider a single piece of Rust code and examine it from each level's perspective. We will use a function that finds the maximum element in a non-empty collection and formats it for display.

```rust
use std::fmt;

/// A non-empty wrapper that guarantees at least one element.
struct NonEmpty<T> {
    head: T,
    tail: Vec<T>,
}

impl<T> NonEmpty<T> {
    fn new(head: T, tail: Vec<T>) -> Self {
        NonEmpty { head, tail }
    }
}

fn format_max<T: Ord + fmt::Display>(collection: NonEmpty<T>) -> String {
    let max = collection.tail
        .into_iter()
        .fold(collection.head, |best, next| {
            if next > best { next } else { best }
        });
    format!("Maximum: {max}")
}
```

Now watch what each level reveals.

**Level 0** sees the concrete ground. When a caller writes `format_max(NonEmpty::new(3_i32, vec![1, 4, 1, 5]))`, Level 0 sees a function that takes a struct containing an `i32` and a `Vec<i32>`, iterates through them comparing 32-bit integers, and produces a heap-allocated `String`. The types describe data layout. The function describes a computation.

**Level 1** sees the parametric structure. `NonEmpty<T>` is not a single type but a family of types — one for each possible `T`. The `impl<T> NonEmpty<T>` block defines behaviour that is *uniform* across this entire family. The constructor `new` can move a value of any type `T` into the struct without knowing anything about what `T` is. At Level 1, `NonEmpty` is a container in the purest sense: it holds a value whose nature is irrelevant to the holding.

**Level 2** sees the propositions. The signature `fn format_max<T: Ord + fmt::Display>` asserts: *for all types T, if T is totally ordered and T is displayable, then a non-empty collection of T values can be reduced to a formatted string*. The bound `Ord` is a proposition asserting that a total ordering exists. The bound `Display` is a proposition asserting that values can be rendered as text. The two bounds together — `T: Ord + fmt::Display` — form a conjunction, a compound proposition asserting both properties simultaneously. The function body constitutes a proof of this compound proposition: it uses `>` (which requires `Ord`) and `format!` (which requires `Display`), and the fact that the body compiles is verification that the proof is valid given the hypotheses.

**Level 3** sees the proof witnesses. For this function to be callable with `i32`, there must exist `impl Ord for i32` and `impl Display for i32` — proof objects witnessing that `i32` satisfies both propositions. These are provided by the standard library. If you define your own type `struct Metres(f64)` and wish to call `format_max` on a collection of `Metres`, you must supply the proofs yourself: `impl Ord for Metres` and `impl Display for Metres`. Each `impl` block is an obligation, a piece of evidence that you construct and that the compiler verifies. The orphan rule ensures that these proof obligations have unique solutions — there is exactly one `impl Ord for i32`, not two conflicting ones.

**Level 4** sees the type-level functions. `NonEmpty<_>` is a type constructor — a function from types to types. `Vec<_>` is another. The composition `NonEmpty<T>` where `T` is constrained by `Ord + Display` carves out a *fibre* of the type constructor: not all possible `NonEmpty<_>` types, but only those whose element type satisfies the stated propositions. If this code used GATs or typestate encoding, Level 4 would also see the type-level state transitions — but even without them, the type constructor structure is present.

**The boundary** performs the final transformation. When the compiler processes `format_max`, it monomorphises: each concrete call site generates a specialised version of the function with all type parameters resolved. `format_max::<i32>` becomes a concrete function that compares `i32` values and formats them. `format_max::<Metres>` becomes a different concrete function that compares `Metres` values and formats them. The trait bounds vanish — they have been verified and serve no further purpose. The generic family collapses into specific functions. What was a universally quantified proposition in 𝒯 becomes a collection of concrete procedures in 𝒱.

All of this happens to the same code, simultaneously. The five levels are not alternative interpretations; they are different views of a single structure, each revealing aspects invisible from the others.

## What This Perspective Gives You

Reading Rust through the type-theoretical lens does not change what the language can do. It changes what you can *see*.

**Types become a design language.** When you model a problem with types, you are not just defining data structures — you are making formal claims about the relationships in your domain. A function signature becomes a theorem statement. An `impl` block becomes a proof that your type satisfies a contract. The question shifts from "what data does this hold?" to "what proposition does this express?"

**Compiler errors become logical feedback.** A type error is not a bug report; it is a notification that your reasoning contains a gap. The bound `T: Ord` that the compiler demands is not a hoop to jump through — it is an identification of a missing hypothesis in your argument. Once you see this, compiler errors become collaborators rather than obstacles.

**Abstraction levels become visible.** Most Rust programmers work at Levels 0 and 2 without distinguishing them. Recognising the level structure lets you choose your altitude deliberately. Some problems are best expressed as concrete data transformations (Level 0). Others are naturally universal statements about families of types (Levels 1–2). Still others require reasoning about proof structure itself (Level 3) or about type-level computation (Level 4). Knowing which level you are working at — and which level a problem *wants* to be expressed at — is a skill this book aims to develop.

**The boundary becomes a tool.** Understanding the forgetful functor from 𝒯 to 𝒱 lets you reason about zero-cost abstractions precisely. A `PhantomData<State>` typestate machine has rich structure in 𝒯 and zero cost in 𝒱 because the boundary erases it completely. A `dyn Trait` object has reduced structure because the boundary preserves behaviour but erases identity. Knowing what survives the boundary crossing and what does not is the key to writing Rust that is both expressive and efficient.

## What This Book Is Not

This book is not an introduction to Rust. It assumes you can read and write Rust comfortably — that you understand ownership, borrowing, lifetimes, traits, generics, and enums at the level of practical use. If you are still learning the language, read *The Rust Programming Language* first and return here once Rust's syntax and basic concepts feel natural.

This book is not a textbook on type theory. It draws on type theory extensively, but it introduces every concept it uses, defines every symbol before deploying it, and always connects the theory to concrete Rust. You will learn real type theory here — but you will learn it as a Rust programmer, not as a mathematics student.

This book is not about making Rust harder. The type-theoretical perspective does not add complexity to the language. It reveals structure that is already present. If anything, it makes certain aspects of Rust *simpler*, because it replaces ad hoc rules ("the orphan rule says you cannot...") with principled explanations ("coherence requires that proofs be unique, therefore...").

## Map of the Book

The book follows the level structure upward through the type hierarchy, with three framing chapters.

**Chapter 1: Background and Foundations** introduces the theoretical tools the book uses: Barendregt's lambda cube, Reynolds' work on polymorphism and parametricity, the type lattice, and the two-category model. This chapter lays the groundwork; it is reference material as much as narrative.

**Chapter 2: Propositions as Types** develops the Curry-Howard correspondence in full. This is the philosophical heart of the book — the chapter that makes the central thesis precise and shows how every major feature of Rust's type system maps onto a logical connective or proof rule.

**Chapters 3 through 7** ascend the level hierarchy:

- **Chapter 3: The Value Domain (Level 0)** — concrete types, plain `impl` blocks, ground propositions.
- **Chapter 4: Parametric Polymorphism (Level 1)** — unconstrained generics, Reynolds parametricity, theorems for free.
- **Chapter 5: Constrained Generics (Level 2)** — trait bounds as predicates, the constraint lattice, associated types.
- **Chapter 6: The Proof Domain (Level 3)** — `impl` blocks as witnesses, blanket impls as functors, coherence.
- **Chapter 7: Type Constructors and GATs (Level 4)** — type-level functions, typestate, the limits of Rust's abstraction.

**Chapter 8: The Boundary** examines the forgetful functor F: 𝒯 → 𝒱 in full depth — monomorphisation, dynamic dispatch, erasure, and the selective permeability of the membrane between compile time and runtime — and uses the boundary as a lens to unify the three proof mechanisms (inherent impls, trait impls, and proof-carrying types) introduced across the preceding chapters.

**Chapter 9: Beyond Rust** places Rust in the landscape of type systems. It maps the full lambda cube, examines what dependent types and effect systems make possible, and reflects on Rust's design position as a principled choice within the space of possible type systems.

## How to Read This Book

The chapters are designed to be read in order. Each one builds on vocabulary and concepts introduced in previous chapters, and the level model accumulates meaning as you ascend through it.

That said, three reading paths suit different starting points:

**The linear path.** Start at Chapter 1, read straight through. This is the intended experience. Each chapter adds one layer of the model, and the cumulative effect is a coherent reframing of Rust's type system from the ground up.

**The impatient path.** Start at Chapter 2 (Propositions as Types) for the core thesis, then skip to whichever level chapter addresses the Rust features you use most. Return to Chapter 1 for foundations when the later chapters reference concepts you want to understand more deeply.

**The comparative path.** Read Chapters 1, 2, 8, and 9 — the framing chapters — for a high-level view of the model and Rust's position in the type-system landscape. Dip into the level chapters as case studies.

Whichever path you choose, Chapter 2 is the one chapter you should not skip. Everything else in the book is an elaboration of the correspondence it establishes.

## A Note on Notation

This book uses mathematical notation where it clarifies and avoids it where it obscures. You will encounter:

- **∀** (for all) — universal quantification, corresponding to generic type parameters
- **∃** (there exists) — existential quantification, corresponding to `impl Trait` in return position
- **→** — implication in logic, function types in Rust
- **∧** — logical conjunction, corresponding to product types `(A, B)` and compound bounds `A + B`
- **∨** — logical disjunction, corresponding to sum types `enum { A, B }`
- **⊥** — falsity / the uninhabited type, corresponding to `!` in Rust
- **𝒯** and **𝒱** — the type category and value category
- **F: 𝒯 → 𝒱** — the forgetful functor (the boundary)

**A note on "value" versus "term".** Type theory conventionally speaks of *terms* — a term inhabits a type, and the lambda cube (introduced in Chapter 1) describes its axes as relationships between *terms* and *types*. Rust programmers speak of *values* — a value has a type, a function returns a value. This book follows Rust's convention: we write 𝒱 for the *value* category and speak of values throughout.

The choice is not merely cosmetic. "Term" in type theory encompasses both the syntactic expression and the thing it evaluates to — it straddles the boundary between 𝒯 and 𝒱. "Value" in Rust refers specifically to the evaluated, runtime entity — an inhabitant of 𝒱, not of 𝒯. The two words meet precisely at the boundary: a value is what remains after a term has been evaluated and its type-level structure erased. When we discuss the lambda cube and other formal frameworks in Chapter 1, we will use "term" in its standard type-theoretical sense and note explicitly where it maps onto Rust's "value".

Every symbol is defined before it is used, and all formal statements are accompanied by their Rust translations. If you encounter notation that has not been explained, it is a bug in the book, not a gap in your preparation.

## Beginning

The type-theoretical perspective on Rust is not esoteric knowledge reserved for language designers and academic researchers. It is, once you see it, the most natural way to understand what Rust's type system is actually doing — and why it is designed the way it is.

The borrow checker is not a safety net. It is a proof checker for a logic of resource ownership.

Trait bounds are not constraints. They are hypotheses in a formal argument.

Generic functions are not templates. They are universally quantified propositions.

And when your code compiles, you have not merely passed a set of checks. You have constructed a proof.

Let us begin with the foundations.
