# Anti-Patterns

> **Read when:** you are reviewing a tree, a pull request, or a proposed target structure and need to name what is wrong, why it costs, and what to change.

Each entry is structural: a shape in the tree or in the import graph. Severity is the cost of leaving it — **high** means the dependency rule is already broken or the migration will not pay off; **medium** means the structure works today but will decay.

## Catalog

### A1. Ports mirror the ORM one-to-one — high

- **Symptom.** The outbound port repeats the ORM surface: `Find`, `FindAll`, `FindBy…`, `Save`, `Delete`, `Query`, `BulkUpdate`. Signatures carry `IQueryable`, `Specification`, `Pageable`, `Order`, or a raw SQL string. The port changes in the same commit as an index or a query.
- **Why it hurts.** The contract is defined by the database's capabilities, not the domain's needs. Every new screen adds a method; every fake must emulate query semantics; the port cannot be implemented by anything except a database.
- **Fix.** Declare one method per application need (`LoadForRenewal`, `Save`) with domain types. Move arbitrary filtering to a separate read/query port or a read model owned by the adapter. A port with more than seven methods is a table gateway.

### A2. One port per entity — high

- **Symptom.** `OrderPort`, `OrderLinePort`, `AddressPort` map one-to-one to tables. A single use case injects four of them to load one aggregate, and each saves its own children.
- **Why it hurts.** The aggregate boundary disappears. The application composes a transaction across four ports and nobody owns consistency. Every new table forces a new port, and partial writes become normal.
- **Fix.** One outbound port per aggregate root or per capability. Children load and save through the root. Read models get a separate query port.

### A3. A use case per getter — medium

- **Symptom.** `application/` contains a folder, request, response, handler, and port method for every read. Seventy percent of the tree is reads with no rule, and a screen change edits four files.
- **Why it hurts.** Ceremony without a decision to protect. The application layer becomes a SQL proxy, reviewers cannot find the operations that matter, and the tree grows linearly with the UI.
- **Fix.** Route reads through a query path — one query handler per screen, or a read-model adapter. Reserve use-case ceremony for operations with rules, transactions, or multiple outbound calls. If more than half the handlers have no branch and one port call, collapse them.

### A4. Anemic domain, orchestration living in adapters — high

- **Symptom.** Domain types are property bags with accessors. Controllers or services in adapter folders load, branch, compute, and persist. The same rule appears in two endpoints. Domain tests do not exist because there is nothing to test.
- **Why it hurts.** Rule changes touch every entry point, a second consumer reimplements the rule, and the domain cannot be tested without HTTP and a database. This is the failure hexagonal is meant to prevent, reproduced inside hex folders.
- **Fix.** Move decisions into domain operations or domain services. Adapters map input and delegate to one inbound port. Application orchestrates; domain decides. Add one test per rule at the domain level before moving code.

### A5. Framework annotations or ORM types in the domain — high

- **Symptom.** `@Entity`, `@Table`, `[Column]`, `[Key]`, `BaseEntity`, `DbContext`, `Repository<T>` inside the domain folder. The domain project references EF Core, Hibernate, Prisma, TypeORM, or Sequelize. Identifiers are typed as `ObjectId` or a library `UUID`.
- **Why it hurts.** The schema dictates the model, domain tests need the ORM, constructors are constrained by the mapper, and the dependency rule is violated at compile time. Any arch test either fails or is written to ignore the worst files.
- **Fix.** Put a persistence model and a mapper in the driven adapter. Remove annotations from the domain. Add a package ban to the arch test in the same PR.

### A6. Adapter calling another adapter — high

- **Symptom.** An HTTP controller injects a repository or gateway. A queue consumer calls `StripeClient` directly. An adapter imports another adapter's namespace, or its constructor takes two adapters.
- **Why it hurts.** The call bypasses the application, its transaction, and its authorization. The flow cannot be reused by the second entry point, and retries and error mapping land in the wrong layer.
- **Fix.** Both callers go through an inbound port → application → outbound port. If the interaction is genuinely transport-level (an auth filter calling an identity client), make it one adapter with one job.

