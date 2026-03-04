# Domain Modelling as Proof Construction

The five levels of Rust's type hierarchy are not merely a taxonomy. They are a vocabulary — a compositional system for expressing domain models as sets of propositions proved by construction. The preceding chapters derived each level independently: newtypes as zero-cost distinctions, generics as universal quantification, trait bounds as predicates, impl blocks as proofs, type constructors as higher-order operations. This chapter shows what happens when you deploy all five levels simultaneously against a single domain problem.

The thesis is practical: a well-modelled domain core in Rust is a collection of propositions about the domain, encoded in 𝒯, verified by the compiler, and erased at the boundary. Invalid states are not caught at runtime — they are unrepresentable. Business rules are not checked by `if` statements — they are proved by type structure. The compiler does not merely prevent memory errors; it prevents *domain* errors, with the same zero-cost guarantee.

To make this concrete, we will build one domain model from start to finish: a procurement request system. Each section adds one level's contribution. By the chapter's end, the model is complete — a domain core that proves its own invariants.

### The Domain

A procurement request follows a lifecycle. Someone drafts a request for goods or services, specifying an amount and a reason. The draft is submitted for approval. A reviewer examines it. The reviewer either approves or rejects the request. An approved request is eventually closed when the goods are received. The rules are:

- Every request has a unique identifier, a positive monetary amount, a reason, and an author.
- Only a draft can be submitted.
- Only a submitted request can be reviewed.
- A review results in approval or rejection — the reviewer cannot defer indefinitely.
- Only an approved request can be closed.
- A rejected request is terminal.

These are domain propositions. The question is where they live: in 𝒯, where the compiler proves them, or in 𝒱, where runtime code checks them.

### The Weak Model

Consider a first attempt — the kind of model that emerges when a programmer thinks only about data:

```rust,ignore
// Deliberately weak — do not emulate
struct ProcurementRequest {
    id: u64,
    amount: f64,
    reason: String,
    status: String,
    reviewer: Option<String>,
}

impl ProcurementRequest {
    fn submit(&mut self) {
        if self.status == "draft" {
            self.status = "submitted".to_string();
        }
    }

    fn approve(&mut self, reviewer: &str) {
        if self.status == "submitted" {
            self.status = "approved".to_string();
            self.reviewer = Some(reviewer.to_string());
        }
    }
}
```

Diagnose this type-theoretically. The domain propositions exist — "only a draft can be submitted" — but they live in 𝒱, encoded as string comparisons at runtime. The type `String` for `status` carries no information in 𝒯; the compiler cannot distinguish a draft from an approved request. The proposition "amount is positive" is unencoded entirely. The relationship between states is implicit in scattered `if` checks, invisible to the type system.

In the two-category model, this design places domain logic below the boundary where the compiler cannot help. The goal of this chapter is to lift it above.

## The Proposition Inventory

Before writing any types, extract the domain's propositions in natural language and classify them. Three kinds of domain statement map to different levels:

**Invariants** are properties that hold of a single value at all times.
- "A request identifier is a valid UUID" — property of a type
- "A monetary amount is positive" — property of a type
- "A reason is non-empty" — property of a type

These are Level 0 propositions. They are proved by the type's construction and maintained by the type's API surface.

**Contracts** are relationships between states or between types.
- "A request can only be submitted if it is in draft state" — state transition rule
- "A request can only be approved if it has been reviewed" — state ordering
- "A rejected request permits no further transitions" — terminal state

These are Level 4 propositions (typestate) or Level 2 propositions (trait bounds constraining which operations are available).

**Capabilities** are behavioural propositions — claims about what operations a type supports.
- "A submitted request can be withdrawn by its author" — method availability
- "Any request can be displayed for audit" — trait bound
- "A request can be persisted to and retrieved from storage" — port contract

These are Level 2–3 propositions: trait definitions (Level 2) and their implementations (Level 3).

The proposition inventory for the procurement domain:

| Proposition | Kind | Level |
|---|---|---|
| Request ID is a valid UUID | Invariant | 0 |
| Amount is positive | Invariant | 0 |
| Reason is non-empty | Invariant | 0 |
| Approver ID is a valid UUID | Invariant | 0 |
| Only drafts can be submitted | Contract | 4 |
| Only submitted requests can be reviewed | Contract | 4 |
| Review produces approval or rejection | Contract | 4 |
| Only approved requests can be closed | Contract | 4 |
| Rejected requests are terminal | Contract | 4 |
| Any request state can be displayed | Capability | 2 |
| Requests can be persisted | Capability | 2–3 |
| Domain errors have a defined vocabulary | Structural | 0 (sum type) |

