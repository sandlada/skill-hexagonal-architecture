# AGENTS.md

## 1. What this repo is

A general-purpose agent **skill library**, not an application. It ships a skill that helps developers adopt hexagonal architecture. Consumers are AI agents and, secondarily, humans reading `README.md`.

The deliverable is never prose alone. Every invocation produces: a **verdict**, a **migration plan**, a **target structure**, and a **way to verify the dependency rule**.

## 2. Scope

In scope (this is what the skill reasons about):

- Architecture style, layers, bounded contexts.
- Directory and file structure.
- Dependency direction and allowed references.
- Export surface: what each module or layer exposes, including barrel files, package `exports`, and project references.
- Structural vocabulary: whether a folder is called `Aggregates` or `Entities`.

Out of scope (do not spend effort here, do not argue about it):

- Class, method, variable, and DTO field naming.
- Interface prefixes (`IFoo` vs `Foo`), file name casing, formatting.
- Framework, database, or messaging choice.

Naming appears in this library only as **structure vocabulary**. Choosing `Aggregates` vs `Entities` is a directory decision and belongs here; naming the class inside it does not.

## 3. Charter

Must:

- **R1 Inspect before advising.** Read the project's real structure, never assume greenfield.
- **R2 Judge fit first** (workflow step 2, the fit gate).
- **R3 Produce the five-part plan** (see §5).
- **R4 Ask before committing to structure options** (workflow step 4, the options gate).
- **R5 State tradeoffs.** Every recommendation names its cost and what it gives up.
- **R6 Make the dependency rule checkable** with a concrete tool for the project's stack, not a diagram.
- **R7 Keep depth in `reference/`.** `SKILL.md` stays thin.

Must not:

- Treat hexagonal architecture as the default answer.
- Recommend a full rewrite of an existing system.
- Skip inspection or the options gate.
- Become a DDD or framework tutorial.
- Emit a scaffolded application. Examples illustrate structure; they are not a generator.
- Override a project's own `AGENTS.md` or established conventions.

## 4. Workflow

```
1 Inspect -> 2 Decide -> 3 Draft plan -> 4 Confirm options -> 5 Emit
```

**1 Inspect.** Read only structural evidence: solution/project files, `package.json` and workspaces, `tsconfig` paths, module graph, entry points, DI registration, directory tree, a small sample of imports. Do not crawl leaf files. Every claim in the verdict must cite a real path.

**2 Decide** (fit gate). One of three verdicts: `full`, `selective` (hexagonal for the core domain only), or `not viable`.

**3 Draft plan.** Fill the five sections in §5 using `reference/plan-template.md`. Mark the option set as a recommendation.

**4 Confirm options** (options gate). Ask the user for the application-layer mode and the directory naming scheme (§6). Do not proceed without an answer.

**5 Emit.** Produce the target tree, dependency rules, and export map using the confirmed vocabulary.

## 5. Plan contract

Every plan answers these five questions, in this order:

1. **Can this project migrate to hexagonal architecture?** Verdict, evidence, confidence, assumptions.
2. **What are the benefits and the drawbacks?** Grounded in this project's size, team, change rate, and coupling. No generic lists.
3. **Are there breaking changes?** Check each: public API and DTOs, DI registration, config keys, serialization formats, DB schema, module export paths, deployment units, build and packaging.
4. **What is the migration plan?** Phases, order, rollback points. Characterization tests before extraction, seams before moves, strangler wrapping instead of replacement.
5. **What does the result look like?** Target directory tree, dependency rule, export boundaries, and which modules actually change.

`selective` keeps all five sections: Section 4 is scoped to the core, Section 5 shows both sides of the boundary. `not viable` replaces Sections 4-5 with the alternative's target shape and boundary; no hexagonal tree is emitted.

## 6. Structure options

Three decisions, all recorded in the options record.

**Application-layer mode:** `usecase` | `command|query` | `input|output|response` | hybrid. Coexistence is allowed, but one module must not mix modes within a single flow.

**Directory naming scheme:** one vocabulary per layer, chosen from the observed alternatives: `Aggregates|Entities|Entries|Objects`, `Common|Commons|Primitives`, `Error|Errors|DomainErrors`, `Adapter|Adapters|Repositories|Ports`. `presentation` and `api` are the same layer; in backend projects the presentation layer may be a dedicated Api port project.

**Port location:** `domain` or `application`. Both satisfy the dependency rule; pick one, record it, and never split the same concern across both.

Rules:

- **Inherit first.** If the project already uses a convention, recommend keeping it. Propose a new scheme only where no pattern exists.
- **Record the choice** in the plan and do not revisit it later in the same session.
- The mapping table in `reference/naming-and-modes.md` is bidirectional: it recognizes an existing vocabulary and generates structure from a chosen one. Use it so a project does not need renaming to be understood.

## 7. File ownership and routing

```
SKILL.md                  entry point: gates, decision flow, output template, routing table
reference/
  concepts.md             ports, adapters, driving/driven, dependency rule; sole terminology source
  fit-assessment.md       decision dimensions, signal checklist, verdicts, alternatives
  plan-template.md        fill-in template for the five-part plan
  naming-and-modes.md     application modes and directory vocabularies
  layouts/
    index.md              selection entry: comparison table and decision flow
    minimal.md
    standard.md
    modular-monolith.md
    enterprise.md
  ports-and-adapters.md   port design, boundaries, DTO mapping
  migration.md            strangler, seams, characterization tests, phasing
  anti-patterns.md
  checklists.md           review checklist and definition of done
  examples/               csharp.md, typescript.md, angular.md, java.md
README.md                 human-facing; never loaded as agent instructions
AGENTS.md                 this file
```

Rules:

- New knowledge goes to `reference/`. `SKILL.md` gains a pointer, not content.
- Every reference file opens with "when to read this". No orphans: everything must be reachable from the `SKILL.md` routing table.
- Layout routing is two-step: `layouts/index.md` to choose a tier, then the specific tier file. Do not load all tiers at once.
- One topic per file. No content duplicated between `SKILL.md` and `reference/`.
- `concepts.md` is the only place terminology is defined. Other files reference it instead of redefining.
- Examples cover only C#, TypeScript, Angular, and Java. Angular is separate from TypeScript because the mapping differs: UI is a driving adapter, the API client is a driven adapter, DI tokens are ports, and module boundaries come from Nx or ESLint rather than compiler references.

## 8. Quality gates and current state

Before calling work done:

- Layout files have four parts: tree, why it is split this way, when **not** to use it, and how the dependency rule is enforced.
- Fit verdicts cite paths, and every recommendation states its cost.
- A new layout tier requires three edits: the tier file, a row in `layouts/index.md`, and a route in `SKILL.md`.
- Changing the layout or tooling means updating this section.

Current state (2026-10): all 20 Markdown files exist — `AGENTS.md`, `SKILL.md`, `README.md`, and 17 files under `reference/` (8 top-level, 5 layouts, 4 examples). There is no git repo, build, test, or lint tooling — do not invent commands. Verification is structural: headings, routing reachability, relative links, table shape.
