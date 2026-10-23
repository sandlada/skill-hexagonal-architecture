# TypeScript Example

> **Read when:** you are structuring a Node/TypeScript backend around the dependency rule and need a workspace layout, an export boundary the runtime enforces, and a CI check that fails on a reversed dependency.

This example assumes a pnpm workspace with four units: `domain`, `application`, `adapter-persistence`, and `apps/api`, which is both the inbound HTTP adapter and the composition root. Ports live in the application layer. If your project puts them in the domain, move the interface and keep the arrows identical.

The example is structural. It does not choose a web framework or an ORM for you; Fastify and Prisma appear because their types make boundary failures concrete.

**Options used in this example** (record your own before generating a tree; see `../naming-and-modes.md` §5):

```
Options record
- Application mode:      usecase
- Domain vocabulary:     model=Entities, shared=Primitives, errors=Errors, outbound ports=Ports
- Port location:         application
- Presentation name:     api (dedicated inbound adapter project, apps/api)
- Inherited from code:   no - greenfield illustration
- Rationale:             one folder per use case, plus a package boundary the Node resolver can enforce
```

## 1. Workspace layout

```text
.
├── apps/
│   └── api/                          inbound adapter + composition root
│       ├── src/
│       │   ├── main.ts
│       │   └── orders.routes.ts
│       ├── package.json
│       └── tsconfig.json
├── packages/
│   ├── domain/                       entities, value objects, errors
│   │   ├── src/
│   │   │   ├── entities/
│   │   │   ├── primitives/
│   │   │   ├── errors/
│   │   │   └── index.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── application/                  use cases + outbound port interfaces
│   │   ├── src/
│   │   │   ├── usecases/place-order.ts
│   │   │   ├── ports/order-repository.ts
│   │   │   └── index.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── adapter-persistence/          Prisma implementation
│       ├── src/
│       │   ├── prisma-order.repository.ts
│       │   ├── order.mapper.ts
│       │   └── index.ts
│       ├── prisma/schema.prisma
│       ├── package.json
│       └── tsconfig.json
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── tsconfig.json                     solution file: references only
└── .dependency-cruiser.mjs
```

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

Dependency direction (`A ──> B` means A may import B):

```text
apps/api ──> application ──> domain
    │             ▲
    └──> adapter-persistence ──┘
```

The API project imports the persistence adapter only to construct it. The application never sees that package.

**Why packages instead of folders.** A folder cannot stop `import '../../adapter-persistence/src/order.mapper.js'`. A package with an `exports` map can, at module resolution time, before review. `tsc -b` can order builds and reject reference cycles only when each layer is its own project. And dependency-cruiser can match resolved package paths instead of guessing from relative import strings.

**Cost.** Every package needs a `package.json`, a `tsconfig.json`, and a build step. A backend under roughly twenty source files with one adapter does not earn that overhead. In that case keep one package and enforce folder rules with the ESLint configuration in §6; you lose runtime export enforcement but keep the dependency check.

## 2. The boundary is `package.json` `exports`

`packages/domain/package.json`:

```json
{
  "name": "@acme/domain",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  },
  "files": ["dist"]
}
```

What this buys:

- `import { Order } from '@acme/domain'` resolves.
- `import { Order } from '@acme/domain/src/entities/order.js'` fails with `ERR_PACKAGE_PATH_NOT_EXPORTED` in Node, and TypeScript cannot resolve it under `"moduleResolution": "nodenext"`. The public surface is one entry point, not a directory tree.
- `files: ["dist"]` means the published artifact contains no source tree. The workspace consumes the package close to the way production does.

The price is a build step: `exports` points at `dist`, so `pnpm build` must run before the API starts. That is deliberate — it tests the boundary that production uses. During development, `tsc -b --watch` keeps `dist` current.

The export map per package:

| Package | Exports | Imported by |
|---|---|---|
| `@acme/domain` | entities, value objects, domain errors | application, adapters |
| `@acme/application` | use-case classes, command types, outbound port interfaces | `apps/api`, adapters |
| `@acme/adapter-persistence` | the repository class the composition root constructs | `apps/api` only |
| `apps/api` | nothing; it is a process entry point | — |