This inventory drives every subsequent design decision. Each row becomes a type, a trait, a bound, or a phantom parameter.

## Level 0: Named Inhabitants

Chapter 3 established that a newtype creates a distinction in 𝒯 that is erased at the boundary — `Metres(f64)` and `Seconds(f64)` have identical runtime representations but occupy different positions in the type lattice. For domain modelling, newtypes do something further: combined with a private field and a validating constructor, they *prove* a property at construction time and carry that proof silently thereafter.

### Smart Constructors

A smart constructor is a function that validates input and returns a newtype only if the validation passes. The type's field is private, so the only way to obtain a value is through the constructor. This means every value of the type satisfies the invariant — not by convention, but by construction.

```rust
use std::fmt;

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct RequestId(uuid::Uuid);

impl RequestId {
    pub fn new() -> Self {
        RequestId(uuid::Uuid::new_v4())
    }

    pub fn parse(s: &str) -> Result<Self, RequestIdError> {
        let id = s.parse::<uuid::Uuid>().map_err(|_| RequestIdError)?;
        Ok(RequestId(id))
    }
}

impl fmt::Display for RequestId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}

#[derive(Debug)]
pub struct RequestIdError;

impl fmt::Display for RequestIdError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "invalid request ID")
    }
}
# // Minimal uuid stub for compilation:
# mod uuid {
#     #[derive(Debug, Clone, PartialEq, Eq, Hash)]
#     pub struct Uuid([u8; 16]);
#     impl Uuid {
#         pub fn new_v4() -> Self { Uuid([0; 16]) }
#     }
#     impl std::str::FromStr for Uuid {
#         type Err = ();
#         fn from_str(s: &str) -> Result<Self, ()> {
#             if s.len() == 36 { Ok(Uuid([0; 16])) } else { Err(()) }
#         }
#     }
#     impl std::fmt::Display for Uuid {
#         fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
#             write!(f, "00000000-0000-0000-0000-000000000000")
#         }
#     }
# }
```

The proposition carried by `RequestId` is: "this value was produced by `new` (a fresh UUID) or `parse` (a validated string)." No code outside the module can construct a `RequestId` by other means — the field is private. The proof is established once at construction time and carried through the program without further checking.

The same pattern applies to the monetary amount:

```rust
use std::fmt;

#[derive(Debug, Clone, Copy, PartialEq)]
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

    pub fn value(&self) -> f64 {
        self.0
    }
}

impl fmt::Display for Amount {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{:.2}", self.0)
    }
}

impl fmt::Display for AmountError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "amount must be positive and finite")
    }
}
```

Every `Amount` in the program is positive and finite. This is not a runtime check that might be forgotten — it is a structural guarantee. The proposition "amount is positive" has been lifted from 𝒱 (an `if` check scattered through business logic) to 𝒯 (a property of the type itself, proved at construction).

The same principle applies to `ApproverId` (a non-empty string newtype) and `Reason` (also a non-empty string newtype):

```rust
use std::fmt;

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct ApproverId(String);

impl ApproverId {
    pub fn new(name: &str) -> Result<Self, ValidationError> {
        if name.trim().is_empty() {
            Err(ValidationError("approver ID must not be empty"))
        } else {
            Ok(ApproverId(name.to_string()))
        }
    }
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Reason(String);

impl Reason {
    pub fn new(text: &str) -> Result<Self, ValidationError> {
        if text.trim().is_empty() {
            Err(ValidationError("reason must not be empty"))
        } else {
            Ok(Reason(text.to_string()))
        }
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

#[derive(Debug)]
pub struct ValidationError(&'static str);

impl fmt::Display for ValidationError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}

impl fmt::Display for ApproverId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}

impl fmt::Display for Reason {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}
```

None of these newtypes carries runtime overhead beyond the underlying representation. The forgetful functor maps `Amount(f64)` to `f64`, `Reason(String)` to `String`. The proofs — positivity, non-emptiness, valid UUID format — exist in 𝒯 and are erased at the boundary. This is the Level 0 contribution: named inhabitants that carry domain invariants at zero cost.

## Levels 2–3: The Trait Lattice as Domain Contract

