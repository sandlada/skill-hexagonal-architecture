# Concepts

> **Read when:** you need the canonical meaning of a term, or you are deciding which layer a file belongs to.

This file is the **only** place terminology is defined. Other files reference it instead of redefining.

## 1. Hexagonal architecture is a dependency rule, not a folder layout

The rule: **source-code dependencies point inward, toward the domain.** Everything else — folder names, project count, module boundaries — is an expression of that rule, and different projects express it differently.

Consequences:

- A project can satisfy the rule with four folders or forty. Structure follows constraints, not fashion.
- Copying a layout without enforcing direction produces the ceremony without the benefit.
- Therefore every recommendation in this library pairs a structure with a way to verify the direction.

## 2. The model

Four concepts plus one wiring point:

| Concept | Contains | May depend on | Must never contain |
|---|---|---|---|
| **Domain** | Entities, value objects, aggregates, domain services, policies, domain events, domain errors | Itself only | Framework, ORM, HTTP, SDK, serialization, clock, filesystem |
| **Application** | Use cases, orchestration, transaction boundaries, authorization decisions | Domain, outbound port interfaces | Framework wiring, SQL, HTTP clients |
| **Ports** | Contracts crossing the boundary | Domain types (never adapter types) | Implementations |
| **Adapters** | Technology translation: HTTP, DB, queue, external API, CLI | Port contracts, their own technology | Other adapters |
| **Composition root** | DI registration, startup, config binding | Everything | Business rules |

## 3. Ports

A port is a **contract owned by the inside**. Its shape is dictated by the domain's needs, not by the technology that will satisfy it.

**Inbound (driving) port** — the outside calls in.

- Declares what the application can do: a use case, a command handler, a query handler.
- Implemented by the application layer.
- Callers: HTTP controllers, CLI commands, message consumers, schedulers, tests.

**Outbound (driven) port** — the application calls out.

- Declares a capability the application needs: persist an aggregate, send a notification, read the clock, call a payment provider.
- Implemented by an adapter.
- Callers: the application layer.

The asymmetry is the point: inbound ports are implemented inside and called outside; outbound ports are declared inside and implemented outside.

### Where port declarations live

Two conventions are in common use:

- Ports in the domain layer — appropriate when the contract is expressed purely in domain terms.
- Ports in the application layer — appropriate when the contract exists to serve a use case.

Both preserve the dependency rule. **Placement is a vocabulary decision, not an architectural one.** Record the choice (see `naming-and-modes.md`) and do not relocate ports as incidental cleanup.

## 4. Adapters

An adapter is a **translation layer with exactly one job**: convert between the outside world's representation and the port's contract.

| Kind | Direction | Typical examples |
|---|---|---|
| Driving / primary | Outside → application | HTTP controller, GraphQL resolver, CLI command, queue consumer, cron trigger, UI component |
| Driven / secondary | Application → outside | Repository, gateway, message publisher, file store, clock, UUID generator |

Rules:

- An adapter contains no business decisions. If it does, the decision belongs in the domain.
- An adapter never talks to another adapter. Cross-adapter calls bypass the domain and hide coupling.
- Framework types stop at the adapter. Nothing framework-shaped crosses a port.

## 5. Dependency matrix

| From | May reference |
|---|---|
| Domain | Domain |
| Application | Domain, outbound port interfaces |
| Driving adapter | Inbound port (application) |
| Driven adapter | Outbound port interface, domain types |
| Composition root | All of the above |

Forbidden, in every layout tier:

- Domain → application, adapters, framework, ORM, HTTP, SDK.
- Application → adapter implementation, ORM, framework wiring.
- Adapter → another adapter.
- Any layer → a concrete instance created with `new` when it should arrive through the composition root.

## 6. Export surface

The **export surface** is what a module or layer makes visible across its boundary: public types, barrel files, package `exports`, project references.

Rules:

- A layer exports its contract, not its internals. Domain exports domain types; application exports use cases; adapters export nothing that another layer imports, except the single registration entry point the composition root calls.
- A barrel file must not widen a boundary. Re-exporting an adapter from an application index silently reverses the dependency rule.
- The export surface is the cheapest thing to check automatically, so it should be the first thing a dependency test asserts.

## 7. Boundary models

Data crossing a boundary gets its own shape:

- **DTO / boundary model** at the edge, owned by the adapter.
- **Domain types** stay inside. An ORM entity is not a domain entity, even when the fields look identical.
- **Mapper** at the adapter converts between them.

Sharing one type across all three is the most common cause of "we tried hexagonal and it did not help": the domain ends up shaped by the database.

## 8. Terms

| Term | Meaning | Not to be confused with |
|---|---|---|
| Domain | The layer holding business rules | The whole application |
| Core domain | The part of the business that differentiates the product | Every module in the repo |
| Use case | One application operation, expressed in domain terms | A controller action |
| Inbound port | Contract the outside calls | An HTTP route |
| Outbound port | Capability the application needs | A repository class |
| Driving adapter | Entry point into the application | The composition root |
| Driven adapter | Implementation of an outbound port | A domain service |
| Composition root | Single place where concretes are wired | A controller |
| Aggregate | Consistency boundary with one root | Any entity |
| Anti-corruption layer | Translation boundary between two models | A DTO mapper |
| Shared kernel | Model deliberately shared by several contexts | A `common` dumping ground |
| Bounded context | Boundary within which a model has one meaning | A deployment unit |
| Modular monolith | Two senses: (1) a non-hexagonal alternative where modules share one deployable and expose interfaces; (2) the layout tier where every module is internally hexagonal | A microservice split |

## 9. What hexagonal architecture is not

- **Not microservices.** One process can be hexagonal; a microservice can be a layered mess.
- **Not DDD.** DDD supplies modeling techniques that fit well; hexagonal supplies the dependency rule. Either can exist without the other.
- **Not a folder convention.** Folders are an output of the decision, never the decision.
- **Not layered architecture with renamed folders.** In classical layering the domain typically depends on data access. Hexagonal inverts that: the domain owns the contracts and the data layer implements them.

## 10. Selective adoption

Hexagonal architecture pays off where business rules are complex and change often. In one repository, that is usually a subset of modules.

`selective` means: apply the rule to the core domain, keep supporting modules in a simpler style, and place an explicit boundary between them. This is a first-class outcome, not a compromise. See `fit-assessment.md` for how to choose the boundary.
