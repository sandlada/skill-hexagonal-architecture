# Angular Example

> **Read when:** you are applying the dependency rule inside an Angular application and need the framework-specific mapping: what is a driving adapter, what is a driven adapter, where DI tokens live, and how to stop a component from talking to `HttpClient`.

Angular does not map onto the backend layout by renaming folders. The arrows invert. A backend's `presentation` or `api` layer is the inbound edge, and its HTTP client, when it has one, is a driven adapter in infrastructure. In Angular the inbound edge is the **component**, and the HTTP client is the **driven adapter**. Copying the backend's `presentation/` and `api/` folders into an Angular app produces a structure where the transport sits in the wrong direction.

This example uses Angular 22 (standalone components, signals, zoneless by default). The domain is plain TypeScript; the framework stops at the application layer's DI.

**Options used in this example** (record your own before generating a tree; see `../naming-and-modes.md` §5):

```
Options record
- Application mode:      usecase
- Domain vocabulary:     model=Entities, shared=Primitives, errors=Errors, outbound ports=Ports
- Port location:         application (interface and InjectionToken together)
- Presentation name:     web - ui/ folders inside each feature, no separate presentation project
- Inherited from code:   no - greenfield illustration
- Rationale:             the component is the driving adapter, so presentation is a layer inside the feature
```

## 1. The mapping, stated explicitly

| Hexagonal concept | Angular artifact | Allowed to import |
|---|---|---|
| Domain | plain classes and types in `domain/` | domain only |
| Application | facade or use-case class in `application/` | domain, `@angular/core` (signals, DI), its ports |
| Inbound port | the facade's public signals and intent methods | — |
| Outbound port | interface + `InjectionToken` in `application/ports/` | domain types |
| Driving adapter | standalone component in `ui/` | application, domain display types, Angular component/router APIs |
| Driven adapter | HTTP gateway in `adapters/http/` | port interface, domain, `HttpClient`, DTOs |
| Composition root | `app.config.ts` or route `providers` | everything |

Two rules make the table enforceable:

1. **Domain imports no `@angular/*` and no `rxjs`.** Not a type, not a token, not `signal`. A framework-free domain is the reason the rest of the layout has a direction.
2. **A component never injects `HttpClient`.** The component is the driving adapter; the HTTP client is a driven adapter. A component that does both collapses the two ends of the hexagon into one file.

The application layer is allowed to import `@angular/core` because a facade is already an Angular construct: it uses `inject`, `signal`, and `resource`. What it must not import is `@angular/common/http`, `@angular/router`, or a component. Those belong to adapters and UI.

### Why backend naming does not transfer

- `presentation` / `api` in a backend means "HTTP controllers, the inbound edge". In Angular, `ui/` is that edge.
- A folder called `api/` in an Angular app almost always means "the API client", which is a **driven** adapter. Using the backend's `api` label for the UI inverts the arrow.
- There is no "presentation project" to place beside `domain` and `application`. The UI is a layer inside the feature, and it depends inward like any other adapter.

If you inherit a folder named `api/`, inspect it before mapping. If it contains `HttpClient` calls and DTOs, it is `adapters/http/`. If it contains components, it is `ui/`. Do not keep the name and hope the direction is obvious.

## 2. Feature tree

One Nx project per feature, layers as folders inside it:

```text
libs/orders/
├── project.json                     tags: ["type:feature", "scope:orders"]
└── src/
    ├── index.ts                     public API: ORDERS_ROUTES only
    └── lib/
        ├── orders.routes.ts         route-level composition root
        ├── domain/
        │   ├── entities/
        │   │   ├── order.ts
        │   │   └── order-line.ts
        │   ├── primitives/
        │   │   ├── order-id.ts
        │   │   └── money.ts
        │   └── errors/
        │       └── order-errors.ts
        ├── application/
        │   ├── ports/
        │   │   └── order-gateway.port.ts
        │   └── orders.facade.ts
        ├── adapters/
        │   └── http/
        │       ├── http-order.gateway.ts
        │       ├── order.dto.ts
        │       └── order.mapper.ts
        └── ui/
            ├── orders-page.component.ts
            └── orders-page.component.html
```