Chapters 5 and 6 established the relationship between trait bounds and impl blocks: a bound states a proposition ("for all `T` satisfying `P`"), and an impl block proves it ("here is the evidence that this specific type satisfies `P`"). In domain modelling, this machinery becomes a tool for expressing domain contracts — the capabilities and relationships that define what the domain can do.

### Domain Traits as Predicates

Chapter 5 showed that a trait bound is a predicate over types: `T: Ord` asserts Ord(`T`), restricting `T` to totally ordered types. The same principle applies to domain-specific traits. A trait `Submittable` is a predicate: Submittable(`T`) asserts that values of type `T` can be submitted.

The design question is not whether to use traits — it is *which propositions to encode as traits*. Not every domain concept needs a trait. A trait is appropriate when:

- Multiple types may satisfy the same capability (a trait as an abstraction boundary).
- A capability must be available generically, without knowing the concrete type (a trait as a port).
- A capability defines a contract that infrastructure must fulfil (a trait as a dependency inversion point).

For the procurement domain, the critical trait is the persistence contract — the port through which the domain core communicates with storage:

```rust
# use std::fmt;
# #[derive(Debug, Clone)]
# pub struct RequestId;
# #[derive(Debug, Clone)]
# pub struct Amount;
# #[derive(Debug, Clone)]
# pub struct Reason;
# #[derive(Debug)]
# pub struct ProcurementError;
# impl fmt::Display for ProcurementError {
#     fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "error") }
# }
# #[derive(Debug, Clone)]
# pub struct RequestSnapshot {
#     pub id: RequestId,
#     pub amount: Amount,
#     pub reason: Reason,
#     pub status: String,
# }
pub trait RequestRepository {
    fn save(&self, snapshot: &RequestSnapshot) -> Result<(), ProcurementError>;
    fn find(&self, id: &RequestId) -> Result<Option<RequestSnapshot>, ProcurementError>;
}
```

The `RequestSnapshot` type (defined fully in the next section) is the serialisable representation of a procurement request — the data that crosses the persistence boundary. The trait is a proposition: "there exists a mechanism that can persist and retrieve procurement requests." The domain core states this proposition; infrastructure proves it with a concrete implementation.

### The ∀ versus ∃ Design Choice

Chapter 5 established the distinction between type parameters and associated types: a type parameter creates a relation in 𝒯 (one implementing type, potentially many parameter instantiations), while an associated type creates a function (one implementing type, exactly one associated type). This distinction, reframed in logical terms, becomes a critical design tool.

A generic type parameter is **universal quantification**: `fn process<E: Event>(e: E)` says "for all types `E` satisfying `Event`, this function can process `E`." The function makes no commitment to a specific event type — it works for any.

An associated type is **existential quantification at the impl level**: `trait Repository { type Entity; }` says "there exists a type `Entity` such that this repository can persist it." The existential is not free-floating — it is *witnessed* by each implementation. When you write `impl Repository for PostgresOrderRepo { type Entity = Order; }`, the impl proves: "there exists an entity type (namely `Order`) for which `PostgresOrderRepo` is a repository." From the consumer's perspective — a function bounded by `fn f(repo: &impl Repository)` — the entity type is determined by whichever impl is provided. The consumer does not choose it; it comes as part of the proof.

The distinction matters for domain design. Consider two formulations of the repository trait:

```rust,ignore
// Universal: Repository is parameterised over entity type
trait Repository<E> {
    fn save(&self, entity: &E) -> Result<(), Error>;
    fn find_by_id(&self, id: &str) -> Result<Option<E>, Error>;
}

// Existential: each repository determines its entity type
trait Repository {
    type Entity;
    fn save(&self, entity: &Self::Entity) -> Result<(), Error>;
    fn find_by_id(&self, id: &str) -> Result<Option<Self::Entity>, Error>;
}
```

The universal formulation (`Repository<E>`) allows a single type to implement `Repository<Order>` *and* `Repository<Invoice>` — it is a relation. This invites an incoherent design: a single database connection that handles every entity type, conflating distinct persistence concerns.

The existential formulation (`type Entity`) creates a functional dependency: each repository type is bound to exactly one entity type. The relationship is a function in 𝒯 — one input, one output — and the design is coherent by construction. A function that accepts `repo: &impl Repository` can use `repo.save(entity)` without knowing the concrete repository type, but the entity type is fixed by whichever implementation arrives:

