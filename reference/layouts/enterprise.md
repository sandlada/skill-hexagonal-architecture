# Enterprise Layout

> **Read when:** `layouts/index.md` landed on `enterprise` — several bounded contexts, several teams, and at least one contract consumed outside the team that owns it.

Contexts are the top-level unit, not layers. Each context is internally hexagonal, exposes a versioned contract package, and talks to other contexts through an anti-corruption layer it owns. The structure is designed so any context can become its own deployable without moving domain code.

This tree names the presentation layer `Adapters.Inbound.*` because a context can have more than one driving adapter; a context with HTTP only may use `Api` instead.

## 1. Target tree

```
commerce/
  contexts/
    billing/                                  # owner: payments team
      contracts/                              # the versioned export surface
        Billing.Contracts/                    # importable package; semver, changelog, deprecation policy
        inbound/
          openapi/billing.v2.yaml             # published HTTP contract, reviewed by consumers
          events/billing-events.v2.json       # published integration events
        consumed/
          shipping.v1.json                    # pinned producer contract this context depends on
      src/
        Billing.Domain/                       # rules only; no other context's types
        Billing.Application/                  # owns use cases and inbound ports; adapters map contracts to them
        Billing.Adapters.Inbound.Http/        # validates and maps published contract -> application port
        Billing.Adapters.Outbound.Sql/        # own schema, own migrations
        Billing.Adapters.Outbound.Shipping/   # ACL: translates shipping.v1 into Billing's ports
      host/
        Billing.Host/                         # optional standalone deployable; same src as in-process
    shipping/                                 # owner: logistics team; same shape
      ...
    catalog/                                  # owner: merchandising team; same shape
      ...
  platform/
    host/                                     # monolith host: composes selected contexts in-process
    governance/
      arch-tests/                             # cross-context rules and the shared test harness
      waivers.md                              # rule | scope | owner | expires | ticket
      adr/                                    # boundary changes, contract majors, context merges
    shared-kernel/                            # platform-owned, semver'd; contexts consume, never edit
```

Notes:

- **The root is context-major on purpose.** There is no top-level `src/Domain`. A layer-first root is the failure mode this tier exists to prevent: it makes every context share a folder and therefore a change queue.
- **Ports live in `<Context>.Application/Ports/`.** Every `Adapters.*` project is either a driving adapter (inbound HTTP, consumers) or a driven adapter (SQL, other contexts through an ACL). Ports are never declared in an adapter.
- **`host/` exists even when everything runs in `platform/host`.** It is the extraction seam, and CI starts it alone. If the host cannot start without the monolith, the optionality is fiction.
- **ACLs are driven adapters inside the consuming context.** `Billing.Adapters.Outbound.Shipping` implements a Billing-owned port and is the only place that names shipping's published types. A shared `integrations/` project would become a second model and defeat the boundary.
- **Cross-context references stop at the ACL.** `Billing.Domain` and `Billing.Application` never name shipping's types; only `Billing.Adapters.Outbound.Shipping` may reference `Shipping.Contracts`, and it maps them into Billing-owned port types.
- **`contracts/consumed/` pins a producer version.** Upgrading is a diff in that file, reviewed by the consumer, not a transitive surprise in a lockfile.
- **`shared-kernel/` is platform-owned and versioned.** No context edits it in a feature branch; promotions go through the gate in §4.

Export surface:

- A context's `<Context>.Contracts` is the only project another context may import, pinned to a major version. It contains DTOs, integration events, facade interfaces, and nothing framework-shaped.
- `contracts/inbound/**` is a wire contract, not a code reference. It is published and reviewed; it is not generated from the database and it does not generate the domain.
- `platform/shared-kernel` is importable by every context, owned by the platform team, and versioned separately from any context.
- Every other project is internal to its context. Make that structural: `internal` visibility, package `exports`, Gradle `implementation`, or a path rule. A convention is not an export surface.

## 2. Why it is split this way

Ownership in this tier:

- **The context team** owns its domain, application, adapters, schema, contracts, host, and on-call. It can change all of it without asking another team, as long as the published contract stays compatible.
- **The platform team** owns the shared kernel, the cross-context test harness, the CI gates, and the monolith host's composition. It does not own any context's model.
- **The ACL owner is the consumer**, because the consumer pays when the producer changes. The producer is not responsible for translating its model into someone else's.
- **Contract review is joint**: the producer approves the change, and at least one consumer approves the version decision.

Contract-first inbound ports, precisely:

- A **published inbound contract** (OpenAPI, protobuf, AsyncAPI, GraphQL schema) is an external commitment. The adapter validates and maps it to an application port; the spec is the source of truth for the wire, and compatibility is checked in CI.
- An **internal inbound port** (a use case handler) stays ordinary and refactorable. It is not generated from the spec.
- The adapter maps between them. Contract-first does not mean the contract owns the application model; it means the wire shape cannot drift by accident.

What the split buys:

- **Team autonomy with explicit coupling.** The only cross-team artifacts are versioned contracts and the platform-owned shared kernel. A breaking change becomes a version and a deprecation window, not a failed build in another team's pipeline.
- **Model protection.** The ACL absorbs producer changes. Billing's domain never learns shipping's field names, so shipping can rename its model without a cross-team refactor.
- **Compatibility is reviewed, not discovered.** A spec diff in CI shows the consumer exactly what changed before merge.
- **Extraction optionality.** Own schema, own migrations, Contracts-only imports, and a per-context host mean moving a context to its own deployable is deployment work, not remodeling.
- **Governance is executable.** Gates are CI jobs, contract diffs, and waivers with expiry dates. A wiki page is not a gate.

