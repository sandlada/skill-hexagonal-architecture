# Naming and Modes

> **Read when:** you have a verdict and must choose the application-layer mode and the directory vocabulary before emitting a target structure.

Two independent choices. Both are **structural** decisions: they change folder names, file placement, and what each module exports. Neither changes class, method, or variable names — that is out of scope for this library.

## 0. Rule zero: inherit first

1. Inspect the existing tree.
2. If a convention already exists for a layer, adopt it and say so.
3. Propose a new scheme only for layers with no existing pattern.
4. Record the choice and stop revisiting it.

If no tree is available, do not claim inheritance. Mark the vocabulary `provisional` and confirm it at the options gate before generating anything.

A migration that renames everything to match a template is a migration that failed. Renames break imports, barrel files, and code review history. Only propose a folder rename when it removes a real boundary violation, and then list it in the plan's breaking-change section.

## 1. Application-layer mode

All four shapes satisfy the dependency rule. They differ in discoverability and in what counts as the exported contract.

### A. `usecase`

```
application/
  usecases/
    <use-case-name>/
```

- One folder per operation; the entry type is the use-case handler.
- **Choose when:** operations have meaningful orchestration, and "where is this feature" should be answerable from the tree.
- **Cost:** many small folders; shared steps can be duplicated or pulled into a shared service that slowly becomes a second domain.
- **Export surface:** the handler per folder. Everything else in the folder is internal.

### B. `command | query`

```
application/
  commands/
    <command-name>/
  queries/
    <query-name>/
```

- Reads and writes are separated at the top level.
- **Choose when:** read and write paths have different shapes, performance profiles, or ownership; or the team already thinks in commands.
- **Cost:** two parallel trees; cross-cutting validation tends to be duplicated.
- **Export surface:** command handlers and query handlers, separately.
- **Hard rule:** a query never mutates. If it does, the mode has already broken down.

### C. `input | output | response`

```
application/
  <feature>/
    input/
    output/
```

or, when contracts are shared across features:

```
application/
  inputs/
  outputs/
```

- The boundary model is the organizing principle.
- **Choose when:** the shape of data crossing the boundary is the main design concern — public APIs, integrations, contract-first teams.
- **Cost:** orchestration has no obvious home and can leak into the input type.
- **Export surface:** the input and output types are the contract; the handler stays internal.

### D. Hybrid

Allowed and common. Rules:

- One **flow** is expressed in exactly one mode. Do not put commands inside a use-case folder or both `input/` and a command handler for the same operation.
- Different modules may use different modes. Record which module uses which.
- A hybrid must be justified by a real split — for example writes as commands, orchestration-heavy flows as use cases — not by indecision.

## 2. Domain-layer vocabulary

These are alternative folder names for the same concepts. Detect what the project uses, then stay consistent.

| Canonical concept | Observed variants | Default generation | Caution |
|---|---|---|---|
| Model grouping | `Aggregates`, `Entities`, `Entries`, `Objects` | `Aggregates` if real aggregates exist, otherwise `Entities` | `Objects` is the loosest name and hides the consistency boundary; do not introduce it |
| Shared primitives | `Common`, `Commons`, `Primitives` | `Primitives` | `Common` reliably becomes a dumping ground; if inherited, keep it but restrict what may enter |
| Errors | `Error`, `Errors`, `DomainErrors` | `Errors` | Singular only when already established |
| Outbound port declarations | `Ports`, `Repositories`, `Adapters`, `Adapter` | `Ports` | `Adapters` inside the domain means port declarations. Infrastructure adapters must still live outside the domain regardless of the name |

`Repositories` is legitimate when the outbound ports really are persistence-shaped. If the domain also needs messaging, clock, or external services, `Ports` scales better than a `Repositories` folder that quietly holds non-repository contracts.

### Where port declarations live

Choose one per project: `domain/ports/` or `application/ports/`. Both are valid; see `concepts.md` §3. Record the choice. Do not split the same concern across both.

## 3. Presentation layer

`presentation`, `api`, `web`, and `adapters/inbound` name the same layer.

| Name | Typical use |
|---|---|
| `presentation` | Full-stack or desktop-oriented projects |
| `api` | Backend services exposing HTTP/GraphQL |
| `web` | Web-only delivery, common in frontend projects |
| `adapters/inbound` | Projects that group all adapters under one parent |

The table is a default, not a rule: an existing project name wins. An ASP.NET project called `Shop.Web` stays `web`; do not rename it to `api`.

Backend note: the presentation layer may be a **dedicated Api project** acting as the inbound adapter host. Project names may combine layer and direction (for example `Billing.Adapters.Inbound.Http` or `adapter-in-web`); record the layer choice, not the project string. When that is the choice:

- The Api project references the application, never the reverse.
- It contains no business rules — no validation that belongs to the domain, no branching on business state.
- It owns transport concerns: routing, auth filters, serialization, status mapping.

Frontend note: Angular maps differently — the UI component is the driving adapter and the API client is a driven adapter. See `examples/angular.md`; do not apply the backend project split there.

## 4. Recognition procedure

Run this before proposing anything:

1. List the children of the existing domain folder. Match each against the variant sets above.
2. List the children of the application folder. Match against the four modes.
3. Record one row per layer: `layer | matched variant | confidence | evidence path`.

Partial matches are normal. Use the intersection and propose only for the gaps.

## 5. Options record

Every plan must contain this block, filled in:

```
Options record
- Application mode:      <usecase | command|query | input|output|response | hybrid>
- Domain vocabulary:     model=<> shared=<> errors=<> outbound ports=<>
- Port location:         <domain | application>
- Presentation name:     <presentation | api | web | adapters/inbound>
- Inherited from code:   <yes | no> (<paths>)
- Rationale:             <one sentence>
```

If the user does not answer, stop and ask. Do not default silently — a wrong vocabulary choice produces a target tree that contradicts the team's own code on day one.

## 6. Consistency rules

- One vocabulary per group per layer. `Entities` and `Aggregates` in the same domain folder means the boundary is unclear, not that both concepts are present.
- Generated examples and target trees must use the recorded vocabulary. Never mix the default names into a project that inherited a convention.
- A mode is a property of a module, not of the repository. Record it per module when a hybrid is used.
- Changing the vocabulary after the plan is emitted means regenerating the target tree and re-checking breaking changes. Treat it as a new decision, not a correction.