```rust
use std::fmt;

#[derive(Debug, Clone)]
pub struct RequestId;
#[derive(Debug, Clone)]
pub struct Amount;
#[derive(Debug, Clone)]
pub struct Reason;

#[derive(Debug)]
pub struct ProcurementError;
impl fmt::Display for ProcurementError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "error") }
}

/// A snapshot of a procurement request's persistent state.
#[derive(Debug, Clone)]
pub struct RequestSnapshot {
    pub id: RequestId,
    pub amount: Amount,
    pub reason: Reason,
    pub status: String,
}

/// A generic repository: each implementor witnesses one Entity type.
pub trait Repository {
    type Entity;
    fn save(&self, entity: &Self::Entity) -> Result<(), ProcurementError>;
    fn find_by_id(&self, id: &str) -> Result<Option<Self::Entity>, ProcurementError>;
}

/// The procurement adapter witnesses Entity = RequestSnapshot.
struct InMemoryRequestRepo;

impl Repository for InMemoryRequestRepo {
    type Entity = RequestSnapshot;

    fn save(&self, _entity: &RequestSnapshot) -> Result<(), ProcurementError> {
        Ok(()) // simplified
    }

    fn find_by_id(&self, _id: &str) -> Result<Option<RequestSnapshot>, ProcurementError> {
        Ok(None) // simplified
    }
}

/// A consumer that is generic over any Repository whose Entity is RequestSnapshot.
fn count_all(repo: &impl Repository<Entity = RequestSnapshot>) -> Result<usize, ProcurementError> {
    // The bound Repository<Entity = RequestSnapshot> constrains the existential:
    // "I accept any repository, provided its witnessed entity type is RequestSnapshot."
    Ok(0) // simplified
}
#
# fn main() {
#     let repo = InMemoryRequestRepo;
#     let _ = count_all(&repo);
# }
```

The bound `Repository<Entity = RequestSnapshot>` is where the ∀ and ∃ meet. The function is universally quantified over repository implementations ("for all repos satisfying this trait"), but the associated type constrains the existential witness ("the entity type must be `RequestSnapshot`"). The caller provides the repository; the repository provides the entity type; the function works with both without knowing either concretely.

**Three design positions.** In practice, the associated-type `Repository` trait is most valuable when a codebase has *many* entity types sharing a common persistence contract. For a single domain like procurement, where only one entity type is persisted through the port, the trait can be specialised further — naming the entity type directly and dropping the associated type:

```rust
# use std::fmt;
# #[derive(Debug, Clone)]
# pub struct RequestId;
# #[derive(Debug, Clone)]
# pub struct Amount;
# #[derive(Debug, Clone)]
# pub struct Reason;
# #[derive(Debug)]
# pub struct ProcurementError;
# impl fmt::Display for ProcurementError {
#     fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "error") }
# }
# #[derive(Debug, Clone)]
# pub struct RequestSnapshot {
#     pub id: RequestId,
#     pub amount: Amount,
#     pub reason: Reason,
#     pub status: String,
# }
pub trait RequestRepository {
    fn save(&self, snapshot: &RequestSnapshot) -> Result<(), ProcurementError>;
    fn find(&self, id: &RequestId) -> Result<Option<RequestSnapshot>, ProcurementError>;
}
```

This is the associated-type pattern taken to its logical conclusion: the functional dependency between repository and entity is so tight that the trait is named for the entity it serves. The entity type appears directly in the method signatures rather than through `Self::Entity`. The single-responsibility guarantee that the associated type enforced structurally is now enforced by the trait's definition — a `RequestRepository` cannot serve invoices because its methods do not mention invoices.

The three positions form a spectrum:

| Design | Trait Form | ∀/∃ Character | When to Use |
|---|---|---|---|
| Type parameter | `Repository<E>` | ∀ over entities — relational | Rarely; invites incoherence |
| Associated type | `Repository { type Entity; }` | ∃ witness per impl — functional | Multiple entity types, shared contract |
| Domain-specific | `RequestRepository` | Entity fixed by definition | Single entity per port — the common case |

For the procurement domain, we use the domain-specific form. The remaining examples in this chapter and the next use `RequestRepository` directly.

### Axioms, Inference Rules, and Derived Theorems

Chapter 6 showed that impl blocks form a proof system: ground impls prove propositions about specific types, and blanket impls derive new proofs from existing ones. This structure has a precise vocabulary:

**Axioms** are primitive type implementations — proofs that the standard library or the language provides without derivation:

```rust,ignore
// Axiom: u32 satisfies Ord (provided by std)
// impl Ord for u32 { ... }
```

**Inference rules** are blanket impls — proof constructors that produce new proofs from existing proofs:

```rust,ignore
// Inference rule: if T satisfies Ord, then Vec<T> satisfies Ord
// impl<T: Ord> Ord for Vec<T> { ... }
```

**Derived theorems** are the proofs that follow from combining axioms and inference rules. The compiler derives these automatically:

```text
Axiom:           Ord(u32)
Inference rule:  ∀T. Ord(T) → Ord(Vec<T>)
Derived theorem: Ord(Vec<u32>)
```

In the procurement domain, the derivation tree looks like this:

```text
Axiom:           Display(String)     [std]
Axiom:           Display(f64)        [std]
Ground proof:    Display(Amount)     [our impl: delegates to f64]
Ground proof:    Display(Reason)     [our impl: delegates to String]
Ground proof:    Display(RequestId)  [our impl: formats UUID]
```

Each ground proof (our `impl Display for Amount`) is a leaf in the derivation tree — it does not depend on other domain proofs, only on axioms from the standard library. As the domain grows, the derivation trees deepen:

```text
Ground proof:    Debug(RequestId)    [derive macro]
Ground proof:    Debug(Amount)       [derive macro]
Ground proof:    Debug(Reason)       [derive macro]
Ground proof:    Debug(RequestSnapshot) [derive macro, depends on above three]
```

The `derive` attribute, as Chapter 6 discussed, is a proof generator: `#[derive(Debug)]` on `RequestSnapshot` generates `impl Debug for RequestSnapshot` by composing the `Debug` proofs of its fields. This is an inference rule instantiated by the compiler: "if all fields satisfy `Debug`, then the struct satisfies `Debug`."

The orphan rule (Chapter 6) keeps this proof system consistent: you can only write `impl Trait for Type` if you own either the trait or the type. This prevents conflicting proofs — two different impls of the same trait for the same type — which would make the proof system unsound. In domain modelling, the orphan rule means your domain types can freely implement your domain traits, and the compiler guarantees no external crate can provide a conflicting implementation.

## Level 4: Typestate for Domain Lifecycles

Chapter 7 introduced the typestate pattern with a `Connection` example: states encoded as phantom type parameters, transitions as functions that consume one state and produce another, invalid transitions rejected at compile time. For domain modelling, typestate encodes the lifecycle rules from the proposition inventory — the contracts that govern which operations are available in which states.

### The Procurement State Machine

The procurement lifecycle has six states: Draft, Submitted, UnderReview, Approved, Rejected, and Closed. The transitions are:

```text
Draft → Submitted → UnderReview → Approved → Closed
                                ↘ Rejected (terminal)
```

In typestate, each state is a zero-sized type, and the procurement request carries its state as a phantom parameter:

```rust
use std::marker::PhantomData;
use std::fmt;

// --- Domain value types (simplified from §9.2) ---
# #[derive(Debug, Clone, PartialEq)]
# pub struct Amount(f64);
# impl Amount {
#     pub fn new(value: f64) -> Result<Self, &'static str> {
#         if value > 0.0 && value.is_finite() { Ok(Amount(value)) } else { Err("invalid") }
#     }
#     pub fn value(&self) -> f64 { self.0 }
# }
# #[derive(Debug, Clone, PartialEq, Eq)]
# pub struct Reason(String);
# impl Reason {
#     pub fn new(text: &str) -> Result<Self, &'static str> {
#         if text.trim().is_empty() { Err("empty") } else { Ok(Reason(text.to_string())) }
#     }
#     pub fn as_str(&self) -> &str { &self.0 }
# }
# #[derive(Debug, Clone, PartialEq, Eq, Hash)]
# pub struct RequestId(u64);
# impl RequestId { pub fn new(id: u64) -> Self { RequestId(id) } }
# #[derive(Debug, Clone, PartialEq, Eq)]
# pub struct ApproverId(String);
# impl ApproverId {
#     pub fn new(name: &str) -> Result<Self, &'static str> {
#         if name.trim().is_empty() { Err("empty") } else { Ok(ApproverId(name.to_string())) }
#     }
# }

// --- States: zero-sized types, exist only in 𝒯 ---
pub struct Draft;
pub struct Submitted;
pub struct UnderReview;
pub struct Approved;
pub struct Rejected;
pub struct Closed;

// --- The procurement request, parameterised by state ---
pub struct ProcurementRequest<State> {
    id: RequestId,
    amount: Amount,
    reason: Reason,
    _state: PhantomData<State>,
}

impl ProcurementRequest<Draft> {
    pub fn new(id: RequestId, amount: Amount, reason: Reason) -> Self {
        ProcurementRequest {
            id,
            amount,
            reason,
            _state: PhantomData,
        }
    }

    pub fn submit(self) -> ProcurementRequest<Submitted> {
        ProcurementRequest {
            id: self.id,
            amount: self.amount,
            reason: self.reason,
            _state: PhantomData,
        }
    }
}

impl ProcurementRequest<Submitted> {
    pub fn begin_review(self) -> ProcurementRequest<UnderReview> {
        ProcurementRequest {
            id: self.id,
            amount: self.amount,
            reason: self.reason,
            _state: PhantomData,
        }
    }
}

/// The outcome of a review: the request branches into one of two states.
pub enum ReviewOutcome {
    Approved(ProcurementRequest<Approved>),
    Rejected(ProcurementRequest<Rejected>),
}

impl ProcurementRequest<UnderReview> {
    pub fn review(self, approved: bool, approver: ApproverId) -> ReviewOutcome {
        if approved {
            ReviewOutcome::Approved(ProcurementRequest {
                id: self.id,
                amount: self.amount,
                reason: self.reason,
                _state: PhantomData,
            })
        } else {
            ReviewOutcome::Rejected(ProcurementRequest {
                id: self.id,
                amount: self.amount,
                reason: self.reason,
                _state: PhantomData,
            })
        }
    }
}

impl ProcurementRequest<Approved> {
    pub fn close(self) -> ProcurementRequest<Closed> {
        ProcurementRequest {
            id: self.id,
            amount: self.amount,
            reason: self.reason,
            _state: PhantomData,
        }
    }
}

// Methods available in any state:
impl<State> ProcurementRequest<State> {
    pub fn id(&self) -> &RequestId {
        &self.id
    }

    pub fn amount(&self) -> &Amount {
        &self.amount
    }

    pub fn reason(&self) -> &Reason {
        &self.reason
    }
}

impl<State> fmt::Debug for ProcurementRequest<State> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("ProcurementRequest")
            .field("id", &self.id)
            .field("amount", &self.amount.value())
            .field("reason", &self.reason.as_str())
            .finish()
    }
}
#
# fn main() {
#     let id = RequestId::new(1);
#     let amount = Amount::new(100.0).unwrap();
#     let reason = Reason::new("Office supplies").unwrap();
#     let request = ProcurementRequest::new(id, amount, reason);
#     let submitted = request.submit();
#     let under_review = submitted.begin_review();
#     let approver = ApproverId::new("Alice").unwrap();
#     match under_review.review(true, approver) {
#         ReviewOutcome::Approved(req) => { let _closed = req.close(); }
#         ReviewOutcome::Rejected(_req) => {}
#     }
# }
```

The `review` method deserves attention. It returns `ReviewOutcome`, an enum of the two possible next states. This is the typestate pattern's answer to branching transitions: when the next state depends on runtime input (the reviewer's decision), the method returns a sum type whose variants are the possible successor states. The caller must match on the outcome, and each arm gives access to a request in the appropriate state. The compiler enforces that no code path can approve a request that was rejected or close one that was not approved.

### The State Transition Category

The typestate encoding has a precise categorical interpretation. The states are objects in a small category. The transition methods are morphisms:

```text
Objects:    Draft, Submitted, UnderReview, Approved, Rejected, Closed
Morphisms:  submit: Draft → Submitted
            begin_review: Submitted → UnderReview
            review(true): UnderReview → Approved
            review(false): UnderReview → Rejected
            close: Approved → Closed
```

The critical property: **missing morphisms are illegal transitions.** There is no morphism Draft → Approved, no morphism Rejected → Closed, no morphism Closed → anything. These transitions do not exist as methods; they cannot be called; the compiler rejects any attempt. The state machine's correctness is not checked — it is *constitutive*. The types define what transitions exist, and everything else is excluded by absence.

### Typestate versus Enum State

