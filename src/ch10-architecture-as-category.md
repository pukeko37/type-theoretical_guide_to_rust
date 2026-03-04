# Architecture as Category

The previous chapter constructed a domain core: newtypes with smart constructors, a typestate lifecycle, trait ports, error types, and algebraic data flows. The result is a collection of propositions proved by construction — but it exists in isolation. A domain core that cannot be wired to a database, exposed through an API, or tested in isolation is an academic exercise. This chapter shows how the two-category model turns architectural patterns into categorical structure.

The onion architecture (equivalently, hexagonal architecture, ports-and-adapters) is the dominant pattern for structuring domain-centric systems. Its central rule — outer layers depend on inner layers, never the reverse — is typically justified by appeal to "clean separation of concerns." The two-category model provides a sharper justification: the dependency rule is isomorphic to the direction of the forgetful functor. The domain core lives high in 𝒯, defining propositions. Infrastructure lives close to 𝒱, providing concrete implementations. The functor F: 𝒯 → 𝒱 flows outward — from abstract to concrete, from proposition to proof to erased execution. The dependency rule is not a convention; it is a consequence of the categorical structure.

## The Ring Topology

The architecture has four rings, each with a category-theoretic role:

| Ring | Contents | Category-Theoretic Role | Levels |
|---|---|---|---|
| Domain Core | Newtypes, typestate, domain traits, error types, commands, events | Objects and morphisms of the domain sub-category | 0, 2, 4 |
| Application (Ports) | Service traits, use-case orchestration, generic function signatures | Functor interfaces — mappings between domain and infrastructure | 2 |
| Infrastructure (Adapters) | Database implementations, HTTP handlers, message queue adapters | Natural transformations — proofs that infrastructure satisfies port contracts | 3 |
| Composition Root | `main.rs`, concrete type wiring, configuration | Site where F: 𝒯 → 𝒱 is finally applied — all generics resolved | All collapse |

The rings have a strict dependency direction: each ring may depend on the rings inside it, but never on the rings outside it. Domain Core depends on nothing. Application depends on Domain Core. Infrastructure depends on Domain Core and Application. Composition Root depends on everything.

This is not arbitrary. The domain core defines propositions (Level 2 traits, Level 0 types, Level 4 typestate). The application layer *uses* these propositions as bounds on generic functions — it works with types that satisfy domain contracts without knowing which concrete types will appear. The infrastructure layer *proves* the propositions — it provides concrete impl blocks for the domain traits. The composition root *instantiates* everything — it resolves all generic parameters to concrete types, the moment where F collapses the type-level structure into runtime execution.

### The Application Layer as Port Definition

The application layer sits between the domain core and infrastructure. Its primary role is defining ports — trait-bounded function signatures that orchestrate domain logic without committing to infrastructure:

```rust
# use std::fmt;
# #[derive(Debug, Clone, PartialEq, Eq, Hash)]
# pub struct RequestId(u64);
# impl RequestId { pub fn new(v: u64) -> Self { RequestId(v) } }
# #[derive(Debug, Clone, PartialEq)]
# pub struct Amount(f64);
# impl Amount { pub fn new(v: f64) -> Result<Self, ProcurementError> {
#     if v > 0.0 { Ok(Amount(v)) } else { Err(ProcurementError::InvalidAmount) }
# }}
# #[derive(Debug, Clone, PartialEq, Eq)]
# pub struct Reason(String);
# impl Reason { pub fn new(t: &str) -> Result<Self, ProcurementError> {
#     if !t.is_empty() { Ok(Reason(t.to_string())) } else { Err(ProcurementError::EmptyReason) }
# }}
# #[derive(Debug)]
# pub enum ProcurementError {
#     InvalidAmount, EmptyReason, NotFound { id: String },
#     StorageUnavailable { cause: String },
# }
# impl fmt::Display for ProcurementError {
#     fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "error") }
# }
# impl std::error::Error for ProcurementError {}
# #[derive(Debug, Clone)]
# pub struct RequestSnapshot {
#     pub id: RequestId, pub amount: Amount, pub reason: Reason, pub status: String,
# }
# pub trait RequestRepository {
#     fn save(&self, snapshot: &RequestSnapshot) -> Result<(), ProcurementError>;
#     fn find(&self, id: &RequestId) -> Result<Option<RequestSnapshot>, ProcurementError>;
# }
# #[derive(Debug, Clone)]
# pub enum ProcurementEvent {
#     DraftCreated { id: RequestId, amount: Amount, reason: Reason },
# }
pub fn create_draft(
    repo: &impl RequestRepository,
    id: RequestId,
    amount: Amount,
    reason: Reason,
) -> Result<ProcurementEvent, ProcurementError> {
    let snapshot = RequestSnapshot {
        id: id.clone(),
        amount: amount.clone(),
        reason: reason.clone(),
        status: "draft".to_string(),
    };
    repo.save(&snapshot)?;
    Ok(ProcurementEvent::DraftCreated { id, amount, reason })
}
```

