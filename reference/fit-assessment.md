# Fit Assessment

> **Read when:** you have inspected a project's structure and entry points and must decide whether hexagonal architecture fits — before writing a migration plan.

This file produces one verdict: `full`, `selective`, or `not viable`. Terminology comes from `concepts.md`; this file does not redefine it. The verdict describes the code and its constraints, not anyone's preference.

## 1. Procedure

1. Collect evidence for the eight dimensions in §2. Each one needs a path, a count, or a command output.
2. Score the checklist in §3.
3. Apply the verdict rules in §4. The score narrows the answer; the rules decide it.
4. Write the result into `plan-template.md` Section 1, with confidence.

A dimension you could not inspect is recorded as `UNKNOWN — needs <specific input>`. It never counts as a positive signal.

## 2. Judgment dimensions

### 2.1 Domain rule complexity

- **Look for:** rules spanning more than one record (proration, eligibility, discount stacking, tax, allocation, entitlements); state machines with five or more states; invariants that must hold on every write; policies that already exist in code rather than only in the schema.
- **Negative reading:** operations are single-table reads and writes; the only rules are nullability, uniqueness, and foreign keys; the domain folder holds data classes with accessors; decisions live in stored procedures or triggers and will not move.
- **Effect:** low complexity caps the verdict at `selective`; without a second entry point or a real test-isolation need, the honest answer is `not viable`. High complexity is necessary for `full`, never sufficient on its own.

### 2.2 Change frequency

- **Look for:** `git log --since="12 months" --name-only -- <module>` with 12 or more commits on rule-bearing files; more than one person changing them; behavior-changing releases at least monthly.
- **Negative reading:** the module has not changed in 12 months except for dependency bumps, or it changes only when a vendor API changes.
- **Effect:** high churn is what repays the boundary's maintenance cost. Low churn means the ports will be correct and unused.

### 2.3 Number of entry points

- **Look for:** distinct driving adapters per operation — HTTP, GraphQL, queue consumer, scheduler, CLI, admin UI, partner API, mobile backend. Count them per operation, not per service.
- **Negative reading:** exactly one HTTP controller per operation, no consumer, no scheduler, and no concrete plan for a second caller.
- **Effect:** the strongest single signal. Two or more entry points on a rule-bearing operation score +3 and are the usual reason a service that looks CRUD still qualifies. One entry point removes most of the reuse argument; test isolation must then carry the verdict alone.

### 2.4 Infrastructure replaceability

- **Look for:** the number of external systems in the core flow (database, broker, payment, tax, identity, files, email); whether tests run without them; whether a vendor swap or a second provider is on the roadmap; how far SDK types have spread.
- **Negative reading:** one database, no external integration, no swap in three years, and tests already run against an in-memory substitute. A port would wrap something nobody will replace and nobody needs to fake twice.
- **Effect:** volatile, multiple, or compliance-driven integrations raise fit. A fixed single-infrastructure system lowers it but does not veto: test isolation still counts.

### 2.5 Team size and lifetime

- **Look for:** 3–15 engineers owning the code, a roadmap of 18 months or more, an on-call rotation, and expected ownership changes.
- **Negative reading:** one or two engineers shipping a four-month MVP; or 60+ engineers whose dominant problem is team autonomy and independent deploys rather than dependency direction.
- **Effect:** small and short-lived teams cannot afford the indirection. Very large organizations should settle service boundaries first; hexagonal inside one service is a smaller, later decision.

### 2.6 Existing coupling

- **Look for:** import edges from domain to infrastructure (`rg` for ORM and framework packages in the core); entities shared across modules; transactions spanning modules; a `common` folder touched by every pull request.
- **Negative reading, dense form:** every module imports every other; no seam can be wrapped without editing callers. Migration cost approaches rewrite cost.
- **Negative reading, empty form:** modules are already isolated and independently deployable, so the rule documents a property the code already has.
- **Effect:** low-to-moderate coupling concentrated on a few edges is ideal — seams exist and the payoff is visible. Dense, even coupling pushes to `selective` with a strangler boundary or to `not viable`.

### 2.7 Test coverage

