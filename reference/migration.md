# Migration

> **Read when:** the system already exists and you are planning how to move it behind ports; you need seam discovery, phasing, coexistence, or rollback rules.

This file covers existing systems only. For greenfield work, phases are slices of the target architecture, not migrations; see `plan-template.md` §4. Terminology is defined in `concepts.md`; this file assumes the verdict is `full` or `selective`. For `selective`, apply it only to the modules inside the boundary; modules outside keep their current style. A `not viable` verdict replaces the migration plan with an alternative architecture (see `fit-assessment.md`).

## 1. Stance: wrap, then move inward

The domain is extracted from working code, not rebuilt beside it. The old code remains the only implementation of a behavior until the new one has served production traffic. Each phase adds a boundary and moves one behavior behind it.

A rewrite is rejected even when its estimate looks smaller:

| Apparent saving | Actual cost |
|---|---|
| No legacy constraints | The old system is the only specification. Customer-specific branches, retry quirks, implicit ordering, and data repairs exist only there. A rewrite rediscovers them through incidents. |
| One cutover instead of many releases | Cutover is the only irreversible step. Every earlier phase can be rolled back; the cutover cannot. |
| No need to understand the old code | The old code is read anyway during reconciliation. The rewrite adds a second system without removing the first. |
| Less coordination | Both systems run in parallel: two deploy pipelines, two on-call surfaces, and reconciliation of writes. |
| A cleaner target | Wrapping reaches the same target shape. The rewrite reaches it and pays for a second system on the way. |
| Faster first release | Value lands at cutover, not at the first commit. Wrapping ships value per phase. |

The estimate usually compares writing the new system against doing nothing. The real comparison is writing the new system, plus running both, plus reconciling data, plus the incidents caused by behavior nobody documented.

A rewrite is defensible only when the system is being replaced for a non-architectural reason (end of vendor support, incompatible runtime) and the business has accepted a parallel run. Even then, the seam work in §3 is required to migrate the data and the behavior.

## 2. Order of operations

A slice is one operation or one aggregate. A phase that moves a whole layer is a rewrite with a schedule.

1. Characterization tests
2. Find the seam
3. Introduce the port
4. Wrap the old path
5. Move logic inward
6. Delete the old path

### Step 1 — Characterization tests

- **Produces:** a deterministic suite pinned to observable behavior at the candidate seam, plus a list of behaviors that could not be tested. Each untestable behavior is either a hidden dependency or a bad seam.
- **Verify:** green on unchanged code; green in random order; a deliberately broken rule makes a test fail; runtime stays within the agreed CI budget. A suite too slow to run per commit will be skipped, and a skipped suite is not a safety net.

Rules:

- Assert responses, persisted state, emitted messages, and side effects. Do not assert internal call sequences.
- Cover every existing branch, including branches that look like bugs. They are behavior; record them and decide later whether to keep them.
- When a path is hard to reach, record production requests and responses at the entry point and replay them as golden-master tests.
- Place tests at the outermost stable boundary (HTTP, message, CLI) so they survive internal moves.
- Do not refactor while writing them. A failure during a refactor cannot tell you whether the test or the code is wrong.

### Step 2 — Find the seam

- **Produces:** a seam inventory (§3) with one chosen seam and a named fallback.
- **Verify:** the call can be intercepted without editing callers; input and output are observable; reverting is a configuration or binding change, not a code change.

Seam candidates come from the inventory in §3. Tests are written against the best candidate; this step confirms it before a port is introduced. If confirmation fails, return to step 1 with the next candidate and keep the tests that still apply.

### Step 3 — Introduce the port

- **Produces:** the inbound port, plus any outbound ports the slice needs. The existing service or repository becomes the first implementation or sits behind one. Nothing moves yet.
- **Verify:** all callers compile against the contract; the old implementation passes the characterization suite through the port; no framework, ORM, or transport type appears in the signature; the dependency test covers the new namespace and still passes.

### Step 4 — Wrap the old path

