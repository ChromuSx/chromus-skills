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

A comprehensive **14-point code review checklist** for full-stack projects (.NET/C#, Angular/React, or any backend+frontend stack).

Goes beyond syntax checking to catch:
- Functional bugs (incomplete domain handling, missing validations)
- Security gaps (OWASP multi-tenant data isolation, cross-tenant leakage)
- Performance anti-patterns (N+1 queries, resource leaks, sync-over-async)
- DTO/Form/Entity misalignment
- Runtime failures (unregistered components, broken reactive chains)
- Accessibility issues (WCAG 2.2 Level AA)

**The 14 checkpoints:**

| # | Check | Focus |
|---|-------|-------|
| 1 | End-to-End User Flow | Simulate every actor's interaction |
| 2 | Domain Completeness | All enum/group values handled everywhere |
| 3 | View Parity | Same data, same rendering across views |
| 4 | Business Invariants | Required fields enforced at every level |
| 5 | Build and Warnings | Zero errors, zero new warnings |
| 6 | Cross-Component Consistency | Same pattern applied everywhere |
| 7 | Serialization and Pre-fill | Data round-trips correctly |
| 8 | i18n and Labels | All keys exist in all languages |
| 9 | DRY and Centralization | No duplicated logic |
| 10 | DTO-Form-Entity Alignment | Fields match across all layers |
| 11 | Security and Data Isolation | OWASP multi-tenant, scope filtering |
| 12 | Runtime Simulation | What happens when it actually runs? |
| 13 | Performance | N+1, leaks, async, indexes |
| 14 | Accessibility (optional) | WCAG 2.2 AA compliance |

**Two modes:**
- **Report mode** - Produces a structured findings table without editing files
- **Fix mode** - Finds issues AND applies fixes, then builds to confirm

**Trigger phrases:** "review", "code review", "check my changes", "controlla il codice", "review di sicurezza"

## Contributing

Feel free to open issues or PRs to suggest new checkpoints or improvements.

## License

MIT
