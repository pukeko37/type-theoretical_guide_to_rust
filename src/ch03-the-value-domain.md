# The Value Domain (Level 0)

Every Rust programmer begins here. Before generics, before trait bounds, before lifetimes and type parameters and associated types — there are concrete types. A struct with named fields. An enum with known variants. A function that takes an `i32` and returns an `i32`. No polymorphism, no abstraction, no quantification. Just fixed types describing fixed data, and fixed functions transforming one piece of data into another.

This is Level 0: the value domain. It corresponds to the **simply typed lambda calculus** (λ→) — the most basic system of typed computation, sitting at the origin of the lambda cube. And while it may seem too simple to warrant a full chapter, it is precisely this simplicity that makes Level 0 the right place to establish the core ideas that the higher levels will generalise.

At Level 0, every concept we need is present in its simplest form. Types are propositions — but ground propositions, with no quantifiers. Impl blocks are proofs — but ground proofs, tethered to specific types. The boundary between 𝒯 and 𝒱 exists — but the mapping is nearly trivial, because there is little type-level structure to erase. By understanding what the model says at Level 0, we build intuition for what it will say at higher levels, where the structure becomes richer and the erasure more dramatic.

## Concrete Types as Points

In the type lattice introduced in Chapter 1, concrete types are the **bottom elements** — fully determined points with no free parameters. `i32` is a point. `String` is a point. `bool` is a point. Each one occupies a fixed position in the lattice, satisfying exactly the trait bounds it satisfies, and no others.

A concrete type is, in a precise sense, a **ground proposition**: a statement that is either true or false, with no variables to bind. The type `i32` asserts "there exist 32-bit signed integers." The existence of values like `0`, `1`, `-42` proves this proposition — they are its witnesses. An uninhabited type like `enum Void {}` is a ground proposition with no proof: it asserts the existence of values that do not exist.

At Level 0, the programmer works entirely with these fixed points. There is no parameterisation, no abstraction over types. Every function signature names its types explicitly, and every type is known at the point of definition.

## Products: Structs as Conjunctions

The struct is Level 0's primary tool for building composite types. A struct with named fields is a **product type** — a conjunction of its field types.

```rust
struct Sensor {
    id: u64,
    latitude: f64,
    longitude: f64,
    active: bool,
}
```

Under Curry-Howard, `Sensor` is the proposition `u64 ∧ f64 ∧ f64 ∧ bool`: there exists a sensor identifier, a latitude, a longitude, and an activity flag, all simultaneously. To construct a `Sensor`, you must supply evidence for all four conjuncts:

```rust
# struct Sensor { id: u64, latitude: f64, longitude: f64, active: bool }
fn new_sensor(id: u64, lat: f64, lon: f64) -> Sensor {
    Sensor {
        id,
        latitude: lat,
        longitude: lon,
        active: true,
    }
}
```

This constructor is a proof of the implication `u64 → f64 → f64 → Sensor`: given an identifier, a latitude, and a longitude, a sensor value can be produced (with `active` fixed to `true` as an internal decision of the proof).

### The Dual Life of a Struct

A struct definition has a **dual existence** in the two-category model. Consider what `Sensor` means in each category:

**In 𝒱** (the value category), `Sensor` is a concrete memory layout: 8 bytes for the `u64`, 8 bytes for each `f64`, and 1 byte for the `bool`, arranged contiguously with alignment padding. A value of type `Sensor` is this specific arrangement of bytes.

**In 𝒯** (the type category), `Sensor` is a named proposition — a node in the type lattice connected to other nodes by trait implementations. It may satisfy `Debug`, `Clone`, `Send`, `Sync`, and other trait bounds, each established by an impl block. These connections exist purely at compile time and are erased at the boundary.

The dual existence is the key insight of Level 0. A concrete type is *simultaneously* a runtime data description and a compile-time logical entity. The `struct` keyword creates both at once: a point in 𝒱 (the memory layout) and a point in 𝒯 (the type identity, with its lattice position). Most Rust programmers think primarily about the 𝒱 side — "what does this look like in memory?" — but the 𝒯 side is where the type-theoretical structure lives.

### Tuple Structs and Newtypes