- **Look for:** line and branch coverage on the target module (`dotnet test --collect:"XPlat Code Coverage"`, `jest --coverage`, JaCoCo); whether core-rule tests need containers; total suite runtime.
- **Thresholds:** at least 50% line coverage on the target module and one end-to-end path per entry point means phase 1 can start. 20–50% means phase 1 is characterization tests and costs one or two sprints. Below 20% with no harness means no safe refactor.
- **Negative reading:** no tests; tests assert only mock interactions; the suite needs a shared staging database that is frequently unavailable.
- **Effect:** coverage is a readiness constraint, not a fit veto, but it sets the first phase and the confidence. A low-coverage core with a hard deadline is `not viable`, with `Revisit when: characterization tests exist`.

### 2.8 Delivery pressure and release cadence

- **Look for:** trunk-based daily or weekly releases, feature flags, automated rollback, and the ability to ship an internal-only change without a customer release.
- **Negative reading:** quarterly manual releases, no flags, a change freeze, or a single environment.
- **Effect:** a migration needs three to eight independently shippable phases. If no phase can ship within the next eight weeks, the verdict is `not viable`, with `Revisit when: one phase can ship inside eight weeks`; the alternative is to shrink scope until one can.

## 3. Signal checklist

Score every signal you can verify and record the evidence beside it. Do not score what you cannot cite. Maximum +12, minimum −17.

### Strong adopt signals

| ID | Signal | Verify | Pts |
|---|---|---|---|
| A1 | Same rule-bearing operation reachable from 2+ transports | map controllers, consumers, schedulers, CLI to operations | +3 |
| A2 | Rule-bearing module has 12+ commits in 12 months | `git log --since="12 months" --name-only` | +2 |
| A3 | Invariants span multiple writes or entities | schema, transaction usage, aggregate candidates | +2 |
| A4 | 2+ outbound integrations in the core flow, or one replacement planned | adapter inventory, SDK imports, roadmap | +2 |
| A5 | Domain vocabulary already exists in code (entities, policies, value objects) | tree and type names | +1 |
| A6 | Core-rule tests currently require containers or a live database | test setup and CI config | +1 |
| A7 | Roadmap of 18+ months, 3+ engineers, on-call exists | constraints from the team | +1 |

### Strong reject signals

| ID | Signal | Verify | Pts |
|---|---|---|---|
| R1 | Every operation is single-table CRUD with fewer than 5 branches and no invariant | sample 5 handlers, read the schema | −3 |
| R2 | Single entry point, single consumer, no plan for another | entry-point inventory | −2 |
| R3 | Repository is a thin wrapper over one vendor API or SDK | adapter tree, dependency list | −3 |
| R4 | Lifetime under 6 months, prototype, or demo | constraints | −3 |
| R5 | Core rules are enforced in the database and cannot move (triggers, procedures) | schema, migration history | −2 |
| R6 | No characterization tests possible before the deadline (no harness, no staging parity) | CI config, test tree | −2 |
| R7 | No independently shippable increment in the next 8 weeks | release log, flags | −1 |
| R8 | No reviewer capacity for structural change (1 engineer, 5 products) | constraints | −1 |

### Reading the score

| Total | Band |
|---|---|
| 6 or more | `full` candidate |
| 2 to 5 | `selective` |
| 1 or less | `not viable` |

Overrides, applied after the total:

- **R3 or R4 present → `not viable`**, regardless of score.
- **R1 with a total below 4 → `not viable`.**
- **Fewer than two of A1–A4 true → never `full`.**
- **Any −3 signal → never `full`.**
- **`full` requires high confidence.** If DI wiring was not inspected, the verdict is at most `selective`.
- A verdict that contradicts the band must name the overriding signal in one sentence.

The score is a forcing function, not a formula. "The team wants to try it" scores zero; see §7.

## 4. Verdicts

| Verdict | Scope | Must be true | Enforcement |
|---|---|---|---|
| `full` | every module that owns business rules | score 6+, two of A1–A4, no −3, high confidence | one repo-wide dependency rule in CI |
| `selective` | a named list of modules | at least one aggregate with invariants; an enforceable boundary | one boundary test, owned by the core |
| `not viable` | none | no module both needs the rule and can be separated below the cost | alternative architecture, stated |

### 4.1 `full`

The dependency rule applies to every module that owns business rules; every entry point becomes a driving adapter; every outbound integration crosses a port; no core module keeps a direct ORM, HTTP, or SDK dependency. Exempt: generated code, migrations, configuration, and reporting queries that declare their own read model.

