# Chromus Skills

A collection of Claude Code skills by [Giovanni Guarino](https://github.com/ChromuSx).

## Installation

### As a marketplace (recommended)

First, add the marketplace:

```bash
claude plugin marketplace add chromus-skills https://github.com/ChromuSx/chromus-skills
```

Then install:

```bash
claude plugin install chromus-skills@chromus-skills
```

### Manual

Clone this repository and point your Claude Code plugin config to the local directory.

## Available Skills

### deep-review

A comprehensive **17-point code review checklist** for full-stack projects (.NET/C#, Angular/React, or any backend+frontend stack). Also works as a **Spec/Design review** sub-mode for reviewing design documents BEFORE implementation — the cheapest moment to catch contract and concurrency errors.

Goes beyond syntax checking to catch:
- Contract-vs-implementation mismatches ("always X" in the spec, best-effort in the code)
- Functional bugs (incomplete domain handling, missing validations)
- Security gaps (OWASP multi-tenant data isolation, cross-tenant leakage)
- Concurrency, retry and idempotency bugs (duplicate requests, parallel actors, lost updates, broken distributed consistency)
- Amplified inherited tech debt (a "pre-existing bug" that the new feature makes routine)
- Framework compromises that could have been eliminated with a built-in primitive (transactional outbox, optimistic concurrency, idempotency key)
- Performance anti-patterns (N+1 queries, resource leaks, sync-over-async)
- DTO/Form/Entity misalignment
- Runtime failures (unregistered components, broken reactive chains)
- Accessibility issues (WCAG 2.2 Level AA)

**The checkpoints:**

| #  | Check                          | Focus                                                          |
|----|--------------------------------|----------------------------------------------------------------|
| 0  | Contract Adherence             | Every declared invariant vs concrete failure modes             |
| 1  | End-to-End User Flow           | Every actor, every mandatory scenario                          |
| 2  | Domain Completeness            | All enum/group values handled everywhere                       |
| 3  | View Parity                    | Same data, same rendering across views                         |
| 4  | Business Invariants            | Required fields enforced at every level                        |
| 5  | Build and Warnings             | Zero errors, zero new warnings                                 |
| 6  | Cross-Component Consistency    | Same pattern applied everywhere                                |
| 7  | Serialization and Pre-fill     | Data round-trips correctly                                     |
| 8  | i18n and Labels                | All keys exist in all languages                                |
| 9  | DRY and Centralization         | No duplicated logic                                            |
| 10 | DTO-Form-Entity Alignment      | Fields match across all layers                                 |
| 11 | Security and Data Isolation    | OWASP multi-tenant, scope filtering                            |
| 12 | Runtime Simulation             | What happens at runtime? Includes concurrency, retry, idempotency |
| 13 | Performance                    | N+1, leaks, async, indexes                                     |
| 14 | Accessibility (optional)       | WCAG 2.2 AA compliance                                         |
| 15 | Amplified Tech Debt            | Does the new feature amplify an inherited bug?                 |
| 16 | Framework Due Diligence        | Is there a built-in primitive that eliminates the compromise?  |

**Three modes:**
- **Report mode** - Produces a structured findings table without editing files
- **Fix mode** - Finds issues AND applies fixes, then builds to confirm
- **Spec/Design Review sub-mode** - Reviews a design document BEFORE implementation; proposes spec edits instead of code changes

**Trigger phrases:** "review", "code review", "deep review", "check my changes", "controlla il codice", "review di sicurezza", "spec review", "design review", "review del design", "review documento", "contract review", "design adherence check"

### verified-planning

A **pre-finalization verification checklist** for implementation plans — runs 3 mandatory checks against real code before declaring a plan "ready to implement".

Prevents the 3 most common planning failures that adversarial review (Codex or similar) catches after the fact:
- Type mismatches between pseudocode and real entity/interface definitions
- Semantic leaks in filter/permission logic (caught only by concrete scenario simulation)
- Technically impossible patterns with no codebase precedent (e.g., async API inside EF Core query filter)

**The 3 checks:**

| # | Check | What it catches |
|---|-------|-----------------|
| 1 | Type-check pseudocode vs real code | `int` vs `Guid`, `string` vs `enum` mismatches from imprecise agent summaries |
| 2 | Simulate concrete filter/permission scenarios | Cross-scope data leaks invisible to abstract reasoning |
| 3 | Verify new patterns have codebase precedent | Infeasible API usages in constrained contexts (EF filters, sync policies, static DI) |

Each check requires concrete evidence (file paths read, scenarios listed, grep results) — not generic "I verified this" statements.

**Trigger phrases:** "piano approvato", "pronto per implementazione", "definitivo", "final plan", "finalizza il piano", "dai il via libera", exit plan mode with non-trivial plan

## Contributing

Feel free to open issues or PRs to suggest new checkpoints or improvements.

## License

MIT