Tuple structs are product types with positional rather than named fields:

```rust
struct Point(f64, f64);
struct Metres(f64);
struct Seconds(f64);
```

The newtype pattern — a struct with a single field — is the simplest illustration of the two-category perspective. As Chapter 1 described, the forgetful functor F maps both `Metres` and `Seconds` to the same representation:

```text
F(Metres)  = f64 (8 bytes, IEEE 754)
F(Seconds) = f64 (8 bytes, IEEE 754)
```

The type distinction lives entirely in 𝒯 and is erased at the boundary — the newtype costs nothing at runtime. Yet in 𝒯, `Metres` and `Seconds` can have completely different impl blocks, different trait implementations, different positions in the type lattice. This is the Level 0 insight: the logical structure is rich; the runtime structure is trivial.

## Sums: Enums as Closed-World Disjunctions

If structs are conjunctions, enums are disjunctions. An enum type asserts: "a value is one of these variants, and nothing else."

```rust
enum Command {
    Start,
    Stop,
    SetSpeed(f64),
    MoveTo { x: f64, y: f64 },
}
```

`Command` is the disjunction `⊤ ∨ ⊤ ∨ f64 ∨ (f64 ∧ f64)`: a command is either a start signal (no data), a stop signal (no data), a speed setting (carrying an `f64`), or a move instruction (carrying two `f64` coordinates). The variants without data carry the unit type implicitly — they are trivially true disjuncts.

### Closed-World Reasoning

Enums enable **closed-world reasoning**: the set of variants is fixed at definition time, known to the compiler, and cannot be extended without modifying the source. This is logically significant. When you pattern-match on an enum, you can enumerate every possibility:

```rust
# enum Command { Start, Stop, SetSpeed(f64), MoveTo { x: f64, y: f64 } }
fn describe(cmd: &Command) -> String {
    match cmd {
        Command::Start => "starting".to_string(),
        Command::Stop => "stopping".to_string(),
        Command::SetSpeed(v) => format!("setting speed to {v}"),
        Command::MoveTo { x, y } => format!("moving to ({x}, {y})"),
    }
}
```

The `match` is exhaustive: every possible variant is handled. The compiler verifies this. If you add a fifth variant to `Command` and forget to update the match, the compiler rejects the program — the proof of disjunction elimination is incomplete.

This exhaustiveness guarantee is a logical property that exists in 𝒯. In 𝒱, the match compiles to a branch table or a chain of comparisons — efficient runtime dispatch. The guarantee that all cases are covered is verified above the boundary and erased below it. The runtime never checks whether a match is exhaustive; the compile-time proof makes that check unnecessary.

### Enums as State Machines

At Level 0, enums naturally model finite state machines with known states:

```rust
enum ConnectionState {
    Disconnected,
    Connecting { address: String },
    Connected { address: String, latency_ms: u32 },
    Error { message: String },
}

impl ConnectionState {
    fn is_connected(&self) -> bool {
        matches!(self, ConnectionState::Connected { .. })
    }

    fn address(&self) -> Option<&str> {
        match self {
            ConnectionState::Connecting { address }
            | ConnectionState::Connected { address, .. } => Some(address),
            _ => None,
        }
    }
}
```

Each variant represents a state, and the associated data represents the information available in that state. The `match` expression forces the programmer to handle each state explicitly. This is a closed-world disjunction used as a state model — and at Level 0, it is the primary tool for expressing "this thing can be in one of several configurations."

We will see in Chapter 7 how Level 4 encodes state machines differently, using phantom types to make invalid state transitions *unrepresentable* rather than merely *unhandled*. The Level 0 encoding is less restrictive: the type system does not prevent you from constructing a `ConnectionState::Connected` directly without going through `Connecting` first. That stronger guarantee requires type-level machinery that Level 0 does not possess.

### Enums versus Traits: Closed versus Open

The distinction between closed-world (enum) and open-world (trait) disjunction is one of the most important design decisions in Rust. At Level 0, only the closed-world option is available:

| Property | Enum (Level 0) | `dyn Trait` (Level 2+) |
|---|---|---|
| Variants/implementors | Fixed at definition | Open to extension |
| Adding new variants | Requires modifying source | Add `impl` block anywhere |
| Pattern matching | Exhaustive, compiler-checked | Not available |
| Method dispatch | Static (match) | Dynamic (vtable) |
| Data layout | Known at compile time | Erased behind pointer |

At Level 0, the programmer trades extensibility for totality. An enum cannot be extended by downstream code — but every function that handles it can be verified to handle all cases. A trait can be implemented by any type — but you cannot enumerate all implementations and handle each one.

This trade-off is not a deficiency of either approach. It is a fundamental duality in logic: **closed-world** reasoning (everything is known) versus **open-world** reasoning (new facts may appear). Enums are closed; traits are open. Level 0 works exclusively with the closed world.

## Ground Propositions: Plain Impl Blocks

At Level 0, an `impl` block contains no generic parameters and no trait bounds. It is a **ground proposition** — a statement about a specific type, with no quantification:

```rust
struct Circle {
    radius: f64,
}

impl Circle {
    fn new(radius: f64) -> Self {
        Circle { radius }
    }

    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }

    fn circumference(&self) -> f64 {
        2.0 * std::f64::consts::PI * self.radius
    }

    fn scale(&self, factor: f64) -> Circle {
        Circle::new(self.radius * factor)
    }
}
```

This `impl` block makes specific claims about `Circle`: that a circle can be constructed from a radius, that its area and circumference can be computed, and that it can be scaled by a factor. These are ground propositions — they hold for `Circle` and no other type. There is no `<T>`, no `where` clause, no generality.

Under Curry-Howard, each method is a proof of an implication:
- `new`: `f64 → Circle` (given a radius, a circle exists)
- `area`: `&Circle → f64` (given a circle, an area exists)
- `circumference`: `&Circle → f64` (given a circle, a circumference exists)
- `scale`: `&Circle → f64 → Circle` (given a circle and a factor, a new circle exists)

These proofs are entirely concrete. They name specific types, use specific operations, and produce specific results. There is nothing to monomorphise, nothing to erase at the boundary beyond the type identity itself. The forgetful functor maps each method to a concrete function in 𝒱 — a specific sequence of machine instructions operating on specific memory layouts.

### Trait Implementations as Ground Proofs

When a concrete type implements a trait without generics, the result is a ground proof of a named proposition:

```rust
use std::fmt;

struct Metres(f64);

impl fmt::Display for Metres {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{:.2}m", self.0)
    }
}
```

This is a proof of the proposition `Display(Metres)`: the type `Metres` can be formatted for display. The proof is ground — it applies to `Metres` and nothing else. It establishes `Metres`'s position in the type lattice: `Metres` now sits below the `Display` bound, in the region of types that satisfy that predicate.

At Level 0, each such proof is written individually. If you have ten types that need `Display`, you write ten impl blocks — ten separate ground proofs. This is the characteristic cost of Level 0: every proposition must be proven for every type, one at a time. The universal quantification of Level 1 and the blanket impls of Level 3 exist precisely to eliminate this repetition — but at Level 0, we have no such tools.

## Functions at Level 0

A function at Level 0 has a fixed signature: every type is concrete, every domain and codomain is known.

```rust
fn clamp(value: f64, min: f64, max: f64) -> f64 {
    if value < min {
        min
    } else if value > max {
        max
    } else {
        value
    }
}
```

This function operates on `f64` and nothing else. It cannot clamp an `i32`, a `u8`, or any other numeric type. Its domain is fixed. Under Curry-Howard, it is a proof of the implication `f64 → f64 → f64 → f64` — a relationship between specific propositions, with no variables.

The limitation is immediately apparent. If you also need to clamp integers, you must write a separate function:

```rust
fn clamp_i32(value: i32, min: i32, max: i32) -> i32 {
    if value < min {
        min
    } else if value > max {
        max
    } else {
        value
    }
}
```

The body is identical. The logic is identical. Only the types differ. At Level 0, this duplication is unavoidable — there is no mechanism to abstract over the type. The programmer working exclusively at Level 0 must write and maintain separate implementations for each concrete type, even when the implementations are structurally identical.

