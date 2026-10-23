# Ports and Adapters

> **Read when:** you are designing or reviewing the contracts that cross the boundary — port granularity, boundary models, error and transaction policy, or adapter tests.

Terminology is defined in `concepts.md` §3–§7. This file assumes the port location (`domain/ports/` or `application/ports/`) is already recorded per `naming-and-modes.md` and is not revisited.

## 1. Two rules and a port record

1. The inside owns the contract. The adapter conforms; the port does not bend to the technology.
2. A port names an operation or a capability. It never names a table, a URL, a framework, or a transport.

For each port, record:

```
Port record
- Kind:            inbound | outbound
- Owner layer:     domain | application
- Operations:      <one line each>
- Implementations: <handlers / adapters / fakes>
- Consumers:       <adapters / use cases>
- Contract test:   <path or "none">
```

A port without a test that pins its contract is an interface, not a contract.

## 2. Inbound port design

### Granularity

| Shape | Choose when | Cost |
|---|---|---|
| One port per use case | Operations differ in input, outcome, authorization, or rate limit, or change independently | Many small contracts; shared validation needs a shared application service, not a shared port |
| One port per module | Operations are cohesive, share an input/output family, and have one caller group (for example an admin surface) | Fat interface: every adapter depends on operations it never calls; one change recompiles all callers; selective wrapping is impossible |
| Hybrid | Reads through a module query surface, writes per operation | Two conventions in one module; record which is which |

Decide with three questions:

- Do the operations have different consumers or authorization rules? Split.
- Must two calls happen inside one transaction to preserve an invariant? Merge — they are one use case.
- Do they always change together and share a caller? Merge.

Do not split below the transaction boundary. Two ports that must be called in order to keep an aggregate consistent are one operation with a misleading interface.

### What belongs in the contract

- The intent, named as an operation the business recognizes.
- An application-owned input: domain-meaningful values, identifiers, and an explicit actor when authorization is decided inside.
- An explicit outcome. Pick one convention — result type, sealed outcome, or typed error — and use it for every inbound port.
- An idempotency key when the caller can retry the operation.
- Paging for queries, expressed as an application-owned page request, not the transport's query string.

### What must not leak

| Leak | Why it breaks the boundary |
|---|---|
| Framework types (request, response, principal, session, ORM context) | The application becomes callable only from that framework; consumers and tests cannot use it |
| ORM entities or persistence rows | The domain ends up shaped by the schema; lazy loading and change tracking leak into use cases |
| Transport DTOs (JSON shape, multipart, protobuf) | Serialization changes become application changes |
| HTTP status, headers, media types, route values | Transport policy moves inside; every new adapter must fake HTTP semantics |
| SQL, query builders, criteria objects tied to a table | The port becomes a second ORM and cannot be faked |
| Framework identity or claims | Authorization tests need an HTTP context; background jobs cannot call the use case |

Leak test: describe the port without naming a framework, a table, a URL, or a status code. If you cannot, something from the table above is in the signature.

## 3. Outbound port design

Name the capability the application needs, not the place the data lives.

| Kind | Question it answers | Shape | Not |
|---|---|---|---|
| Repository | Which aggregate do I load or store? | Collection-like, one per aggregate root; returns domain objects | Per-table CRUD, joins, reporting, query builders |
| Gateway | What does another system or context tell me, in my terms? | Translated external model; anti-corruption at the adapter | Raw SDK passthrough, transport client |
| Provider | Can you perform this technical capability? | Narrow: clock, IDs, email, payments, files, tax | A static helper with the same name |

### Why a port per table is a smell

A table is a storage decision. A port is a domain need. When the port mirrors tables:

- Every schema change becomes a contract change: adding an index, splitting a table, or denormalizing forces edits to the domain-facing interface.
- Methods become row operations (`insert`, `update column`, `select by foreign key`), so invariants move to the caller and the domain goes anemic.
- Cross-table invariants cannot be expressed; the caller sequences writes and the transaction leaks upward.
- Fakes become row-level mocks, so tests assert storage choreography instead of behavior.
- Reporting and joins creep in until the port is a second ORM with worse typing.

| Method | Verdict | Reason |
|---|---|---|
| `orders.save(order)` | Good | Aggregate-level capability |
| `orders.findPendingFor(customer)` | Good | Named domain question; no table or column |
| `orders.findByStatusAndCreatedBetween(from, to)` | Suspect | Query-builder shape unless it is a named domain concept |
| `orderRows.updateStatus(id, status)` | Bad | Row and column exposed |
| `db.query(sql, params)` | Bad | Storage escape hatch; unfakeable and unenforceable |