The function `create_draft` is generic over `impl RequestRepository`. It states a proposition: "given any proof of `RequestRepository`, I can create a draft and produce an event." The function does not know whether the repository is backed by PostgreSQL, an in-memory map, or a test fake. It operates at Level 2, working with the bound, not the concrete type.

## Crate Topology as Proof Structure

In Rust, the ring boundaries can be encoded as Cargo workspace crates. The Cargo dependency graph makes boundary violations mechanically detectable:

```text
procurement/
├── Cargo.toml              (workspace root)
├── procurement-domain/     (pure 𝒯: newtypes, traits, typestate, events)
│   ├── Cargo.toml          (no infrastructure dependencies)
│   └── src/lib.rs
├── procurement-app/        (Level 2: generic orchestration)
│   ├── Cargo.toml          (depends on: procurement-domain)
│   └── src/lib.rs
├── procurement-infra/      (Level 3: concrete impl blocks)
│   ├── Cargo.toml          (depends on: procurement-domain, procurement-app, sqlx, etc.)
│   └── src/lib.rs
└── procurement-bin/        (composition root)
    ├── Cargo.toml          (depends on: all of the above)
    └── src/main.rs
```

The critical rule: `procurement-domain/Cargo.toml` must have **no infrastructure dependencies**. If the domain crate imports `sqlx`, `tokio`, `reqwest`, or any infrastructure library, a ring boundary has been violated. The Cargo dependency graph makes this enforceable by inspection — and by CI policy if desired.

Each crate's `Cargo.toml` encodes the dependency direction:

```text
procurement-domain  →  (nothing)
procurement-app     →  procurement-domain
procurement-infra   →  procurement-domain, procurement-app, sqlx, tokio
procurement-bin     →  procurement-domain, procurement-app, procurement-infra, tokio
```

The arrows flow outward: from abstract to concrete, from proposition to proof. No arrow points inward. The domain core is a source in the dependency graph — it has in-degree zero.

### Visibility as Proof Scope

Rust's visibility modifiers restrict *where proofs are available*. A `pub(crate)` type or method is visible within its crate but not outside it — the proof of its existence is scoped to the crate boundary. This is useful for infrastructure internals:

```rust,ignore
// In procurement-infra/src/lib.rs:
pub struct PostgresRequestRepository {
    pool: sqlx::PgPool,  // pub struct, but pool is private
}

impl PostgresRequestRepository {
    pub(crate) fn new(pool: sqlx::PgPool) -> Self {
        PostgresRequestRepository { pool }
    }
}
```

The `pool` field is private — no code outside the struct can access the database connection directly. The constructor is `pub(crate)` — only the infrastructure crate can create a `PostgresRequestRepository`. The domain core cannot construct it (it does not even know the type exists). Only the composition root, which depends on the infrastructure crate, can obtain one — and it does so through the infrastructure crate's public API.

This is information hiding at the proof level. The domain core knows the proposition `RequestRepository`. The infrastructure crate knows the proof `impl RequestRepository for PostgresRequestRepository`. The composition root knows both and connects them. Each ring sees exactly the proofs it needs.

## The `dyn` versus Generic Decision

Chapter 8 showed that `dyn Trait` is a partial projection through the boundary: the concrete type is erased, but the trait's interface survives as a vtable. For architectural design, the choice between generic dispatch and `dyn Trait` is a choice about when proofs are resolved.

| Form | Level | What Survives F | Runtime Cost |
|---|---|---|---|
| `fn f<R: RequestRepository>(r: &R)` | Level 2 | Full monomorphisation | Zero |
| `fn f(r: &impl RequestRepository)` | Level 2 (sugar) | Same | Zero |
| `fn f(r: &dyn RequestRepository)` | Partial 𝒱 | Interface; identity erased | vtable indirection |
| `Box<dyn RequestRepository>` | 𝒱 | Only vtable pointer | Heap + indirection |