This is not a hypothetical limitation. It is the daily reality of C programming, where the absence of parametric polymorphism leads to either code duplication or `void*` erasure (which abandons type safety). Rust programmers encounter the same limitation whenever they write concrete functions where generic ones would serve — and recognising this pattern is the first step toward understanding why Level 1 exists.

## The Simply Typed Lambda Calculus

Level 0 corresponds to **λ→**, the simply typed lambda calculus. In λ→:

- Types are built from base types (like `i32`, `bool`) and function types (like `i32 → bool`).
- Every term has a unique type, determined by the typing rules.
- There are no type variables, no polymorphism, no type-level computation.
- Type checking is decidable, straightforward, and fast.

This is the origin of the lambda cube — the vertex where no axes of dependency have been activated. Functions take values and produce values. Types classify values. That is all.

The simply typed lambda calculus has strong properties precisely because of its limitations:

**Strong normalisation:** every well-typed term in λ→ terminates. There are no infinite loops in a purely simply typed system. (Rust breaks this property with `loop {}`, recursion, and other general-recursion mechanisms — but the *type system* at Level 0 does not introduce non-termination. It is the value-level language that adds it.)

**Decidable type checking:** given a term and a candidate type, it is always possible to determine whether the term has that type. This remains true throughout Rust's type system, but it is most obviously true at Level 0, where there are no type variables to unify and no bounds to satisfy.

**Unique typing:** in the simplest form of λ→, every term has exactly one type. Rust relaxes this through coercions and subtyping (a `&'static str` can be used where a `&'a str` is expected), but at Level 0 with concrete types, the typing is essentially unique.

These properties make Level 0 the safest, most predictable part of the type system. They also make it the least expressive. The entire purpose of the higher levels is to sacrifice some of this simplicity in exchange for the ability to state more general propositions.

## The Distinction in 𝒯 and 𝒱

Level 0 is where the distinction between 𝒯 and 𝒱 is easiest to see — because at this level, the two categories are almost in correspondence.

Consider the type `Circle` from our earlier example. In 𝒱, it is a block of memory containing an `f64`. The function `Circle::area` is a sequence of machine instructions that reads this `f64`, multiplies it by itself and by π, and writes the result somewhere. Everything is concrete. Everything is bytes and instructions.

In 𝒯, `Circle` is a named node in the type lattice. It has associated impl blocks that position it relative to traits. If we add `impl Display for Circle`, then `Circle` acquires a connection to the `Display` node. If we add `impl Clone for Circle`, it acquires a connection to `Clone`. These connections are logical structure — they say what *can be done* with a `Circle`, what propositions it satisfies.

At Level 0, the forgetful functor F: 𝒯 → 𝒱 is nearly trivial:

| 𝒯 | F | 𝒱 |
|---|---|---|
| `Circle` (type identity) | → | `f64` (memory layout) |
| `impl Display for Circle` | → | Concrete `fmt` function |
| `impl Clone for Circle` | → | Concrete `clone` function |
| Type identity: `Circle ≠ f64` | → | (erased — same bytes) |

The functor erases type identity (in 𝒱, `Circle` is just `f64`) and resolves trait implementations to concrete functions. At Level 0, there is no polymorphism to monomorphise and no lifetimes to erase. The boundary crossing is minimal.

But it is *not* trivial. The type identity — the fact that `Circle` is not `f64`, even though they have the same representation — is real structure in 𝒯 that vanishes in 𝒱. This identity prevents a `Circle` from being used where a raw `f64` is expected, or vice versa. It is a logical distinction with no runtime cost — zero-cost abstraction at its simplest.

## What Level 0 Cannot Express

The characteristic limitation of Level 0 is **repetition without abstraction**. Whenever the same logical pattern applies to multiple types, the Level 0 programmer must restate it for each type individually.

Consider a pattern that arises constantly in practice: formatting a value for display.

```rust
struct Celsius(f64);
struct Fahrenheit(f64);
struct Kelvin(f64);

impl std::fmt::Display for Celsius {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{:.1}°C", self.0)
    }
}

impl std::fmt::Display for Fahrenheit {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{:.1}°F", self.0)
    }
}

impl std::fmt::Display for Kelvin {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{:.1}K", self.0)
    }
}
```

