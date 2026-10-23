# Checklists

> **Read when:** you are reviewing a structure or a migration pull request and need a pass/fail gate, not an explanation.

Rules of use:

- Every item is pass or fail. Record the path or command output that proves it; an unverified item is a fail.
- `N/A` needs a reason. "Not applicable yet" is not a reason.
- Verify the layers the change touches, plus every layer it imports through.
- Commands use ripgrep (`rg`) and build-file inspection. Substitute the stack equivalent and keep the check.
- In tables, `\|` escapes the cell separator; the command itself uses `|`.
- Terminology is defined in `concepts.md`; this file does not restate it.

## 1. Per-layer checklists

### 1.1 Domain

| # | Check | Verify | Fail if |
|---|---|---|---|
| D1 | No framework, ORM, HTTP, or SDK dependency | `rg -n "EntityFramework\|Hibernate\|jakarta.persistence\|typeorm\|prisma\|mongoose\|express\|axios\|Stripe" <domain>`; read build file | any production match |
| D2 | Domain build unit references only itself | `dotnet list <Domain>.csproj reference`; `npm ls --prod`; Gradle `dependencies` | any reference to application, infrastructure, or API |
| D3 | No ambient state or I/O | `rg -n "DateTime.Now\|Instant.now\|new Date\(\)\|Math.random\|process.env\|File\.\|fs\.\|fetch\(" <domain>` | any match |
| D4 | No serialization or transport types | `rg -n "JsonProperty\|Serializable\|\[Json\|HttpStatus\|statusCode\|ResponseEntity" <domain>` | any match |
| D5 | Invariants hold on construction and on every state change | write a test that attempts three invalid constructions or mutations | any invalid object is created |
| D6 | Domain tests need no container, database, or network | test project references; run the suite with infrastructure stopped | tests fail, skip, or hang |
| D7 | Errors are domain errors | `rg -n "throw new (Http\|Sql\|Db\|Stripe)" <domain>` | transport or infrastructure exceptions raised |
| D8 | Export surface is domain types only | read the barrel, index, or public types | a mapper, client, config, or adapter type is exported |

### 1.2 Application

| # | Check | Verify | Fail if |
|---|---|---|---|
| A1 | One handler per operation; the inbound port may be per operation or per module | map entry points to handlers and ports | two handlers for one operation, or an entry point imports a concrete handler |
| A2 | One transaction boundary per operation, owned here | `rg -n "@Transactional\|BeginTransaction\|session.begin\|commit\("` outside application | an entry point or adapter opens or commits |
| A3 | No SQL, HTTP, or SDK calls | `rg -n "SELECT \|DbContext\|HttpClient\|RestTemplate\|axios\|fetch\(" <application>` | any match |
| A4 | Depends on outbound port interfaces only | constructor signatures; `rg "new .*Repository\|new .*Client"` | concrete infrastructure constructed or imported |
| A5 | Authorization decided here | pick three operations, disable the HTTP filter, and confirm each operation still denies | denial exists only in middleware |
| A6 | No persistence type crosses the boundary | handler and port signatures | ORM entity in a public signature |
| A7 | Query paths never mutate | `rg -n "Save\|Update\|Delete\|publish\|emit"` in query handlers | any write or event |
| A8 | No duplicated domain rule | compare handler branches with domain operations | the same condition exists in two or more handlers |
| A9 | Cross-module calls use published contracts | module graph and imports | direct import of another module's internals |

Coarse authentication may stay in the adapter. The business authorization decision must be here or behind an explicit policy port.

### 1.3 Ports

| # | Check | Verify | Fail if |
|---|---|---|---|
| P1 | Contract is capability-shaped; more than seven methods is a review trigger, not an automatic fail | count methods | a method per screen or query |
| P2 | Types are domain or boundary types | read signatures | ORM, HTTP, or SDK types |
| P3 | No vendor or wire format in the contract | `rg -n "Stripe\|S3\|Kafka\|Redis\|Postgres\|HttpRequest\|JSON" <ports>` | any production match |
| P4 | One port per capability, not per entity or table | map ports to aggregates and capabilities | one-to-one with tables |
| P5 | Every port has a production caller and a production implementation | `rg "<PortName>"`; exclude tests | only tests reference it |
| P6 | Port location matches the recorded choice | options record vs tree | split across domain and application |
| P7 | Inbound ports expose operations, not data access | read signatures | `GetAll()`, `IQueryable`, `Find(spec)` |
| P8 | Port errors are declared, not leaked infrastructure exceptions | read throws and result types | `SqlException`, `HttpRequestException` in the contract |

### 1.4 Adapters

| # | Check | Verify | Fail if |
|---|---|---|---|
| AD1 | Translation only | read the changed adapter; count business branches | a rule or policy is implemented here |
| AD2 | No adapter-to-adapter dependency | `rg -n "Infrastructure\|Adapter"` per adapter; read constructors | one adapter injects another |
| AD3 | Framework types stop at the boundary | signatures crossing the port | ORM entity, HTTP request, or SDK type crosses |
| AD4 | One adapter implements one port; one driving adapter calls one inbound port | map adapters to ports | an adapter implements several ports or calls several |
| AD5 | Errors mapped at the edge, cause preserved | exception handler | domain error swallowed or re-typed without cause |
| AD6 | No transaction control | `rg` for transaction APIs | adapter commits or rolls back |
| AD7 | Contract test per port, run against every implementation; integration test per adapter | test tree | only mocked tests, or a port with no contract suite |
| AD8 | Adapter exports nothing another layer imports | barrel and package exports; `rg` imports | another layer imports adapter internals |

### 1.5 Composition root