`packages/application/src/index.ts` lists exports explicitly:

```ts
export { PlaceOrder, type PlaceOrderCommand } from './usecases/place-order.js';
export type { OrderRepository } from './ports/order-repository.js';
```

Do not use `export *` here. It turns every future internal into public API without a review, and it is how adapter types end up re-exported from the application package. The cost is one line per public type and a deliberate edit whenever the contract grows.

The persistence package exports only its wiring surface:

```ts
// packages/adapter-persistence/src/index.ts
export { PrismaOrderRepository } from './prisma-order.repository.js';
```

Nothing in the application may import that symbol; §5 enforces it.

## 3. Project references

`tsconfig.base.json`:

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "lib": ["ES2023"],
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "strict": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "baseUrl": ".",
    "paths": {
      "@acme/domain": ["packages/domain/src/index.ts"],
      "@acme/application": ["packages/application/src/index.ts"],
      "@acme/adapter-persistence": ["packages/adapter-persistence/src/index.ts"]
    }
  }
}
```

`packages/application/tsconfig.json`:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "rootDir": "src",
    "outDir": "dist",
    "tsBuildInfoFile": "dist/.tsbuildinfo"
  },
  "include": ["src/**/*.ts"],
  "references": [{ "path": "../domain" }]
}
```

`apps/api/tsconfig.json` references `../packages/application` and `../packages/adapter-persistence`. The root `tsconfig.json` is a solution file:

```json
{
  "files": [],
  "references": [
    { "path": "packages/domain" },
    { "path": "packages/application" },
    { "path": "packages/adapter-persistence" },
    { "path": "apps/api" }
  ]
}
```

Build with `pnpm exec tsc -b`; watch with `pnpm exec tsc -b --watch`.

What the compiler proves:

- Every referenced project builds in dependency order.
- A reference cycle fails with `TS6202: Project references may not form a circular graph`.
- With `composite: true`, importing a source file that is not in the current project and not reached through a declared reference fails with `TS6307`. This catches the accidental "I will just point at the other package's source" import.

What it does not prove:

- Layer direction. `domain` can reference `application` in a one-way, wrong-direction edge and `tsc` is satisfied.
- Deep imports, once a path alias resolves them.
- That a public signature is free of ORM types. Declaration emit makes such a type visible in `.d.ts`, but it does not reject it.

That gap is why §5 exists.

## 4. Path aliases: where cycles hide

Aliases are the most common way a correct-looking tree compiles with a reversed arrow. Two layouts show the failure.

### 4.1 One package, one alias per layer

```jsonc
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@domain/*": ["src/domain/*"],
      "@application/*": ["src/application/*"],
      "@adapters/*": ["src/adapters/*"]
    }
  }
}
```

This is the common single-package backend. A domain file imports `@application/ports/order-repository`; TypeScript resolves it, because `paths` apply to every file in the program. There is no project reference to violate and no cycle for `tsc` to see. The code compiles, and at runtime the ESM cycle shows up as `Cannot access 'X' before initialization` or as an `undefined` binding in one import order but not another.

A wildcard also makes a deep import look legitimate:

```ts
import { toOrderRow } from '@adapters/persistence/order.mapper';
```

That is the domain importing a mapper. It type-checks. Nothing in the compiler knows what `adapters` means.

### 4.2 Multi-package workspace, wildcard alias

```jsonc
{
  "paths": {
    "@acme/domain": ["packages/domain/src/index.ts"],
    "@acme/domain/*": ["packages/domain/src/*"]
  }
}
```

The first entry points at the same file `exports` points at; it is mostly a convenience. The second entry is the boundary hole. It lets `@acme/domain/src/prisma-order.mapper` resolve in the type checker even though Node refuses the same specifier at runtime. The result is green CI and a production `ERR_PACKAGE_PATH_NOT_EXPORTED`.

### 4.3 What to do instead

