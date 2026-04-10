---
name: verified-planning
description: >
  Pre-finalization verification for implementation plans — runs 3 mandatory checks against real code
  BEFORE declaring a plan "ready to implement". Catches type mismatches, semantic leaks in filter logic,
  and technically impossible patterns that adversarial review (Codex or similar) would flag later.
  MUST be invoked before writing "piano approvato", "pronto per implementazione", "definitivo", "final plan",
  or any equivalent phrase. Also trigger when the user says "finalizza il piano", "chiudi il piano",
  "dai il via libera", or before exiting plan mode with a non-trivial plan.
  The 3 checks are non-negotiable prerequisites for plan delivery — not optional follow-ups.
  Do NOT declare a plan final until all 3 checks are explicitly completed with concrete evidence.
---

# Verified Planning — Pre-Finalization Plan Verification

You are about to declare an implementation plan final. Stop. Run the 3 checks below before presenting the plan as ready to implement. Each check requires concrete evidence — not "I believe this is correct", but actual file reads or scenario simulations.

## Before You Start

1. **Identify the plan scope**: What entities, interfaces, query filters, migrations, and services does the plan reference?
2. **List all technical claims**: Every type reference, API usage, and filter semantic is a claim that must be verified.
3. **Do not trust agent summaries**: Intermediate agents in long threads lose type precision. Go back to the source files.

---

## The 3 Mandatory Checks

### Check 1 — Type-check pseudocode against real code

**What to do:**

For every piece of pseudocode in the plan that references types (tuples, interfaces, query filters, pattern matches, migrations), read the actual source files for the entities and interfaces cited and verify that EVERY type matches exactly.

**How to execute:**

- Use `Read` or `Grep` directly on the source files — do not rely on summaries from earlier in the conversation
- For each field cited in the plan, confirm: is it `Guid` or `int`? `string` or `enum`? nullable or required?
- If the plan says `(Guid? CompanyId, int? CountryId)` — open the entity file and confirm the actual field types

**Why this fails without the check:**

Intermediate agents summarize entities as prose. Prose loses precision. A summary saying "the entity has CountryId" does not tell you whether it's `int` or `Guid`. If you used the summary instead of the file, your pseudocode has wrong types.

**Real failure pattern:**

Plan proposed `(Guid? CompanyId, int? CountryId, int? CityId)`. Real entities used `Guid` for all three. A verification agent had provided the correct information in the same thread, but it was not propagated to the final plan document. The plan was wrong.

**Evidence required to mark complete:**

State which files you read, and for each type reference in the plan, confirm the exact type found in the source.

---

### Check 2 — Simulate concrete scenarios for every filter/permission rule

**What to do:**

For every filter predicate or authorization rule in the plan, construct 2–3 concrete examples in the form:
`(role=X, record with fields=Y) → should this user see the record? Does the filter allow it?`

If the logical answer and the filter behavior don't match, the filter is wrong — even if it "sounds reasonable" in the abstract.

**How to execute:**

- Pick real roles from the system (e.g., SystemAdmin, CountryManager, CompanyUser)
- Pick edge-case records: null scope fields, records belonging to a different scope, global records (all scope fields null)
- Walk each record through the filter predicate step by step
- If any scenario surprises you, the filter has a semantic bug

**Why this fails without the check:**

"Upward visibility" or "hierarchical scoping" sounds correct in abstract. It breaks when you simulate: CountryManager for Italy should NOT see records with `CountryId=null` if those records are company-scoped globals used across all countries. Abstract reasoning misses this. Concrete simulation catches it.

**Real failure pattern:**

Plan proposed "upward visibility" semantics for CountryManager: a CountryManager could see records with `CountryId=null`. Simulation showed this exposed all company-only records (with `CountryId=null`) to every CountryManager in the company — a cross-scope data leak. The abstract semantics sounded correct. Three concrete scenarios made the bug obvious.

**Evidence required to mark complete:**

List the 2–3 scenarios you simulated per rule, with the role, the record shape, the expected behavior, and the actual filter outcome.

---

### Check 3 — Verify every new technical pattern has a precedent in the codebase