A generic bound (`R: RequestRepository`) is a compile-time proof obligation: the caller must provide a concrete type that satisfies the trait. The compiler resolves this at monomorphisation, producing specialised code for each concrete type. The proof is fully discharged in 𝒯 and completely erased in 𝒱.

A `dyn Trait` is a **deferred proof**: "I cannot prove at compile time which implementation will be provided." The vtable carries enough information to dispatch method calls at runtime, but the concrete type's identity is lost. This is appropriate when:

- **Runtime configuration**: the repository implementation is determined by a configuration file at startup.
- **Plugin systems**: adapters are loaded dynamically or selected at runtime.
- **Heterogeneous collections**: you need a `Vec<Box<dyn EventListener>>` holding different listener implementations.

It is **inappropriate** when the concrete type is known at the composition root and fixed for the lifetime of the program. In that case, generic dispatch gives you zero-cost abstraction — the proof is resolved at compile time and erased without a trace.

For the procurement system, the repository implementation is typically fixed: either PostgreSQL in production or an in-memory implementation in tests. Generic dispatch is appropriate:

```rust
# use std::fmt;
# #[derive(Debug, Clone, PartialEq, Eq, Hash)]
# pub struct RequestId(u64);
# #[derive(Debug, Clone)]
# pub struct RequestSnapshot { pub id: RequestId, pub status: String }
# #[derive(Debug)]
# pub struct ProcurementError;
# impl fmt::Display for ProcurementError {
#     fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "error") }
# }
# impl std::error::Error for ProcurementError {}
# pub trait RequestRepository {
#     fn save(&self, snapshot: &RequestSnapshot) -> Result<(), ProcurementError>;
#     fn find(&self, id: &RequestId) -> Result<Option<RequestSnapshot>, ProcurementError>;
# }
pub struct ProcurementService<R: RequestRepository> {
    repo: R,
}

impl<R: RequestRepository> ProcurementService<R> {
    pub fn new(repo: R) -> Self {
        ProcurementService { repo }
    }

    pub fn find_request(&self, id: &RequestId) -> Result<Option<RequestSnapshot>, ProcurementError> {
        self.repo.find(id)
    }
}
```

The service is generic over `R`. At the composition root, `R` is resolved to a concrete type. The monomorphiser produces `ProcurementService<PostgresRequestRepository>` in production and `ProcurementService<InMemoryRequestRepository>` in tests. No vtable, no heap allocation, no indirection. The architectural abstraction is zero-cost.

The `dyn` alternative becomes appropriate if the system must support switching repository implementations at runtime — for instance, a multi-tenant system where different tenants use different storage backends. In that case, the deferred proof is justified: the runtime must carry enough information (the vtable) to dispatch correctly.

## The Composition Root

The composition root is where F: 𝒯 → 𝒱 is finally applied. All generic parameters are resolved. All trait bounds are discharged with concrete types. All type-level structure collapses into runtime execution.

```rust,ignore
// procurement-bin/src/main.rs
use procurement_domain::*;
use procurement_app::ProcurementService;
use procurement_infra::PostgresRequestRepository;

#[tokio::main]
async fn main() {
    let pool = sqlx::PgPool::connect("postgres://...").await.unwrap();
    let repo = PostgresRequestRepository::new(pool);
    let service = ProcurementService::new(repo);

    // service is ProcurementService<PostgresRequestRepository>
    // All generics resolved. All proofs discharged. Pure V from here.
}
```

The composition root is intentionally the thickest, least abstract, most concrete part of the system. It names specific types: `PostgresRequestRepository`, not `impl RequestRepository`. It calls constructors with configuration values. It wires everything together.

This is not a design flaw — it is the design working correctly. The domain core is abstract (Level 2–4, pure 𝒯). The infrastructure layer provides proofs (Level 3, impl blocks). The composition root instantiates (Level 0, concrete types). The functor F acts at this point, collapsing the entire type-level architecture into a running program. After this point, no compile-time proof machinery is operating. The binary is pure 𝒱.

The composition root is also the narrowest part of the system in terms of reuse: it is specific to one deployment configuration. A test harness is an alternative composition root, wiring different concrete types (in-memory repositories, stub services) against the same generic domain and application layers.

## Testing as Alternative Proof

A test double — a mock, stub, or fake — is an alternative proof of a port's proposition. The trait `RequestRepository` states a proposition: "there exists a mechanism that can persist and retrieve procurement requests." A production implementation (`PostgresRequestRepository`, introduced in the composition root at §10.4) proves this proposition with a database. A test implementation proves the same proposition with a `HashMap`:

```rust
use std::collections::HashMap;
use std::cell::RefCell;
use std::fmt;

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct RequestId(u64);
impl RequestId {
    pub fn new(id: u64) -> Self { RequestId(id) }
}

#[derive(Debug, Clone)]
pub struct RequestSnapshot {
    pub id: RequestId,
    pub status: String,
}

#[derive(Debug)]
pub struct ProcurementError;
impl fmt::Display for ProcurementError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "error") }
}
impl std::error::Error for ProcurementError {}

pub trait RequestRepository {
    fn save(&self, snapshot: &RequestSnapshot) -> Result<(), ProcurementError>;
    fn find(&self, id: &RequestId) -> Result<Option<RequestSnapshot>, ProcurementError>;
}

// --- The test proof: InMemoryRequestRepository ---

pub struct InMemoryRequestRepository {
    store: RefCell<HashMap<u64, RequestSnapshot>>,
}

impl InMemoryRequestRepository {
    pub fn new() -> Self {
        InMemoryRequestRepository {
            store: RefCell::new(HashMap::new()),
        }
    }
}

impl RequestRepository for InMemoryRequestRepository {
    fn save(&self, snapshot: &RequestSnapshot) -> Result<(), ProcurementError> {
        self.store.borrow_mut().insert(snapshot.id.0, snapshot.clone());
        Ok(())
    }

    fn find(&self, id: &RequestId) -> Result<Option<RequestSnapshot>, ProcurementError> {
        Ok(self.store.borrow().get(&id.0).cloned())
    }
}

// --- The service, generic over any proof of RequestRepository ---

pub struct ProcurementService<R: RequestRepository> {
    repo: R,
}

impl<R: RequestRepository> ProcurementService<R> {
    pub fn new(repo: R) -> Self {
        ProcurementService { repo }
    }

    pub fn find_request(
        &self,
        id: &RequestId,
    ) -> Result<Option<RequestSnapshot>, ProcurementError> {
        self.repo.find(id)
    }

    pub fn create_draft(
        &self,
        id: RequestId,
        reason: &str,
    ) -> Result<(), ProcurementError> {
        let snapshot = RequestSnapshot {
            id,
            status: "draft".to_string(),
        };
        self.repo.save(&snapshot)
    }
}

// --- A test using the alternative proof ---

fn test_create_and_retrieve() {
    let repo = InMemoryRequestRepository::new();
    let service = ProcurementService::new(repo);

    let id = RequestId::new(42);
    service.create_draft(id.clone(), "Office supplies").unwrap();

    let found = service.find_request(&id).unwrap();
    assert!(found.is_some());
    assert_eq!(found.unwrap().status, "draft");
}

fn main() {
    test_create_and_retrieve();
}
```

The test wires `ProcurementService` with `InMemoryRequestRepository` — the alternative proof. In production, the same service is wired with `PostgresRequestRepository` (§10.4). Both implementations satisfy `RequestRepository`; the compiler verifies each is a valid proof of the same proposition. The domain core and application layer cannot distinguish between them — they operate generically over `R: RequestRepository`, and the type system guarantees that any valid proof will work.

This is the practical content of "testing as alternative proof." The test does not verify the service by inspecting its internals or mocking its dependencies. It provides a *different proof* of the same port contract and runs the service against it. If the service works with one valid proof, and the production implementation is also a valid proof (verified by the compiler), the architectural guarantee holds.

### Fakes versus Mocks

The `InMemoryRequestRepository` is a **fake**: a genuine alternative implementation with real behaviour. It stores data, retrieves data, and maintains the save-then-find contract. It is a complete proof of the `RequestRepository` proposition.

A mock is different. A mock records interactions and asserts on them — "did the code call `save` with these arguments?" A mock is a *partial proof*, valid only under specific interaction assumptions. It does not prove that `RequestRepository` can persist and retrieve data; it proves that certain method calls occurred in a certain order.

In type-theoretic terms: a fake is a valid proof that lives in a different model (in-memory rather than PostgreSQL, but still a model of persistence). A mock is a proof about *interaction patterns*, not about the proposition itself. Both are useful; the distinction matters for understanding what your tests actually prove.

### Parametricity and Property Testing

Chapter 4 showed that parametricity — Reynolds' abstraction theorem — constrains what a generic function can do. A function `fn f<T>(x: T) -> T` *must* be the identity. A function `fn sort<T: Ord>(xs: Vec<T>) -> Vec<T>` must produce a permutation of the input (it cannot fabricate new `T` values).