- **Use exact package-root aliases, never `/*` across packages.** The block in §3 does this. Type-checking follows `src/index.ts`; runtime follows `exports`. The public surface is the same file in both cases.
- **Keep intra-package aliases inside the package** with Node's `imports` field, for example `"#/*": "./src/*"`. Those cannot cross a package boundary, and Node enforces them.
- **If aliases must remain, make the checker resolve them before matching.** dependency-cruiser needs `options.tsConfig`; without it, `@acme/domain` looks like an external npm package and every `^packages/domain` rule silently misses.
- **Treat `tsc -b` as a cycle detector, not a layer checker.** It catches `TS6202` and, under `composite`, `TS6307`. It does not know that domain must not import application.

If you want editor navigation without aliases at all, a `"development"` condition in `exports` plus `"customConditions": ["development"]` in the tsconfig keeps one source of truth. It also means the runtime and the type checker read the same map. The cost is one more condition to document.

## 5. Enforcement with dependency-cruiser

`.dependency-cruiser.mjs`:

```js
/** @type {import('dependency-cruiser').IConfiguration} */
export default {
  forbidden: [
    {
      name: 'no-circular',
      severity: 'error',
      comment: 'A cycle means two layers own each other; one of them is in the wrong place.',
      from: {},
      to: { circular: true },
    },
    {
      name: 'domain-self-contained',
      severity: 'error',
      comment: 'Domain may depend on itself only: no framework, ORM, HTTP, clock, or other layer.',
      from: { path: '^packages/domain/src', pathNot: '\\.spec\\.ts$' },
      to: { pathNot: '^packages/domain/src' },
    },
    {
      name: 'domain-has-no-npm-dependencies',
      severity: 'error',
      comment: 'A domain package that imports npm code is shaped by that code.',
      from: { path: '^packages/domain/src', pathNot: '\\.spec\\.ts$' },
      to: { dependencyTypes: ['npm', 'npm-dev', 'npm-optional', 'npm-peer'] },
    },
    {
      name: 'only-composition-root-imports-adapters',
      severity: 'error',
      comment: 'Application and domain must not see a concrete adapter.',
      from: { pathNot: '^(apps/|packages/adapter-)' },
      to: { path: '^packages/adapter-' },
    },
    {
      name: 'adapters-do-not-import-adapters',
      severity: 'error',
      comment: 'Cross-adapter calls bypass the domain and hide coupling.',
      from: { path: '^packages/adapter-([^/]+)/src' },
      to: {
        path: '^packages/adapter-[^/]+/src',
        pathNot: '^packages/adapter-$1/src',
      },
    },
    {
      name: 'cross-package-imports-use-entry-points',
      severity: 'error',
      comment: 'Reach another package through its exports map, not through files under src/.',
      from: { path: '^(?:packages|apps)/([^/]+)/src/.+' },
      to: {
        path: '^packages/[^/]+/src/.+',
        pathNot: [
          '^packages/$1/src/.+',
          '^packages/[^/]+/src/index\\.ts$',
        ],
      },
    },
  ],
  options: {
    doNotFollow: { path: 'node_modules' },
    tsConfig: { fileName: 'tsconfig.base.json' },
    tsPreCompilationDeps: true,
    enhancedResolveOptions: {
      exportsFields: ['exports'],
      conditionNames: ['import', 'node', 'default'],
    },
  },
};
```

Why each switch matters:

- `tsConfig` resolves the exact-root aliases to source files. Without it, `@acme/domain` is an external module and `domain-self-contained` cannot see the edge.
- `tsPreCompilationDeps: true` counts type-only imports. A domain type that imports an ORM type is still coupled to the ORM; excluding `import type` would hide exactly the leak this rule is meant to catch.
- `pathNot: '\\.spec\\.ts$'` keeps test files out of the domain purity rules. Test code may import Vitest; production domain code may not.
- The two capture groups in `cross-package-imports-use-entry-points` use dependency-cruiser's group matching: `$1` is the importing package, so a package may import its own files and any package's `src/index.ts`, but not another package's internals.