Chapter 7 presented the tradeoffs between typestate and enum-based state machines. For the procurement domain, this choice has practical consequences:

| Criterion | Typestate | Enum state |
|---|---|---|
| Compile-time transition safety | Yes — invalid transitions unrepresentable | No — requires runtime checks |
| Runtime state queries | Difficult — state is erased at boundary | Natural — `match` on discriminant |
| Serialisation/persistence | Requires conversion to serialisable form | Direct — enum serialises naturally |
| Heterogeneous collections | Cannot store `Vec<ProcurementRequest<_>>` | Can store `Vec<ProcurementRequest>` |

The typestate approach excels for command-side logic where the state machine is traversed linearly: create, submit, review, close. The enum approach excels for query-side concerns: displaying a list of requests in various states, serialising to a database, reporting.

A well-designed domain core often uses *both*: typestate for the command path (where invalid transitions must be prevented) and an enum snapshot for the query path (where state must be inspected and stored). The `RequestSnapshot` type from §9.3 serves this purpose — it is the serialisable, queryable representation that complements the typestate model.

## Error Types as Logical Connectives

Chapter 2 established the correspondence between logical connectives and Rust types: `Result<T, E>` is the disjunction `T ∨ E`, and `?` is monadic bind that propagates the right disjunct. For domain modelling, this correspondence transforms error handling from an operational concern into a logical one.

### Domain Errors as Propositions

A domain error type is a proposition about the ways an operation can fail. Each variant names a failure mode and carries the evidence:

```rust
use std::fmt;

#[derive(Debug)]
pub enum ProcurementError {
    /// The amount was not positive or not finite.
    InvalidAmount,
    /// The reason text was empty.
    EmptyReason,
    /// The requested state transition is not permitted.
    InvalidTransition { from: &'static str, to: &'static str },
    /// The request was not found in storage.
    NotFound { id: String },
    /// Storage is unavailable.
    StorageUnavailable { cause: String },
}

impl fmt::Display for ProcurementError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::InvalidAmount => write!(f, "amount must be positive and finite"),
            Self::EmptyReason => write!(f, "reason must not be empty"),
            Self::InvalidTransition { from, to } =>
                write!(f, "cannot transition from {from} to {to}"),
            Self::NotFound { id } => write!(f, "request {id} not found"),
            Self::StorageUnavailable { cause } =>
                write!(f, "storage unavailable: {cause}"),
        }
    }
}

impl std::error::Error for ProcurementError {}
```

The error enum is a closed-world disjunction (Chapter 3): the domain defines exactly the ways operations can fail, and the `match` on `ProcurementError` is exhaustive. If a new failure mode emerges, adding a variant forces every handler to be updated — the compiler tracks the proof obligation.

### The Ring Boundary for Errors

Notice that `ProcurementError` has a `StorageUnavailable` variant. This is a domain-level description of an infrastructure failure. It is *not* `sqlx::Error` or `std::io::Error` — those are infrastructure types that the domain core must not know about. The adapter layer (Chapter 10) transforms infrastructure errors into domain errors:

```rust,ignore
// In the infrastructure adapter (not the domain core):
impl From<sqlx::Error> for ProcurementError {
    fn from(e: sqlx::Error) -> Self {
        ProcurementError::StorageUnavailable {
            cause: e.to_string(),
        }
    }
}
```

This `From` impl is a proof transformation: it converts evidence of an infrastructure failure (the proposition sqlx::Error) into evidence of a domain failure (the proposition ProcurementError::StorageUnavailable). The domain core never sees the infrastructure proof — it operates only with its own error vocabulary. The `?` operator composes these proof transformations automatically: a function returning `Result<T, ProcurementError>` can use `?` on any operation that returns a `Result<_, E>` where `E: Into<ProcurementError>`.

### `Result` as Disjunction Elimination

A function returning `Result<T, ProcurementError>` presents the caller with a disjunction: either the operation succeeded (with evidence of type `T`) or it failed (with evidence of type `ProcurementError`). The caller must eliminate this disjunction — handle both cases — which is disjunction elimination in the Curry-Howard sense.

The `?` operator is shorthand for a specific elimination strategy: "if the left disjunct, continue; if the right disjunct, propagate." This is monadic bind specialised to the `Result` monad, and it composes: a chain of `?` operations propagates the first failure, threading the success path through each step.

## Algebraic Data Flows