### A7. A `common` folder that becomes a second domain — high

- **Symptom.** `common/` or `shared/` is imported by every module. It holds `OrderStatus`, `Money`, `Customer`, validation, and helpers. It changes every sprint, and a change to it retests everything.
- **Why it hurts.** It is a shared kernel with no governance. Every module depends on every change, real boundaries are hidden, and in a later service split it becomes a distributed monolith.
- **Fix.** Split into language-level primitives with no business meaning, an explicit shared kernel with one owner, a version, and a short member list, and per-module copies. Ban domain types from `common` with a module dependency test.

### A8. Hexagonal + microservices + CQRS + event sourcing adopted at once — high

- **Symptom.** A new repository with 40 projects, a broker, an event store, a command bus, and one aggregate. No end-to-end path in the first release. Every change touches six layers, and the team debates serialization before the first rule exists.
- **Why it hurts.** Four independent decisions are conflated, so failures cannot be attributed. The team learns the dependency rule while debugging infrastructure, and event sourcing adds versioning and projection concerns before the domain is understood.
- **Fix.** Sequence them. Enforce the rule in one process first, as a modular monolith. Extract a service only when a deployment boundary is justified. Add CQRS or event sourcing per module, after a proven read/write or audit need.

### A9. Mocking every port so tests verify nothing — high

- **Symptom.** Every handler test builds five mocks and asserts `verify(save).save(any())`. Mocks return the argument they received. No test asserts domain state or emitted events, and changing a rule breaks no test.
- **Why it hurts.** The suite tests call sequences, not behavior. Refactors break tests while defects pass, and ports exist mainly to be mocked — which is a design smell in itself.
- **Fix.** Domain tests with no mocks. Application tests against in-memory fakes that implement real semantics. One contract test per port, shared by all implementations. Assert outcomes and state, not interactions.

### A10. Domain returning transport DTOs — high

- **Symptom.** Domain methods return `OrderResponse` or `OrderDto` with JSON annotations, status codes, or links. The domain imports the API package. There is no mapper at the edge.
- **Why it hurts.** The wire format freezes the domain. A second entry point must adopt the first one's contract, a field rename becomes a domain change, and serialization concerns leak inward.
- **Fix.** Domain returns domain types. The adapter owns the DTO and the mapper. Contract tests pin the payload at the edge.

### A11. Transaction boundaries in controllers — high

- **Symptom.** `@Transactional` on a controller method, `using var tx` in a route handler, `session.begin()` in a resource. The use case calls two ports with no transaction, or a repository saves and commits internally.
- **Why it hurts.** One HTTP request becomes one transaction even when it contains several operations. Queue, CLI, and scheduler paths get no transaction at all, so partial writes appear only on non-HTTP entry points. Retries duplicate committed work.
- **Fix.** One unit of work per application operation, opened by the application layer or a decorator around the inbound port. Adapters expose the capability but never decide. Entry points never open transactions.

### A12. Barrel files that re-export adapters — high

- **Symptom.** `application/index.ts` re-exports `../infrastructure/prisma`. `ports/index` exports concrete gateways. A package `exports` map exposes the adapter entry point. The arch test passes because it scans only direct imports.
- **Why it hurts.** The import graph now says application → adapter. The arch test is gamed by importing the barrel, consumers couple to internals, and the boundary disappears at the next import.
- **Fix.** Each layer's barrel exports its contract only. Adapters have no cross-layer barrel. The arch test asserts on the resolved module graph, not on source lines. Prefer explicit imports over wildcard re-exports.

### A13. Port shaped around a vendor SDK — high

