# Minimal Layout

> **Read when:** `layouts/index.md` landed on `minimal` — one bounded context, one deployable, thin domain, and a folder boundary is the honest amount of structure.

One project, one bounded context, four folders, one composition file. Direction is preserved by imports and checked by a lint rule; the compiler cannot help. This tier exists because a small service with real rules does not need a project graph. It is not a starting point for a large system.

## 1. Target tree

```
billing/                              # one deployable, one package, one test run
  src/
    domain/                           # owns the rules; imports only itself and the standard library
      Aggregates/
      Primitives/                     # shared inside this context only
      Errors/
    application/                      # owns orchestration and transaction boundaries
      usecases/
        issue-invoice/                # inbound port = the handler in this folder
        void-invoice/
      Ports/                          # outbound ports; declared here, implemented outside
    adapters/
      inbound/
        http/                         # transport DTOs <-> use-case input; no business decisions
      outbound/
        persistence/                  # implements Ports/; owns ORM mapping and schema access
    composition/                      # the only place that constructs adapters
      Program.cs | main.ts | App.java # one entry file, whichever the platform uses
  tests/
    unit/                             # domain and use cases against fake ports
    architecture/                     # the import rule, executable (§4)
```

Notes:

- `inbound`/`outbound` is the only sub-split worth keeping. It encodes direction, not technology; `http` and `persistence` are examples, not required names.
- `composition/` may be the project-root entry file if the platform expects it there. The rule is one wiring file, not one folder.
- Ports sit in `application/` because the contracts serve use cases (`concepts.md` §3). Moving them to `domain/Ports/` is a vocabulary decision and does not change the rule.
- If the codebase is Angular: the UI component is the driving adapter and the API client is the driven adapter. `domain/` and `application/` are plain TypeScript folders in the same app; do not create a backend-shaped `api/` folder that has no server behind it.

Export surface, inside the project:

- `domain/` exposes domain types to `application/` and `adapters/`. Nothing outside the project imports it; no adapter type may appear in a domain signature.
- `application/` exposes use-case handlers and port interfaces to `adapters/`.
- `adapters/` expose nothing upward. An adapter type in an application signature is a reversed dependency, even when both live in one assembly.
- Nothing outside the project imports these folders. If the project is published as a library, the folder layout is not an export surface: use the `standard` split and publish only the project that carries the contract.

Forbidden additions inside `src/`: `common/`, `shared/`, `utils/`, `helpers/`, `models/`. Each becomes either a second domain or a bypass around the layer it sits next to. A step shared by two use cases is a private collaborator in one of their folders, or its own use case.

## 2. Why it is split this way

Ownership in this tier:

- `domain/` owns every rule that would survive a UI rewrite. It has no framework imports. If it needs time or an identifier, the port is in `application/Ports/` and the implementation is in `adapters/outbound/`.
- `application/` owns the order of operations, the transaction boundary, and authorization decisions. It calls ports; it never constructs an adapter.
- `adapters/` own translation. The inbound adapter maps transport models to use-case input; the outbound adapter maps port contracts to a database or an external API. Neither branches on business state.
- `composition/` owns `new` and lifetimes. It is the only file allowed to name a concrete adapter.

What the split buys:

- One deployable, one version, one test command. There is no release coordination to design.
- The rule fits in one import list. A reviewer can check a pull request without opening a project file.
- Use-case folders answer "where is this feature" without a project graph or a navigation tool.
- Extracting a layer later is a move, not a redesign: the folders already contain the right code.

What it costs:

- No compiler guarantee. Until the check in §4 runs in CI, `domain/` can import the ORM and nothing stops it. The reviewer pays for a skipped check.
- No package isolation. One project has one dependency set, so the domain cannot be **proven** framework-free. See §3, signal 1.
- `internal` is not a boundary. Every type in the assembly can see every other type; IDE navigation crosses layers freely and so do reviewers' assumptions.
- Upgrading to `standard` moves every file once and rewrites every import. The team pays in one disruptive release. Schedule it; do not discover it during a feature freeze.

