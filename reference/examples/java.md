# Java Example

> **Read when:** you are applying the standard tier to a Gradle or Maven multi-module service and need the module graph, the ArchUnit rules, the Spring wiring, and the JPA mapping split for one bounded context.

One bounded context, one deployable, six Gradle modules. `bootstrap` is the only module allowed to see both adapters; `domain` and `application` contain no framework code. Maven produces the same graph with one `<module>` per layer and the same `<dependency>` edges in the same direction.

**Options used in this example** (record your own before generating a tree; see `../naming-and-modes.md` §5):

```
Options record
- Application mode:      usecase
- Domain vocabulary:     model=Aggregates (package per aggregate), shared=Primitives, errors=Errors, outbound ports=Ports
- Port location:         application; inbound port is the use-case class, outbound under port/out
- Presentation name:     web (adapter at the edge, hosted by bootstrap)
- Inherited from code:   no — greenfield illustration
- Rationale:             one package per use case keeps transaction and authorization boundaries visible
```

## 1. Tree

```
shop/
  settings.gradle.kts
  build.gradle.kts
  domain/                                     zero dependencies
    src/main/java/com/example/shop/domain/
      order/
        Order.java                            aggregate root
        OrderId.java
        OrderLine.java
        OrderLineDraft.java
        OrderStatus.java
        CustomerId.java
      primitives/
        Money.java
        Sku.java
      errors/
        DomainException.java
  application/                                depends on domain
    src/main/java/com/example/shop/application/
      order/place/
        PlaceOrderService.java                inbound port: the contract the controller calls
        PlaceOrderCommand.java
        PlaceOrderLine.java
        PlaceOrderResult.java
      port/out/
        OrderRepository.java
        UnitOfWork.java
  adapter-in-web/                             depends on application
    src/main/java/com/example/shop/adapter/in/web/
      WebConfiguration.java                   public; the adapter's entry point
      order/
        PlaceOrderController.java             package-private
        PlaceOrderMapper.java                 package-private, MapStruct
        PlaceOrderRequest.java                public record, wire contract
        PlaceOrderResponse.java               public record, wire contract
        PlaceOrderLineRequest.java            public record, wire contract
      error/
        DomainExceptionHandler.java           package-private
  adapter-out-persistence/                    depends on application, domain
    src/main/java/com/example/shop/adapter/out/persistence/
      PersistenceConfiguration.java           public; the adapter's only entry point
      order/
        OrderJpaEntity.java                   package-private @Entity
        OrderLineJpaEntity.java               package-private @Entity
        OrderJpaRepository.java               package-private Spring Data interface
        JpaOrderRepository.java               package-private, implements OrderRepository
        OrderPersistenceMapper.java           package-private, hand-written
      tx/
        TransactionalUnitOfWork.java          package-private
  bootstrap/                                  depends on application, both adapters
    src/main/java/com/example/shop/bootstrap/
      ShopApplication.java                    @SpringBootApplication
      UseCaseConfiguration.java               bean definitions for use cases
    src/main/resources/application.yml
  architecture-tests/                         test-only; no production code
    src/test/java/com/example/shop/architecture/
      HexagonalArchitectureTest.java
```

Java renders the `Aggregates` grouping as one package per aggregate (`domain/order`), which is the idiomatic Java form of the same option; C# nests the aggregate folders under `Aggregates/`. The shape is identical either way: no framework type may appear under `domain`.

Module dependencies and what each edge buys:

| Module | Depends on | Declared as | Consequence |
|---|---|---|---|
| `domain` | nothing | — | No Spring, JPA, Jackson, or logging facade on its compile classpath. |
| `application` | `domain` | `api` | Domain types appear in port signatures, so consumers must see them. |
| `adapter-in-web` | `application` | `implementation` | It cannot name a repository or an entity — neither module is on its path. |
| `adapter-out-persistence` | `application`, `domain` | `implementation` | Implements the port; mapping needs domain rehydration. |
| `bootstrap` | `application`, both adapters | `implementation` | The only module that can wire a port to its implementation. |
| `architecture-tests` | all | `testImplementation` | Test-only; never packaged. |

## 2. Enforcing the direction

### Build time: the module graph

