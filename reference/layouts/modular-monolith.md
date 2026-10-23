# Modular Monolith Layout

> **Read when:** `layouts/index.md` landed on `modular-monolith` — several bounded contexts in one deployable, each internally hexagonal, with a boundary that survives deadlines.

Several modules, each with its own domain and adapters, one process. A module may see another module only through that module's `Contracts` project. This is the tier that makes cross-context coupling visible in the project graph instead of hiding it in a shared service class.

**Term warning:** `fit-assessment.md` §5 uses "modular monolith" for the *non-hexagonal* alternative to adoption. This file uses the term for a deployable in which every module is hexagonal. If the plan's verdict is `not viable` and recommends the modular-monolith alternative, do not use this tree; that alternative keeps modules in their existing style.

## 1. Target tree

```
commerce/
  Commerce.sln
  src/
    Modules/
      Billing/
        Billing.Contracts/                 # published language; the only part other modules may import
        Billing.Domain/                    # rules; may use SharedKernel, nothing else outside
        Billing.Application/               # orchestration; implements Billing.Contracts facades
        Billing.Adapters.Inbound.Http/     # module routes and consumers; -> Billing.Application
        Billing.Adapters.Outbound.Persistence/   # ORM, schema, migrations, outbox
      Shipping/
        Shipping.Contracts/
        Shipping.Domain/
        Shipping.Application/
        Shipping.Adapters.Inbound.Http/
        Shipping.Adapters.Outbound.Persistence/
      Catalog/
        Catalog.Contracts/
        Catalog.Domain/
        Catalog.Application/
        Catalog.Adapters.Inbound.Http/
        Catalog.Adapters.Outbound.Persistence/
    SharedKernel/                      # cross-module types only; not a per-module Primitives folder
    Host/
      Program.cs                       # module registration and route prefixes; no business logic
  tests/
    ArchitectureTests/                 # module boundary rules (§4)
    Billing.Tests/
    Shipping.Tests/
```

Ports live in `<Module>.Application/Ports/`. `<Module>.Adapters.Inbound.*` and `<Module>.Adapters.Outbound.*` are the driving and driven adapters; neither contains a port declaration, and neither references the other.

Dependency rules, in full:

| From | May reference | Must never reference |
|---|---|---|
| `<Module>.Contracts` | `SharedKernel` | Any other project of any module, including other Contracts |
| `<Module>.Domain` | Its own domain types, `SharedKernel` | Everything else outside `<Module>.Domain`, including its own Contracts |
| `<Module>.Application` | Its own Domain, its own Contracts, other modules' Contracts, `SharedKernel` | Any adapter; any other module's Domain or Application |
| `<Module>.Adapters.Inbound.*` | Its own Application, its own Contracts, `SharedKernel` | Its own outbound adapter, or any other module's internals |
| `<Module>.Adapters.Outbound.*` | Its own Application, its own Domain types, its own Contracts, `SharedKernel` | Its own inbound adapter, or any other module's internals |
| `Host` | Every module's Application, inbound and outbound adapters, `SharedKernel` | — |
| `SharedKernel` | Nothing | Any module |

Additional rules:

- **Contracts do not reference Contracts.** If two published languages need the same type, it either belongs to one module (and the other imports that module's Contracts) or it is a shared-kernel candidate. A contract graph with cycles is a modeling error, not a build configuration problem.
- **Cross-module calls are contracts-first.** The caller depends on `<Other>.Contracts`; the implementation lives in `<Other>.Application`; `Host` wires them. The caller never sees the implementation type.
- **Use events where no answer is needed, a Contracts facade where one is.** Do not do both for the same interaction; two consistency stories for one edge is how teams lose track of ordering.
- **One schema per module.** Its own migrations, its own tables, no cross-module joins, no shared `DbContext`. This is the rule that makes later extraction possible; everything else is decoration without it.
- **`SharedKernel` is domain code, not a layer between modules.** A module's Domain may reference it because the kernel is part of the model every adopter deliberately shares. It is still not a place for workflows, integration events, or utilities.
- **`SharedKernel` is not `Primitives`.** `Primitives` is per-module domain vocabulary. `SharedKernel` is the cross-module kernel described in `concepts.md` §8. Keep them in different folders and never merge them.

Export surface:

- `<Module>.Contracts` is the only importable project. It contains DTOs, integration event types, and facade interfaces. No module-internal domain types, no ORM types, no framework types; a shared-kernel value type is allowed only when the kernel already owns it.
- Every other module project is internal, even though the compiler may allow a reference if someone adds one. The rules in §4 make that reference fail the build.
- `SharedKernel` is importable by all modules, owned by the solution, and changed only with every consumer's review.

## 2. Why it is split this way

Ownership in this tier:

- `Contracts` owns the published language and nothing else. If it contains behavior, it is an application service with a misleading name.
- `Domain` owns the module's invariants. It does not know that other modules exist.
- `Application` owns orchestration inside the module, the local transaction boundary, and implementations of the module's own facade interfaces.
- `Adapters.Inbound.*` owns the module's driving adapters: routes, consumers, and their DTO mapping.
- `Adapters.Outbound.*` owns driven adapters, schema, migrations, and the outbox that publishes integration events.
- `SharedKernel` owns types whose meaning is identical in every module. It owns no workflow.
- `Host` owns composition and route prefixes. It must remain boring enough that no one needs to review it carefully. It is the only project allowed to see every module's Application and adapters.

What the split buys:

- **Change isolation is visible.** A feature inside one module touches one folder. Cross-module work appears as a Contracts diff, which reviewers can see and price.
- **Extraction is prepared, not performed.** The module already has its own schema, contracts, migrations, and host registration. Moving it to a process is deployment work rather than a modeling project.
- **Coupling is countable.** Every cross-module edge is a reference to a Contracts project. The project graph is the architecture diagram, and it is generated rather than drawn.
- **One deployable.** No network partitions, no distributed tracing, no eventual consistency except where you deliberately chose events. Local transactions still exist inside a module.
- **Read paths are designed.** Cross-module joins have no sanctioned path, so reporting gets a read model instead of a query that quietly couples two schemas.

What it costs:

- Five projects per module before extra adapters. Five modules means twenty-five projects plus SharedKernel, Host, and tests; build time and IDE indexing are noticeably worse.
- No cross-module ACID transaction. Consistency moves into outbox, sagas, or explicit compensation, and that code has to be written and operated.
- Contracts are a versioned surface inside one repository. Every change has a consumer to notify, even when that consumer sits three desks away.
- Mapping is duplicated at every contract boundary: producer domain → contract, consumer contract → domain. That is two mappers per edge and two sets of tests.
- `Host` becomes a startup bottleneck. Every module edits its registration, and one bad module fails the whole process.
- The first module boundary is usually wrong. Expect one merge or split within the first six months; budget it as learning, not failure.

## 3. When NOT to use it

1. **The second "module" is a feature, not a context** — same vocabulary, same invariants, same owner. Add a use case to the existing module. Two genuine contexts justify this tier; a third is not a prerequisite.
2. **Any two modules share one ACID invariant or write the same tables.** Then they are one module. Splitting creates an in-process distributed transaction with none of the operational honesty of a real one.
3. **Cross-module joins are the dominant read path and there is no read-model plan.** The first reporting deadline will produce a join across schemas, and the boundary will leak where it is hardest to see.
4. **The "modules" are technical layers** — Reporting, Notifications, Integrations, Shared — with no domain vocabulary and no owner. That is a folder split wearing module names; use `standard.md`.
5. **The codebase is one compilation unit.** One `.csproj`, one `tsconfig` with no library boundaries, one source root. You cannot enforce this tier; a diagram is not a boundary.
6. **No one can name an owner per module.** Unowned boundaries are crossed by whoever is on call, and the Contracts project becomes a suggestion.
7. **The only way to make it work is to put domain entities in `SharedKernel`.** Sharing an entity means the contexts share a model; they are one context.
8. **No one can articulate a change that should stop at a module boundary.** If every change legitimately touches four modules, the boundaries are hypothetical and the projects are overhead.

## 4. Enforcement

The project graph is the primary check; the architecture test is the backstop for namespace-level leaks that the compiler cannot see (for example, a type placed in the wrong project).

| Stack | Tool | Rule lives in | What it asserts |
|---|---|---|---|
| .NET | Project references plus NetArchTest, plus a solution-level `.csproj` scan | `tests/ArchitectureTests` | No module project references another module's non-Contracts project, including unused references |
| Java | Gradle `api`/`implementation` plus ArchUnit slices | `build.gradle` and `src/test/java/.../ModuleTest` | Module packages are cycle-free; cross-module edges are checked separately against the Contracts-only rule |
| TypeScript | Nx tags, or one generated dependency-cruiser rule per module pair | ESLint config or `.dependency-cruiser.js` | `scope:<module>` only reaches its own scope, `scope:public` Contracts, and `scope:shared` |
| Angular | `@nx/enforce-module-boundaries` with `scope:` and `layer:` tags | ESLint config, `depConstraints` | Scope and layer are checked independently; a Contracts lib is `scope:public` |

On .NET, prefer the `.csproj` scan over a namespace test for cross-module references: it catches a reference that no code uses yet, which is exactly the failure mode that precedes coupling.

Schema ownership is invisible to all four tools. Enforce it with one migration directory per module and one database principal per module; a grant that reaches another module's tables is the violation that a project reference cannot catch.

Rule example:

```js
// eslint.config.js (fragment) — tag every project with exactly one scope and one layer
module.exports = [
  {
    rules: {
      '@nx/enforce-module-boundaries': ['error', {
        depConstraints: [
          { sourceTag: 'layer:domain',      onlyDependOnLibsWithTags: ['layer:domain', 'layer:shared-kernel'] },
          { sourceTag: 'layer:contracts',   onlyDependOnLibsWithTags: ['layer:shared-kernel'] },
          { sourceTag: 'layer:application', onlyDependOnLibsWithTags: ['layer:domain', 'layer:application', 'layer:contracts', 'layer:shared-kernel'] },
          { sourceTag: 'layer:adapter-inbound',  onlyDependOnLibsWithTags: ['layer:application', 'layer:contracts', 'layer:shared-kernel'] },
          { sourceTag: 'layer:adapter-outbound', onlyDependOnLibsWithTags: ['layer:domain', 'layer:application', 'layer:contracts', 'layer:shared-kernel'] },
          { sourceTag: 'scope:billing',     onlyDependOnLibsWithTags: ['scope:billing', 'scope:public', 'scope:shared'] },
          { sourceTag: 'scope:shipping',    onlyDependOnLibsWithTags: ['scope:shipping', 'scope:public', 'scope:shared'] },
        ],
      }],
    },
  },
];
```

With tags: `Billing.Domain` = `scope:billing`, `layer:domain`; `Billing.Contracts` = `scope:public`, `layer:contracts`; `Billing.Adapters.Inbound.Http` = `scope:billing`, `layer:adapter-inbound`; `Billing.Adapters.Outbound.Persistence` = `scope:billing`, `layer:adapter-outbound`; `SharedKernel` = `scope:shared`, `layer:shared-kernel`. `Host` carries no scope or layer tag on purpose: it is the one project allowed to see every module. A project with no tags is unconstrained, so add a CI check that every project **except the host** has exactly one `scope:` and one `layer:` tag, and keep the exemption list at one entry. That check catches more violations than the dependency rule itself.

Tags enforce scope and layer independently; they cannot express "billing may use shipping's Contracts but not catalog's". When that precision matters, add a generated dependency-cruiser rule per allowed pair, or encode the dependency in a third tag dimension. Do not pretend the two-dimensional tag set proves more than it does.

### Adding a module without breaking direction

1. **Prove it is a context, not a feature.** It needs its own vocabulary, its own invariants, and an owner. If it is a feature, add a use case to the existing module.
2. **Create the five projects.** Contracts first, with no references to any module.
3. **Define the published language.** DTOs, integration events, facade interfaces. No domain types, no ORM types, no behavior.
4. **Implement inside the module.** Own schema and migrations from the first commit. No shared tables, no edits to another module's database.
5. **Register in `Host`.** One line plus a route prefix. If adding a module requires edits in three other modules, the boundaries are wrong.
6. **Add tags and the module list.** Update the architecture test's module list and the Nx/depcruise configuration in the same change.
7. **Prove independence.** Delete the module folder, its host line, and every reference to its Contracts; the solution must still build. If it does not, the boundary leaked.

Two rules for the edges:

- Existing modules never gain a reference to the new module's internals. If the new module must call back, define a contract in the caller's Contracts project and wire it in `Host`. A cycle between two modules means they should be one module.
- A module that cannot be deleted in one pull request was never a module.