### Repository rules

- One per aggregate root. Two tables behind one aggregate is still one repository; one table read by two aggregates is two access paths, split by need.
- Return domain objects or an application read model. Never an ORM entity, a lazy proxy, or a tracked object.
- Save the aggregate as a unit. No commit or save-changes inside the repository; the transaction belongs to the application (§6).
- Query methods answer domain questions. No `IQueryable`, no expression trees built from persistence types, no eager-loading graph in the signature.

### Gateway and provider rules

- The adapter maps the external model into domain terms. The SDK type never appears in the port signature.
- Timeouts, retries, circuit breaking, and authentication are adapter policy. The port states the semantics the application relies on: idempotent or not, ordered or not, strongly consistent or eventual.
- A provider has one capability. A `PlatformServices` port holding clock, email, files, and SMS is a static class with a constructor.

## 4. Boundary models and mapping

| Model | Owner | Lives in | Crosses | Mapped by |
|---|---|---|---|---|
| Transport DTO | Inbound adapter | Inbound adapter | HTTP, queue, or CLI boundary | Adapter ↔ application input/output |
| Application input/output | Application | Application layer | Inbound port | Adapter maps in; handler returns it |
| Domain model | Domain | Domain | Port boundaries; never transport | Domain factory for new instances; adapters translate at their boundary |
| Persistence model | Driven adapter | Persistence adapter | Database boundary | Adapter: domain ↔ row |
| External model | Driven adapter | Gateway adapter | External API boundary | Adapter: external → domain |
| Read model | Application, query side | Application layer | Query port | Adapter projects directly; must not be an ORM entity |

One line per boundary:

```
transport DTO -> [inbound adapter] -> application input -> [handler] -> domain -> [outbound adapter] -> row / external DTO
```

Rules:

- Mapping happens only in the adapter that owns the boundary. A mapper inside a use case means a transport or persistence model leaked.
- The domain has one creation path (constructor or factory) that enforces invariants. Adapters call it; they do not assemble invalid objects and validate later.
- Inbound maps once. Outbound maps in both directions when the shapes differ: domain → row and row → domain, or domain ↔ external.
- Read-only projections may skip the domain. The query port returns an application-owned read model produced by the adapter. This is the one legitimate shortcut, and it must not return a tracked or ORM type.

Cost:

- Two to four types per concept, plus mappers, plus the tests that keep them aligned.
- Drift appears when a field is added to one model and forgotten in another. Contract tests on mapped fields catch it; review does not.
- Collapsing models is legitimate for a CRUD module with no rules (`selective`). It is not hexagonal if the domain type is the ORM entity — that is a layered module with a port-shaped name.

## 5. Error translation

Three hops, one owner each: domain → application → transport.

| Condition | Domain error | Application outcome | Transport |
|---|---|---|---|
| Invariant or input rule violated | `InvalidRule`, with violation detail | `Rejected` | 400, or 422 when the payload parsed but the rule failed |
| Aggregate or record absent | `NotFound` | `NotFound` | 404 |
| Unique constraint or duplicate command | `AlreadyExists` | `Conflict` | 409 |
| Stale version or optimistic lock | `ConcurrencyConflict` | `Conflict`, retryable | 409 or 412 |
| Actor lacks permission | `NotPermitted` | `Forbidden` | 403 |
| No authenticated actor | — (adapter) | `Unauthenticated` | 401 |
| Business state forbids the operation | `StateConflict` | `Rejected` or `Conflict` | 409 or 422; choose one per operation and document it |
| Dependency unavailable or timed out | — (adapter translates) | `Unavailable`, retryable | 503; 504 only when a gateway timed out and the operation is safe to retry |
| Upstream rate limit | — (adapter translates) | `Throttled` | 429 |
| Unhandled exception | — | `Unexpected` | 500, with a correlation id logged |

Rules:

- Domain errors carry no status codes, no HTTP vocabulary, and no serialization attributes. They describe the rule that failed.
- The application maps domain errors to outcomes and decides retryability and rollback. It does not know the transport.
- One inbound adapter component owns the outcome → status mapping. Per-endpoint try/catch produces inconsistent codes across an API.
- An unknown exception maps to 500. It never becomes 400, and never a 200 with an error body.
- Outbound adapters translate technology failures (SQL exception, SDK error, timeout) into port-level errors. The application never catches a driver exception.
- The error envelope is a transport concern, owned by the inbound adapter, identical across endpoints.