```kotlin
// settings.gradle.kts
rootProject.name = "shop"

include(
    "domain",
    "application",
    "adapter-in-web",
    "adapter-out-persistence",
    "bootstrap",
    "architecture-tests",
)
```

```kotlin
// build.gradle.kts
subprojects {
    apply(plugin = "java")

    repositories { mavenCentral() }

    dependencies {
        // Version alignment only: a BOM contributes constraints, not classes, so domain stays empty.
        "implementation"(platform("org.springframework.boot:spring-boot-dependencies:3.5.4"))
    }

    extensions.configure<JavaPluginExtension> {
        toolchain { languageVersion = JavaLanguageVersion.of(21) }
    }

    tasks.withType<Test>().configureEach { useJUnitPlatform() }
}
```

```kotlin
// application/build.gradle.kts
plugins {
    `java-library`
}

dependencies {
    // api, not implementation: port signatures expose domain types to every adapter.
    api(project(":domain"))
}
```

```kotlin
// domain/build.gradle.kts
plugins {
    `java-library`
}

dependencies {
    // Deliberately empty. Adding anything here is the review signal.
}
```

Three properties of the graph do the real work:

1. **Cycles are unbuildable.** Gradle fails with `Circular dependency between the following tasks` (Maven fails the reactor build). A domain that calls a repository, or an adapter that calls another adapter in both directions, cannot be expressed.
2. **`api` versus `implementation` is the export surface, and it is enforced by the compiler.** If `application` declared `implementation(project(":domain"))` while `OrderRepository.findById` returns `Order`, every adapter fails with `cannot find symbol: class Order`. That compile error is what keeps the port contract honest.
3. **The classpath is minimal by construction.** `domain` has no `spring-context`, so `@Service` on a domain class does not compile. `adapter-in-web` has no `spring-boot-starter-data-jpa`, so no `EntityManager` can appear in a controller. These are the cheapest possible failures: they happen before a test exists.

The graph still does not express "no adapter may depend on another adapter" as a *rule* — it only makes the reverse edge fail. Someone can add `implementation(project(":adapter-out-persistence"))` to `adapter-in-web` and the build stays green. That gap belongs to ArchUnit.

### Test time: ArchUnit

```java
// architecture-tests/src/test/java/com/example/shop/architecture/HexagonalArchitectureTest.java
package com.example.shop.architecture;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;
import static com.tngtech.archunit.library.Architectures.layeredArchitecture;
import static com.tngtech.archunit.library.dependencies.SlicesRuleDefinition.slices;

import com.tngtech.archunit.core.importer.ImportOption;
import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;

@AnalyzeClasses(packages = "com.example.shop", importOptions = ImportOption.DoNotIncludeTests.class)
class HexagonalArchitectureTest {

    @ArchTest
    static final ArchRule layers = layeredArchitecture()
        .consideringAllDependencies()
        .layer("Domain").definedBy("com.example.shop.domain..")
        .layer("Application").definedBy("com.example.shop.application..")
        .layer("Inbound").definedBy("com.example.shop.adapter.in..")
        .layer("Outbound").definedBy("com.example.shop.adapter.out..")
        .layer("Bootstrap").definedBy("com.example.shop.bootstrap..")
        .whereLayer("Domain").mayOnlyBeAccessedByLayers("Application", "Outbound")
        .whereLayer("Application").mayOnlyBeAccessedByLayers("Inbound", "Outbound", "Bootstrap")
        .whereLayer("Inbound").mayOnlyBeAccessedByLayers("Bootstrap")
        .whereLayer("Outbound").mayOnlyBeAccessedByLayers("Bootstrap")
        .whereLayer("Bootstrap").mayNotBeAccessedByAnyLayer()
        .because("dependencies point inward, and only the composition root knows both sides");

    @ArchTest
    static final ArchRule domain_is_technology_free = noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat().resideInAnyPackage(
            "org.springframework..",
            "jakarta.persistence..",
            "org.hibernate..",
            "com.fasterxml.jackson..",
            "java.sql..",
            "javax.sql..");

    @ArchTest
    static final ArchRule no_cycles_between_modules = slices()
        .matching("com.example.shop.(*)..")
        .should().beFreeOfCycles()
        .because("a cycle between modules defeats the inward rule even when every single edge looks legal");
}
```