Parametricity is also what justifies test doubles at a formal level. `ProcurementService<R: RequestRepository>` is parametric over `R`. The service cannot inspect which concrete repository it holds — it can only call the methods that `RequestRepository` exposes. This means: if two implementations of `RequestRepository` behave identically on the same method calls, the service *must* behave identically with both. The parametricity theorem guarantees this. A test that passes with `InMemoryRequestRepository` proves a property of the service that holds for *any* valid implementation, including `PostgresRequestRepository` — provided the two implementations agree on the observable behaviour of the trait methods.

This insight generates property tests. Consider any function with a constrained generic signature:

```rust
fn sort_and_dedup<T: Ord + Clone>(mut xs: Vec<T>) -> Vec<T> {
    xs.sort();
    xs.dedup();
    xs
}
```

Parametricity tells us, before looking at the implementation:
- The output is a sub-sequence of the input (the function cannot fabricate new `T` values — it has no constructor for `T`).
- The output is sorted (the `Ord` bound is the only ordering available).
- The output contains no duplicates.

Each constraint derived from the type signature becomes a testable property:

```rust
# fn sort_and_dedup<T: Ord + Clone>(mut xs: Vec<T>) -> Vec<T> {
#     xs.sort();
#     xs.dedup();
#     xs
# }
fn test_output_is_sorted() {
    let input = vec![3, 1, 4, 1, 5, 9, 2, 6];
    let output = sort_and_dedup(input);
    for pair in output.windows(2) {
        assert!(pair[0] <= pair[1]);
    }
}

fn test_output_is_subset_of_input() {
    let input = vec![3, 1, 4, 1, 5, 9, 2, 6];
    let input_clone = input.clone();
    let output = sort_and_dedup(input);
    for item in &output {
        assert!(input_clone.contains(item));
    }
}

fn test_no_duplicates() {
    let input = vec![3, 1, 4, 1, 5, 9, 2, 6];
    let output = sort_and_dedup(input);
    let len_before = output.len();
    let mut deduped = output.clone();
    deduped.dedup();
    assert_eq!(len_before, deduped.len());
}
#
# fn main() {
#     test_output_is_sorted();
#     test_output_is_subset_of_input();
#     test_no_duplicates();
# }
```

The same principle applies to the procurement service. The signature `fn find_request(&self, id: &RequestId) -> Result<Option<RequestSnapshot>, ProcurementError>` on `ProcurementService<R: RequestRepository>` tells us: the service can only return what the repository provides or a `ProcurementError`. It cannot fabricate request data. A property test can verify this — save a snapshot, retrieve it, confirm the result matches — and parametricity guarantees the property holds regardless of which `R` is supplied.

## The Anti-Pattern Catalogue

The five levels and the ring topology provide a diagnostic vocabulary for common architectural failures. Each anti-pattern below is a violation of a specific type-theoretic principle; each fix lifts a proposition from 𝒱 to 𝒯.

### Stringly-Typed Domain

**Symptom:** `status: String`, `id: u64`, `currency: &str` — domain values represented as primitive types.

**Diagnosis:** Propositions not encoded. The distinction between a request ID and an invoice ID, or between "draft" and "submitted", exists only in programmer intent — in 𝒱 rather than 𝒯. The compiler cannot prevent mixing a request ID with an invoice ID because they are both `u64`.

**Fix:** Newtypes with smart constructors (Level 0, Chapter 9 §9.2). `RequestId(Uuid)`, `Amount(f64)`, `Reason(String)` — each carries a domain invariant proved at construction time.

### Infrastructure Leaking into Domain

**Symptom:** Domain types import `sqlx::Row`, `serde_json::Value`, or `reqwest::Error`. Repository methods return database-specific types.

**Diagnosis:** Infrastructure proofs crossing the ring boundary. The domain core, which should contain only propositions, has become entangled with specific proof implementations. This couples the domain to infrastructure and prevents independent testing.

**Fix:** Adapter mapping at the ring boundary (Chapter 10 §10.1). Domain types define their own vocabulary (`RequestSnapshot`, `ProcurementError`). Infrastructure adapters convert between infrastructure types and domain types. The `From` trait serves as the proof transformation mechanism.

### Incoherent Service Traits

**Symptom:** A trait with fifteen methods spanning multiple concerns — persistence, validation, notification, logging. Impossible to implement a test double without implementing every method.