The app shell stays empty of feature knowledge:

```text
apps/shop/src/app/
├── app.config.ts
├── app.routes.ts
├── app.component.ts
└── main.ts
```

```ts
// apps/shop/src/app/app.routes.ts
import type { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: 'orders',
    loadChildren: () => import('@shop/orders').then((m) => m.ORDERS_ROUTES),
  },
  { path: '', pathMatch: 'full', redirectTo: 'orders' },
];
```

`libs/orders/src/index.ts` exports the route array and nothing else:

```ts
export { ORDERS_ROUTES } from './lib/orders.routes';
```

That is the export surface of the feature. Another feature cannot import `OrdersPageComponent`, reach into its facade, or depend on its DTOs. The cost is that cross-feature UI reuse must go through a shared UI library rather than a relative import; that is the intended boundary, not an accident.

**Why split the folders at all.** The domain can be tested without `TestBed`. The application can be tested with a fake port and no `HttpTestingController`. The adapter can be replaced with a GraphQL or offline implementation without touching the component. The UI can be rendered against a fake facade.

**When not to use it.** A three-screen feature with no rules beyond rendering does not need four folders. A single component plus a thin gateway is cheaper and easier to review. Adopt the split when the domain has invariants that change independently of the screens, or when the transport is likely to change.

## 3. When a shared domain package is justified

A shared domain library is a **shared kernel**: one model deliberately used by several features. It is justified when all of the following hold:

- Two or more features enforce the same invariant on the same concept — for example, `Money` rounding and currency mismatch rules.
- The model has behavior, not only fields. A bag of properties with no rules is a DTO, not a domain model.
- The features are in the same bounded context. Different contexts need an anti-corruption layer, not a shared model.
- A change must land in every consumer at once; duplicating it would let the copies drift.

```text
libs/shared/domain/
├── project.json                     tags: ["type:domain", "scope:shared"]
└── src/
    ├── index.ts
    └── lib/
        └── primitives/
            ├── money.ts
            └── currency.ts
```

It is **not** justified when:

- Only the JSON shape is shared. Share a generated API client or the DTOs in `adapters/http/`, tagged as an adapter, not as domain.
- The shared model is a set of fields used by two screens. Duplicate the view model; a shared kernel that exists for convenience becomes a coordination point for every feature.
- The two features are different bounded contexts with different meanings for the same word. Translate at the boundary instead.

**Cost.** A shared domain library serializes delivery: every consumer rebuilds and retests when it changes, and a wrong abstraction is much more expensive to unwind than duplicated code. Keep it small, stable, and owned. If it starts importing Angular or a feature's adapter, it has stopped being a shared kernel and become a second application layer.

## 4. The port: an interface plus an `InjectionToken`

```ts
// libs/orders/src/lib/application/ports/order-gateway.port.ts
import { InjectionToken } from '@angular/core';
import type { Order } from '../../domain/entities/order';
import type { OrderId } from '../../domain/primitives/order-id';

export interface OrderGateway {
  list(): Promise<readonly Order[]>;
  place(order: Order): Promise<OrderId>;
}

export const ORDER_GATEWAY = new InjectionToken<OrderGateway>('OrderGateway');
```

The token lives in the **application** layer, next to the interface. It cannot live in the domain: `InjectionToken` is an `@angular/core` import, and a domain that imports Angular has already broken the one rule the layout exists to enforce.

This is a recorded vocabulary decision (`application/ports`). Do not also declare the interface in `domain/ports`; one contract, one location. If the application later needs to run outside Angular, that is a new port decision, not a reason to split this one across two folders.

## 5. Composition root: `app.config.ts` or the route