## 6. Transaction boundaries

The application layer owns the transaction because it owns the use case. One use case is one unit of work unless it explicitly coordinates a process manager.

| Layer | May open or commit? | Responsibility | Failure if it does |
|---|---|---|---|
| Inbound adapter | No | Parse, authenticate, call the port, map the outcome | One transaction spans several use cases; partial commits; an HTTP retry replays half an operation |
| Application | Yes | Start before the first persistence call, commit after the last; define rollback and outbox | — |
| Domain | No | Enforce invariants in memory | Cannot roll back; I/O in the domain breaks isolation |
| Outbound adapter — persistence | No; joins the ambient transaction | Translate and execute; flush when the unit of work asks | Repository-level commits hide partial writes and break composition |
| Outbound adapter — gateway or provider | No | Call outside the transaction, or enqueue through the outbox | A slow external call holds a transaction open; a failure after commit leaves inconsistent state |
| Composition root | No | Wire the unit of work and the adapters | Configuration starts deciding consistency |

Rules:

- Repositories join the ambient transaction and never commit as a side effect. If the ORM needs an explicit flush, the unit-of-work adapter performs it at the application boundary.
- A controller must not begin a transaction "to save a round trip". That couples two use cases and makes the second one non-reusable.
- A use case spanning two systems uses an outbox, not a distributed transaction. Publish after commit; the consumer is idempotent.
- Do not hold a transaction across a slow external call. Model the operation as state transitions with an idempotency key.
- Read-only queries use a read-only transaction or none. Do not reuse the write unit of work.
- Framework mapping: `@Transactional` on the application handler, not the controller or repository; `transaction.atomic` around the use case, not the view; save-changes in the unit-of-work adapter, not in every repository method. Where the application must stay framework-free, a transaction decorator wired at the composition root implements the same boundary.
- Verify with a fake unit of work: one commit on success; zero commits and one rollback on a domain error or an outbound failure.

## 7. Cross-cutting capabilities as ports

A static is a dependency that does not appear in the signature. It cannot be replaced per test, cannot vary per tenant or request, and hides that the domain touches the outside world.

| Capability | Port returns | Why not a static | Test consequence |
|---|---|---|---|
| Clock | Current instant in UTC, plus an explicit zone policy where the business needs one | Time-dependent rules become untestable and fail at month boundaries | Freeze time; assert expiry, scheduling, and cutoff rules deterministically |
| IDs | A new identifier | Static UUIDs make assertions non-deterministic and block tenant-scoped or sequential strategies | Inject fixed IDs; assert identity and deduplication |
| Current user | Application-owned actor: id, tenant, permissions | Thread-locals and HTTP principals fail in consumers, jobs, and tests; authorization becomes implicit | Run the same use case as any actor without an HTTP context |
| Outbox | Enqueue a message within the current transaction | A static publisher sends before commit or outside it; a crash loses the message or publishes a phantom | Assert the message is recorded atomically with the state change |
| Unit of work | Commit/rollback boundary, or `run(work)` | Framework transactions leak into adapters; nested commits go unnoticed | Assert one commit per use case and rollback on failure |

Rules:

- **Clock:** inject it into the application layer and pass the resulting instant into domain operations. Domain rules that read time directly are untestable and violate the dependency table.
- **Current user:** authentication is an inbound adapter concern. Authorization is an application decision; domain policies receive the actor value when the rule is business-owned. Do not pass a framework principal through the port.
- **Outbox:** the port writes to an outbox table in the same transaction; a separate relay publishes and marks sent. The relay is an adapter or process, not part of the use case.
- **Unit of work:** expose it only when the application must control commit explicitly. Otherwise a transaction decorator around the handler is the implementation and the application never sees it.
- **Feature flags are not an application port.** Flag evaluation belongs to the composition root or the inbound adapter; the application receives the selected behavior.
- Keep each capability narrow. A `PlatformServices` port with ten methods is a static class with extra steps.

## 8. Testing

