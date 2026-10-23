# C# Example

> **Read when:** you are applying the standard tier to a .NET solution and need the project split, the reference direction, the architecture tests, and the composition root for one bounded context.

One bounded context, one deployable, five production projects plus four test projects. `Shop.Api` is both the host and the composition root; every other project is a layer or an adapter. If the solution hosts several bounded contexts, repeat this shape per context instead of growing one shared `Domain` project — a shared domain project becomes the place where every context's rules meet, and the dependency rule stops meaning anything.

**Options used in this example** (record your own before generating a tree; see `../naming-and-modes.md` §5):

```
Options record
- Application mode:      usecase
- Domain vocabulary:     model=Aggregates, shared=Primitives, errors=Errors, outbound ports=Ports
- Port location:         application; the inbound port sits with its use case, outbound ports under Ports/Outbound
- Presentation name:     Shop.Adapters.Inbound.Http, hosted by Shop.Api
- Inherited from code:   no — greenfield illustration
- Rationale:             one folder per use case makes "where is this feature" answerable from the tree
```

## 1. Tree

```
Shop.sln
Directory.Packages.props                  one version per package; a new dependency is a reviewable diff
src/
  Shop.Domain/                            no project references, no packages
    Aggregates/
      Orders/
        Order.cs                          aggregate root
        OrderId.cs
        OrderLine.cs
        OrderLineDraft.cs
        OrderStatus.cs
    Primitives/
      Money.cs
      Sku.cs
      CustomerId.cs
    Errors/
      DomainException.cs
  Shop.Application/                       references Shop.Domain
    UseCases/
      PlaceOrder/
        PlaceOrderCommand.cs
        PlaceOrderLine.cs
        PlaceOrderResult.cs
        PlaceOrderHandler.cs              inbound port: the contract the endpoint calls
    Ports/
      Outbound/
        IOrderRepository.cs
        IUnitOfWork.cs
  Shop.Adapters.Inbound.Http/             references Shop.Application
    InboundHttpRegistration.cs            public: AddInboundHttp()
    OrderEndpoints.cs                     public: MapOrders()
    Endpoints/
      PlaceOrderEndpoint.cs               internal
    Contracts/
      PlaceOrderRequest.cs                public wire contract
      PlaceOrderResponse.cs               public wire contract
    Errors/
      DomainExceptionHandler.cs           internal
  Shop.Adapters.Outbound.Persistence/     references Shop.Application, Shop.Domain
    PersistenceRegistration.cs            public: AddPersistence()
    ShopDbContext.cs                      internal
    Orders/
      OrderRecord.cs                      internal ORM model
      OrderLineRecord.cs                  internal ORM model
      OrderConfiguration.cs               internal IEntityTypeConfiguration<OrderRecord>
      OrderMapper.cs                      internal record <-> aggregate
      OrderRepository.cs                  internal, implements IOrderRepository
      EfUnitOfWork.cs                     internal, implements IUnitOfWork
  Shop.Api/                               references Shop.Application and both adapters
    Program.cs                            composition root
    appsettings.json
tests/
  Shop.Domain.Tests/                      references Shop.Domain
  Shop.Application.Tests/                 references Shop.Application
  Shop.Adapters.Outbound.Persistence.Tests/   Testcontainers, references the adapter
  Shop.Architecture.Tests/                references every production project, ships nothing
    HexagonalRules.cs
```

Reference direction, and the reason each edge is legal:

| Project | References | Why |
|---|---|---|
| `Shop.Domain` | nothing | The rule's anchor. If it references anything, the rule is already broken. |
| `Shop.Application` | `Shop.Domain` | Use cases orchestrate domain types. |
| `Shop.Adapters.Inbound.Http` | `Shop.Application` | Calls the inbound port; knows HTTP. |
| `Shop.Adapters.Outbound.Persistence` | `Shop.Application`, `Shop.Domain` | Implements an outbound port whose signatures use domain types. The edge points inward, which is allowed; what is forbidden is any edge out of `Shop.Domain`. |
| `Shop.Api` | `Shop.Application`, both adapters | The composition root is the only project allowed to see both sides. |
| `Shop.Architecture.Tests` | everything | Test-only; not a deployment unit. |