| # | Check | Verify | Fail if |
|---|---|---|---|
| C1 | One composition root per deployable | locate DI modules and startup files | several, or features build their own graphs |
| C2 | Concretes named only here | `rg -n "new .*(Adapter\|Repository\|Client\|Gateway)"` outside it | constructed in domain, application, or adapter |
| C3 | Lifetimes are correct | review each registration | scoped `DbContext` as singleton; stateful singleton |
| C4 | Every port has exactly one registration; startup fails on a missing binding | `ValidateOnBuild`, `validateDependencies`, or a resolve-all test | a missing registration is discovered at runtime |
| C5 | Config is bound and validated at startup | options validation | environment read inside a layer |
| C6 | No business branching in wiring | read the DI module | `if` on business state selects an implementation |
| C7 | No service locator | `rg -n "GetService\(\|resolve\(\|container\."` | resolved inside a layer |
| C8 | Health and startup checks do not write domain state | read the probes | a probe triggers a write or a migration |

## 2. PR review checklist for a migration change

| # | Check | Evidence required |
|---|---|---|
| 1 | The PR moves or wraps code; it does not change behavior | diff plus green characterization tests; any behavior change split out |
| 2 | Characterization tests land before the move | commit order: `git log --oneline --reverse <base>..HEAD` |
| 3 | Every moved operation has a test at its new boundary | operation-to-test list in the PR |
| 4 | The dependency test is added or updated and fails without the change | revert the import locally, run the test, restore |
| 5 | No schema change | empty migration diff |
| 6 | Public API and DTOs unchanged, or versioned | contract test against a recorded payload |
| 7 | Export surface is not widened | diff of barrels, package `exports`, project references |
| 8 | No new `common` or `shared` folder | tree diff |
| 9 | Old path retained behind a flag, or deleted only after production parity | flag name and removal date |
| 10 | Rollback is documented and executable | revert commit or flag flip, exercised in staging |
| 11 | Vocabulary matches the options record | names in the diff vs `naming-and-modes.md` §5 |
| 12 | Structural diff is one module or at most 400 changed lines excluding pure renames | `git diff -M --stat` |
| 13 | CI runs build, tests, and the arch test on this PR | pipeline link |
| 14 | The owner of the boundary reviewed it | reviewer name and CODEOWNERS entry |
| 15 | No new exemption without an owner and an expiry date | diff of the exclusion list |

A PR that cannot satisfy items 1–4 is not a migration change; it is a rewrite disguised as one.

## 3. Definition of done for a migration phase

- [ ] Phase exit criteria from `plan-template.md` §4 are met, with one link per criterion.
- [ ] Every operation in scope has a characterization test that failed before the move and passes after.
- [ ] The dependency-rule test runs in CI and has been proven to fail on a deliberate violation.
- [ ] Exclusion count is unchanged or lower; every entry has an owner and an expiry date.
- [ ] The old path is removed, or behind a flag with a removal date within two releases or one full business cycle (month-end, quarter-end), whichever is longer.
- [ ] No behavior change: contract tests are green; error codes, payloads, and status codes are unchanged.
- [ ] Rollback has been executed once in staging, with the time to roll back recorded.
- [ ] Metrics, logs, and traces for the moved operations match the baseline for seven days or one release cycle, whichever is longer.
- [ ] No port lacks a production implementation; no adapter lacks a contract or integration test.
- [ ] The project's architecture rules in `AGENTS.md` match the recorded vocabulary and name the enforcement command.
- [ ] Options record unchanged; if it changed, the target tree was regenerated and breaking changes re-checked.
- [ ] Module documentation or ADR states the boundary and the command that verifies it.
- [ ] Debt register updated: every temporary ACL, duplicate path, and TODO has an owner and a date.

A phase that cannot be released alone is not a phase. Stop and split it.

## 4. Red flags that block merge

| Red flag | Why it blocks | Required action |
|---|---|---|
| Domain build unit references an ORM or framework package | the rule is broken at compile time | remove the reference; persistence model and mapper in the adapter |
| Domain or application imports an adapter type | the rule is broken | route through a port |
| An adapter injects another adapter | bypasses the application and its transaction | collapse into one adapter or route through a port |
| Transaction opened in a controller or consumer | non-HTTP entry points lose atomicity | move it to the application operation |
| New port per table or entity, or more than seven methods | the contract is a table gateway | redesign per capability before merge |
| Port exposes ORM, HTTP, or SDK types | the boundary leaks | domain types plus a mapper |
| Arch test disabled, skipped, or exclusion list grew | enforcement was removed | restore it; new exclusions need owner, expiry, and approval |
| Schema migration inside a structural PR | two risks in one rollback | split into two plans |
| Public payload changed without versioning | a breaking change ships silently | version it or make the change additive, plus a contract test |
| Old and new paths both live with no removal date | permanent duplication | add a flag and a date, or delete now |
| A barrel re-exports adapters | reverses the dependency rule | narrow the export surface |
| A port or adapter exists only for tests | the design is driven by the test double | use a fake in test code, not a production port |
| The PR says "hexagonal" but changes no dependency edge | the claim is unverifiable | reject the label; require an edge change or a test |

## 5. Gate summary

Merge only when every touched layer passes, the PR checklist is complete, and there are zero red flags. A phase closes only when §3 is complete. Two consecutive phases with a growing exclusion list stop the migration and trigger a re-assessment with `fit-assessment.md` §4.

## 6. Reviewer output

Report in this shape; it is the review artifact, not a summary of the diff.

```
Verdict:       pass | block
Blocking:      <item ID> <path> — <one line>
Non-blocking:  <item ID> <path> — <one line>
Phase ready:   yes | no (<missing DoD item>)
Evidence gap:  <what was not inspected>
```