The domain types defined so far — newtypes, state machines, error types — model the domain's *structure*. But a domain also has *flows*: commands enter the system, events leave it, queries retrieve projections. These flows are algebraic types — sum types that define the vocabulary of interaction.

### Commands as Sum Types

A command is a request to change the domain's state. The set of possible commands is closed (the domain defines exactly what changes are permitted):

```rust
# #[derive(Debug, Clone, PartialEq, Eq, Hash)]
# pub struct RequestId(u64);
# #[derive(Debug, Clone, PartialEq)]
# pub struct Amount(f64);
# #[derive(Debug, Clone, PartialEq, Eq)]
# pub struct Reason(String);
# #[derive(Debug, Clone, PartialEq, Eq)]
# pub struct ApproverId(String);
#[derive(Debug)]
pub enum ProcurementCommand {
    CreateDraft {
        id: RequestId,
        amount: Amount,
        reason: Reason,
    },
    Submit {
        id: RequestId,
    },
    BeginReview {
        id: RequestId,
    },
    Review {
        id: RequestId,
        approved: bool,
        approver: ApproverId,
    },
    Close {
        id: RequestId,
    },
}
```

### Events as Sum Types

An event records something that happened. Commands are imperative ("do this"); events are declarative ("this happened"). The event type mirrors the command type but in the past tense:

```rust
# #[derive(Debug, Clone, PartialEq, Eq, Hash)]
# pub struct RequestId(u64);
# #[derive(Debug, Clone, PartialEq)]
# pub struct Amount(f64);
# #[derive(Debug, Clone, PartialEq, Eq)]
# pub struct Reason(String);
# #[derive(Debug, Clone, PartialEq, Eq)]
# pub struct ApproverId(String);
#[derive(Debug, Clone)]
pub enum ProcurementEvent {
    DraftCreated {
        id: RequestId,
        amount: Amount,
        reason: Reason,
    },
    Submitted {
        id: RequestId,
    },
    ReviewStarted {
        id: RequestId,
    },
    Approved {
        id: RequestId,
        approver: ApproverId,
    },
    Rejected {
        id: RequestId,
        approver: ApproverId,
    },
    Closed {
        id: RequestId,
    },
}
```

These sum types are closed-world disjunctions — Chapter 3's insight applied to the flow layer. The compiler guarantees that every command handler covers every command, and every event listener covers every event. Adding a new command variant without updating all handlers is a compile-time error.

### The Vertical Seam

In a conventional layered architecture, the primary seams are horizontal — between layers (controller, service, repository). The domain's message types suggest a different organisation: the primary seams are *vertical*, around data flows.

A `ProcurementCommand` flows from the outside world (an HTTP handler, a CLI, a message queue) through an adapter that parses and validates it, into the domain core that processes it, producing `ProcurementEvent` values that flow outward to subscribers. The seam is the algebraic type boundary: the command enum defines what can enter; the event enum defines what can leave. The domain core's public API is a function from commands to events:

```rust,ignore
fn handle(
    cmd: ProcurementCommand,
    repo: &impl RequestRepository,
) -> Result<Vec<ProcurementEvent>, ProcurementError> {
    // ...
}
```

This signature is the domain core's complete contract: it accepts a procurement command, consults storage through a repository port, and produces either a list of events or a domain error. Everything the domain can do is expressed in this type. The types *are* the architecture.

## What We Have Built

The procurement domain core is now a collection of propositions proved by construction:

| Component | Proposition | Level |
|---|---|---|
| `Amount`, `RequestId`, `Reason` | Values satisfy domain invariants | 0 |
| `ProcurementRequest<State>` | State machine transitions are valid | 4 |
| `RequestRepository` trait | Persistence capability exists | 2 |
| `ProcurementError` | Failure modes are enumerated | 0 |
| `ProcurementCommand`, `ProcurementEvent` | Interaction vocabulary is closed | 0 |
| Impl blocks (Chapter 10) | Capabilities are proved for concrete types | 3 |

The domain core has no infrastructure imports. It does not know about databases, HTTP frameworks, or message queues. It defines propositions and requires proofs (via trait bounds). The proofs arrive from outside — from infrastructure adapters — and the compiler verifies that every proof obligation is discharged.

What this chapter has not addressed is *how to wire it*: how to organise the crate structure, how to connect infrastructure to the domain core, how to choose between generic dispatch and dynamic dispatch, how to test the assembly. These are architectural questions, and they are the subject of the next chapter.
