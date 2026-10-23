# Layouts

> **Read when:** you have a fit verdict of `full` or `selective` and must choose the structure to emit. Read this file, run the decision flow, then open exactly one tier file.

A tier answers two questions: **how many boundaries exist** and **what enforces them**. It does not change the dependency rule (`concepts.md` §1). Tiers are not a maturity ladder: a five-person CRUD service on the enterprise tier pays contract tax with no counterparty, and a 300-use-case system on the minimal tier has a boundary nobody can check.

## 0. Vocabulary

Every tree in this catalog uses the defaults from `naming-and-modes.md`:

- model grouping `Aggregates`, shared `Primitives`, errors `Errors`
- outbound ports in `application/Ports/`
- driving adapters grouped by direction (`adapters/inbound` in folder tiers, `Adapters.Inbound.*` in project tiers), with one composition root (`Api`, `Host`, `bootstrap`, or `composition/` depending on tier)
- application mode `usecase`

**This vocabulary is configurable per the options record (`naming-and-modes.md` §5).** Substitute the inherited or chosen names throughout the tree you emit; the shape, dependency rules, and enforcement do not change. Never mix two vocabularies in one tree.

## 1. Comparison

| Tier | Project size | Team shape | Bounded contexts | Module enforcement | Migration cost | Ceremony |
|---|---|---|---|---|---|---|
| [minimal](minimal.md) | 1 deployable, up to ~15 use cases, thin domain | 1–2 developers, one owner | 1 | Folder rules in a lint/architecture test; the compiler sees one assembly | **Low** — folders move inside one project | **Low** — four folders, one composition file |
| [standard](standard.md) | 1 service, 5–60 use cases, a domain with real invariants | 1 team, 3–8 developers | 1 | Project/package references plus a build-blocking architecture test | **Medium** — every file moves into a layer project, imports rewritten | **Medium** — five production projects (including the host), four test projects |
| [modular-monolith](modular-monolith.md) | 2–8 modules, 40–300 use cases, one deployable | 1–5 teams sharing one release | 2–8 | Compiler project graph, Contracts-only crossing, CI architecture tests | **High** — extract modules, introduce Contracts, split schema ownership | **High** — five projects per module, cross-module contracts, outbox |
| [enterprise](enterprise.md) | 8+ contexts, several deployables, external contract consumers | 3+ teams, one named owner per context | 8+ | Versioned contract packages, per-context CI gates, ACL confinement, expiring waivers | **Very high** — contracts, ACLs, per-context hosts, release pipelines | **Very high** — contract reviews, deprecation windows, governance gates |

Migration cost means converting an existing codebase **into** this tier, not upgrading from the tier above. Ceremony is what the team maintains every sprint afterward. Read the size numbers as signals, not admission criteria: when size and ownership disagree, ownership wins. A 40-use-case system owned by four teams needs module boundaries more than a 400-use-case system owned by one.

## 2. Decision flow

Answer in order. Stop at the first terminal.

1. **Is the fit verdict `not viable`?**
   → **STOP — use a simpler architecture.** Do not apply any tier. Use the alternative named in the plan (layered, transaction script, vertical slice, pipeline, or the non-hexagonal modular monolith in `fit-assessment.md` §5). Hexagonal folders around code with no independent domain rules add indirection and buy nothing.

   **`selective` is not a tier.** Its structural home is §4: choose the core module's internal tier with steps 3–4, keep supporting modules in their current shape, and enforce the seam.
2. **Can you name at least two bounded contexts — distinct vocabulary and distinct reasons to change?**
   - No → 3.
   - Yes → 5.
3. **Can you name one domain invariant that is not a database constraint and changes independently of the UI and the schema?**
   - No → **STOP.** This is CRUD. Use a transaction script or vertical slices; revisit when a second invariant appears.
   - Yes → 4.
4. **Does the domain's isolation need to be enforced by the build graph (project or package references) rather than a lint rule, or will there be more than one inbound adapter?**
   - No → **[minimal](minimal.md)**.
   - Yes → **[standard](standard.md)**.
5. **Do any two contexts share one ACID transaction or write the same tables?**
   - Yes → they are one context. Merge them, then go to 3.
   - No → 6.