Add the check to CI:

```json
{
  "scripts": {
    "build": "tsc -b",
    "depcruise": "depcruise --config .dependency-cruiser.mjs packages apps",
    "depcruise:ci": "depcruise --config .dependency-cruiser.mjs --output-type err packages apps"
  }
}
```

The `err` reporter exits non-zero on an `error`-severity violation. Run `depcruise:ci` after `build`, so package entry points resolve.

**Cost.** Six rules are cheap; thirty are not. Start with the six above, add a rule only after a real violation, and keep the reason in `comment` so the next person can tell whether the rule still applies.

## 6. The ESLint alternative

If the project already runs ESLint and does not want a second graph tool, `eslint-plugin-boundaries` can express the same layer matrix. Flat config:

```js
import boundaries from 'eslint-plugin-boundaries';

export default [
  {
    files: ['**/*.ts'],
    plugins: { boundaries },
    settings: {
      'boundaries/elements': [
        { type: 'domain', pattern: 'packages/domain/src/**' },
        { type: 'application', pattern: 'packages/application/src/**' },
        { type: 'adapter', pattern: 'packages/adapter-*/src/**' },
        { type: 'app', pattern: 'apps/*/src/**' },
      ],
      'import/resolver': { typescript: { project: 'tsconfig.base.json' } },
    },
    rules: {
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
              from: { element: { type: 'app' } },
              allow: {
                to: { element: { type: ['domain', 'application', 'adapter', 'app'] } },
              },
            },
          ],
        },
      ],
    },
  },
  {
    files: ['packages/domain/src/**/*.ts', '!packages/domain/src/**/*.spec.ts'],
    rules: {
      'no-restricted-imports': [
        'error',
        { patterns: ['@prisma/*', 'fastify', 'express', 'node:*'] },
      ],
    },
  },
];
```

`eslint-import-resolver-typescript` is required for the resolver entry. Without it, aliases resolve to nothing and the boundary rule sees only relative paths.

Tradeoff: ESLint runs per file and gives fast editor feedback, but it cannot prove that the graph is acyclic. Add `import/no-cycle` if you need that too. dependency-cruiser cruises the whole graph once and reports the cycle; ESLint is the cheaper daily check, not a replacement for a graph check on a growing codebase.

## 7. Inbound adapter: HTTP calls a use case

The use case is the inbound port. `@acme/application` exports the class; the only public operation is `execute`.

```ts
// packages/application/src/usecases/place-order.ts
import { CustomerId, Money, Order, OrderId, OrderLine } from '@acme/domain';
import type { OrderRepository } from '../ports/order-repository.js';

export interface PlaceOrderCommand {
  orderId: OrderId;
  customerId: CustomerId;
  lines: ReadonlyArray<{
    sku: string;
    quantity: number;
    unitPriceMinor: number;
    currency: string;
  }>;
}

export class PlaceOrder {
  constructor(private readonly orders: OrderRepository) {}

  async execute(command: PlaceOrderCommand): Promise<OrderId> {
    const order = Order.place({
      id: command.orderId,
      customerId: command.customerId,
      lines: command.lines.map((line) =>
        OrderLine.of({
          sku: line.sku,
          quantity: line.quantity,
          unitPrice: Money.ofMinor(line.unitPriceMinor, line.currency),
        }),
      ),
    });

    await this.orders.save(order);
    return order.id;
  }
}
```

The outbound port:

```ts
// packages/application/src/ports/order-repository.ts
import type { Order, OrderId } from '@acme/domain';

export interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  save(order: Order): Promise<void>;
}
```

The HTTP route translates transport to the command and the result back to transport. It makes no business decision.

