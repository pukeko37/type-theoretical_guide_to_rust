# Type-Driven Rust Design Prompt

Encode domain invariants in the type system so the compiler proves them at zero runtime cost. Invalid states should be unrepresentable, not merely unchecked. Every domain rule that can be expressed as a type, trait, bound, or phantom parameter should be — runtime checks are a fallback for what the type system cannot yet express.

This document is a set of design rules. No theoretical derivation is included. For the reasoning behind each rule, see *Types as Propositions: A Type-Theoretical Guide to Rust*, Chapters 9 and 10.

---

## Phase 1: Proposition Inventory

Before writing code, list the domain's propositions in natural language and classify them into three categories:

- **Invariants** — properties of a single value that hold at all times. "Amount is positive." "Reason is non-empty." Each becomes a newtype with a smart constructor.
- **Contracts** — state-transition rules. "Only a draft can be submitted." "A rejected request is terminal." Each becomes a typestate transition (phantom parameter + consuming method).
- **Capabilities** — behavioural operations the domain requires. "Requests can be persisted." "Notifications can be sent." Each becomes a trait (port).

Every proposition in the inventory becomes a type, trait, bound, or phantom parameter. If a proposition has no corresponding type-level encoding, it will be enforced only by convention — flag it explicitly.

---

## Phase 2: Domain Types

### Newtypes with smart constructors

Wrap every domain value in a newtype with a private field. Provide a constructor that validates input and returns `Result<Self, Error>`. Never use raw `String`, `u64`, or `f64` for domain concepts. Every value of the type satisfies its invariant by construction — no further runtime checks needed.

### Typestate for lifecycles

Encode lifecycle states as zero-sized types. Parameterise the entity by `PhantomData<State>`. Implement transition methods only on the relevant state — `submit(self)` exists only on `ProcurementRequest<Draft>`, returning `ProcurementRequest<Submitted>`. Transitions consume `self` and return the next state. Invalid transitions are absent, not checked.

For branching transitions (where the next state depends on runtime input), return an enum of possible next states:

```rust,ignore
pub enum ReviewOutcome {
    Approved(Request<Approved>),
    Rejected(Request<Rejected>),
}
```

The caller must match on the outcome. The compiler enforces exhaustive handling.

### Dual representation

Use typestate for the command path (compile-time transition safety). Use a plain enum or snapshot struct for the query/persistence path (runtime state inspection, serialisation). Convert between the two at the boundary.

### Domain errors

Define a closed enum with one variant per failure mode. No infrastructure error types (`sqlx::Error`, `io::Error`) in the domain. Map infrastructure errors to domain error variants using `From` impls in the adapter layer. Propagate errors with `?`, never `.unwrap()` in domain logic.

### Commands and events

Define commands as a closed enum (sum type) with imperative naming: `CreateDraft`, `Submit`, `Review`. Define events as a closed enum with past-tense naming: `DraftCreated`, `Submitted`, `Approved`. Exhaustive `match` on both is enforced by the compiler. Adding a variant without updating handlers is a compile error.

### Domain core API

Express the domain's contract as a function signature:

```rust,ignore
fn handle(
    cmd: Command,
    repo: &impl Repository,
) -> Result<Vec<Event>, DomainError>
```

The types are the contract.

---

## Phase 3: Trait Design

- Define one trait per domain capability (single-responsibility). Do not bundle persistence, validation, and notification into one trait.
- Prefer domain-specific trait names (`RequestRepository`) over generic parameterised ones (`Repository<E>`). Use `Repository<E>` only when genuinely multiple entity types share a common persistence contract.
- Use associated types (`type Entity`) when the relationship between implementor and entity is one-to-one. Use a type parameter on the trait only when the relationship is one-to-many (e.g. `From<T>`).
- Port traits live in the domain crate. Implementations live in the infrastructure crate.

---

## Phase 4: Architecture

### Four rings

Organise the system into four layers with strict inward-only dependencies:

1. **Domain Core** — newtypes, typestate, domain traits, error types, commands, events. Zero infrastructure dependencies.
2. **Application** — generic functions bounded by domain traits (`repo: &impl RequestRepository`). Orchestrates domain logic without knowing concrete types.
3. **Infrastructure** — concrete `impl RequestRepository for PostgresRepo`. Provides `impl From<sqlx::Error> for DomainError`. This is where proofs are supplied.
4. **Composition Root** (`main.rs`) — all generics resolved to concrete types. The most concrete, least abstract code. This is correct by design.