- **Produces:** an adapter that implements the inbound port by delegating to the old path. The old entry point (controller, consumer, scheduler) calls the port.
- **Verify:** recorded requests produce identical responses, rows, and messages; the flag selects old or new at the edge; no business rule is duplicated.

This is the strangler skin: new boundary, old core. If the new adapter cannot call the old core, the seam is wrong, not the adapter.

### Step 5 — Move logic inward

- **Produces:** domain rules extracted in behavior-sized slices, an application handler that owns the transaction, and outbound ports replacing direct data access.
- **Verify:** the characterization suite is unchanged and green; the old path delegates to the new core (or both call a shared core); a shadow comparison shows zero differences for the agreed window.

Rules:

- Move one decision per change: a pricing rule, an eligibility check, a state transition. Moving a thousand-line service in one commit is a rewrite with extra steps.
- Invert the delegation only after the new path is proven. Until then the new core is called by the old path, not the other way around.
- Keep the old entry point alive. Deleting it is step 6.

### Step 6 — Delete the old path

- **Produces:** old adapter, service branches, flag, DI registration, configuration, and tests of old internals removed. The module exports only the new contract.
- **Verify:** telemetry shows zero calls to the old path for a full business cycle; no imports of the old namespace; build, dependency test, and characterization suite green; the deletion ships alone.

Keep the characterization suite: it now runs against the new path. Remove only tests that asserted old internals. A dormant path behind a permanently off flag is not deleted — it still compiles, appears in reviews, and gets edited by mistake.

## 3. Seam discovery

A seam is an existing call boundary that can be intercepted without editing callers. Look first at boundaries the framework already provides, then at boundaries the code created by accident.

| Seam | Where it appears | How to wrap it | Watch |
|---|---|---|---|
| Entry point | Controller action, route handler, CLI command, queue consumer, scheduler job, middleware terminus | New inbound adapter calls the existing handler | Auth and tenant middleware: wrap after it, or the new path bypasses policy |
| Framework callback | ORM lifecycle hook, event subscriber, application event listener, middleware filter | Register a new listener that calls the port; keep the old listener during coexistence | Ordering and implicit writes; callbacks often fire inside the transaction |
| Transaction boundary | Service method with commit, `@Transactional`, `atomic`, `SaveChanges` | Treat the method as the use case; put the port in front of it | Implicit commits inside ORM calls; nested transactions |
| Duplicated branch | Same condition on status, role, or tenant in three or more handlers | Extract the decision; route every path through it | Branches often differ per channel in ways nobody documented |
| Direct data access in a handler | SQL string, query builder, or ORM query in a controller or service | Put an outbound port in front; keep the query text unchanged | Dynamic filters and tenant predicates are behavior |
| God service | One service with many public methods touching many tables | Cluster methods by aggregate touched and by caller; extract one cluster | Private helpers shared across clusters hide coupling |
| Existing module boundary | Separate project, package, or workspace with few inbound imports | Cheapest seam: enforce the export surface first, then wrap | A `common` or `shared` project usually defeats the boundary |
| External integration wrapper | Payment, email, ERP, or tax client already behind a class | Declare the port inside; the wrapper becomes the adapter | SDK types in the signature; retries hidden inside the wrapper |
| Reporting path | Dashboard or export queries | Low-risk first slice: leave it as a read model behind a query port | Do not force it through the domain; it has no invariants to protect |

Rank seams by four properties, in this order:

1. **Callers unchanged** — a seam that needs ten call sites edited is a refactor, not a seam.
2. **Observable** — input and output can be recorded. An unobservable seam cannot be characterized.
3. **Reversible** — a flag or binding switch returns to the old path.
4. **Small** — one operation or one aggregate, reviewable in one sitting.

Record the inventory before choosing:

| Path | Seam kind | Callers | Behavior pinned? | Flaggable? | Notes |
|---|---|---|---|---|---|
| `<path>` | entry point / transaction / ... | `<count and paths>` | yes / no | yes / no | |

## 4. Prioritization

Score each candidate slice on three dimensions, 1–3:

```
Priority = change frequency × test weakness × business value
```