What it costs:

- **Contract tax dominates.** Every cross-context change needs a version decision, a changelog entry, a deprecation window, and two-team review. A one-line field addition is no longer a one-line change.
- **ACL duplication.** N consumers means N translations of the same producer, each with its own tests. There is no shared mapper, by design.
- **Host multiplication.** Per-context host projects multiply CI jobs and configuration even while everything runs in one process.
- **Version skew.** Consumers upgrade on their own schedule, so the producer supports several majors at once and must test them.
- **Decision latency.** A boundary change needs two owners and an ADR. A startup cannot afford this cadence, and a small team inside a large org still pays it.
- **Governance overhead.** Waivers, contract reviews, and ADRs are work. Without automation they become a document nobody reads and a rubber stamp.

## 3. When NOT to use it

1. **Fewer than three teams with independent ownership,** or one release train for the next twelve months. Use `modular-monolith.md`; introduce contracts when a consumer outside the owning team appears.
2. **No contract consumer outside the owning team.** Versioning has no counterparty. A `Contracts` package consumed only by its producer is a folder with a version number.
3. **Any two contexts share an ACID transaction or write the same tables.** They are one context. Merge them; an ACL between two halves of one transaction is an admission that the boundary is wrong.
4. **No named owner per context.** Unowned boundaries are crossed by whoever is on call, and the contract review has no one to approve it.
5. **CI cannot block a merge.** If architecture tests and contract diffs are advisory, governance is theater. Fix the pipeline before adopting this tier.
6. **One team wants microservices "for scale".** This tier adds contract tax without adding autonomy. If one team owns everything, `standard.md` or `modular-monolith.md` delivers the same code organization for less.
7. **The "contract" is generated from the database schema.** Then the schema is the real contract, the ACL is decorative, and the first migration breaks a consumer.
8. **Every context uses a different stack and there is no platform team.** Federation without a shared enforcement layer is N private standards; the gates in §4 will not run anywhere.

## 4. Enforcement

| Concern | Mechanism | Tool examples | Blocks merge |
|---|---|---|---|
| Layer direction inside a context | Project/package references plus architecture tests | NetArchTest, ArchUnit, dependency-cruiser, Nx `enforce-module-boundaries` | Yes |
| Cross-context imports | Other contexts' `Contracts` only, pinned to a major; only the consuming ACL project may reference them | Solution graph scan, Nx `scope:` tags, Gradle `api` vs `implementation` | Yes |
| Schema isolation | One migration set and one database principal per context; no cross-context grants | Per-context migration diff, database grant review | Yes |
| ACL confinement | Only the consuming context's ACL project may name a producer's types | NetArchTest `DoNotResideInNamespace`, ArchUnit package rules | Yes |
| Published contract compatibility | Spec diff against the target branch | `oasdiff breaking`, `buf breaking`, `asyncapi diff`, `graphql-inspector diff` | Yes |
| Export surface drift | API surface snapshot | `Microsoft.CodeAnalysis.PublicApiAnalyzers`, `api-extractor`, `japicmp` | Yes |
| Waivers | Expiry check over `platform/governance/waivers.md` | Scheduled CI job; expired row fails the build | Yes |

Contract diff example:

```yaml
# CI step (GitHub Actions syntax; the command is portable)
- name: Published contract must not break without a major version
  run: |
    git show origin/main:contexts/billing/contracts/inbound/openapi/billing.v2.yaml \
      > /tmp/billing.base.yaml
    oasdiff breaking /tmp/billing.base.yaml \
      contexts/billing/contracts/inbound/openapi/billing.v2.yaml \
      --fail-on ERR
```

Use the equivalent for the contract format: `buf breaking --against '.git#branch=main'` for protobuf, `asyncapi diff` for event documents, `graphql-inspector diff` for schemas. The rule is the same in every format: a removal or a type change is a major version; additions are minor. A deprecated member stays until the next major, and its replacement ships in the same minor. The deprecation window is recorded in the contract's changelog, not assumed.

### Governance and review gates

| Gate | Who decides | Artifact | Blocks |
|---|---|---|---|
| Fit: does this context adopt hexagonal at all? | Context owner + architect | Plan §1 in `plan-template.md` | The first structural pull request |
| Contract change | Producer + at least one consumer | Spec diff and version bump | Merge |
| Boundary change: new cross-context dependency, context merge or split | Owners of both contexts | ADR in `platform/governance/adr/` | Merge |
| Architecture tests | CI | Green build | Merge |
| Waiver expiry | CI | Row in `waivers.md` past `expires` | Merge and release |
| Shared-kernel promotion | Platform owner + every consumer | Pull request to `shared-kernel` | Merge |

Six gates. If any of them is advisory, delete it: an advisory gate teaches people to ignore gates. Waivers are the pressure valve — each row gets an owner, an expiry date, and a ticket; there are no permanent waivers, and an expired row fails CI.

### Deployable-per-context optionality

Optionality is preserved by structure, not by building the split early. All four must hold while the contexts still run in `platform/host`:

1. No shared tables or schema. Each context migrates independently.
2. No transaction spans two contexts. Cross-context consistency is events or explicit compensation.
3. No context imports another context's `src`. Only pinned Contracts cross.
4. Each `host/` starts alone in CI — a smoke test, even if it is never deployed alone.

If any of the four is false, extraction is a rewrite, not a deployment change. Test the fourth one on every pull request; it is the only one that cannot be verified by reading the project graph.
