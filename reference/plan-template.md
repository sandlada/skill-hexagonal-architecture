# Plan Template

> **Read when:** you are about to write the migration plan. Fill this in; do not invent a different shape.

The five sections are the contract. Keep the headings and the order. Unknowns get `UNKNOWN — needs <specific input>`, never a confident guess.

## Rules

- Every claim about the current system cites a real path. A plan without paths is a blog post.
- Every benefit is tied to a named constraint of *this* project. "Cleaner code" and "better separation" are not benefits.
- Every drawback names who pays for it: the team, the delivery date, the on-call rotation, or the reviewer.
- Confidence is defined by evidence coverage, not by tone:
  - `high` — structure, entry points, and DI wiring all inspected.
  - `medium` — structure and entry points inspected, wiring inferred.
  - `low` — partial tree only.

- Section 0 records evidence; it is not one of the five contract sections.
- With no repository access, mark each row `reported by the user` and cap confidence at `low`. Never fabricate a path.

---

## Section 0 — Evidence

| Path | What it shows |
|---|---|
| `<path>` | `<layer structure, coupling, entry point, DI registration, ...>` |

Also state the scope you did **not** inspect.

## Section 1 — Fit verdict

```
Verdict:     full | selective | not viable
Confidence:  high | medium | low
Constraints: <team size, delivery pressure, release cadence, lifetime, compliance>
```

Then:

- **Why this verdict.** Three to five sentences tied to the evidence above.
- **What would change the verdict.** The specific fact that would move it, and in which direction.
- **If `selective`:** the modules inside the boundary, the modules outside it, and where the boundary is enforced.
- **If `not viable`:** the recommended alternative — layered, modular monolith, transaction script, vertical slice, or pipeline — and why it fits better.

## Section 2 — Benefits and drawbacks

| Benefit | Who gains | Evidence |
|---|---|---|
| `<benefit>` | `<role>` | `<path or metric>` |

| Drawback | Who pays | Mitigation |
|---|---|---|
| `<drawback>` | `<role>` | `<action, or "accepted">` |

Drawbacks that must be considered, and answered explicitly even when the answer is "not applicable here":

- More indirection: more files and one extra hop per operation.
- Boundary maintenance: the rule decays without an automated check.
- Mapping cost: separate domain, DTO, and persistence models.
- Team learning curve and review overhead.
- Migration period where both shapes coexist.

## Section 3 — Breaking changes

Check every row. Write `none` only after checking.

| Area | Risk | Evidence | Mitigation |
|---|---|---|---|
| Public API and DTOs | Shape, field names, nullability, status codes | | Versioning, additive change, compatibility layer |
| DI registration | Lifetimes, registration order, missing bindings | | Register through the composition root; keep the old registration until parity |
| Config keys | Renamed or relocated keys | | Support both keys for one release, log a deprecation |
| Serialization | JSON/protobuf shape, enum representation, date formats | | Contract tests against recorded payloads |
| Database schema | Table or column moves, ORM mapping | | No schema change in a structural refactor; use views if unavoidable |
| Module export paths | Import paths, package `exports`, project references, barrel files | | Re-export shims for one release |
| Deployment units | Process count, startup order, health endpoints | | Deploy behind a flag; keep the old unit until verified |
| Build and packaging | Assembly/package names, entry assemblies | | Update pipelines in the same change |

Verdict line: `Breaking changes: none | internal only | public | UNKNOWN (needs <access>)` — with the list of public ones.

**Structural refactors should not change the schema.** If a migration requires a schema change, split it into two plans.

## Section 4 — Migration plan

Ordering rules, non-negotiable:

1. Characterization tests around the behavior being moved.
2. Identify seams — existing boundaries that can be wrapped without editing callers.
3. Wrap, then move logic inward.
4. Delete the old path only after the new one is proven in production.

| Phase | Scope | Exit criteria | Rollback |
|---|---|---|---|
| 1 | `<modules>` | `<tests green, flag shipped>` | `<revert commit / flip flag>` |
| 2 | | | |

Each phase must be independently shippable. A phase that cannot be released alone is not a phase; it is a rewrite.

For existing systems, use `migration.md` for the technique. For greenfield, phases are slices of the target architecture rather than migrations.

## Section 5 — Target architecture

- **Tree.** The target directory structure, using the recorded vocabulary. Show only the parts that change, plus enough context to locate them.
- **Dependency rule.** The allowed references, as a table or a short list.
- **Enforcement.** The concrete tool and rule for this stack: NetArchTest or ArchUnitNET for C#, ArchUnit for Java, dependency-cruiser or ESLint boundaries for TypeScript, Nx `enforce-module-boundaries` for Angular.
- **Export map.** What each changed module exposes after migration.
- **Modules changed.** Explicit list. Anything not listed is out of scope.

If the verdict is `not viable`, replace this section **and Section 4** with the target shape and boundary of the recommended alternative. If `selective`, keep both sections: Section 4 covers only the core, Section 5 shows both sides.

## Options record

Copy the block from `naming-and-modes.md` §5. It must be filled before Section 5 is written.

## Self-check before emitting

- [ ] Every section is present, in order.
- [ ] Every current-state claim cites a path.
- [ ] No benefit or drawback is generic.
- [ ] All eight breaking-change rows were checked.
- [ ] The migration plan has rollback for every phase.
- [ ] The target tree uses the recorded vocabulary only.
- [ ] Enforcement names a real tool, not "code review".
- [ ] Confidence matches the evidence coverage.