Application-wide binding:

```ts
// apps/shop/src/app/app.config.ts
import type { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient } from '@angular/common/http';
import { API_BASE_URL } from '@shop/shared/config';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(),
    { provide: API_BASE_URL, useValue: '/api' },
  ],
};
```

Feature-scoped binding, in the feature's route:

```ts
// libs/orders/src/lib/orders.routes.ts
import type { Routes } from '@angular/router';
import { OrdersFacade } from './application/orders.facade';
import { ORDER_GATEWAY } from './application/ports/order-gateway.port';
import { HttpOrderGateway } from './adapters/http/http-order.gateway';

export const ORDERS_ROUTES: Routes = [
  {
    path: '',
    providers: [
      OrdersFacade,
      { provide: ORDER_GATEWAY, useClass: HttpOrderGateway },
    ],
    loadComponent: () =>
      import('./ui/orders-page.component').then((m) => m.OrdersPageComponent),
  },
];
```

Which one to use:

- **Route-level** when the state belongs to the feature and should die with it. The route creates an injector; navigating away destroys the facade and its `resource`, cancelling in-flight work. This is the default for feature state.
- **`app.config.ts`** when the state is genuinely application-wide and must survive navigation — session, cart, theme. The cost is that it also survives every feature teardown, so stale data becomes possible.

Exactly one provider per token per injector branch. Providing `OrdersFacade` in `app.config.ts` and again on the route creates two instances; the route one wins for the component, the root one keeps its own state, and the two diverge silently. The same applies to `HttpOrderGateway` if it carries `providedIn: 'root'` and is also bound with `useClass`.

## 6. State: a facade with `resource`, not a component with `HttpClient`

```ts
// libs/orders/src/lib/application/orders.facade.ts
import { computed, inject, Injectable, resource } from '@angular/core';
import type { Order } from '../domain/entities/order';
import { ORDER_GATEWAY } from './ports/order-gateway.port';

@Injectable()
export class OrdersFacade {
  private readonly gateway = inject(ORDER_GATEWAY);

  readonly orders = resource({
    loader: () => this.gateway.list(),
  });

  readonly count = computed(() => this.orders.value()?.length ?? 0);

  async place(order: Order): Promise<void> {
    await this.gateway.place(order);
    await this.orders.reload();
  }

  reload(): void {
    void this.orders.reload();
  }
}
```

`@Injectable()` without `providedIn` is deliberate. The facade is stateful and feature-scoped; the route decides its lifetime. `resource` is created in the facade's injection context, so it is destroyed with the route injector and its pending request is cancelled.

Where state belongs:

- **Feature state** — the facade provided at the feature route. It dies with the feature and cannot leak into another route.
- **Application state** — a root-provided service, only for data that must survive navigation. Keep it small and explicit.
- **Component state** — a local `signal` for open/closed, hover, draft input. It does not belong in the facade.
- **Server cache** — the `resource` in the facade, or a query library in the adapter/application boundary. Do not keep a second copy in a component field.

### Why the component must not call `HttpClient`

```ts
// What not to do
private readonly http = inject(HttpClient);
readonly orders = toSignal(this.http.get<OrderDto[]>('/api/orders'));
```

That one line makes the component both the driving adapter and the driven adapter. The consequences are concrete:

- The port is bypassed, so swapping the transport for a fake requires `provideHttpClientTesting` and URL matching instead of a one-line provider.
- The DTO shape reaches the template. A snake_case field or a nested envelope becomes a UI concern.
- Two components that show orders fetch independently; there is no single place to cache, cancel, or invalidate.
- Error and loading handling fork per component, and SSR has no single pending task to track.
- The component now depends on `@angular/common/http`, which the lint rule in §8 can ban for `ui/` without banning it for `adapters/http/`.

## 7. The HTTP adapter and the mapper

The adapter converts a transport payload into a domain model. The domain side is plain TypeScript:

```ts
// libs/orders/src/lib/domain/entities/order.ts
import { EmptyOrderError, InvalidOrderStatusError } from '../errors/order-errors';
import { Money } from '../primitives/money';
import type { OrderId } from '../primitives/order-id';
import { OrderLine } from './order-line';

const ORDER_STATUSES = ['draft', 'placed', 'cancelled'] as const;
export type OrderStatus = (typeof ORDER_STATUSES)[number];

export function parseOrderStatus(value: string): OrderStatus {
  if (!ORDER_STATUSES.includes(value as OrderStatus)) {
    throw new InvalidOrderStatusError(value);
  }
  return value as OrderStatus;
}

export interface OrderProps {
  id: OrderId;
  status: OrderStatus;
  lines: readonly OrderLine[];
}

export class Order {
  private constructor(private readonly props: OrderProps) {}

  static rehydrate(props: OrderProps): Order {
    if (props.lines.length === 0) throw new EmptyOrderError();
    return new Order(props);
  }

  get id(): OrderId {
    return this.props.id;
  }

  get status(): OrderStatus {
    return this.props.status;
  }

  get lines(): readonly OrderLine[] {
    return this.props.lines;
  }

  get total(): Money {
    return this.props.lines.reduce(
      (sum, line) => sum.add(line.lineTotal),
      Money.ofMinor(0, this.props.lines[0].unitPrice.currency),
    );
  }
}
```

`Money` and `OrderLine` validate their inputs — `Money.ofMinor` rejects a negative amount or a malformed currency, `OrderLine.of` rejects a non-positive quantity — and neither imports Angular. `OrderId` wraps a non-empty string. None of these types has a decorator, an `@Injectable`, or an `rxjs` import.

The DTO is owned by the adapter. Its `status` is `string`, not a domain union, because the server is outside the model and its response must be validated before it becomes a domain value.

```ts
// libs/orders/src/lib/adapters/http/order.dto.ts
export interface OrderDto {
  id: string;
  status: string;
  total: { currency: string };
  lines: Array<{ sku: string; quantity: number; unit_price_minor: number }>;
}

export interface PlaceOrderResponseDto {
  id: string;
}
```

```ts
// libs/orders/src/lib/adapters/http/order.mapper.ts
import { Money } from '../../domain/primitives/money';
import { Order, parseOrderStatus } from '../../domain/entities/order';
import { OrderId } from '../../domain/primitives/order-id';
import { OrderLine } from '../../domain/entities/order-line';
import type { OrderDto } from './order.dto';

export function toOrder(dto: OrderDto): Order {
  return Order.rehydrate({
    id: OrderId.of(dto.id),
    status: parseOrderStatus(dto.status),
    lines: dto.lines.map((line) =>
      OrderLine.of({
        sku: line.sku,
        quantity: line.quantity,
        unitPrice: Money.ofMinor(line.unit_price_minor, dto.total.currency),
      }),
    ),
  });
}
```

```ts
// libs/orders/src/lib/adapters/http/http-order.gateway.ts
import { HttpClient } from '@angular/common/http';
import { inject, Injectable } from '@angular/core';
import { firstValueFrom } from 'rxjs';
import { API_BASE_URL } from '@shop/shared/config';
import type { OrderGateway } from '../../application/ports/order-gateway.port';
import type { Order } from '../../domain/entities/order';
import { OrderId } from '../../domain/primitives/order-id';
import { toOrder } from './order.mapper';
import type { OrderDto, PlaceOrderResponseDto } from './order.dto';

@Injectable()
export class HttpOrderGateway implements OrderGateway {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = inject(API_BASE_URL);

  async list(): Promise<readonly Order[]> {
    const dtos = await firstValueFrom(
      this.http.get<OrderDto[]>(`${this.baseUrl}/orders`),
    );
    return dtos.map(toOrder);
  }

  async place(order: Order): Promise<OrderId> {
    const response = await firstValueFrom(
      this.http.post<PlaceOrderResponseDto>(`${this.baseUrl}/orders`, {
        lines: order.lines.map((line) => ({
          sku: line.sku,
          quantity: line.quantity,
          unit_price_minor: line.unitPrice.minor,
        })),
      }),
    );
    return OrderId.of(response.id);
  }
}
```