```ts
// apps/api/src/orders.routes.ts
import type { FastifyInstance } from 'fastify';
import { z, ZodError } from 'zod';
import { CustomerId, DomainError, OrderId } from '@acme/domain';
import type { PlaceOrder } from '@acme/application';

const placeOrderBody = z.object({
  customerId: z.string().min(1),
  lines: z
    .array(
      z.object({
        sku: z.string().min(1),
        quantity: z.number().int().positive(),
        unitPriceMinor: z.number().int().nonnegative(),
        currency: z.string().length(3),
      }),
    )
    .min(1),
});

export async function orderRoutes(
  app: FastifyInstance,
  options: { placeOrder: PlaceOrder },
): Promise<void> {
  app.post('/orders', async (request, reply) => {
    const body = placeOrderBody.parse(request.body);
    const idempotencyKey = z
      .string()
      .min(1)
      .parse(request.headers['idempotency-key']);

    const orderId = await options.placeOrder.execute({
      orderId: OrderId.of(idempotencyKey),
      customerId: CustomerId.of(body.customerId),
      lines: body.lines,
    });

    return reply.code(201).send({ id: orderId.value });
  });

  app.setErrorHandler((error, request, reply) => {
    if (error instanceof ZodError) {
      return reply.code(400).send({ code: 'invalid_request', issues: error.issues });
    }
    if (error instanceof DomainError) {
      return reply.code(422).send({ code: error.code, message: error.message });
    }
    request.log.error(error);
    return reply.code(500).send({ code: 'internal_error' });
  });
}
```

The composition root is the only place that constructs a concrete adapter:

```ts
// apps/api/src/main.ts
import Fastify from 'fastify';
import { PrismaClient } from '@prisma/client';
import { PlaceOrder } from '@acme/application';
import { PrismaOrderRepository } from '@acme/adapter-persistence';
import { orderRoutes } from './orders.routes.js';

const prisma = new PrismaClient();
const app = Fastify({ logger: true });

await app.register(orderRoutes, {
  placeOrder: new PlaceOrder(new PrismaOrderRepository(prisma)),
});

await app.listen({ port: Number(process.env.PORT ?? 3000) });
```

Two rules are doing the work here. `new PrismaOrderRepository` appears once, at startup; no route and no use case constructs an adapter. And the route imports the application package, not the persistence package. If either changes, dependency-cruiser fails before the code ships.

## 8. Outbound adapter: Prisma stays outside

The adapter converts between two models of the same concept. The domain side is plain TypeScript and does not know a database exists:

```ts
// packages/domain/src/entities/order.ts
import {
  CurrencyMismatchError,
  EmptyOrderError,
  InvalidOrderStatusError,
} from '../errors/order-errors.js';
import type { CustomerId } from '../primitives/customer-id.js';
import { Money } from '../primitives/money.js';
import type { OrderId } from '../primitives/order-id.js';
import type { OrderLine } from './order-line.js';

export const ORDER_STATUSES = ['draft', 'placed', 'cancelled'] as const;
export type OrderStatus = (typeof ORDER_STATUSES)[number];

export function parseOrderStatus(value: string): OrderStatus {
  if (!ORDER_STATUSES.includes(value as OrderStatus)) {
    throw new InvalidOrderStatusError(value);
  }
  return value as OrderStatus;
}

export interface OrderProps {
  id: OrderId;
  customerId: CustomerId;
  status: OrderStatus;
  lines: readonly OrderLine[];
}

export class Order {
  private constructor(private readonly props: OrderProps) {}

  static place(input: {
    id: OrderId;
    customerId: CustomerId;
    lines: readonly OrderLine[];
  }): Order {
    if (input.lines.length === 0) throw new EmptyOrderError();

    const currency = input.lines[0].unitPrice.currency;
    if (input.lines.some((line) => line.unitPrice.currency !== currency)) {
      throw new CurrencyMismatchError();
    }

    return new Order({ ...input, status: 'placed' });
  }

  static rehydrate(props: OrderProps): Order {
    if (props.lines.length === 0) throw new EmptyOrderError();
    return new Order(props);
  }

  get id(): OrderId {
    return this.props.id;
  }

  get customerId(): CustomerId {
    return this.props.customerId;
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

`Money` and `OrderLine` are small validating types: `Money.ofMinor` rejects a negative or non-integer amount and a malformed currency; `OrderLine.of` rejects a non-positive quantity. `OrderId` and `CustomerId` wrap a non-empty string. None of them imports anything outside the domain.

The database side is the Prisma schema:

`packages/adapter-persistence/prisma/schema.prisma`:

```prisma
model Order {
  id         String      @id
  customerId String
  status     String
  currency   String      @db.Char(3)
  lines      OrderLine[]

  @@index([customerId])
}