Notes on these rules:

- The layered rule expresses the whole matrix, including adapter-to-adapter: `Inbound` may only be accessed by `Bootstrap`, so a controller calling a public type in the persistence adapter is a violation, not a style question. Today the module's package-private classes already block that call; the rule is the guard for the day someone makes a DAO public.
- `Domain` may be accessed by `Outbound` because port signatures and rehydration use domain types. That edge still points inward. What the rule forbids is anything from the domain pointing out.
- Annotation usage is a bytecode dependency, so `@Service` or `@Entity` in a domain package trips `domain_is_technology_free` even if the type is otherwise untouched.
- The slices rule covers cycles at package level inside a module. Gradle already prevents cycles between modules, but `com.example.shop.domain.order` and `com.example.shop.domain.billing` can still form one.
- ArchUnit's own analysis is not free: import only `com.example.shop`, exclude tests, and load the rules once per class.

Two more checks that cost nothing:

```bash
# The domain jar must need nothing but the JDK base module.
jdeps --print-module-deps --multi-release 21 domain/build/libs/domain.jar
# expected output: java.base
```

```kotlin
// build.gradle.kts (root)
plugins {
    id("com.autonomousapps.dependency-analysis") version "2.1.0"
}
```

The dependency-analysis plugin fails the build on *declared but unused* and *used but undeclared* module dependencies. It catches the slow drift the compiler cannot see: a module that keeps a dependency after the last import was deleted, or an adapter that compiles only because a transitive edge happens to be on the path.

### Maven equivalent

One `<module>` per directory, the same `<dependency>` edges, and `maven-enforcer-plugin` with a `bannedDependencies` rule on the `domain` module so that `org.springframework:*` and `jakarta.persistence:*` fail the build. `maven-dependency-plugin:analyze` plays the role of dependency-analysis.

## 3. Composition root

Spring wiring lives in `bootstrap` and in one configuration class per adapter. No domain or application class carries a stereotype annotation.

```java
// bootstrap/src/main/java/com/example/shop/bootstrap/ShopApplication.java
@SpringBootApplication(scanBasePackages = "com.example.shop.bootstrap")
@Import({WebConfiguration.class, PersistenceConfiguration.class})
public class ShopApplication {

    public static void main(String[] args) {
        SpringApplication.run(ShopApplication.class, args);
    }
}
```

`scanBasePackages` is narrowed to the bootstrap package on purpose. The application class happens to sit there today; if it ever moves to `com.example.shop`, the explicit value stops the move from silently turning the whole codebase into a component-scan target. Everything outside `bootstrap` is imported explicitly, which is the only way an adapter's beans get created.

```java
// bootstrap/src/main/java/com/example/shop/bootstrap/UseCaseConfiguration.java
@Configuration(proxyBeanMethods = false)
class UseCaseConfiguration {

    @Bean
    Clock clock() {
        return Clock.systemUTC();
    }

    @Bean
    PlaceOrderService placeOrderService(OrderRepository orders, UnitOfWork unitOfWork, Clock clock) {
        return new PlaceOrderService(orders, unitOfWork, clock);
    }
}
```

This is the whole point of the split. `PlaceOrderService` is a plain class with constructor arguments, and it is also the inbound port: the controller depends on this class, not on an interface added for symmetry. It is a bean because the composition root says so, not because it is annotated. Adding `@Service` to it would force `spring-context` into `application`, and the module rule would fail. If a project decides to accept that trade, it must also accept that the application layer can now use `@Transactional` — a real convenience with a real cost: the framework's proxy semantics become part of the application's behaviour, and `application` stops being testable as plain Java.

Each adapter ships exactly one configuration class and never scans outside its own package:

```java
// adapter-out-persistence/src/main/java/com/example/shop/adapter/out/persistence/PersistenceConfiguration.java
@Configuration(proxyBeanMethods = false)
@ComponentScan(basePackageClasses = PersistenceConfiguration.class)
@EnableJpaRepositories(basePackageClasses = OrderJpaRepository.class)
@EntityScan(basePackageClasses = OrderJpaEntity.class)
public class PersistenceConfiguration {

    @Bean
    UnitOfWork unitOfWork(PlatformTransactionManager transactionManager) {
        return new TransactionalUnitOfWork(transactionManager);
    }
}
```