The port returns domain types. `firstValueFrom` is an adapter detail; the facade never sees an `Observable`. The DTOs and the mapper are internal to `adapters/http/`, not re-exported from the feature's `index.ts`.

`HttpClient` is available because `provideHttpClient()` is in `app.config.ts`. In a component test, the fake gateway replaces the whole adapter, so no HTTP provider is needed. If a test does need the real adapter, add `provideHttpClientTesting()` there — not in the application config.

## 8. Enforcement

### 8.1 What Nx can and cannot see

`@nx/enforce-module-boundaries` compares **projects**, not folders. The tree in §2 is one project (`@shop/orders`), so the rule can stop another feature from importing it, but it cannot see the `domain/` versus `ui/` split inside it. Tagging the lib `type:feature` and calling the inner layers "enforced" is a common false claim.

Two options, with different costs:

**Keep one project per feature.** Use Nx for feature scope and the ESLint element rules in §8.3 for the inner layers. Cheap; lint-only for the layers.

**Promote each layer to a project.** Nx then enforces the layer tags directly, at the cost of four projects per feature:

```text
libs/orders/domain/            @shop/orders-domain          type:domain, scope:orders
libs/orders/application/       @shop/orders-application     type:application, scope:orders
libs/orders/adapters-http/     @shop/orders-http            type:adapter, scope:orders
libs/orders/ui/                @shop/orders-ui              type:ui, scope:orders
```

Promote when the feature is large enough that the layer boundary is crossed by several teams, or when the domain is reused outside the feature. Otherwise the extra `project.json`, `tsconfig`, and build target per layer is ceremony.

### 8.2 Nx configuration

Each project carries its tags in `project.json`:

```json
{
  "name": "orders-domain",
  "tags": ["type:domain", "scope:orders"]
}
```

```js
// eslint.config.mjs
import nx from '@nx/eslint-plugin';

export default [
  ...nx.configs['flat/base'],
  ...nx.configs['flat/typescript'],
  ...nx.configs['flat/javascript'],
  {
    files: ['**/*.ts', '**/*.tsx', '**/*.js', '**/*.jsx'],
    rules: {
      '@nx/enforce-module-boundaries': [
        'error',
        {
          enforceBuildableLibDependency: true,
          allow: [],
          depConstraints: [
            {
              sourceTag: 'type:domain',
              onlyDependOnLibsWithTags: ['type:domain'],
              bannedExternalImports: ['@angular/*', 'rxjs'],
            },
            {
              sourceTag: 'type:application',
              onlyDependOnLibsWithTags: ['type:domain', 'type:application'],
              bannedExternalImports: ['@angular/common/http', '@angular/router'],
            },
            {
              sourceTag: 'type:adapter',
              onlyDependOnLibsWithTags: ['type:domain', 'type:application', 'type:config'],
              bannedExternalImports: ['@angular/router'],
            },
            {
              sourceTag: 'type:ui',
              onlyDependOnLibsWithTags: ['type:domain', 'type:application', 'type:ui'],
              bannedExternalImports: ['@angular/common/http'],
            },
            {
              sourceTag: 'type:config',
              onlyDependOnLibsWithTags: ['type:config'],
              bannedExternalImports: ['@angular/common/http', '@angular/router'],
            },
            {
              sourceTag: 'type:feature',
              onlyDependOnLibsWithTags: [
                'type:domain',
                'type:application',
                'type:adapter',
                'type:ui',
                'type:config',
              ],
            },
            {
              sourceTag: 'type:app',
              onlyDependOnLibsWithTags: ['*'],
            },
            {
              sourceTag: 'scope:shared',
              onlyDependOnLibsWithTags: ['scope:shared'],
            },
            {
              sourceTag: 'scope:orders',
              onlyDependOnLibsWithTags: ['scope:orders', 'scope:shared'],
            },
          ],
        },
      ],
    },
  },
];
```