| Target | Double | Assert | Do not assert |
|---|---|---|---|
| Application / use case | Hand-written fakes for outbound ports | Outcome, domain state, outbox entries, one commit or rollback | SQL, HTTP, framework annotations, call order across ports |
| Inbound adapter | Real handler plus fake outbound ports, or a transport-level test | Status mapping, payload shape, authentication, validation → error envelope | Domain rules, persistence |
| Persistence adapter | Real database (container), production engine version | Roundtrip, aggregate mapping, query semantics, optimistic concurrency, transaction join | Domain decisions |
| Gateway or provider adapter | Stub server or recorded responses | Request shape, authentication, timeout and retry, error translation, idempotency | Domain rules |
| Contract suite | Same tests parameterized over fake and real adapter | Identical observable semantics per port operation | Implementation details |

Fakes vs mocks:

- Prefer a fake for stateful outbound ports: in-memory repository, fixed clock, recording notifier. Tests read as behavior, not as choreography.
- Use a mock only for an interaction with no observable state. Even then, a recording fake is usually cheaper to maintain and asserts the same thing.
- Never mock the inbound port you are testing. Call the real handler.
- A mock that asserts cross-port call order is testing the implementation. If order matters, make it an outcome, not an assertion on collaborators.

Contract tests:

- One suite per port, run against every implementation, including the fake.
- Cover success, absence, duplicate or conflict, error translation, promised ordering, idempotency, and transaction visibility.
- Run the real adapter against the same engine version as production. A documented difference is acceptable; an undocumented one is a production bug.
- If the fake cannot pass the suite, the port contract is underspecified. Fix the contract, not the fake.

What each boundary must prove:

- **Inbound:** a transport request becomes exactly one application call, and every outcome maps to one documented response.
- **Application:** the use case is the transaction, and it never touches a concrete adapter.
- **Outbound:** the adapter satisfies the contract, including its failure semantics, with no domain logic.

## 9. Evolution

### Adding a method vs creating a port

| Condition | Action |
|---|---|
| Same capability, all implementations can satisfy it, no new failure semantics | Add the method |
| New capability with different consumers | New port |
| Only some adapters can implement it | New port; do not add a `NotSupported` method |
| Different consistency, ordering, or retry semantics | New port; one contract cannot promise two behaviors |
| The port already has many methods and mixed callers | Split by caller before adding anything |
| A fake would need to throw for the new method | Wrong port |

A default method that throws `NotSupported` is a runtime failure waiting for the one adapter that forgot. Split the interface instead.

### Versioning an inbound contract

- Additive first: a new optional field, a new operation, a new outcome variant. Ensure unknown fields are ignored by every consumer before adding.
- Never change the meaning of an existing field, status code, or default silently. Add a new operation or a new media type instead.
- Version at the adapter, not in the domain. `v1` and `v2` are transport concerns; the application keeps one operation unless the behavior genuinely differs.
- Deprecate with telemetry: log usage per client and version, announce the window, remove only after usage reaches zero.
- Internal inbound ports, called only by your own adapters, need no versioning: change the contract and all adapters in one commit.
- Published message contracts follow the same rules as public APIs: additive, optional, never renumber enum values.

### Outbound port evolution

- Outbound ports are internal. Change them freely, but update every adapter and fake in the same commit; a stale adapter should be a compile error, not a runtime surprise.
- Do not add a method with a default implementation to avoid updating adapters. That hides a capability split.
- When two adapters interpret the same method differently, the contract is underspecified. Write the contract test first, then fix the adapters.

## 10. Design review checklist

- [ ] The contract is readable without naming a framework, table, URL, or status code.
- [ ] Every port has one reason to exist, stated as an operation or a capability.
- [ ] Inbound granularity matches the transaction boundary; no invariant needs two ports called in order.
- [ ] No ORM entity, transport DTO, SDK type, or framework type appears in a port signature.
- [ ] Repository ports are per aggregate root, not per table; no query builder or eager-loading graph in the contract.
- [ ] The domain model has no persistence or serialization annotations.
- [ ] Mapping happens only in adapters; no use case maps transport or persistence types.
- [ ] Domain errors carry no transport codes; one adapter component maps outcomes to responses.
- [ ] The application owns commit and rollback; no controller transaction, no repository commit.
- [ ] Clock, IDs, current user, outbox, and unit of work are injected ports, not statics.
- [ ] Feature flags are evaluated at the edge, not inside the application.
- [ ] Every outbound port has a fake that passes the same contract suite as the real adapter.
- [ ] No `NotSupported` methods; capabilities only some adapters provide have their own port.
- [ ] Published inbound changes are additive or versioned, with usage telemetry before removal.
- [ ] The dependency test covers the port package and fails if an adapter type is imported, using the stack-specific tool recorded in the plan.
