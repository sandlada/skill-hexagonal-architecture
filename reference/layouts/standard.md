# Standard Layout

> **Read when:** `layouts/index.md` landed on `standard` — one service, one bounded context, and the dependency rule must survive without a reviewer watching every import.

One service, one bounded context, one project per layer, adapters split by direction, one host that is only a composition root. The unit of enforcement is the project or package reference, not a folder convention. This is the default tier for a backend with real rules, one deployable, and one owning team.

## 1. Target tree

```
billing/
  Billing.sln
  Directory.Packages.props             # one version per package; a new dependency is a reviewable diff
  src/
    Billing.Domain/                    # owns the rules. Zero project references, zero framework packages.
      Aggregates/
      Primitives/
      Errors/
    Billing.Application/               # -> Domain. Owns use cases, transaction boundary, outbound ports.
      usecases/
        IssueInvoice/
        VoidInvoice/
      Ports/
        Outbound/                      # implemented by the outbound adapter
    Billing.Adapters.Inbound.Http/     # -> Application. Driving adapter: HTTP, DTOs, status mapping.
      Endpoints/
      Contracts/                       # request/response models; never domain types
    Billing.Adapters.Outbound.Persistence/   # -> Application, Domain. Driven adapter: ORM, schema access.
      Persistence/
    Billing.Api/                       # -> Application and both adapters. Composition root only.
      Program.cs
      appsettings.json
  tests/
    Billing.Domain.Tests/
    Billing.Application.Tests/         # use cases against fake ports; no container, no database
    Billing.Adapters.Outbound.Persistence.Tests/   # against a real database
    Billing.ArchitectureTests/         # the rule, executable (§4)
```

Notes:

- In `usecase` mode the handler is the inbound port. Add a separate inbound interface only when a second implementation or a test seam needs one; do not add marker interfaces to make the tree look symmetrical.
- The inbound adapter references `Application` only. An endpoint that needs the outbound adapter is a wiring bug: it must go through a port.
- The outbound adapter references `Application` and `Domain`, because port signatures use domain types. It never references the inbound adapter.
- `Billing.Api` has no business logic, no DTOs, and no service classes. It binds configuration, orders middleware, and invokes each adapter's registration entry point. If it grows a controller, that controller belongs in the inbound adapter project.
- The host project is named `Api` here, `bootstrap` in some Java codebases, and `Host` elsewhere. The name is vocabulary; the role is composition root, and exactly one project has it.
- Single-inbound variant (common in Node/TypeScript): collapse the inbound adapter and the host into `apps/api`, as the TypeScript example does. The outbound adapter may then be imported only from the composition file; enforce that with a path-scoped dependency-cruiser rule and accept the weaker guarantee.
- Java: Gradle or Maven modules with the same roles. `implementation` dependencies, not `api`, except where a type appears in a published signature; `application` exposes `domain` as `api` so outbound adapters can name domain types.
- A second inbound adapter (worker, CLI, scheduler) is another project referencing `Application`; the host composes both. It does not go inside the HTTP adapter, and it does not get its own copy of the use cases.

Export surface:

- `Domain` exposes domain types. Implementation types are `internal`/package-private; making everything public is how a domain becomes a library by accident.
- `Application` exposes use-case entry types (the handler and, when present, its inbound interface) and outbound port interfaces. Validators, mappers, and helpers inside a use-case folder stay internal.
- The inbound adapter exposes exactly one registration entry point per host. Endpoint classes, wire models, and mappers stay internal.
- The outbound adapter exposes exactly one registration entry point per adapter group. It exposes no repository, gateway, or ORM types.
- `Api` exposes nothing; no production project references it.
- A barrel or index that re-exports an outbound adapter from `Application` reverses the rule more quietly than a `using` statement. The architecture test in §4 asserts the export surface first because it is the cheapest check (`concepts.md` §6).

## 2. Why it is split this way

Ownership in this tier:

- `Domain` owns the invariants, domain events, and domain errors. It has no project references, and its package list must stay empty; the guard in §4 makes an accidental `PackageReference` fail the build.
- `Application` owns one folder per operation: the order of calls, the transaction boundary, authorization decisions, and idempotency. It declares the outbound ports it needs.
- The inbound adapter owns transport: routing, authentication filters, request/response models, status mapping. Validation here is shape validation; business validation belongs inward.
- The outbound adapter owns the translation from port contracts to a database, queue, or external API, plus ORM entities and mappings. It contains no branch on business state.
- `Api` owns composition: which adapter satisfies which port, lifetimes, configuration binding, middleware order.

What the split buys:

- **Direction is a build error.** Domain has no reference to Application, so `using Billing.Application` does not compile. The inbound adapter cannot name a repository, and the outbound adapter cannot name an endpoint. On .NET and Java this is stronger than any lint rule; on TypeScript it requires project references plus the rule in §4.
- **Dependency isolation is factual.** The ORM is in the outbound adapter's package list, not Domain's. "The domain is framework-free" stops being a code-review opinion.
- **Independent test seams.** Application tests run against fake ports with no container and no database. The outbound adapter's tests run against a real database. Neither waits for the other, and the inbound adapter is tested at the transport boundary.
- **Replacement is a project swap.** A second database adapter is a new project implementing existing ports; Application and Domain do not change.
- **Ownership is assignable.** CODEOWNERS can point at projects; a review boundary expressed as a folder is a suggestion.