## 2. Enforcing the direction

### Build time: project references (primary)

`ProjectReference` is the main mechanism. A type in `Shop.Adapters.Inbound.Http` cannot name `OrderRepository`, because the assembly is not on its compile path. Cyclic project references do not build at all, which removes the whole class of "domain calls the repository it was supposed to own" designs. `Shop.Application` does not reference `Microsoft.Extensions.DependencyInjection`, so a service-locator cannot appear in a use case either.

### Why that is not enough

A project reference constrains *assemblies*, not *packages*, and it constrains nothing at all inside the referenced assembly. Three concrete gaps:

1. `Shop.Domain.csproj` can add `<PackageReference Include="Microsoft.EntityFrameworkCore" />` in a one-line PR and still build. Attributes, base classes, or `ILogger` then pull the domain toward the database or the host.
2. A namespace-level violation inside one assembly — an application handler reaching into a persistence namespace — is invisible to the compiler if the two live in the same project, and it appears the moment someone "temporarily" merges projects.
3. Reflection, DI container strings, and serialization attributes are not references at all. Only a rule that inspects compiled types or the build graph can see them.

So the domain project also refuses dependencies at build time:

```xml
<!-- src/Shop.Domain/Shop.Domain.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <Target Name="ForbidOutwardDependencies" BeforeTargets="CollectPackageReferences;ResolveProjectReferences">
    <ItemGroup>
      <_Forbidden Include="@(PackageReference)" />
      <_Forbidden Include="@(ProjectReference)" />
    </ItemGroup>
    <Error Condition="'@(_Forbidden)' != ''"
           Text="Shop.Domain must stay dependency-free. Found: @(_Forbidden). Move the type inward or declare a port." />
  </Target>
</Project>
```

If a source generator or analyzer is genuinely required, add it with `PrivateAssets="all"` and an explicit allowlist condition. A silent exception is how the guard dies.

### Test time: NetArchTest

Build-time guards cover packages and references. The rules below cover *usage* — which type mentions which namespace — and they run in CI on every pull request, not in review.

```csharp
// tests/Shop.Architecture.Tests/HexagonalRules.cs
using System.Reflection;
using NetArchTest.Rules;
using Shop.Adapters.Inbound.Http;
using Shop.Adapters.Outbound.Persistence;
using Shop.Application.UseCases.PlaceOrder;
using Shop.Domain.Aggregates.Orders;
using Xunit;

namespace Shop.Architecture.Tests;

public sealed class HexagonalRules
{
    private static readonly Assembly Domain = typeof(Order).Assembly;
    private static readonly Assembly Application = typeof(PlaceOrderHandler).Assembly;
    private static readonly Assembly Inbound = typeof(InboundHttpRegistration).Assembly;
    private static readonly Assembly Persistence = typeof(PersistenceRegistration).Assembly;

    [Fact]
    public void Domain_does_not_depend_on_frameworks()
    {
        var result = Types.InAssembly(Domain)
            .ShouldNot()
            .HaveDependencyOnAny(
                "Microsoft.EntityFrameworkCore",
                "Microsoft.AspNetCore",
                "Microsoft.Extensions.DependencyInjection",
                "System.Data",
                "System.Net.Http")
            .Because("business rules must be testable without a host, a database, or a container")
            .GetResult();

        Assert.True(result.IsSuccessful, Describe(result));
    }

    [Fact]
    public void Application_does_not_depend_on_adapters_or_orm()
    {
        var result = Types.InAssembly(Application)
            .ShouldNot()
            .HaveDependencyOnAny("Shop.Adapters", "Microsoft.EntityFrameworkCore", "Microsoft.AspNetCore")
            .GetResult();

        Assert.True(result.IsSuccessful, Describe(result));
    }

    [Fact]
    public void Inbound_adapter_does_not_reference_driven_adapters()
    {
        var result = Types.InAssembly(Inbound)
            .ShouldNot()
            .HaveDependencyOn("Shop.Adapters.Outbound")
            .GetResult();

        Assert.True(result.IsSuccessful, Describe(result));
    }

    [Fact]
    public void Driven_adapter_does_not_reference_driving_adapters()
    {
        var result = Types.InAssembly(Persistence)
            .ShouldNot()
            .HaveDependencyOn("Shop.Adapters.Inbound")
            .GetResult();

        Assert.True(result.IsSuccessful, Describe(result));
    }

    [Fact]
    public void Http_contracts_do_not_expose_domain_types()
    {
        var result = Types.InAssembly(Inbound)
            .That().ResideInNamespace("Shop.Adapters.Inbound.Http.Contracts")
            .ShouldNot().HaveDependencyOn("Shop.Domain")
            .GetResult();

        Assert.True(result.IsSuccessful, Describe(result));
    }

    private static string Describe(TestResult result) =>
        result.FailingTypeNames is { } failing
            ? string.Join(Environment.NewLine, failing)
            : "rule failed without reporting types";
}
```