### Cargo workspace

```text
workspace/
├── domain/        # Cargo.toml: NO infrastructure deps
├── app/           # depends on: domain
├── infra/         # depends on: domain, app, sqlx, etc.
└── bin/           # depends on: all
```

The domain crate must have no infrastructure dependencies in `Cargo.toml`. This is mechanically enforceable by CI.

### Dispatch strategy

Default to generic bounds (`<R: RequestRepository>`), not `dyn Trait` or `Arc<dyn Trait>`. Use `dyn` only for: runtime configuration, plugin systems, heterogeneous collections. When the concrete type is fixed at the composition root and never changes, generic dispatch gives zero-cost abstraction.

### Visibility

Use `pub(crate)` for infrastructure internals. The domain core sees the trait. The infrastructure crate sees the impl. The composition root connects them. Each layer sees exactly what it needs.

---

## Phase 5: Testing

- Test doubles are alternative implementations of the same trait. `InMemoryRepo` implements `RequestRepository` just as `PostgresRepo` does.
- Prefer **fakes** (genuine alternative implementations with real behaviour, e.g. backed by a `HashMap`) over **mocks** (interaction recordings). Fakes prove the same proposition in a different model; mocks prove only that certain calls occurred.
- Wire the service with the fake in tests: `ProcurementService::new(InMemoryRepo::new())`.
- Derive property tests from type signatures: a function generic over `T: Ord` cannot fabricate `T` values, so the output must be drawn from the input. Each constraint derived from the signature becomes a testable property.

---

## Anti-Pattern Checklist

| Symptom | Fix |
|---|---|
| `status: String`, `id: u64` | Newtypes with smart constructors |
| Domain imports `sqlx::Row` | Adapter mapping at ring boundary |
| Trait with 15 methods | One trait per capability |
| `.clone()` everywhere at boundaries | Accept references; design for ownership |
| `.unwrap()` in domain logic | Propagate `Result` with `?` |
| `Arc<dyn Trait>` by default | Generic bounds; resolve at composition root |

---

## Skeleton

```rust
use std::marker::PhantomData;

// -- Newtype with smart constructor (Level 0) --
#[derive(Debug, Clone, PartialEq)]
pub struct Amount(f64);

#[derive(Debug)]
pub struct AmountError;

impl Amount {
    pub fn new(value: f64) -> Result<Self, AmountError> {
        if value > 0.0 && value.is_finite() {
            Ok(Amount(value))
        } else {
            Err(AmountError)
        }
    }
    pub fn value(&self) -> f64 { self.0 }
}

// -- Typestate (Level 4) --
pub struct Draft;
pub struct Submitted;

pub struct Request<State> {
    amount: Amount,
    _state: PhantomData<State>,
}

impl Request<Draft> {
    pub fn new(amount: Amount) -> Self {
        Request { amount, _state: PhantomData }
    }
    pub fn submit(self) -> Request<Submitted> {
        Request { amount: self.amount, _state: PhantomData }
    }
}

// -- Domain error (Level 0, closed enum) --
#[derive(Debug)]
pub enum DomainError {
    InvalidAmount,
    NotFound,
    // Infrastructure adapters map their errors into these variants
    // via `impl From<sqlx::Error> for DomainError` in the infra crate.
    StorageUnavailable,
}

// -- Port trait (Level 2) — lives in domain crate --
pub trait RequestRepository {
    fn save(&self, amount: f64) -> Result<(), DomainError>;
    fn find(&self, id: u64) -> Result<Option<f64>, DomainError>;
}

// -- Application layer: generic over port --
pub fn create_draft(
    repo: &impl RequestRepository,
    amount: Amount,
) -> Result<(), DomainError> {
    repo.save(amount.value())
}

// Infrastructure impl (in infra crate):
//   impl RequestRepository for PostgresRepo { ... }
//
// Composition root (in bin crate):
//   let repo = PostgresRepo::new(pool);
//   create_draft(&repo, amount)?;

fn main() {
    let amount = Amount::new(42.0).expect("test value");
    let draft = Request::new(amount);
    let _submitted = draft.submit();
    // draft.submit() here would fail: value moved
}
```
