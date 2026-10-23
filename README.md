# Hexagonal Architecture Skill

An agent skill that decides whether a project should adopt hexagonal architecture, then produces an incremental migration plan and a concrete target structure.

Most architecture material starts with a folder tree. This one starts with a verdict. Not every project should be hexagonal, and the skill says so when that is the case.

## What it produces

Every run returns five sections, plus an optional sixth artifact:

1. **Fit verdict** — `full`, `selective`, or `not viable`, with evidence and confidence.
2. **Benefits and drawbacks** — grounded in the project's size, team, change rate, and coupling.
3. **Breaking changes** — public API, DI, config, serialization, schema, export paths, deployment, build.
4. **Migration plan** — phases with exit criteria and rollback. Existing systems are wrapped and moved inward, never rewritten.
5. **Target architecture** — directory tree, dependency rule, enforcement tool, export map.
6. **Project rules** *(optional)* — a ≤ 60-line block for the project's `AGENTS.md`, plus a detail-file outline, written only after the structure is confirmed.

## How it works

```
inspect -> decide -> draft -> confirm options -> emit
```

- **Inspect** reads structural evidence only: manifests, entry points, DI wiring, the tree, a sample of imports. No claim without a path.
- **Decide** applies `reference/fit-assessment.md`, which scores eight dimensions and allows `not viable`.
- **Draft** fills the five-section template.
- **Confirm** asks the user for the application-layer mode and the directory vocabulary, preferring whatever the project already uses.
- **Emit** selects exactly one layout tier and generates the tree with the confirmed vocabulary.

## Structure

```
SKILL.md                    entry point: gates, workflow, routing
AGENTS.md                   rules for maintaining this repo
reference/
  concepts.md               terminology; the dependency rule
  fit-assessment.md         verdicts and alternatives
  plan-template.md          the five-section output contract
  naming-and-modes.md       application modes and folder vocabularies
  layouts/
    index.md                tier selection
    minimal.md
    standard.md
    modular-monolith.md
    enterprise.md
  ports-and-adapters.md     port design and boundary models
  migration.md              strangler, seams, characterization tests
  anti-patterns.md          what goes wrong and how to spot it
  checklists.md             review gates
  agent-rules.md            rules encoded into a project's AGENTS.md
  examples/                 csharp.md, java.md, typescript.md, angular.md
```

## Install

Copy this folder into your agent's skills directory so that the entry point is `<skills>/hexagonal-architecture/SKILL.md`.

- OpenCode, global: `~/.config/opencode/skills/hexagonal-architecture/`
- OpenCode, project-local: `.opencode/skills/hexagonal-architecture/`

No build step, dependencies, or scripts. It is Markdown only.

## Design principles

- **Fit first.** A structure answer before a verdict is a guess.
- **Evidence over assumption.** Greenfield is never assumed; every claim cites a path.
- **Proportionality.** Structure scales with domain complexity and ownership, not with ambition. The catalog is explicitly not a maturity ladder.
- **Selective adoption is first-class.** Applying the rule to one core module and leaving CRUD modules alone is a valid, recorded outcome.
- **Naming is structural.** `Aggregates` vs `Entities` is a directory decision. Class and method names are out of scope.
- **Enforcement or it did not happen.** Every layout names a real check: NetArchTest, ArchUnit, dependency-cruiser, Nx module boundaries.

## Scope

In: architecture style, layers and bounded contexts, directory structure, dependency direction, export surface.

Out: class/method/field naming, interface prefixes, framework or database choice, code generation.

## Extending

- **New layout tier:** add the tier file, a row in `reference/layouts/index.md`, and a route in `SKILL.md`. All three, or the tier is unreachable.
- **New stack example:** add one file under `reference/examples/` and one routing row in `SKILL.md`.
- **New knowledge:** goes into `reference/`; `SKILL.md` gains a pointer, not content.

See `AGENTS.md` for the full maintenance contract.

## Status

The skill content is complete. The repository has no build, test, or lint tooling by design.

## License

MIT.