If the team already uses ArchUnitNET, the same domain rule reads like this. Note that the architecture must be loaded once — building it per test is slow enough to get the tests disabled:

```csharp
private static readonly Architecture Architecture =
    new ArchLoader().LoadAssemblies(typeof(Order).Assembly).Build();

private static readonly IObjectProvider<IType> DomainLayer =
    ArchRuleDefinition.Types().That().ResideInNamespace("Shop.Domain").As("Domain layer");

[Fact]
public void Domain_must_not_use_a_framework()
{
    IArchRule rule = ArchRuleDefinition.Types().That().Are(DomainLayer)
        .Should().NotDependOnAny(ArchRuleDefinition.Types()
            .That().ResideInNamespace("Microsoft.EntityFrameworkCore", true));

    rule.Check(Architecture);
}
```

What each mechanism can and cannot see:

| Mechanism | Runs at | Catches | Misses |
|---|---|---|---|
| `ProjectReference` | build | outward assembly references, cycles | package references, namespace edges, reflection |
| `ForbidOutwardDependencies` target | build | any package or project added to `Shop.Domain` | everything added to the other projects |
| NetArchTest | test | framework types in the domain, application → adapter, adapter → adapter | reflection, DI strings, configuration |
| `ValidateScopes` / `ValidateOnBuild` | startup | lifetime mismatches | nothing directional — those never compile |

## 3. Composition root

`Program.cs` is the only file where a port and its implementation meet. Lifetimes are chosen here, once.

```csharp
// src/Shop.Api/Program.cs
using Shop.Adapters.Inbound.Http;
using Shop.Adapters.Outbound.Persistence;
using Shop.Application.UseCases.PlaceOrder;

var builder = WebApplication.CreateBuilder(args);

builder.Host.UseDefaultServiceProvider(options =>
{
    options.ValidateScopes = true;
    options.ValidateOnBuild = true;
});

var connectionString = builder.Configuration.GetConnectionString("Shop")
    ?? throw new InvalidOperationException("Connection string 'Shop' is not configured.");

builder.Services.AddInboundHttp();                 // ProblemDetails + exception handler
builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddScoped<PlaceOrderHandler>();
builder.Services.AddPersistence(connectionString);

var app = builder.Build();

app.UseExceptionHandler();
app.MapOrders();
app.Run();
```

`Shop.Api` has no EF Core reference: the adapter takes a connection string, not a `DbContextOptions`. The host cannot build a query even if someone tries.

Lifetimes and the failure they invite:

| Service | Lifetime | Reason |
|---|---|---|
| `ShopDbContext` | scoped (`AddDbContext` default) | One unit of work per request; it is not thread-safe. |
| `IOrderRepository`, `IUnitOfWork` | scoped | They must share the request's `DbContext` instance, or the unit of work commits nothing. |
| `PlaceOrderHandler` | scoped | The inbound port. Depends on the two scoped ports above. |
| `TimeProvider` | singleton | Stateless. |