```java
// adapter-in-web/src/main/java/com/example/shop/adapter/in/web/WebConfiguration.java
@Configuration(proxyBeanMethods = false)
@ComponentScan(basePackageClasses = WebConfiguration.class)
public class WebConfiguration {
}
```

The transaction port is implemented here, not in `application`, so the use case never names a Spring type:

```java
// adapter-out-persistence/.../tx/TransactionalUnitOfWork.java
final class TransactionalUnitOfWork implements UnitOfWork {

    private final TransactionTemplate template;

    TransactionalUnitOfWork(PlatformTransactionManager transactionManager) {
        this.template = new TransactionTemplate(transactionManager);
    }

    @Override
    public <T> T inTransaction(Supplier<T> work) {
        return template.execute(status -> work.get());
    }
}
```

Lifetimes are simple here, and for a reason worth knowing: Spring beans default to singleton, and `EntityManager` is a shared proxy that binds to the current transaction rather than a per-request object. There is no direct equivalent of .NET's captive `DbContext`; a singleton repository is correct. The failure this stack invites instead is a lazily loaded entity graph escaping the adapter, which is why the configuration below is not optional:

```yaml
# bootstrap/src/main/resources/application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/shop
  jpa:
    open-in-view: false
```

With open-session-in-view left on, a controller can touch a lazy collection after the repository returned, the transaction is already committed, and the query silently runs during serialization. That is the adapter boundary being bypassed at runtime, with no compile error and no failing architecture test. Turning it off converts the leak into a `LazyInitializationException` at the exact line that broke the rule.

## 4. The driving adapter

The controller translates HTTP to the inbound port. Transaction boundaries, invariants, and error codes are not its business.

```java
// adapter-in-web/.../order/PlaceOrderController.java
@RestController
class PlaceOrderController {

    private final PlaceOrderService placeOrder;
    private final PlaceOrderMapper mapper;

    PlaceOrderController(PlaceOrderService placeOrder, PlaceOrderMapper mapper) {
        this.placeOrder = placeOrder;
        this.mapper = mapper;
    }

    @PostMapping("/orders")
    ResponseEntity<PlaceOrderResponse> place(@RequestBody PlaceOrderRequest request) {
        var result = placeOrder.handle(mapper.toCommand(request));
        var location = URI.create("/orders/" + result.orderId());

        return ResponseEntity.created(location).body(new PlaceOrderResponse(result.orderId()));
    }
}
```

```java
// adapter-in-web/.../order/PlaceOrderMapper.java
@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
interface PlaceOrderMapper {

    PlaceOrderCommand toCommand(PlaceOrderRequest request);
}
```

MapStruct is a good fit here: flat request records in, a command record out, no invariants involved. It is a poor fit for rehydrating an aggregate — a generated constructor call bypasses the factory that validates it, so the persistence mapper below stays hand-written.

The domain error code becomes an HTTP status in the adapter, not in `DomainException`:

```java
// adapter-in-web/.../error/DomainExceptionHandler.java
@RestControllerAdvice
class DomainExceptionHandler {

    @ExceptionHandler(DomainException.class)
    ProblemDetail onDomainException(DomainException exception) {
        var status = switch (exception.code()) {
            case "order.empty" -> HttpStatus.BAD_REQUEST;
            case "order.not-cancellable" -> HttpStatus.CONFLICT;
            default -> HttpStatus.UNPROCESSABLE_ENTITY;
        };

        var problem = ProblemDetail.forStatusAndDetail(status, exception.getMessage());
        problem.setTitle("Request rejected.");
        problem.setProperty("code", exception.code());
        return problem;
    }
}
```

The use case owns the transaction, and the port stays framework-free:

```java
// application/.../order/place/PlaceOrderService.java
public final class PlaceOrderService {

    private final OrderRepository orders;
    private final UnitOfWork unitOfWork;
    private final Clock clock;

    public PlaceOrderService(OrderRepository orders, UnitOfWork unitOfWork, Clock clock) {
        this.orders = orders;
        this.unitOfWork = unitOfWork;
        this.clock = clock;
    }

    public PlaceOrderResult handle(PlaceOrderCommand command) {
        return unitOfWork.inTransaction(() -> {
            var order = Order.place(
                OrderId.newId(),
                new CustomerId(command.customerId()),
                command.lines().stream()
                    .map(line -> new OrderLineDraft(line.sku(), line.quantity(), line.unitPrice(), line.currency()))
                    .toList(),
                clock.instant());

            orders.save(order);
            return new PlaceOrderResult(order.id().value());
        });
    }
}
```

## 5. The mapping problem

Four models, four owners, and one mapper that is allowed to know two of them.

| Model | Lives in | Owned by | Shaped by |
|---|---|---|---|
| `Order`, `OrderLine`, `Money` | `domain` | domain | Invariants; no annotations, no parameterless constructor, no setters. |
| `PlaceOrderCommand`, `PlaceOrderResult` | `application` | application | What the use case needs and returns. |
| `PlaceOrderRequest`, `PlaceOrderResponse` | `adapter-in-web` | inbound adapter | The HTTP contract; versioned with the API. |
| `OrderJpaEntity`, `OrderLineJpaEntity` | `adapter-out-persistence` | persistence adapter | Tables, joins, column types, lazy loading. |

`@Entity` in the domain leaks in five ways, and all five are structural rather than cosmetic:

1. **JPA needs a no-argument constructor.** The aggregate gets one, and with it a way to exist in an invalid state.
2. **JPA needs mutable, non-final fields.** Value objects stop being immutable, and the compiler stops helping.
3. **JPA proxies subclass the entity.** Lazy associations arrive as `PersistentBag` and `HibernateProxy`; `equals`/`hashCode` and `instanceof` start behaving differently depending on session state.
4. **The domain module needs `jakarta.persistence-api`.** The `jdeps` check stops printing `java.base` alone, and the ArchUnit rule fails. The rule is not the problem; the dependency is.
5. **Schema changes become domain changes.** Renaming a column edits the aggregate, so a database migration now needs a domain review.

The persistence model is a separate, deliberately dumb class:

```java
// adapter-out-persistence/.../order/OrderJpaEntity.java
//
// Fields are package-private on purpose: the class is package-private too, so nothing outside
// this package can see them, and the mapper avoids a getter/setter layer that exists only for JPA.
@Entity
@Table(name = "orders")
class OrderJpaEntity {

    @Id
    UUID id;

    @Column(name = "customer_id", nullable = false)
    UUID customerId;

    @Column(nullable = false, length = 32)
    String status;

    @Column(name = "placed_at", nullable = false)
    Instant placedAt;

    @Column(name = "total_amount", nullable = false, precision = 18, scale = 2)
    BigDecimal totalAmount;

    @Column(nullable = false, length = 3)
    String currency;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "order_id", nullable = false)
    List<OrderLineJpaEntity> lines = new ArrayList<>();

    OrderJpaEntity() {
        // required by JPA; the application never calls it
    }
}
```

The mapper is package-private, sits next to the entity, and is the only class that mentions both shapes:

```java
// adapter-out-persistence/.../order/OrderPersistenceMapper.java
final class OrderPersistenceMapper {

    private OrderPersistenceMapper() {
    }

    static Order toDomain(OrderJpaEntity entity) {
        var lines = entity.lines.stream()
            .map(line -> new OrderLine(Sku.of(line.sku), line.quantity, Money.of(line.unitPrice, line.currency)))
            .toList();

        return Order.rehydrate(
            new OrderId(entity.id),
            new CustomerId(entity.customerId),
            OrderStatus.valueOf(entity.status),
            entity.placedAt,
            lines);
    }

    static OrderJpaEntity toEntity(Order order) {
        var entity = new OrderJpaEntity();
        entity.id = order.id().value();
        entity.customerId = order.customerId().value();
        entity.status = order.status().name();
        entity.placedAt = order.placedAt();
        entity.totalAmount = order.total().amount();
        entity.currency = order.total().currency();

        order.lines().forEach(line -> {
            var lineEntity = new OrderLineJpaEntity();
            lineEntity.sku = line.sku().value();
            lineEntity.quantity = line.quantity();
            lineEntity.unitPrice = line.unitPrice().amount();
            lineEntity.currency = line.unitPrice().currency();
            entity.lines.add(lineEntity);
        });

        return entity;
    }
}
```