**What to do:**

For every API, library, or framework feature the plan proposes using in a specific context, grep the codebase to see if anyone already does it. If nobody does: stop and ask whether it's because it's impractical in that context.

**How to execute:**

- Run targeted greps: `HasQueryFilter`, `RequireAssertion`, `IsEnabledAsync`, `IFeatureManager`, `async` in EF filter contexts
- Check if the proposed usage appears anywhere in the codebase
- If not found: treat it as a yellow flag and verify it's technically feasible before including it in the plan

**Typically mined areas:**

| Context | Risk |
|---------|------|
| EF Core `HasQueryFilter` | Must be translatable to SQL — no service calls, no async, no method invocations on injected services |
| ASP.NET `RequireAssertion` in policies | Synchronous context — async-hostile |
| Dependency injection in static or construction-time contexts | DI may not be available |
| Expression trees vs executable code | EF needs expression trees, not compiled delegates |
| `IFeatureManager` / feature flags inside query filters | API is async; EF filter context is sync; SQL translation is impossible |

**Why this fails without the check:**

The pattern looks correct in isolation. The API exists. The method has the right signature. But the CONTEXT forbids it. Nobody in the codebase does it because it doesn't work in that context — and you missed it because you didn't look.

**Real failure pattern:**

Plan proposed calling `_featureManager.IsEnabled("flag")` inside an EF Core `HasQueryFilter`. Nobody in the codebase did this. Reason: EF cannot translate it to SQL, and the API is async. The correct pattern (resolved with middleware + scoped service) was findable with a 30-second grep. The plan shipped the broken approach.

**Evidence required to mark complete:**

List the grep commands run, the results found (or not found), and for any pattern with no precedent, confirm it is technically feasible before including it.

---

## Red Flags — Thoughts That Mean STOP

| Thought | Reality |
|---------|---------|
| "I tipi sembrano giusti, non serve verificare" | Verifica comunque. I summary degli agenti perdono dettagli di tipo. |
| "La semantica è intuitiva, chi la userebbe male" | Simula 3 scenari. Se uno ti sorprende, il filtro è sbagliato. |
| "Questa API esiste, quindi posso usarla ovunque" | Verifica i pattern esistenti. Ogni contesto ha vincoli diversi. |
| "Il piano è già abbastanza lungo, non aggiungo verifica" | Lungo ≠ verificato. La lunghezza non sostituisce il rigore. |
| "Faccio i check dopo, intanto consegno" | No. I check sono prerequisito della consegna, non follow-up. |
| "Un agent ha già verificato questo" | Gli agent intermedi perdono precisione di tipo. Torna ai file sorgente. |
| "Ho letto il codice prima, mi ricordo i tipi" | Le entità cambiano. Rileggi i file nel momento della verifica. |

---

## Checklist Output

When this skill activates, create a visible checklist before presenting the final plan:

```
## Verified Planning — Pre-Finalization Checklist

- [ ] Check 1: Type-check — read source files for every type cited in pseudocode
      Files to read: [list them]
      
- [ ] Check 2: Scenario simulation — 2-3 concrete examples per filter/permission rule
      Rules to simulate: [list them]
      
- [ ] Check 3: Pattern precedent — grep codebase for every new technical pattern
      Patterns to verify: [list them]
```

Mark each item complete only after providing concrete evidence (file names read, scenarios listed, grep results shown).

**Do not present the plan as final until all three boxes are checked with evidence.**

---

## Principles

**Evidence over confidence**: "I believe the types are correct" is not evidence. A file path and the actual type found in that file is evidence.

**Scenarios over semantics**: "This role should have upward visibility" is semantics. `(role=CountryManager Italy, record with CountryId=null, CompanyId=ACME) → visible?` is a scenario. Only scenarios reveal bugs.

**Precedent over possibility**: "This API can do X" is not sufficient. "This API is used in this way elsewhere in the codebase" is precedent. No precedent = investigate before committing.

**Delivery blocker**: These checks are not recommendations. They are delivery gates. A plan without these checks completed is not a final plan — it is a draft.