| Dimension | 1 | 2 | 3 | Measure from |
|---|---|---|---|---|
| Change frequency | Less than once a quarter | Monthly | Weekly or a known hotspot | VCS history for the directory, incident tickets, backlog |
| Test weakness | Covered at the seam by fast, deterministic tests | Partial, integration-only, or slow | No coverage, flaky, or manual QA | Test suite, branch coverage at the seam, release checklist |
| Business value | Supporting or internal | Revenue-adjacent | Core differentiator or regulated | Product owner, cost of an incident in this module |

Bands: 18–27 first, 8–17 second, 1–7 last or never. Within a band, choose the better seam and the smaller slice.

The number is ordinal, not precise:

- It ranks exposure, not effort. A high score means "most risk if left alone", not "easiest to move". High scores require characterization before any extraction.
- One dimension at 1 collapses the product. Strong tests make a core module score low: that means it is cheap to move when touched, not that it is unimportant.
- Do not average scores across teams or compare them across quarters. The scales are judgments, not measurements.
- Re-score after each phase. Change frequency moves once a module is wrapped and once the team stops avoiding it.
- If two dimensions are guesses, write `UNKNOWN` and choose by seam quality. A fabricated 2 is worse than an explicit gap.
- Never convert the score into a schedule. It orders work; it does not size it.

Example: `orders` 3×3×3 = 27, `invoices` 2×2×1 = 4, `catalog` 1×1×2 = 2. Start with `orders`; its harness makes `invoices` cheaper later.

## 5. Coexistence

Two paths will run at once. These rules keep them from becoming two systems.

| Mode | Reads | Writes | Allowed |
|---|---|---|---|
| Delegate | New path calls the old core | Old core | Yes — step 4 |
| Shadow read | New path computes, old path answers the caller | Old only | Yes, with diff telemetry |
| Canary | Split by cohort or operation | One path per request | Yes, with reconciliation |
| Dual write | Both | Both | No |
| Parallel logic | Both implement the rules | One or both | Only for a bounded comparison window |

Rules:

- **One writer per aggregate.** If both paths can write, one delegates to the other. Two independent writers on the same tables are the failure mode that makes rollback impossible.
- **One source of truth.** The schema does not change (§6). Once the slice has an outbound adapter, both paths use that one adapter; do not give the new path its own repository implementation during coexistence.
- **Flags live at the inbound edge.** One flag per operation, default old, named owner, expiry date, kill switch. The application layer must not read migration flags; the adapter selects the handler. Log flag state with the request id.
- **Shadow reads only.** A shadow write to the primary store is a dual write with a different name. Use a replica or a sandbox.
- **Canary writes need reconciliation.** Define the divergence threshold, the query that detects it, and the automatic flip-back before enabling the canary.
- **No dual writes to two stores.** If a phase seems to need one, the phase is wrong or it is a schema plan. The legal version is a transactional outbox with one writer and an idempotent consumer.
- **Keep the old path compiling.** A path broken behind an off flag cannot serve as a rollback.
- **Deletion protocol:** flag off for a full business cycle (month-end and quarter-end when money is involved) → telemetry at zero → remove entry point, flag, DI registration, configuration, and tests of old internals in one release → remove dead code and unused tables only under a separate schema plan.
- **Record parity evidence.** Diff counts, canary results, and a named sign-off. "It looks fine" is not an exit criterion.

Every migration flag has an expiry date and an owner. A flag missing either becomes permanent branching.

## 6. Data and schema

A structural migration preserves the schema: table names, columns, constraints, indexes, and ownership. If the target requires a schema change, split it into a second plan with its own phases, verification, and rollback.

The split is mandatory because a schema change makes rollback a data problem. A code revert restores behavior only if the data still fits the old shape.

| Allowed in a structural phase | Not allowed |
|---|---|
| New code paths over existing tables | Renames, splits, or merges of tables and columns |
| Read-only compatibility views | New required columns |
| Mapping changes inside an adapter, deployed with the code | Changed nullability or types |
| Moving code ownership without moving data ownership | Moving a table to a new owner and switching the mapping in the same release |