The pitfall is the **captive dependency**: a singleton that receives a scoped service in its constructor keeps that instance for the process lifetime, so every request shares one `DbContext` and the first concurrent request fails with "A second operation was started on this context instance". It usually arrives as `AddSingleton<PlaceOrderHandler>()` after a profiling session, or as a static `IServiceProvider`. `ValidateScopes` turns it into a startup exception:

```
Cannot consume scoped service 'Shop.Application.Ports.Outbound.IOrderRepository'
from singleton 'Shop.Application.UseCases.PlaceOrder.PlaceOrderHandler'.
```

`ValidateOnBuild` forces that check for every registration at startup rather than on the first request that happens to hit the bad path. Both are on in Development by default; turn them on everywhere except when startup time is measured in minutes.

A second, quieter version: constructing an adapter with `new` inside a handler. Here it does not compile — the application project cannot name `OrderRepository` — which is the point of the project split.

## 4. The driving adapter

The endpoint calls a use case that knows nothing about HTTP. The handler is the inbound port — no marker interface is added to make the tree symmetrical — so it is public, together with the command and result types that cross the boundary; everything else in the application project is `internal`.

```csharp
// src/Shop.Application/UseCases/PlaceOrder/PlaceOrderHandler.cs
public sealed class PlaceOrderHandler(
    IOrderRepository orders,
    IUnitOfWork unitOfWork,
    TimeProvider timeProvider)
{
    public async Task<PlaceOrderResult> HandleAsync(
        PlaceOrderCommand command, CancellationToken cancellationToken)
    {
        var order = Order.Place(
            OrderId.New(),
            CustomerId.From(command.CustomerId),
            command.Lines
                .Select(line => new OrderLineDraft(line.Sku, line.Quantity, line.UnitPrice, line.Currency))
                .ToArray(),
            timeProvider.GetUtcNow());

        await orders.AddAsync(order, cancellationToken);
        await unitOfWork.SaveChangesAsync(cancellationToken);

        return new PlaceOrderResult(order.Id.Value);
    }
}
```

Note what is absent: no `DbContext`, no HTTP type, and no `SaveChangesAsync` on the repository — only the unit of work exposes it. The transaction boundary is a port call, so a queue consumer could drive the same use case without touching this file.

The endpoint does three things: translate an HTTP request into the port's command, call the port, and translate the result back. No branching on business state, no `DbContext`, no `try/catch` for domain errors.

```csharp
// src/Shop.Adapters.Inbound.Http/Endpoints/PlaceOrderEndpoint.cs
using Shop.Adapters.Inbound.Http.Contracts;
using Shop.Application.UseCases.PlaceOrder;

namespace Shop.Adapters.Inbound.Http.Endpoints;

internal static class PlaceOrderEndpoint
{
    public static RouteHandlerBuilder Map(IEndpointRouteBuilder app) =>
        app.MapPost("/orders", async (
                PlaceOrderRequest request,
                PlaceOrderHandler handler,
                CancellationToken cancellationToken) =>
            {
                var command = new PlaceOrderCommand(
                    request.CustomerId,
                    request.Lines
                        .Select(line => new PlaceOrderLine(line.Sku, line.Quantity, line.UnitPrice, line.Currency))
                        .ToArray());

                var result = await handler.HandleAsync(command, cancellationToken);

                return TypedResults.Created($"/orders/{result.OrderId}", new PlaceOrderResponse(result.OrderId));
            })
            .WithName("PlaceOrder")
            .Produces<PlaceOrderResponse>(StatusCodes.Status201Created)
            .ProducesProblem(StatusCodes.Status400BadRequest)
            .ProducesProblem(StatusCodes.Status409Conflict);
}
```

```csharp
// src/Shop.Adapters.Inbound.Http/OrderEndpoints.cs
using Shop.Adapters.Inbound.Http.Endpoints;

namespace Shop.Adapters.Inbound.Http;

public static class OrderEndpoints
{
    public static IEndpointRouteBuilder MapOrders(this IEndpointRouteBuilder app)
    {
        PlaceOrderEndpoint.Map(app);
        return app;
    }
}
```

Domain failures are reported by one handler in the same adapter. The domain decides *what* failed; the adapter decides *how* to report it, which is why the status-code table lives here and not in `DomainException`.