**Diagnosis:** Incoherent proposition. The trait bundles multiple unrelated capabilities into a single predicate. In the constraint lattice (Chapter 5), this is a conjunction so large that no useful type can satisfy it without being a god object.

**Fix:** Trait segregation into single-capability ports. One trait per domain capability: `RequestRepository` for persistence, `NotificationSender` for notifications. Each trait is a focused proposition that can be independently proved and independently tested.

### Gratuitous Cloning at Boundaries

**Symptom:** `.clone()` on every value crossing a layer boundary. Functions take owned values when they only need references.

**Diagnosis:** Port designed for interface-orientation (method calls with owned parameters) rather than data-orientation (borrowing and returning). The `clone` is the cost of misaligned ownership at the boundary.

**Fix:** Design ports with Rust's ownership model in mind. Accept references when the port does not need ownership. Return owned values only when ownership transfer is the intent. Consider accepting `impl Into<String>` or `&str` rather than `String` when the port may or may not need to allocate.

### `unwrap()` in Domain Logic

**Symptom:** Domain functions calling `.unwrap()` on `Result` or `Option` values, converting type-level error handling into runtime panics.

**Diagnosis:** Proof obligations discharged in 𝒱 instead of 𝒯. The `Result` type encodes a disjunction (success ∨ failure), and `unwrap` is an unsound disjunction elimination — it asserts "the right disjunct cannot occur" without proof. A panic is the runtime consequence of an unproved assertion.

**Fix:** Propagate `Result` with `?` (Chapter 9 §9.5). Define domain error types that name every failure mode. The `?` operator is sound disjunction elimination — it handles the right disjunct by propagating it. The caller, not the callee, decides how to handle failure.

### `Arc<dyn Trait>` by Default

**Symptom:** Every port is `Arc<dyn Trait>`, every service holds `Arc<dyn Repository>`, heap allocation and vtable indirection throughout the hot path.

**Diagnosis:** Deferred proof where static dispatch was possible. `dyn Trait` is appropriate when the concrete type is genuinely unknown at compile time. When the concrete type is fixed at the composition root and never changes, `dyn` pays the cost of runtime dispatch for no benefit.

**Fix:** Generic bounds (Chapter 10 §10.3). Use `<R: RequestRepository>` on the service struct. The composition root resolves `R` to the concrete type. Monomorphisation erases the generic parameter, producing zero-cost dispatch. Reserve `dyn Trait` for genuinely dynamic cases — runtime configuration, plugin systems, heterogeneous collections.

### Summary Table

| Anti-Pattern | Type-Theoretic Diagnosis | Fix | Reference |
|---|---|---|---|
| Stringly-typed domain | Propositions unencoded in 𝒯 | Newtypes + smart constructors | §9.2 |
| Infrastructure leaking into domain | Proofs crossing ring boundary | Adapter mapping | §10.1 |
| Incoherent service trait | Incoherent proposition (over-large conjunction) | Trait segregation | §9.3 |
| Gratuitous cloning | Ownership misaligned at boundary | Borrow-aware port design | §10.3 |
| `unwrap()` in domain logic | Unsound disjunction elimination | `Result` propagation | §9.5 |
| `Arc<dyn Trait>` by default | Deferred proof without justification | Generic bounds | §10.3 |

## The Title Discharged

The book's title makes a claim: types are propositions. The preceding chapters derived this claim theoretically — through the Curry-Howard correspondence, through the five levels of Rust's type hierarchy, through the forgetful functor and the boundary. This chapter and the last have discharged the claim practically.

A well-architected Rust system is a proof system. The domain core defines propositions. Infrastructure provides proofs. The composition root applies the forgetful functor — collapsing 𝒯 into 𝒱, erasing the type-level structure, producing a running binary that carries no trace of the proofs that validated it. Tests are alternative proofs. Anti-patterns are proof system failures.

This is not metaphor. When the compiler rejects a programme because a trait bound is not satisfied, it has found a gap in the proof — a proposition that no impl block discharges. When a typestate machine prevents an invalid state transition, it has enforced a theorem — a structural property that no runtime check could match for reliability. When a newtype with a smart constructor guarantees that an amount is positive, it has proved an invariant — once, at construction time, and never again.

Rust's type system can prove a great deal about a domain at compile time, but it cannot express everything. The propositions you cannot yet prove — full refinement types, dependent types, effect annotations — map precisely onto the territories the next chapter explores.