- **Symptom.** The outbound port's signatures contain `Stripe.Charge`, `AmazonS3Client`, `KafkaProducerRecord`, or a vendor error type. Swapping the vendor would change every caller. Two ports exist for the same capability with different vendors.
- **Why it hurts.** The abstraction hides the implementation's name but not its types. The domain compiles against the SDK, and a second provider duplicates the application path.
- **Fix.** Define the port in domain terms — amount, currency, result, failure. Keep the SDK entirely inside the adapter and translate errors at the boundary. One capability, one port, N adapters.

### A14. Domain events published as integration events — high

- **Symptom.** The domain raises `OrderPlaced` with a JSON schema and a version, publishes to a broker, and consumers import the producer's domain package. A domain refactor breaks downstream services.
- **Why it hurts.** The domain depends on messaging and on consumers it cannot see. Internal model changes become public contract changes, and the outbound port is bypassed.
- **Fix.** Domain events stay in-process. The application translates them into versioned integration contracts, and an outbound port publishes. Consumers own their copy of the contract.

### A15. Business logic in the composition root — medium

- **Symptom.** The DI module branches on business state to choose an implementation. `container.resolve()` appears inside a handler. Startup selects a strategy from a config value that is really a business rule.
- **Why it hurts.** The rule lives where no domain test reaches it, behavior differs by environment, and the object graph is only knowable at runtime.
- **Fix.** The composition root binds. Selection happens through an injected policy or a factory port. Config maps to typed options at startup and carries no business branching.

### A16. "Hexagonal" with no dependency check — high

- **Symptom.** Folders renamed, no arch test, no project separation. The domain imports the ORM in three files. "We review it" is the enforcement story, and the first deadline reverses an edge.
- **Why it hurts.** The rule decays within weeks, the claim cannot be verified, and renaming without enforcement bought pure cost. A new hire learns the structure from an import that should not compile.
- **Fix.** Add one failing arch test in CI before any move, then ratchet exclusions down. An exclusion list that grows is a failed migration.

### A17. Modules share tables instead of contracts — high

- **Symptom.** Two modules read and write the same tables. A shared `DbContext` or connection spans modules. A module joins another's tables in a query, and schema changes require coordinated releases.
- **Why it hurts.** The module boundary is cosmetic. The database is the real contract and it is unversioned, so `selective` adoption becomes impossible to enforce.
- **Fix.** One schema and one owner per module. Cross-module reads go through a published query contract or an ACL. No shared `DbContext`. If the tables cannot be split, keep the modules together.

## Triage

| Symptom in the tree | Pattern | Read next |
|---|---|---|
| Port has a method per screen or query | A1 | `ports-and-adapters.md` |
| One use case injects four ports for one aggregate | A2 | `ports-and-adapters.md` |
| Application tree is mostly getters | A3 | `naming-and-modes.md` §1 |
| Controller or service holds the rules | A4 | `concepts.md` §2, `layouts/standard.md` |
| Domain imports ORM or framework packages | A5 | `concepts.md` §5, `checklists.md` §1.1 |
| Adapter imports adapter | A6 | `concepts.md` §4 |
| `common` grows every sprint | A7 | `layouts/modular-monolith.md` |
| Four patterns shipped in one release | A8 | `fit-assessment.md` §5, `migration.md` |
| Tests only verify mocks | A9 | `checklists.md` §1.2, `migration.md` |
| Domain returns DTOs or JSON | A10 | `concepts.md` §7 |
| `@Transactional` on a controller | A11 | `ports-and-adapters.md` |
| Barrel exposes infrastructure | A12 | `concepts.md` §6 |
| Port mentions a vendor SDK in its signatures | A13 | `ports-and-adapters.md` |
| Domain publishes to a broker | A14 | `ports-and-adapters.md` |
| DI module branches on business state | A15 | `concepts.md` §2 |
| No arch test, or exclusions growing | A16 | `plan-template.md` §5, `checklists.md` §3 |
| Modules share tables | A17 | `layouts/modular-monolith.md` |
| Structure is heavy but nothing got easier | — | `fit-assessment.md` §2–4 |