## 3. When NOT to use it

**Many projects should not start here at all.** If the code is request → validate → write row → return row, with no invariant you can state without reading the schema, hexagonal folders are decoration. Use a transaction script or vertical slices. A minimal layout is still indirection; CRUD does not repay it.

Concrete signals — any one is enough to move to `standard.md`:

1. **The domain must be provably framework-free.** Compliance, a published package, or reuse by a second host requires excluding ORM/HTTP packages from `domain/`. One project cannot exclude a package from a folder.
2. **More than one inbound adapter that must run independently** — HTTP plus a queue consumer plus a scheduler. In one project they share startup, dependencies, and failure modes.
3. **More than one team commits to the project,** or layers need different reviewers. Folder ownership is a convention; project ownership is visible in the build.
4. **Past roughly 15 use cases or 5 aggregates with independent lifecycles.** The folder listing stops fitting on one screen, and shared steps start migrating into a service that becomes a second domain.
5. **Each layer needs a different rule.** Domain: zero dependencies; application: no adapters; adapters: no sibling adapters. In one assembly the rule set is the union, so the strongest assertion is unavailable.
6. **The team keeps importing ORM types into the domain.** That is not a discipline problem to solve with review; it is a missing build boundary. Move to `standard.md` and let the compiler enforce it.

Do not choose minimal as a placeholder for standard. If you already know a second adapter or a second team arrives within two quarters, start at `standard`; the split you defer is the split you will perform under delivery pressure.

## 4. Enforcement

Rules to assert:

1. `domain/**` imports only `domain/**` and the language standard library.
2. `application/**` imports `domain/**`, `application/**`, and declared port types; never `adapters/**`.
3. `adapters/**` imports `application/**` and `domain/**`; never another adapter subtree (`adapters/inbound` must not import `adapters/outbound`, or the reverse).
4. `composition/**` may import everything; nothing imports `composition/**`.

| Stack | Tool | Rule lives in | What it asserts |
|---|---|---|---|
| .NET | NetArchTest.Rules (xUnit/NUnit) | `tests/architecture` | The domain namespace depends on nothing outside itself; no dependency on ORM/HTTP namespaces |
| Java | ArchUnit | `src/test/java/.../ArchitectureTest` | `..domain..` depends only on `..domain..` and `java..` |
| TypeScript | dependency-cruiser | `.dependency-cruiser.js` | `src/domain` reaches nothing outside `src/domain`; `src/application` reaches no adapter |
| Angular | Nx `@nx/enforce-module-boundaries` in an Nx workspace; `eslint-plugin-boundaries` in a single Angular CLI app | ESLint flat config, `boundaries/element-types` or Nx `depConstraints` | `domain` element imports only `domain`; `adapter` imports `application` and `domain` |

Rule example:

```js
// .dependency-cruiser.js — run with `depcruise src --config .dependency-cruiser.js`
// Paths are relative to the working directory; drop the `src/` prefix if you cruise from inside `src/`.
/** @type {import('dependency-cruiser').IConfiguration} */
module.exports = {
  forbidden: [
    {
      name: 'domain-imports-only-domain',
      severity: 'error',
      from: { path: '^src/domain/' },
      to: { pathNot: '^src/domain/' },
    },
    {
      name: 'application-never-imports-adapters',
      severity: 'error',
      from: { path: '^src/application/' },
      to: { path: '^src/adapters/' },
    },
  ],
};
```

Two closing rules:

- The check must run in CI and block the merge. An architecture test that only runs locally is documentation with a pass rate.
- Add a second test that asserts `src/` contains exactly `domain`, `application`, `adapters`, `composition`. Minimal layouts fail by folder proliferation, not by dramatic import violations, and the inventory test catches the drift early.