Three properties of the rule matter:

- Constraints are **ANDed**. `orders-domain` has `type:domain` and `scope:orders`; it must satisfy both constraints. It may depend on `shared-domain` (type:domain, scope:shared) and not on `billing-domain` (type:domain, scope:billing).
- `bannedExternalImports` stops the framework at the outer layers: `@angular/*` and `rxjs` in the domain, `@angular/common/http` in the application and UI. It works on package names/globs, and by default it checks direct imports; `checkNestedExternalImports: true` extends it to packages a project pulls in transitively.
- A project with no tags matches no constraint and is therefore unconstrained. Tag every project and add a CI check that fails on an untagged one; otherwise a new `libs/core` silently becomes a shared dumping ground.

`bannedExternalImports` is not a substitute for the layer rule. A domain project can avoid Angular and still import the application package if `onlyDependOnLibsWithTags` is missing.

### 8.3 ESLint-only alternative

Without Nx, `eslint-plugin-boundaries` classifies files by folder and applies the same matrix. This is also the right tool for the inner layers of a single-project feature.

```js
// eslint.config.mjs
import boundaries from 'eslint-plugin-boundaries';

export default [
  {
    files: ['libs/**/*.ts', 'apps/**/*.ts'],
    plugins: { boundaries },
    settings: {
      'boundaries/elements': [
        { type: 'domain', pattern: 'libs/*/src/lib/domain/**' },
        { type: 'domain', pattern: 'libs/shared/domain/src/**' },
        { type: 'application', pattern: 'libs/*/src/lib/application/**' },
        { type: 'adapter', pattern: 'libs/*/src/lib/adapters/**' },
        { type: 'ui', pattern: 'libs/*/src/lib/ui/**' },
        { type: 'config', pattern: 'libs/shared/config/src/**' },
        { type: 'app', pattern: 'apps/*/src/**' },
      ],
    },
    rules: {
      'boundaries/no-unknown-files': ['error'],
      'boundaries/dependencies': [
        2,
        {
          default: 'disallow',
          policies: [
            {
              from: { element: { type: 'domain' } },
              allow: { to: { element: { type: 'domain' } } },
            },
            {
              from: { element: { type: 'application' } },
              allow: { to: { element: { type: ['domain', 'application'] } } },
            },
            {
              from: { element: { type: 'adapter' } },
              allow: { to: { element: { type: ['domain', 'application'] } } },
            },
            {
              from: { element: { type: 'ui' } },
              allow: {
                to: { element: { type: ['domain', 'application', 'ui'] } },
              },
            },
            {
              from: { element: { type: 'app' } },
              allow: {
                to: {
                  element: {
                    type: ['domain', 'application', 'adapter', 'ui', 'config', 'app'],
                  },
                },
              },
            },
          ],
        },
      ],
    },
  },
  {
    files: [
      'libs/*/src/lib/domain/**/*.ts',
      'libs/shared/domain/src/**/*.ts',
    ],
    rules: {
      'no-restricted-imports': [
        'error',
        { patterns: ['@angular/*', 'rxjs', '@angular/common/http'] },
      ],
    },
  },
  {
    files: ['libs/*/src/lib/application/**/*.ts'],
    rules: {
      'no-restricted-imports': [
        'error',
        { patterns: ['@angular/common/http', '@angular/router'] },
      ],
    },
  },
  {
    files: ['libs/*/src/lib/ui/**/*.ts'],
    rules: {
      'no-restricted-imports': [
        'error',
        { patterns: ['@angular/common/http'] },
      ],
    },
  },
];
```