`full` does **not** mean four projects per folder, one project per layer, or a service per module. A single project with enforced namespaces can satisfy it. It means the rule is uniform and tested everywhere.

- **Cost:** mapping at every boundary; one extra hop per operation; two to four times the files for a simple feature; review overhead on every PR; an enforcement rule that will be argued with at every deadline.
- **Choose when:** most business code sits in rule-bearing modules, and the team can pay that cost across the whole repository.

### 4.2 `selective`

The rule applies to a named list of modules. Everything else keeps its current style. The two sides meet at one enforced boundary, not at shared types. `selective` is a destination, not a phase of `full`; do not promise that the rest will follow.

Choosing the boundary:

1. **Score per module, not per repository.** A large CRUD monolith with one volatile billing domain scores `not viable` in aggregate and 7/12 for `Billing`; the repository verdict is `selective`.
2. **Shortlist one to five modules** that each contain an aggregate with invariants, plus at least two entry points or high churn.
3. **Cut on an existing boundary** — project, package, workspace package, namespace, or top-level folder. Do not split an existing project in the first phase.
4. **Include the module's ports, application, and adapters. Exclude** shared utilities, reporting, and generated clients.
5. **Check size.** The core should be at most 40% of production code and at most five modules. Above 40%, the destination is `full`; if confidence is below `high`, keep `selective` for now, name the largest enforceable core, and list the evidence needed to price `full`. One module with one entry point and no invariants is a weak `selective`; prefer `not viable`.
6. **Make traffic one-directional.** Legacy calls the core only through inbound ports. The core calls legacy only through an anti-corruption outbound port. No shared entities, no shared `DbContext`, no shared transaction.
7. **Enforce with the strongest available mechanism:** a separate build unit with a reference rule, then a lint zone, then a test. "Code review" is not enforcement.
8. **Add a CI test** that fails when the core imports legacy (excluding the ACL) and when legacy imports core internals. Every exclusion needs an owner and an expiry date.

- **Cost:** two styles in one repository; reviewers must know which side a file belongs to; an ACL to maintain; duplicated build and test configuration.

### 4.3 `not viable`

No module both contains rules worth protecting and can be separated at a cost below the benefit, or the migration cannot be made safe within the delivery constraint. The correct output is an alternative architecture, not a smaller hexagonal structure.

`not viable` is a judgment about fit, not about code quality. A clean CRUD service is not a failed system.

A `not viable` plan must still answer: which alternative, where its boundary sits, and what would have to change for the verdict to flip.

Every `not viable` verdict carries a `Revisit when:` line naming the concrete condition that flips it.

## 5. Alternatives for non-fit cases

| Alternative | When it wins | What it gives up | Keep from this library |
|---|---|---|---|
| **Layered** | One delivery channel; the framework carries routing, validation, and persistence; rules are thin and localized; team turnover is low | Domain independence from the database; a seam for a second consumer | Transaction boundary in the service layer; DTOs at the edge; no SQL in controllers |
| **Modular monolith** | Many feature teams, one deployable; the pain is coupling between modules, not infrastructure | Per-module infrastructure substitution | Module export surfaces; cross-module dependency tests; no shared entities |
| **Transaction script** | Each operation is a procedure with fewer than five branches; rules are local to one request; CRUD plus workflow | Rule reuse across entry points; complex invariants | One script per operation; explicit transaction per script; no shared mutable state |
| **Vertical slice** | Features are independent; delivery speed matters; duplication is cheaper than abstraction; each slice may own its data access | Consistency across features; shared rules; cross-slice transactions | Slice boundaries; no cross-slice imports; contract tests at the edge |
| **Pipeline / ETL** | The system is a sequence of transformations; batch or streaming; no interactive invariants | Request/response modeling; domain objects | Typed stage contracts; idempotency; dead-letter handling |

When the real problem is independent deployment rather than dependency direction, service boundaries come first. Hexagonal inside each service is a separate decision and can be `selective` there.

Minimal shapes for the alternatives — enough to emit Sections 4–5 without inventing a tree:

- **Layered:** `api -> service -> repository`. One project or three; enforce `api ↛ repository`.
- **Modular monolith (non-hexagonal):** `modules/<name>/{api,internal}`. Cross-module access only through `api`; enforce with build references.
- **Transaction script:** `handlers/<operation>`, each owning exactly one transaction. Shared code limited to primitives.
- **Vertical slice:** `features/<name>/{route,handler,data}`. No cross-slice imports; duplicate rather than abstract.
- **Pipeline:** `stages/<n>-<name>` with a typed input and output per stage. Enforce stage order in one orchestrator.

## 6. Worked examples

### (a) Pricing and rules engine, four entry points → `full`

- **Evidence.** `src/Pricing/` holds 9k lines of rate cards, eligibility, and discount stacking. The same operations are reachable from `Api/Controllers/QuoteController.cs`, `Jobs/RepriceJob.cs`, `Messaging/QuoteConsumer.cs`, and `Tools/Reprice/Program.cs`. `Pricing.Domain` references `Billing.Data` for lookups, so rules and SQL are mixed. 61 commits in 12 months.
- **Score.** A1 +3, A2 +2, A3 +2, A4 +2 = 9. No reject signals.
- **Verdict.** `full`. Four entry points already duplicate orchestration; the boundary removes three copies, not one.
- **Cost.** Three adapters and a mapper to maintain; roughly two sprints before the first entry point is deleted.

### (b) CRUD admin panel → `not viable`

- **Evidence.** `admin-api/src/routes/` has 14 route files; 12 map to one table each. Median handler is 18 lines. No domain folder. The only rules are `NOT NULL` and unique indexes. One SPA consumes it, with no queue or scheduler. Tests: 41 supertest cases against a disposable Postgres.
- **Score.** R1 −3, R2 −2, R5 −2 = −7.
- **Verdict.** `not viable`. Hexagonal would add two mappings per table and a port per query, with no rule to protect.
- **Alternative.** Layered CRUD with a thin service per resource; keep validation at the schema and the edge. Revisit when a second consumer (mobile, partner API) or a real rule (billing, entitlement) appears.

### (c) 400k-line monolith, one volatile domain → `selective`

- **Evidence.** `src/` has 23 projects. `Billing` (4 projects) changes three times as often as anything else and holds the only real invariants — proration, tax, credits. The rest is CRUD, reporting, and imports. `Billing` is called from HTTP, a monthly job, and a webhook consumer.
- **Score.** `Billing`: A1 +3, A2 +2, A3 +2 = 7. The other 19 projects score R1 −3 each; scoring the repository as a whole would hide the only module that matters.
- **Verdict.** `selective`, core = `Billing.*`.
- **Boundary.** `Billing.Domain`, `Billing.Application`, and `Billing.Ports` reference no `Shared.Data`. `Billing.Api` and `Billing.Infrastructure` are the only projects allowed to reference both sides. `Shared.Data` may not reference `Billing.*`.
- **Enforcement.** Project-reference rule plus one NetArchTest test in `Billing.Tests`; CI fails on violation.
- **Cost.** Two build graphs, an ACL over `Shared.Data`, and reviewers who must know which side a new file belongs to.

## 7. What is not evidence

- **"The team wants to try it."** A preference is not a property of the code. It scores zero, cannot appear in "Why this verdict", and belongs only in Section 2 as a constraint on cost and morale.
- A conference talk, a book, or a competitor's architecture page.
- Folder names such as `Domain/` or `Application/`. Check imports; the names prove nothing.
- A README claiming Clean, Hexagonal, or DDD.
- "We are moving to microservices." That is a deployment-boundary decision, not this one (`concepts.md` §9).
- One architect's conviction, or a past failure of a different architecture.
- A dependency graph image without a rule that fails the build.

What counts instead: paths, counts, import edges, churn, entry points, coverage, release history, and team constraints.

## 8. Handoff

| Verdict | Section 1 must state | Sections 4–5 must show |
|---|---|---|
| `full` | score, signals, confidence, repo-wide cost | phases across all rule-bearing modules; one enforcement rule |
| `selective` | modules inside and outside the boundary, why each side was chosen | both sides of the boundary; boundary test; phases only inside the core |
| `not viable` | alternative and the evidence that rules it in | the alternative's target shape and boundary; no hexagonal tree |

Confidence: `high` requires structure, entry points, and DI wiring inspected. `full` is not emitted at `medium` or `low`. If the score indicates `full` but confidence is below `high`, emit `selective` with the likely boundary and list the inspections that would upgrade it.