6. **Does each context have a named owner, and is at least one contract consumed by someone outside that owner's team?**
   - No → **[modular-monolith](modular-monolith.md)**.
   - Yes → **[enterprise](enterprise.md)**.

Flow rules:

- If evidence is missing, answer **no**. Overbuilding is paid every sprint; upgrading is paid once and can be scheduled.
- Choose per repository, not per company. Two services in the same organization can sit on different tiers.
- A context with no invariant you can state without its schema does not get hexagonal folders, even inside a modular or enterprise tree. Keep it CRUD and put the boundary at its Contracts project; §4 covers the mixed case.
- If two tiers seem to fit, take the lower one unless the higher tier's enforcement is required by an external constraint: compliance, a published package, or multiple owners.
- A "context" whose vocabulary you cannot distinguish from its neighbor is a folder. Answer no at step 2.

## 3. Tier files

Open exactly one. Each tier file follows the same four-part contract: target tree, why it is split this way, when not to use it, enforcement. Read only the one you chose.

- **[minimal](minimal.md)** — one project, folder-level layering, lint-enforced direction.
- **[standard](standard.md)** — one service, one project per layer and adapter direction, build-enforced direction, executable architecture tests.
- **[modular-monolith](modular-monolith.md)** — several modules, each internally hexagonal, crossing only through Contracts.
- **[enterprise](enterprise.md)** — several contexts and teams, ACLs, contract-first inbound ports, versioned exports, governance gates.

No full trees appear in this file. Each tree lives in exactly one tier file; duplication is how catalogs start contradicting themselves.

## 4. Selective adoption: mixing tiers

`selective` is a scope verdict, not a fifth tier. Its structural home is a **modular-monolith tree in which the core module uses the `standard` split (Domain, Application, inbound and outbound adapters) and supporting modules use the `minimal` folder set (domain, application, adapters) without a composition root**.

- The core module is chosen by rule density and change rate, not by lines of code.
- The seam is enforced, not drawn. When other modules call the core, the seam is a `Contracts` project. When the core sits inside one application with no other modules, the seam is a build unit or namespace plus one architecture test: legacy may not import the core's internals, and the core may not import legacy except through one anti-corruption port.
- Supporting modules that own rules use the minimal folder set; pure CRUD supporting modules stay CRUD with no hexagonal folders. Do not wrap them to look consistent; the asymmetry is the decision.
- A supporting module exposes a `Contracts` project only when another module calls it. A leaf module needs none; adding one "for symmetry" creates a surface nobody consumes.
- One repository, one enforcement run. A module without a check is not selectively adopted, it is unenforced.
- The boundary is recorded in the plan (`plan-template.md` §1). Moving it later is a new decision, not cleanup.

## 5. Upgrade and downgrade

| Move | What changes | Cost | Reversible |
|---|---|---|---|
| minimal → standard | Every file moves into a layer project; imports rewritten; architecture test added | 1–3 days per 10k LOC, one release of churn | Yes |
| standard → modular monolith | Modules extracted around proven boundaries; Contracts introduced; schema ownership split | 2–6 weeks; highest risk is the data layer | Partially |
| modular monolith → enterprise | Per-context hosts, contract versions, ACLs, governance gates | 1–2 quarters | No — the review process follows the structure |
| enterprise → modular monolith | Delete hosts and version folders; keep ACLs and contracts as internal packages | Weeks | Yes |
| standard → minimal | Merge projects; add folder lint | Days | Yes |

Upgrade only after a boundary has survived a real change: two features delivered through it, or one incident contained by it. Pre-building the next tier is how teams end up with enterprise ceremony and startup delivery.

## 6. What this catalog does not decide

- Framework, database, messaging, and deployment topology are out of scope by charter (`AGENTS.md` §2). The catalog sizes structure; it does not pick technology.
- Microservices are not a fifth tier. The enterprise tree already supports per-context deployables; a separate tier would rename an operational decision as an architecture decision.
- Class, method, and field naming are out of scope. `Aggregates` vs `Entities` is in scope because it is a directory decision; the type inside is not.