`boundaries/no-unknown-files` is the cheap defense against a new `core/` or `services/` folder that no rule knows about. Every file must match an element descriptor, so an architectural folder cannot appear silently.

**Cost.** ESLint checks one file at a time and cannot prove the graph is acyclic. It also depends on glob accuracy: a pattern that misses a folder silently exempts it, which is why `no-unknown-files` matters. Nx builds a project graph and catches circular project dependencies; ESLint gives faster editor feedback. Using both is reasonable; using neither is the failure mode.

## 9. The component and its template

```ts
// libs/orders/src/lib/ui/orders-page.component.ts
import { ChangeDetectionStrategy, Component, inject } from '@angular/core';
import { OrdersFacade } from '../application/orders.facade';

@Component({
  selector: 'app-orders-page',
  templateUrl: './orders-page.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class OrdersPageComponent {
  private readonly facade = inject(OrdersFacade);

  readonly orders = this.facade.orders;
  readonly count = this.facade.count;

  reload(): void {
    this.facade.reload();
  }
}
```

No `standalone: true`: standalone is the default from Angular 19 onward. No `imports` array because the template uses only built-in control flow. `OnPush` is the recommended baseline for a zoneless application.

```html
<!-- libs/orders/src/lib/ui/orders-page.component.html -->
@if (orders.isLoading()) {
  <p>Loading…</p>
} @else if (orders.error()) {
  <p role="alert">Could not load orders.</p>
} @else {
  <p>{{ count() }} orders</p>
  <ul>
    @for (order of orders.value() ?? []; track order.id.value) {
      <li>{{ order.id.value }} — {{ order.total.minor }} {{ order.total.currency }}</li>
    }
  </ul>
}
```

The component reads signals and calls one intent method. It does not format money with domain logic, map a DTO, or know a URL. Formatting belongs to a UI pipe; the domain exposes minor units and currency and stays out of presentation.

## 10. Testing with a fake port

```ts
// libs/orders/src/lib/ui/orders-page.component.spec.ts
import { TestBed } from '@angular/core/testing';
import { Money } from '../domain/primitives/money';
import { Order } from '../domain/entities/order';
import { OrderId } from '../domain/primitives/order-id';
import { OrderLine } from '../domain/entities/order-line';
import { OrdersFacade } from '../application/orders.facade';
import {
  ORDER_GATEWAY,
  type OrderGateway,
} from '../application/ports/order-gateway.port';
import { OrdersPageComponent } from './orders-page.component';

function orderFixture(): Order {
  return Order.rehydrate({
    id: OrderId.of('order-1'),
    status: 'placed',
    lines: [
      OrderLine.of({
        sku: 'sku-1',
        quantity: 1,
        unitPrice: Money.ofMinor(1999, 'EUR'),
      }),
    ],
  });
}

describe('OrdersPageComponent', () => {
  it('renders the orders returned by the port', async () => {
    const gateway: OrderGateway = {
      list: async () => [orderFixture()],
      place: async () => OrderId.of('order-2'),
    };

    await TestBed.configureTestingModule({
      imports: [OrdersPageComponent],
      providers: [
        OrdersFacade,
        { provide: ORDER_GATEWAY, useValue: gateway },
      ],
    }).compileComponents();

    const fixture = TestBed.createComponent(OrdersPageComponent);
    fixture.detectChanges();
    await fixture.whenStable();
    fixture.detectChanges();

    expect(fixture.nativeElement.textContent).toContain('order-1');
  });
});
```

The test proves the port is real: it supplies a fake gateway and never configures `HttpClient`. If someone adds `inject(HttpClient)` to the component or the facade, this test fails to compile or fails at runtime, and the boundary violation surfaces before review.

`fixture.whenStable()` waits for the `resource` loader in a zoneless application. If the facade used a raw promise outside `resource`, the test would need an explicit await or a `PendingTasks` wrapper; that is one more reason to keep async reads in `resource`.