model OrderLine {
  id             String @id @default(cuid())
  orderId        String
  sku            String
  quantity       Int
  unitPriceMinor Int
  order          Order  @relation(fields: [orderId], references: [id], onDelete: Cascade)
}
```

The adapter owns the row type and the mapper. Neither appears in the port:

```ts
// packages/adapter-persistence/src/prisma-order.repository.ts
import { Prisma, PrismaClient } from '@prisma/client';
import {
  CustomerId,
  Money,
  Order,
  OrderId,
  OrderLine,
  parseOrderStatus,
} from '@acme/domain';
import type { OrderRepository } from '@acme/application';

type OrderRow = Prisma.OrderGetPayload<{ include: { lines: true } }>;

export class PrismaOrderRepository implements OrderRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: OrderId): Promise<Order | null> {
    const row = await this.prisma.order.findUnique({
      where: { id: id.value },
      include: { lines: true },
    });
    return row ? toDomain(row) : null;
  }

  async save(order: Order): Promise<void> {
    const row = toPersistence(order);

    await this.prisma.$transaction(async (tx) => {
      await tx.order.upsert({
        where: { id: row.id },
        create: row.order,
        update: row.order,
      });
      await tx.orderLine.deleteMany({ where: { orderId: row.id } });
      await tx.orderLine.createMany({ data: row.lines });
    });
  }
}

function toDomain(row: OrderRow): Order {
  return Order.rehydrate({
    id: OrderId.of(row.id),
    customerId: CustomerId.of(row.customerId),
    status: parseOrderStatus(row.status),
    lines: row.lines.map((line) =>
      OrderLine.of({
        sku: line.sku,
        quantity: line.quantity,
        unitPrice: Money.ofMinor(line.unitPriceMinor, row.currency),
      }),
    ),
  });
}

function toPersistence(order: Order) {
  const currency = order.lines[0]?.unitPrice.currency;
  if (!currency) {
    throw new Error('Order invariant violated: an order always has at least one line');
  }

  return {
    id: order.id.value,
    order: {
      id: order.id.value,
      customerId: order.customerId.value,
      status: order.status,
      currency,
    },
    lines: order.lines.map((line) => ({
      orderId: order.id.value,
      sku: line.sku,
      quantity: line.quantity,
      unitPriceMinor: line.unitPrice.minor,
    })),
  };
}
```

`OrderRow` is declared in the adapter and never exported. `parseOrderStatus` is a domain function because the allowed set is a domain rule; the adapter supplies the raw string and accepts the domain's answer. `toDomain` is the only place that knows both representations.

The failure this prevents: a port signature such as `save(order: Prisma.OrderGetPayload<...>)`. It compiles, it is shorter, and it makes the application depend on the database shape. Once that happens, the domain is a projection of the schema and the architecture is decorative.

## 9. Testing with in-memory adapters

The fake implements the port and lives outside `src`, so it is not built into the package and cannot be imported through `exports`:

```ts
// packages/application/test/fakes/in-memory-order.repository.ts
import type { Order, OrderId } from '@acme/domain';
import type { OrderRepository } from '../../src/ports/order-repository.js';

export class InMemoryOrderRepository implements OrderRepository {
  private readonly orders = new Map<string, Order>();

  async findById(id: OrderId): Promise<Order | null> {
    return this.orders.get(id.value) ?? null;
  }

  async save(order: Order): Promise<void> {
    this.orders.set(order.id.value, order);
  }
}
```

```ts
// packages/application/test/place-order.spec.ts
import { describe, expect, it } from 'vitest';
import { CustomerId, OrderId } from '@acme/domain';
import { PlaceOrder } from '../src/usecases/place-order.js';
import { InMemoryOrderRepository } from './fakes/in-memory-order.repository.js';