An index or a nullable column is a data change, not a structural one: ship it separately and do not bundle it with a behavior move.

When a schema change is unavoidable, use expand/contract, one release per step:

1. **Expand** — add the new structure; keep the old. One writer; dual-write only inside that writer or through triggers.
2. **Backfill** — idempotent, resumable, batched; checkpoint progress; verify counts and spot-check records.
3. **Switch reads** — move readers over; keep writing both shapes until reads are proven.
4. **Contract** — stop writing the old shape, wait a full cycle, drop it in a separate release.

Every step needs a rollback that does not require restoring a backup. A backfill without an idempotency key is a one-shot operation, not a migration.

Data ownership follows the same rule: after a split, exactly one module owns writes for a table or aggregate; others read through a port or a published read model.

## 7. Stack notes

### Layered monolith

Typical shape: controller → service → repository/DAO, transaction on the service.

- The service's public method is the use-case seam. Put the inbound port in front of it and let the service be the first implementation.
- Extraction target: decisions (eligibility, pricing, state transitions) move to the domain; orchestration and the transaction stay in the application; data access moves behind outbound ports implemented by the existing DAO.
- Do not move CRUD. A service that only forwards fields is already the application layer; wrapping it adds a hop and no rule.
- A large service is usually several clusters. Split by aggregate and by caller, not by technical layer, and migrate one cluster.
- Existing DAO tests become outbound adapter contract tests. Keep them and point them at the adapter.

### Framework-heavy applications (Spring, Rails, Django)

Rule: framework types stop at the adapter, and the domain is framework-free. The one exception is a transaction annotation on the application handler: it marks the use-case boundary but must not pull framework behavior into the domain. Where the application must stay framework-free, a transaction decorator at the composition root replaces it.

| Stack | Framework types stay in | Watch |
|---|---|---|
| Spring | `@RestController`, `@Repository`, JPA entities, `@Transactional` on the application handler | JPA entities in domain signatures; Spring events treated as domain events; `@Transactional` on controllers or repositories |
| Rails | Controllers, ActiveJob, ActiveRecord models as the driven adapter | `after_save` / `after_commit` callbacks are hidden writes; validations split between ActiveRecord and the domain; `update_column` bypasses callbacks |
| Django | Views, forms, serializers, models as the driven adapter, management commands, Celery tasks | Signals are implicit seams; QuerySets crossing into the application; `transaction.atomic` in views; overridden `save()` with side effects |

Notes:

- **Spring:** map JPA entities to domain objects inside the persistence adapter. Do not annotate the domain to save mapping code; that reverses the dependency rule.
- **Rails:** ActiveRecord fuses persistence, validation, and domain. Either accept it as the driven adapter and keep rules in plain objects (`selective`), or map ActiveRecord to domain (`full`). Do not claim `full` while domain objects are ActiveRecord subclasses.
- **Django:** replace signals with explicit outbound port calls during extraction. A signal handler is invisible to the caller and cannot be ordered reliably.
- In all three, the DI container is the composition root. An annotation does not make a class part of a layer.

The cost is a mapping layer at each framework boundary. That mapping is the price of a testable domain; skipping it leaves the ORM or the framework in charge of the model.

### Frontend

- The UI component is the driving adapter. The router is an inbound adapter. The API client is a driven adapter. A store or facade is the closest thing to an application layer.
- Framework types (component decorator, route params, framework HTTP client, framework forms) stop at the adapter boundary.
- One use case per user flow, not per component. A component that calls three endpoints for one action is hiding a use case.
- The transaction boundary is the optimistic update and its rollback, or the multi-request flow. If it spans requests, model the compensation explicitly.
- Test the component against a fake use case, the use case against a fake API port, and the API adapter against recorded payloads.
- Angular: the DI token is the port, `HttpClient` stays in the adapter, and the standalone component is the driving adapter. See `examples/angular.md`.

## 8. Rollback, stop conditions, metrics

### Rollback per phase

