---
name: hexagonal-architecture
description: Assess whether a project fits hexagonal architecture, then produce an incremental migration plan and a concrete target structure. Use when asked to design, review, or migrate to hexagonal or ports-and-adapters architecture; to choose a directory structure for a service, modular monolith, or enterprise multi-context system; or to judge whether a codebase should adopt hexagonal architecture at all.
---

# Hexagonal Architecture

A decision skill, not a tutorial. It inspects a project, produces a verdict, and only then produces structure.

## Output contract

Every invocation that produces a verdict returns these five sections, in this order, plus an optional sixth when rules are requested:

1. **Fit verdict** — `full`, `selective`, or `not viable`, with evidence and confidence.
2. **Benefits and drawbacks** — tied to this project's constraints, including who pays each cost.
3. **Breaking changes** — checked against all eight areas, with `none` only if verified.
4. **Migration plan** — phases, exit criteria, rollback; wrap and move, never rewrite.
5. **Target architecture** — tree, dependency rule, enforcement tool, export map.
6. **Project rules** *(optional)* — the rule block to paste into the project's `AGENTS.md`, plus the detail-file outline.

A prose answer without all five is incomplete. `selective` keeps all five: Section 4 covers only the core, Section 5 shows both sides of the boundary. `not viable` replaces Sections 4–5 with the recommended alternative and emits no hexagonal tree. A request with no project evidence gets the tier comparison plus an explicit "no verdict possible", never a fabricated plan.

The sixth artifact is not part of the five-section contract. Emit it on request, or by default once the target is accepted; never in place of Section 5.

## Workflow

### 0. Establish scope

Identify the module or service under discussion. If the user names the whole repository but the question is about one module, confirm the boundary before inspecting.

If the target repository has its own `AGENTS.md`, conventions, or architecture rules, read them first; they override this skill's defaults.

### 1. Inspect (mandatory)

Read structural evidence only:

- solution/project files, `package.json`, workspaces, `tsconfig` paths, Gradle/Maven modules
- entry points and composition root
- DI registration
- directory tree, two levels deep
- 5–10 representative imports crossing layers

Do not crawl leaf files. Every architectural claim must cite a path.

If there is no repository access, ask for the tree, the manifest, and one entry point. Do not proceed on assumptions; lower the confidence if the evidence is partial.

### 2. Decide

Read `reference/fit-assessment.md`. Produce exactly one verdict.

- `not viable` → stop the hexagonal path. Name the alternative (layered, modular monolith, transaction script, vertical slice, pipeline). Do not emit hexagonal folders.
- `selective` → name the modules inside the boundary and the ones outside it.
- `full` → continue.

"Cleaner" and "the team wants to try it" are not evidence.

### 3. Draft the plan

Read `reference/plan-template.md`. Fill Sections 0–5. Mark the option set as a recommendation; it is not yet confirmed.

### 4. Confirm options (mandatory gate)

Read `reference/naming-and-modes.md`. Ask for three decisions:

1. Application-layer mode: `usecase`, `command|query`, `input|output|response`, or hybrid?
2. Directory vocabulary: which variant per layer, and `presentation` vs `api`?
3. Port location: `domain` or `application`?

Recommend inheriting whatever the project already uses. If the user does not answer, stop. Do not default silently.

### 5. Emit the target

Read `reference/layouts/index.md`, run its decision flow, then open **exactly one** tier file. Apply the options record from step 4. Include the enforcement tool for the project's stack.

### 6. Emit the project rules (on request)

Read `reference/agent-rules.md`. Produce the rule block for the project's `AGENTS.md` from the confirmed options record, plus the detail-file outline. The block is ≤ 60 lines, path-specific, and names one enforcement command.

Emit rules only after the target is confirmed. Rules written before the structure settles encode a structure that will change.

### 7. Self-check

Run the checklist at the end of `reference/plan-template.md`, then the gate in `reference/checklists.md` §5. If a check fails, fix the plan before returning it.

## Routing

| Need | Read |
|---|---|
| Meaning of a term; which layer a file belongs to | `reference/concepts.md` |
| Fit verdict and alternatives | `reference/fit-assessment.md` |
| Output shape and breaking-change checklist | `reference/plan-template.md` |
| Application mode and folder vocabulary | `reference/naming-and-modes.md` |
| Choose a structure tier | `reference/layouts/index.md` |
| Tier detail | `reference/layouts/minimal.md`, `standard.md`, `modular-monolith.md`, `enterprise.md` |
| Port and adapter design | `reference/ports-and-adapters.md` |
| Existing system migration | `reference/migration.md` |
| Smells and failure modes | `reference/anti-patterns.md` |
| Review gate and definition of done | `reference/checklists.md` |
| Encode rules into the project's `AGENTS.md` | `reference/agent-rules.md` |
| Stack-specific structure | `reference/examples/csharp.md`, `java.md`, `typescript.md`, `angular.md` |

Load only what the current step needs. Layouts are two-step: index first, then one tier. Examples are opt-in: one file, matching the project's stack.

## Hard rules

- **Fit before structure.** A structure answer before a verdict is a guess.
- **No rewrites.** Existing systems get wrapped and moved inward.
- **Structure only.** Folder and module names are in scope; class, method, and field names are not.
- **Every recommendation states its cost.**
- **Every layout names an enforcement tool**, not "code review".
- **`selective` is a first-class outcome**, not a compromise.
- **Do not mix vocabularies** in one target tree.
- **Rules are written after the structure is confirmed**, in the recorded vocabulary, with a command that proves them.

## Refusals

- "Give me the enterprise folder structure" for a CRUD admin panel → refuse, explain, offer the alternative.
- "Rewrite it as hexagonal" → refuse, offer the phased migration.
- "Just tell me the folders" without a project → give the tier comparison, but state that no verdict is possible.
- A project with no tests, no seams, and a change freeze → `not viable`, with `Revisit when: a seam exists`; the prerequisite is characterization tests, not architecture.