Three impl blocks, each proving `Display` for a specific type. Each is a ground proof — true for exactly one type. The pattern is obvious: each temperature unit is an `f64` with a unit suffix. But at Level 0, there is no way to state this pattern *once* and apply it to all three types. The abstraction — "any type that wraps an `f64` and has a unit suffix can be displayed" — requires the quantification and trait bounds of Levels 1 and 2.

Similarly, consider a function that should work with any of these temperature types:

```rust
# struct Celsius(f64);
fn celsius_above_boiling(temp: &Celsius) -> bool {
    temp.0 > 100.0
}
```

This is hopelessly specific. At Level 0, you would need `fahrenheit_above_boiling`, `kelvin_above_boiling`, and so on — each with its own threshold, each as a separate function. The observation that "checking whether a temperature exceeds a threshold" is a *general pattern* cannot be expressed at Level 0.

These limitations motivate the ascent through the levels:

- **Level 1** provides quantification over types: `fn identity<T>(x: T) -> T` works for all types, not just one.
- **Level 2** provides bounded quantification: `fn max<T: Ord>(a: T, b: T) -> T` works for all orderable types.
- **Level 3** provides proof construction: `impl<T: Display> Printable for T` derives proofs from other proofs.
- **Level 4** provides type-level functions: `Vec<_>` maps types to types.

Each level adds a form of abstraction that eliminates a class of repetition that the level below cannot avoid.

## The Level 0 Worldview

Every programming style reflects an implicit position in the level hierarchy. The Level 0 programmer's worldview has a distinctive character:

**Types describe data.** A struct is a memory layout. An enum is a tagged union. A function signature specifies the byte-level contract between caller and callee. Types are fundamentally *about* runtime representation.

**Impl blocks add behaviour.** Methods are functions attached to types. An `impl` block gives a type its API — the operations it supports. Traits are interfaces that types can optionally implement.

**The compiler checks correctness.** Type errors mean you are passing the wrong kind of data. The compiler catches mistakes that would otherwise cause runtime failures.

This worldview is not wrong. It is *incomplete*. It is the view from 𝒱 — the value category — projected upward through the boundary. The Level 0 programmer sees types as they appear after the forgetful functor has been applied: as data descriptions, with the logical structure stripped away.

The type-theoretical view sees the same types from above the boundary — from 𝒯:

**Types are propositions.** A struct is a conjunction. An enum is a disjunction. A function type is an implication. Types exist to make formal claims about the program's domain.

**Impl blocks are proofs.** They are evidence that a type satisfies a proposition. The compiler does not just "check correctness" — it verifies that your logical claims are substantiated by your code.

**Compiler errors are logical feedback.** They identify gaps in your reasoning, not just mistakes in your data handling.

The Level 0 worldview and the type-theoretical worldview are not contradictory. They are different views of the same structure — one from 𝒱, the other from 𝒯. At Level 0, the difference is subtle: ground propositions and data descriptions look almost the same, and the boundary barely distorts anything. But as we ascend through the levels, the gap widens. At Level 2, trait bounds are clearly propositions, not data descriptions. At Level 3, blanket impls are clearly proof constructions, not mere "behaviour." At Level 4, phantom types exist purely in 𝒯, with no 𝒱 counterpart at all.

Level 0 is where the two views are closest together. It is also where most Rust programmers spend most of their time. Understanding what it offers — and where it stops — is the foundation for understanding everything that follows.

## Ascending

The value domain is the ground floor of Rust's type system. It is expressive enough for many programs: concrete types, concrete functions, closed-world disjunctions, and ground proofs cover a vast range of practical software. Much production Rust — configuration parsing, protocol implementation, state machine drivers — lives primarily at Level 0.

But Level 0 is also where you notice the ceilings. The function that could be generic but is not. The impl block that repeats a pattern you have already written for three other types. The enum that models states but does not prevent invalid transitions. Each of these ceilings marks the boundary of what Level 0 can express — and points toward the level that lifts it.

In the next chapter, we ascend to Level 1: the domain of unconstrained parametric polymorphism. There, a single function can operate on all types, a single struct can hold any value, and the type signature alone determines the implementation. We leave the world of ground propositions and enter the world of universal quantification.