| Phase | Rollback | Data implication |
|---|---|---|
| Characterization | Delete or revert the tests | None |
| Port | Revert the commit; restore the previous DI binding | None |
| Wrap | Flip the flag to the old entry point; redeploy the previous release | None |
| Move inward | Flip the operation flag to the old implementation | None |
| Delete | Redeploy the previous release; restore from VCS | None |

The only irreversible step in a migration is a data change the old code cannot read, or a dropped structure. That is why §6 splits schema work into its own plan. Code phases stay reversible even after deletion, as long as the schema is untouched. Design the flag and the rollback before the first write, not after.

### Stop conditions

Hold the phase and escalate when any of these is true:

- Characterization tests cannot be made deterministic after two attempts. A hidden dependency remains.
- The seam requires editing callers in more than three modules. The seam is wrong.
- A second writer appears — including a batch job, a trigger, or an admin script.
- Boundary violations rise for two consecutive phases.
- Module defect rate exceeds the agreed baseline (for example 2× for two weeks).
- The phase reaches twice its estimate before the first shippable slice. Split it; do not extend it.
- The diff cannot be reviewed by one reviewer in a day. It is a rewrite in disguise.
- A schema change turns up mid-phase. Stop and split the plan (§6).
- No named owner for the module. Unowned migrations stall at the first incident.

### Metrics

| Metric | Definition | Source | Direction |
|---|---|---|---|
| Change failure rate | Releases in the module that cause an incident or rollback | Pipeline and incident tracker | Down |
| Cycle time | Commit to production for module changes | VCS and deploy records | Down |
| Test runtime at the seam | p95 of the characterization suite | CI | Flat or down |
| Boundary violations | Count from the dependency test | CI | To zero; new violations fail the build |
| Escaped defects | Incidents traced to a moved behavior | Tracker | Not up |
| Old-path traffic | Calls to the pre-migration path | Telemetry | To zero before deletion |
| Flag age | Days since a migration flag was enabled | Flag system | Under one business cycle |

Metrics detect regression during migration; they do not prove the architecture. The dependency test — the stack-specific tool recorded in the plan — is the proof.

## 9. Worked example: four phases in a mid-size monolith

Context: commerce monolith, ~150k LOC, 8 developers, weekly release, one database. Modules: `orders`, `invoices`, `catalog`, `admin`. `orders` holds a controller, a ~1,800-line service, a DAO mixing SQL and ORM, and an invoice client. Sixty percent of changes and most incidents touch `orders`.

Scores: `orders` 3×3×3 = 27, `invoices` 2×2×1 = 4, `catalog` 1×1×2 = 2. Start with `orders`; `invoices` follows once the harness exists.

| Phase | Scope | Produces | Exit criteria | Rollback |
|---|---|---|---|---|
| 1 Pin | HTTP characterization tests for place, cancel, and status change; seam inventory | Deterministic suite, golden payloads, branch list with reasons for gaps, chosen seam | Green on unchanged code, random order, within the CI budget | Tests only |
| 2 Port and wrap | Inbound ports for the three operations; adapter delegates to the existing service; flag `orders.new-core` defaulting to old | Controllers depend on ports; one implementation; no logic moved; flag wired | Identical responses, rows, and messages; p95 within 10% of baseline | Revert the controller binding |
| 3 Move inward | Pricing, eligibility, cancellation policy, and status transitions to the domain; transaction to an application handler; DAO behind an outbound port | One flag per operation; the old service delegates per operation once proven | Characterization suite unchanged and green; shadow diff at zero for two weeks; canary 10% then 100%; outbox counts match order counts | Flip the operation flag to the old implementation |
| 4 Delete | Old branches, old DAO calls, flag, DI registration, tests of old internals | Module exports ports and domain; one writer; dependency test; short ADR recording the boundary | Zero old-path traffic for 30 days including month-end; no imports of the old service; CI green | Redeploy the previous release; data untouched |

Phase 3 takes most of the time. If the service is too large, repeat phase 3 per operation rather than growing the phase. The second module is cheaper: the test harness, flag convention, and dependency test already exist.

What this example deliberately does not do: no table renames, no framework upgrade, no service extraction, no DTO cleanup, no microservice split. Each would add a second variable to a change that already carries risk.