## 11. What breaks first

### 11.1 A component injects `HttpClient`

The shortest path for one screen is `private readonly http = inject(HttpClient)` in the component. It works, the screen ships, and the port now has two implementations: the gateway for most screens and the component for this one.

Symptom: a component test imports `provideHttpClientTesting`; the same endpoint is fetched twice on one page; a DTO field appears in a template; changing the API envelope requires editing a component. Catch: `bannedExternalImports: ['@angular/common/http']` for `type:ui`, plus `no-restricted-imports` for `ui/**`. Fix: move the call into the gateway, expose a facade signal or `resource`, and let the component read it.

### 11.2 The domain imports `@angular/*`

An `InjectionToken` is moved into `domain/` because the port needs a token and the token "belongs to the port". Or a `signal` is used in an entity because the UI needs reactivity. The domain now needs Angular to compile, and domain tests need the framework.

Symptom: `@angular/core` in a domain import; `TestBed` in a domain spec; the model cannot be reused in a Node script or a shared package. Catch: Nx `bannedExternalImports: ['@angular/*', 'rxjs']` on `type:domain`; ESLint `no-restricted-imports` on the domain folders. Fix: keep the interface pure and put the token in `application/ports/`; keep reactivity in the facade, not in the entity.

### 11.3 A shared `core` becomes a dumping ground

`libs/core` starts as "shared utilities", then imports a feature's facade, then a feature imports `core`, and `core` imports the feature back. It has no type tag, or it is tagged `type:core`, which no constraint mentions. It becomes a second application layer with singletons nobody owns.

Symptom: every feature imports `core`; `core` imports from at least one feature; a change in `core` requires rebuilding everything; `providedIn: 'root'` services with hidden mutable state. Catch: no `type:core` tag; every project has a `type:*` tag; untagged projects cannot depend on anything (Nx); `boundaries/no-unknown-files` for folder-level layouts. Fix: split into `domain` (rules), `application` (shared orchestration), `ui` (design system), and `util` (pure functions). A library everyone may use must depend on nothing but `util`; if it needs a domain type, it is not a util.

### 11.4 Route-level providers create duplicate state

A parent route provides `OrdersFacade`; a child route provides it again "so the child can inject it". Angular creates a child injector, resolves the nearest provider, and the parent and child now hold different facades. The child places an order; the parent list does not change. A variant is `providedIn: 'root'` plus a route provider: two instances, one of which never gets destroyed.

Symptom: a list does not update after a detail action; navigating away and back resets or preserves state unexpectedly; a facade constructor runs twice for one feature. Catch: grep for every `providers: [SomeFacade]` and confirm exactly one per injector branch; a test that resolves the facade from the route injector and from the component and compares identity (or, more simply, a review rule that stateful facades never use `providedIn`). Fix: provide the facade once at the route that owns the feature. If state must survive navigation, hoist it to a parent route that stays active, or to the root, and accept the longer lifetime. Do not provide it in both places.

## 12. What this costs

| Cost | Who pays | Mitigation |
|---|---|---|
| Four folders or four projects per feature | the feature team | start with one project and ESLint; promote layers only when the boundary is crossed often |
| DTO + mapper per endpoint | the feature team | keep the mapper next to the DTO; test it at the port boundary |
| Signals and `resource` instead of a familiar `subscribe` | developers new to Angular's reactive style | keep async reads in the facade; components only read signals |
| Nx tags and constraints to maintain | the platform owner | start with the type/scope matrix above; add a tag only when a real violation needs it |
| A shared domain library that everyone must rebuild | all consumers | keep it small and stable; duplicate until the invariant is genuinely shared |

The layout pays off when the application has more than one transport or more than one team touching the same feature, and when the domain rules are worth testing without a browser. For a small internal app with static screens, a single `features/` folder and a typed API service is the cheaper correct answer.