```java
// adapter-out-persistence/.../order/JpaOrderRepository.java
@Repository
class JpaOrderRepository implements OrderRepository {

    private final OrderJpaRepository jpa;

    JpaOrderRepository(OrderJpaRepository jpa) {
        this.jpa = jpa;
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpa.findById(id.value()).map(OrderPersistenceMapper::toDomain);
    }

    @Override
    public void save(Order order) {
        jpa.save(OrderPersistenceMapper.toEntity(order));
    }
}
```

Rules that keep this from rotting:

- The mapper is package-private and lives with the entity. A shared `mapping` module would need both sides and become a hub through which every layer reaches every other layer.
- `Order.rehydrate` takes primitives and domain types, never an entity. If it takes an `OrderJpaEntity`, the domain compiles against JPA and the module boundary is decorative.
- Reads that return a screen shape should not load the aggregate. Put them in a query port with its own row type, and do not reuse `Order` as a view model.
- The use case opens the transaction through `UnitOfWork`; the repository never commits. The adapter owns the mechanics, the application owns the boundary.
- `open-in-view: false` is part of the mapping design, not a performance tweak. Without it, "the entity never leaves the adapter" is unenforced.

## 6. JPMS or build modules?

Use build modules. JPMS would give compile-time `requires`, a hard `exports` surface, and a real `module-info.java` per layer — but the cost lands in the wrong place:

- Spring, Hibernate, and Jackson rely on reflection and deep access. Every adapter's `module-info.java` grows into an `opens` allowlist of framework internals, which is exactly the kind of configuration that silently drifts.
- Parts of the stack still ship as non-modular jars, so you end up with automatic modules and `--add-opens` flags. That is enforcement nobody can reason about.
- The rule that matters — domain has no framework, adapters do not see each other, wiring happens once — is already a property of the build graph plus ArchUnit. JPMS adds a third mechanism that checks roughly the same thing at a higher maintenance cost.

Keep a JPMS-grade check without JPMS: run `jdeps --print-module-deps` on the domain jar in CI and fail if the output is anything other than `java.base`. That single line catches the most expensive drift in the whole design — a framework reaching the core — and it costs one command.

## 7. What this costs

- Six modules, one build file each plus the root build. A new module means editing `settings.gradle.kts` and every module that must see it.
- One configuration class per adapter plus the use-case configuration in `bootstrap`, and the wiring is invisible from the use case. Reading `PlaceOrderService` no longer tells you who calls it.
- Four model shapes and a hand-written persistence mapper. Every field added to the aggregate is a field that can be forgotten in the mapper, and the failure is silent.
- ArchUnit rules need maintenance and are slow enough that teams disable them under deadline pressure.
- For a CRUD service with no invariants this is overhead. Keep one module until the invariants justify the split.

## 8. What breaks first

1. **`@Entity` moves into the domain.** Someone avoids the mapper by annotating the aggregate, and the aggregate gains a no-arg constructor and mutable fields. Detect: the `domain` module does not compile until `jakarta.persistence-api` is added to its build file; if it is added, `domain_is_technology_free` fails on the first run, and `jdeps --print-module-deps` stops printing `java.base` alone.
2. **Domain classes become Spring beans.** A domain service needs a repository, so it gets `@Service` and constructor injection, and the domain now depends on the container. Detect: it does not compile without `spring-context` in `domain/build.gradle.kts`. If the dependency is added to make it compile, the ArchUnit rule fails, and the narrowed `scanBasePackages` means the bean is not created at all — if something injects it, startup fails instead of quietly running a domain object under Spring's control.
3. **An adapter reaches into another adapter.** The web adapter needs a read query, so it depends on the persistence module and calls a public type there — a query DAO, or an entity someone made public to avoid the mapper. Detect: `layers` fails because `Outbound` may only be accessed by `Bootstrap`. In the Gradle graph the edge is legal, so the module build stays green — this is the one violation that only the architecture test catches, which is why the test is not optional.