describe('PlaceOrder', () => {
  it('persists a placed order through the port', async () => {
    const orders = new InMemoryOrderRepository();
    const useCase = new PlaceOrder(orders);

    const id = await useCase.execute({
      orderId: OrderId.of('order-1'),
      customerId: CustomerId.of('customer-1'),
      lines: [
        { sku: 'sku-1', quantity: 2, unitPriceMinor: 1999, currency: 'EUR' },
      ],
    });

    const saved = await orders.findById(id);
    expect(saved?.total.minor).toBe(3998);
    expect(saved?.status).toBe('placed');
  });
});
```

No Prisma, no Fastify, no container. The test fails if the use case starts reaching for a concrete adapter, because the constructor no longer accepts one.

**Cost.** An in-memory adapter is not the real database. It does not reproduce unique constraints, transaction isolation, or `cascade` behavior. Keep a small integration suite against `PrismaOrderRepository` for those properties; do not grow the fake until it is a second ORM.

## 10. What breaks first

### 10.1 A barrel widens the boundary

`packages/application/src/index.ts` gains `export * from './internal/order-mapper.js'`, or an adapter is re-exported "for convenience". One import later, a controller is using a persistence type, and the package graph still looks correct because the edge points at the application package.

Symptom: a type from `@prisma/client` appears in an application or API signature; a change to a mapper breaks a controller. Check: explicit named exports, no `export *` across layer packages, and a dependency-cruiser rule that flags adapter-to-application edges. A barrel can also create a runtime cycle that TypeScript accepts; the `no-circular` rule catches it.

### 10.2 An alias bypasses `exports`

A wildcard alias resolves a deep import in the type checker; Node rejects the same specifier at runtime. Worse, the alias can resolve a domain import of an adapter, so the wrong arrow compiles.

Symptom: `tsc` green, `node dist/main.js` fails with `ERR_PACKAGE_PATH_NOT_EXPORTED` or `MODULE_NOT_FOUND`; the failing specifier does not appear in any relative-import search. Check: no `/*` aliases across packages; dependency-cruiser runs with `tsConfig`; CI starts the built process once, not only the test suite.

### 10.3 ORM types leak into the domain

The port returns `Prisma.OrderGetPayload<...>`, or a domain entity imports a Prisma enum because "it is the same set of values". The domain now changes when the schema changes, and the mapper disappears because it looks redundant.

Symptom: `@prisma/client` in `packages/domain/package.json`; a `.d.ts` in `dist` that references a generated type; a domain test that needs a database. Check: `domain-has-no-npm-dependencies` and `domain-self-contained`; code review of any new dependency in the domain package.

### 10.4 A circular import

`Order` imports a policy from `application`, and `PlaceOrder` imports `Order`. ESM loads one of them first; the other sees an uninitialized binding. The failure depends on entry order, so it appears in one test file and not another.

Symptom: `Cannot access 'X' before initialization`, or a constructor that receives `undefined` only in production. Check: `no-circular` in CI, plus `madge --circular packages apps` as a second opinion when adding a package. Fix by moving the shared type down to the domain or by inverting the call through a port; do not fix it by changing import order.

## 11. What this costs

| Cost | Who pays | Mitigation |
|---|---|---|
| A build step before the API starts | every developer and the CI pipeline | `tsc -b --watch` locally, cached `tsc -b` in CI |
| Mapper code per adapter | the feature team | one mapper per adapter, tested at the port boundary |
| Four packages for a small service | reviewers and new hires | collapse to one package below ~20 files; keep the ESLint rules |
| dependency-cruiser maintenance | the platform owner | start with six rules, add one per incident, keep `comment` current |
| Type-checking source through exact aliases | the team debugging a publish issue | build and start the real entry point in CI; do not rely on `tsc` alone |

The layout earns its keep when the domain has rules that change independently of the database and the transport, and when more than one adapter is plausible. If neither is true, the cheaper single-package layout with folder rules is the correct answer, not a compromised version of this one.
