# Agent Rules

> **Read when:** the target structure is confirmed and you must encode the architecture as rules an agent (or a new developer) can follow — typically into the project's `AGENTS.md`.

A plan describes a destination. Rules keep the codebase at that destination after the agent that wrote the plan is gone. This file defines what to write, where, and how to keep it true.

## 1. Two artifacts, two audiences

| Artifact | Location | Size budget | Loaded |
|---|---|---|---|
| **Rule block** | project root `AGENTS.md` | ≤ 60 lines | every agent session, automatically |
| **Rule detail** | `docs/architecture/rules.md` (or the repo's docs folder) | ≤ 150 lines | on demand, linked from the block |

The split is not cosmetic. `AGENTS.md` is loaded into context on every task, including tasks that have nothing to do with architecture. A 400-line architecture essay there is skimmed, then ignored, and it taxes every unrelated request.

The block carries **only what an agent must know before touching a file**: dependency direction, the layer map with real paths, where new code goes, what is forbidden, and the command that proves compliance. Everything else — rationale, examples, migration history — lives in the detail file.

When the block exceeds 60 lines, cut in this order: layer descriptions to one line each, then the placement table to four rows, then the options record to a pointer at the detail file. Never cut the Verify command or the Ask-before list.

If the project has no docs folder and no appetite for one, the detail file is optional. The block is not.

## 2. What belongs in the block

| Include | Exclude |
|---|---|
| Dependency rule, one sentence, with the allowed direction | Why hexagonal architecture is valuable |
| Layer map: folder → responsibility, using the recorded vocabulary | Tutorials, definitions, diagrams |
| Placement table: change type → location | Class, method, or field naming |
| Forbidden imports, named by package or path | Framework or database advocacy |
| One enforcement command that actually exists | Aspirational rules with no check |
| The options record (mode, vocabulary, port location) | Migration phase history |
| "Ask before" list | The five-section plan |
| Link to the detail file | Duplicated detail |

A rule that cannot be checked is a preference. Mark preferences as preferences, or leave them out.

## 3. The block template

Fill every `<>`. The paths must match the target tree exactly — a rule that names a folder which does not exist teaches the agent to invent it.

```md
## Architecture

This project follows hexagonal architecture. The dependency rule is enforced in CI.

Options: mode `<usecase | command|query | input|output|response | hybrid>`; domain vocabulary
`<model/shared/errors/ports>`; port location `<domain | application>`; presentation `<name>`.

### Layers

- `<domain path>` — entities, value objects, domain services, domain errors.
  No framework, ORM, HTTP client, SDK, clock, or filesystem.
- `<application path>` — use cases and port contracts. May import the domain only.
- `<inbound adapter path>` — HTTP/queue/CLI entry points. May import the application.
  Must not import another adapter.
- `<outbound adapter path>` — persistence and external gateways. May import the
  application and domain types.
- `<composition path>` — wiring only. No business rules.

### Where new code goes

| Change | Location |
|---|---|
| A business rule or invariant | `<domain path>` |
| A new operation | `<application path>/<mode folder>/<name>/` |
| A new external system | port in `<port path>`, adapter in `<outbound adapter path>` |
| A new transport | `<inbound adapter path>` |
| Wiring | `<composition path>` |

### Rules

- Business rules never live in a controller, handler, repository, or UI component.
- Domain types never cross the transport boundary; map to a DTO in the adapter.
- Do not add a package to `<domain path>` without an approved exception.
- Do not call another adapter. Route through the application.
- Structural changes (new module, new layer, moving a boundary) require a plan first.
- If this file conflicts with an architecture test, the test wins; fix this file.

### Ask before

- Adding a module or project, or a dependency to `<domain path>`.
- Moving a type across a boundary.
- Changing the options record.

### Verify

Run `<exact command>`. A violation fails the build. Do not merge with a skipped
architecture test.

Detail: `<path to detail file>`.
```

For a `selective` verdict, the block opens with the boundary instead of claiming the whole repo. State the boundary in **target-tree paths**, not in the pre-migration module names — translate the core's logical modules to the projects or folders that own them after the move.

```md
Hexagonal rules apply to `<core paths>` only. Everything under `<legacy path>` keeps its
current structure. Do not introduce domain/application folders outside the core.

Legacy code may reference exactly one core contract: `<ACL port>`, declared in `<port path>`
and implemented by the core. It may not reference `<domain path>` or `<application path>`
otherwise.
```

For `enterprise`, add one line: other contexts are reachable only through their published `Contracts`; the local anti-corruption layer is the only translator.

## 4. What the detail file adds

Only material that would bloat the block:

- the full allowed/forbidden import matrix
- port design rules: capability-shaped, no ORM types, where errors are translated
- boundary models: domain vs DTO vs persistence, and who maps
- transaction ownership
- the exceptions register (see §6)
- links to the architecture tests, one per rule

Keep the detail file subordinate: if it contradicts the block, the block is wrong or the detail file is stale. Fix both in the same change.

## 5. Wording rules for agent-facing text

An agent follows text that is imperative, scoped, and checkable. It ignores text that hedges.

| Write | Not |
|---|---|
| "Business rules live in `src/domain`." | "Business logic should ideally be separated." |
| "`src/domain` must not import `@prisma/client`." | "Keep the domain clean." |
| "Run `pnpm arch:test`; a violation fails the build." | "Architecture is verified in review." |
| "Add a port in `application/ports/`." | "Consider abstracting the dependency." |

Also required:

- **One scope per rule.** State the path or the package. A rule with no scope is applied everywhere and enforced nowhere.
- **No duplication of the linter.** If the tool already forbids an import, the block states the intent and the command; it does not restate every pattern.
- **Precedence.** Write one line: "If this file conflicts with an architecture test, the test wins; fix this file." Otherwise the two drift silently.
- **The "ask before" list.** Name the changes an agent must not make unilaterally: adding a module or project, adding a dependency to the domain, moving a type across a boundary, changing the options record. This is the cheapest guard against an agent "helpfully" restructuring a repository.

## 6. Exceptions

Every real codebase needs them, and an unrecorded exception becomes precedent.

| Field | Meaning |
|---|---|
| Rule | Which rule is waived |
| Scope | Exact path, package, or type |
| Reason | The constraint that justifies it |
| Owner | A person, not a team alias |
| Expiry | A date or a condition; `permanent` needs a stronger reason |
| Check | How the exception is encoded so the tool does not fail on it |

Put the register in the detail file. The block gets one line: "Approved exceptions are listed in `<detail file>`; do not add one without an owner and an expiry."

An exception with no expiry is a silent architecture change.

## 7. Keeping the rules true

- **Same-change rule.** A PR that changes structure updates the block in the same PR. A follow-up documentation ticket is a broken rule.
- **Test first.** The enforcement test is the source of truth. If a rule cannot be expressed as a test, say so in the block and mark it `manual`.
- **Options record is authoritative.** The block's folder names come from the recorded vocabulary. If the project renames a layer, the block changes in that commit.
- **Review checkpoint.** Add one item to the PR checklist: "Architecture rules in `AGENTS.md` still match the tree and the options record." One line, checked every time.
- **No orphan rules.** When a rule's subject disappears (a port removed, a module merged), delete the rule.

## 8. Anti-patterns

| Symptom | Consequence | Fix |
|---|---|---|
| 300+ lines of architecture prose in `AGENTS.md` | Loaded every session, read once, ignored | Keep the block ≤ 60 lines; move detail out |
| Rules without paths | Agent guesses which folder applies | Name the path or package |
| Rules that restate the linter | Two sources of truth; one goes stale | State intent + command, not patterns |
| "Best practices" with no check | Unenforceable, ignored under pressure | Make it checkable or drop it |
| Rules contradicting `README.md` or the tests | Agents follow whichever they read first | One source: the test; align the rest |
| Vocabulary drift (`Entities` in the block, `Aggregates` in the tree) | Agent creates a parallel folder | Copy the options record verbatim |
| Exceptions with no owner or expiry | Permanent, invisible | Register with owner + expiry |

## 9. Output checklist

Before returning the rules artifact:

- [ ] Every path in the block exists in the target tree or is explicitly marked as created in phase 1.
- [ ] Vocabulary matches the options record exactly.
- [ ] The block is ≤ 60 lines.
- [ ] At least one rule names the enforcement command.
- [ ] A `selective` block states the boundary and what is explicitly outside it.
- [ ] The "ask before" list is present.
- [ ] The exceptions policy is stated, even if the register is empty.
- [ ] No class, method, or field naming is prescribed.