What it costs:

- Five production projects and four test projects before the first feature ships. Onboarding time and build time are real, and the solution file is now part of the design.
- A cross-layer feature is a three-project diff. "Small" changes stop being small, and reviewers must hold the whole path in view.
- Mapping happens at two seams: wire model ↔ use-case input, and domain ↔ persistence. Teams try to skip it by returning domain types from endpoints; that is the failure mode this tier exists to prevent, not a shortcut.
- The architecture tests are code. Renaming a project or namespace breaks them, and they need an owner.
- Adapter projects multiply with delivery mechanisms: two inbound adapters and two outbound integrations means four adapter projects before any feature work.

## 3. When NOT to use it

1. **Fewer than ~5 use cases and a single inbound adapter.** The project graph costs more than the boundary buys; use `minimal.md`.
2. **Two or more bounded contexts with separate vocabularies or owners.** Repeating this tree per context without Contracts produces a distributed monolith inside one process; go to `modular-monolith.md`.
3. **The architecture test will not run in CI.** Without the gate, standard is minimal with five projects and slower builds.
4. **The domain is CRUD.** No invariant independent of the schema means the split only relocates the same code. Use vertical slices; do not wrap a table editor in ports.
5. **One context must be split into independently deployable processes now.** That is `enterprise.md`, not "standard plus microservices". Splitting by project and calling it a service adds network failure modes without contract discipline.
6. **Application is a pass-through.** If every use case is `return await _repository.Get(id)`, there is no orchestration to own. Minimal is the honest tier; a multi-project split around a pass-through is pure ceremony.
7. **The team cannot say what belongs in Domain versus Application.** The boundary will drift with the first deadline. Do the modeling pass first; the tier will not supply it.

## 4. Enforcement

Rules to assert:

1. Domain → nothing: no project references, no framework packages.
2. Application → Domain only.
3. Inbound adapter → Application only.
4. Outbound adapter → Application and Domain.
5. Api → Application and both adapters; no production project references Api.
6. Inbound adapter ↛ outbound adapter; outbound adapter ↛ inbound adapter.

| Stack | Tool | Rule lives in | What it asserts |
|---|---|---|---|
| .NET | NetArchTest.Rules, or ArchUnitNET if the team already uses ArchUnit | `tests/Billing.ArchitectureTests` | Assembly references match the list above; Domain has no namespace dependency on outer layers or ORM/HTTP packages |
| Java | ArchUnit | `src/test/java/.../LayerTest` | `..domain..` depends only on itself and `java..`; `..application..` does not touch `..adapter..`; the two adapters do not touch each other |
| TypeScript | dependency-cruiser plus `tsconfig` project references | `.dependency-cruiser.js` | Layer edges match the list; outbound adapter imports are confined to the composition file |
| Angular | `@nx/enforce-module-boundaries` with `layer:` and `scope:` tags | ESLint config, `depConstraints` | One project per feature by default; promote layers to separate projects only when the boundary is crossed often; adapter layers cannot see each other |

Rule example:

```csharp
using NetArchTest.Rules;

// tests/Billing.ArchitectureTests/LayerTests.cs
public class LayerTests
{
    [Fact]
    public void Domain_depends_on_nothing_outside_itself()
    {
        var result = Types.InAssembly(typeof(Invoice).Assembly)
            .ShouldNot()
            .HaveDependencyOnAny(
                "Billing.Application",
                "Billing.Adapters.Inbound.Http",
                "Billing.Adapters.Outbound.Persistence",
                "Billing.Api",
                "Microsoft.EntityFrameworkCore", "Microsoft.AspNetCore")
            .GetResult();

        Assert.True(result.IsSuccessful, Failures(result));
    }

    [Fact]
    public void Application_does_not_reference_an_adapter()
    {
        var result = Types.InAssembly(typeof(IssueInvoiceHandler).Assembly)
            .ShouldNot()
            .HaveDependencyOnAny(
                "Billing.Adapters.Inbound.Http",
                "Billing.Adapters.Outbound.Persistence")
            .GetResult();

        Assert.True(result.IsSuccessful, Failures(result));
    }

    [Fact]
    public void Http_contracts_do_not_expose_domain_types()
    {
        var result = Types.InAssembly(typeof(InboundHttpRegistration).Assembly)
            .That().ResideInNamespace("Billing.Adapters.Inbound.Http.Contracts")
            .ShouldNot().HaveDependencyOn("Billing.Domain")
            .GetResult();

        Assert.True(result.IsSuccessful, Failures(result));
    }

    static string Failures(TestResult result) =>
        string.Join(", ", result.FailingTypeNames ?? Enumerable.Empty<string>());
}
```

The architecture test project references every production project. It is the one project allowed to; that reference is how it loads the assemblies, not a dependency of the application.

Project references stop assemblies, not packages. Add the `ForbidOutwardDependencies` build target from `examples/csharp.md` so `Billing.Domain.csproj` cannot acquire a `PackageReference` in a one-line pull request; NetArchTest only sees what has already compiled.

Two closing rules:

- Run the tests in CI on every pull request and fail the build. A rule that produces a warning is a suggestion.
- Assert the export surface before asserting behavior: no Domain type in the inbound adapter's contracts, no outbound-adapter type in an Application signature, no adapter type in a port. Those three catch most regressions, and they are cheaper to write than the dependency rules.