```csharp
// src/Shop.Adapters.Inbound.Http/Errors/DomainExceptionHandler.cs
internal sealed class DomainExceptionHandler(IProblemDetailsService problemDetails) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
    {
        if (exception is not DomainException domain) return false;

        var status = domain.Code switch
        {
            "order.empty" => StatusCodes.Status400BadRequest,
            "order.not-cancellable" => StatusCodes.Status409Conflict,
            _ => StatusCodes.Status422UnprocessableEntity
        };

        var problem = new ProblemDetails { Status = status, Title = "Request rejected.", Detail = domain.Message };
        problem.Extensions["code"] = domain.Code;

        return await problemDetails.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            ProblemDetails = problem
        });
    }
}
```

## 5. The mapping problem

Four models, four owners. Collapsing any two is what makes a hexagonal codebase feel like extra work with no payoff.

| Model | Lives in | Owned by | Shaped by |
|---|---|---|---|
| `Order`, `OrderLine`, `Money` | `Shop.Domain` | domain | Invariants: no invalid order can be constructed. |
| `PlaceOrderCommand`, `PlaceOrderResult` | `Shop.Application` | application | What the use case needs and returns. |
| `PlaceOrderRequest`, `PlaceOrderResponse` | `Shop.Adapters.Inbound.Http` | inbound adapter | The HTTP contract: versioning, nullability, wire format. |
| `OrderRecord`, `OrderLineRecord` | `Shop.Adapters.Outbound.Persistence` | persistence adapter | Tables, keys, column types, migrations. |

The domain aggregate carries no persistence attributes and no parameterless constructor. Its only concession to the database is a rehydration factory that takes primitives, not EF types:

```csharp
// src/Shop.Domain/Aggregates/Orders/Order.cs
public sealed class Order
{
    private readonly List<OrderLine> _lines;

    private Order(OrderId id, CustomerId customerId, OrderStatus status, DateTimeOffset placedAt, List<OrderLine> lines)
    {
        Id = id;
        CustomerId = customerId;
        Status = status;
        PlacedAt = placedAt;
        _lines = lines;
    }

    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public OrderStatus Status { get; private set; }
    public DateTimeOffset PlacedAt { get; }
    public IReadOnlyList<OrderLine> Lines => _lines;

    public Money Total =>
        Money.Of(_lines.Sum(line => line.LineTotal.Amount), _lines[0].LineTotal.Currency);

    public static Order Place(
        OrderId id, CustomerId customerId, IReadOnlyCollection<OrderLineDraft> drafts, DateTimeOffset placedAt)
    {
        if (drafts.Count == 0)
            throw new DomainException("order.empty", "An order needs at least one line.");

        if (drafts.Select(draft => draft.Currency).Distinct().Count() > 1)
            throw new DomainException("order.mixed-currency", "All order lines must use the same currency.");

        var lines = drafts
            .Select(draft => OrderLine.Create(
                Sku.From(draft.Sku), draft.Quantity, Money.Of(draft.UnitPrice, draft.Currency)))
            .ToList();

        return new Order(id, customerId, OrderStatus.Placed, placedAt, lines);
    }

    // The only persistence-shaped entry point: primitives in, no ORM types, no validation bypass for new orders.
    public static Order Rehydrate(
        OrderId id, CustomerId customerId, OrderStatus status, DateTimeOffset placedAt, IEnumerable<OrderLine> lines) =>
        new(id, customerId, status, placedAt, lines.ToList());

    public void Cancel()
    {
        if (Status != OrderStatus.Placed)
            throw new DomainException("order.not-cancellable", $"Order {Id.Value} is {Status}.");

        Status = OrderStatus.Cancelled;
    }
}
```

EF Core configuration lives in the adapter and targets the record, so no attribute is needed in the domain:

```csharp
// src/Shop.Adapters.Outbound.Persistence/Orders/OrderConfiguration.cs
internal sealed class OrderConfiguration : IEntityTypeConfiguration<OrderRecord>
{
    public void Configure(EntityTypeBuilder<OrderRecord> builder)
    {
        builder.ToTable("orders");
        builder.HasKey(order => order.Id);
        builder.Property(order => order.Status).HasMaxLength(32);
        builder.Property(order => order.TotalAmount).HasPrecision(18, 2);
        builder.Property(order => order.Currency).HasMaxLength(3).IsFixedLength();
        builder.HasMany(order => order.Lines)
               .WithOne()
               .HasForeignKey(line => line.OrderId)
               .OnDelete(DeleteBehavior.Cascade);
    }
}
```

The adapter maps in both directions, in one file, and nothing else in the system knows both shapes:

```csharp
// src/Shop.Adapters.Outbound.Persistence/Orders/OrderMapper.cs
internal static class OrderMapper
{
    public static Order ToDomain(OrderRecord record) =>
        Order.Rehydrate(
            OrderId.From(record.Id),
            CustomerId.From(record.CustomerId),
            Enum.Parse<OrderStatus>(record.Status),
            record.PlacedAt,
            record.Lines.Select(line => OrderLine.Rehydrate(
                Sku.From(line.Sku), line.Quantity, Money.Of(line.UnitPrice, line.Currency))));

    public static OrderRecord ToRecord(Order order) => new()
    {
        Id = order.Id.Value,
        CustomerId = order.CustomerId.Value,
        Status = order.Status.ToString(),
        PlacedAt = order.PlacedAt,
        TotalAmount = order.Total.Amount,
        Currency = order.Total.Currency,
        Lines = order.Lines.Select(line => new OrderLineRecord
        {
            OrderId = order.Id.Value,
            Sku = line.Sku.Value,
            Quantity = line.Quantity,
            UnitPrice = line.UnitPrice.Amount,
            Currency = line.UnitPrice.Currency
        }).ToList()
    };
}
```

Rules that keep this from rotting:

- The mapper is `internal` and lives next to the model it owns. A shared `Mapping` project would need references to both sides and become a hub through which every layer reaches every other layer.
- Do not use a convention-based object mapper for aggregates. Rehydration must go through a factory, and a mapper that sets properties reflectively will eventually construct an invalid aggregate.
- Read models may skip the aggregate entirely: a query that returns a screen-shaped row belongs in a query port, not in `Order`.
- The unit of work is committed by the use case, through `IUnitOfWork`, not by the repository. The adapter owns the transaction mechanics; the application owns where the boundary is.

## 6. What this costs

- Five production projects and four test projects. Adding a feature usually means touching four of them, and the reference graph is reviewed like code.
- Four model types in play and a hand-written mapper. Partial mapping bugs are real: a field added to `Order` and not to `OrderRecord` fails silently on round-trip.
- Architecture tests are a maintenance surface. They fail on legitimate additions until the namespace list is extended, and a team that disables them has paid the cost and lost the benefit.
- For a CRUD service with no invariants, this shape is pure overhead. Keep the API, application, and persistence in one project until the rules get complicated enough to be worth protecting.

## 7. What breaks first

1. **A package lands in `Shop.Domain`.** Someone needs `ILogger`, an EF Core base class, or a Newtonsoft attribute, adds one `PackageReference`, and the domain quietly becomes host-shaped. Detect: the `ForbidOutwardDependencies` target fails the build; if it was removed, `dotnet list src/Shop.Domain/Shop.Domain.csproj package --include-transitive` in CI shows the drift before the new type is even used, and the NetArchTest rule fails as soon as it is.
2. **The inbound adapter reaches the database.** A "simple read" endpoint gets a `ProjectReference` to persistence and a `DbContext`, and the use case is bypassed. Detect: `Inbound_adapter_does_not_reference_driven_adapters` fails on the first pull request; the `ProjectReference` itself is the tell in review, because the adapter has no other reason to want that assembly.
3. **A singleton captures a scoped `DbContext`.** A cache, a background worker, or a handler registered with the wrong lifetime holds one context for the process. Detect: `ValidateScopes` and `ValidateOnBuild` fail at startup with the exact dependency chain. Without them the symptom is intermittent concurrency exceptions in production, which is the most expensive possible detector.
