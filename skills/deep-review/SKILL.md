---
name: deep-review
description: >
  Deep functional audit — a SECOND PASS that complements standard code reviews.
  Catches what syntax checks and standard reviews miss: cross-layer data flow gaps, incomplete domain coverage,
  multi-tenant security violations (OWASP), N+1 queries, DTO/Form/Entity misalignment, and runtime failures.
  Covers .NET/C#, Angular/React, and any backend+frontend stack with a 14-point checklist.
  MUST be invoked as a complementary second check AFTER or ALONGSIDE any other code review skill or agent.
  Trigger whenever the user asks for "review", "code review", "deep review", "check my changes",
  "verify the implementation", "review di sicurezza", "controlla il codice", "performance review",
  or after completing a significant implementation task.
  This is NOT a replacement for superpowers:requesting-code-review — it runs IN ADDITION to it.
  Also trigger proactively when a large cross-cutting change has been made (new field added across layers,
  new filter/scope introduced, refactoring touching multiple services, new query patterns introduced).
  DO NOT use for simple typo fixes or single-file cosmetic changes.
---

# Deep Review — Full-Stack Code Review Checklist

You are performing a deep code review. This is not a syntax check — it's a functional, security, performance, and consistency audit that catches the bugs compilers and linters miss.

## Mode Selection

Ask the user once at the start:

> **Modalita review:** Vuoi un **report** (tabella con findings per ogni punto) o preferisci che **trovi e corregga** direttamente i problemi?

If the user has already indicated a preference (e.g., "fai una review e correggi"), skip the question and proceed accordingly.

- **Report mode**: Produce a structured table with status per checkpoint. Do NOT edit files.
- **Fix mode**: Find issues AND apply fixes. Build after fixes to confirm zero errors. Report a summary of what was fixed at the end.

## Before You Start

1. **Identify the scope**: What changed? Use `git diff` or the conversation context to understand which files, entities, and layers were modified.
2. **Map the data flow**: For each change, trace the path: Entity → Repository → Service → Controller → DTO → Frontend Model → Form Component → UI Template. This map guides every checkpoint below.
3. **Identify all actors**: Who interacts with this code? (admin, end-user, API consumer, background job). Each actor's flow must be verified.
4. **Work on diffs, not whole files**: Focus your review on the changed code and its immediate context. Reviewing entire files leads to noise and missed issues in the actual changes.

---

## The 14-Point Checklist

Execute ALL checks in order. Do not skip any, even if it seems like there are no issues. For each check, document what you verified and what you found.

Points 1-12 are always mandatory. Point 13 (Performance) is mandatory. Point 14 (Accessibility) is optional — run it only if the user requests it or if the changes touch UI components.

### 1. End-to-End User Flow

Mentally simulate every actor interacting with the modified code.

- For every configuration an admin can make, ask: "What does the end user see? Is the behavior correct?"
- Follow data from the entry point (UI form) through persistence (DB) and back (read/display)
- Don't stop at backend logic: if backend adds a value to an internal list (e.g., `effectiveMandatoryIds`), verify that same data reaches the frontend for rendering. An internal check-list is not the same as what the frontend renders.
- **Test inverse operations**: if a field can be added, test removal. If a value can be set, test reset to null/empty.
- **Ask "what if this fails?"**: for every external call, async operation, or state change, consider the failure path. Is there error handling? Does the UI show a meaningful message?

### 2. Domain Completeness

- List ALL possible values for every domain involved (enums, groups, types, states)
- Verify every value is handled in ALL code paths that use it
- If there's a set/list/map, compare it against the source of truth (DB, backend enum, i18n keys)
- Watch for partial handling: if 5 out of 6 enum values are handled, that's a bug, not a feature

### 3. View Parity

- If the same information is rendered in multiple contexts (admin preview vs user form vs detail view), verify all use the same logic
- Compare property by property: label, description, placeholder, validation, select options, visibility
- Compare how similar grids/lists handle the same field (e.g., if vehicle-type-list uses `field="country.name"`, document-type-list must use the same pattern)
- Verify shared components (language-switcher, translatable-input) are connected with ALL required bindings

### 4. Business Invariants

- For every required field: does something prevent proceeding without it? At EVERY level (frontend form, backend validation, completion logic, state transition)?
- For every state transition: are all preconditions verified?
- For every saved value: does it return correctly when re-read? (pre-fill, admin display)
- For conditional logic (visibility conditions, mandatory conditions): does evaluation work correctly with edge-case inputs (null context, empty values)?

### 5. Build and Warnings

- Read the full build output, not just the last line
- Warnings are potential bugs — investigate and resolve any new warnings introduced by the changes
- If the project has both backend and frontend builds, run BOTH
- Zero errors is the minimum. Zero NEW warnings is the target.

### 6. Cross-Component Consistency

- If two components do the same thing (e.g., Work section and Address section), they must use IDENTICAL patterns
- If a pattern was improved in one component, verify the same improvement is applied everywhere
- Search the codebase for similar patterns: if you fixed a query in ServiceA, check if ServiceB has the same query unfixed

### 7. Serialization and Pre-fill

- For every value saved via API: is the serialization format (JSON casing, array vs string, date format) consistent between send and receive?
- For every form with pre-fill: do saved values return exactly as the form expects them?
- Edge cases: single value in a multi-select, null vs empty fields, date timezone issues

### 8. i18n and Labels

- Every i18n key referenced in code must exist in ALL language files
- If a field supports multilingual JSON (labels, descriptions), is language resolution applied EVERYWHERE the field is displayed?
- No references to obsolete concepts in text (e.g., "Step 1" when steps no longer exist)

### 9. DRY and Centralization

- If the same logic is repeated in multiple files, assess whether it can be extracted into a shared function/helper/component
- Check if the codebase already has utilities that do the same thing
- If two components build similar structures with similar logic, verify if they can share a builder function or if the differences are justified

### 10. DTO ↔ Form ↔ Entity Alignment

This is where the most insidious bugs hide — invisible to compilers, invisible to syntax reviews.

- For every field the frontend form can modify: verify it exists in BOTH the CREATE and UPDATE DTOs
- For every field made nullable/optional in the entity: verify the update DTO supports it (not just create)
- For every grid/list: verify column field names in the template match exactly the property names returned by the backend
- If backend returns entities via OData (IQueryable), grid columns must use entity property names with navigation (e.g., `country.name`), not flat DTO names
- For Mapperly/AutoMapper: verify `MapperIgnoreTarget` is correctly set — fields that shouldn't change on update must be ignored

### 11. Security and Data Isolation

This check is critical for multi-tenant systems or any system where data must be scoped. Follows OWASP Multi-Tenant Security guidelines.

**Query-level isolation:**
- For EVERY query on scoped entities, verify that ALL scope dimensions are filtered. If an entity has `CountryId`, `CityId`, AND `CompanyId`, all three must be checked where applicable.
- The pattern `(ScopeField == null || ScopeField == value)` means: null = global (visible to all), specific value = scoped.
- Search systematically: find all `Set<EntityName>()` or repository calls for each scoped entity and verify each one.
- Tenant context must be derived from authenticated tokens or server-side session, never from user-supplied input (OWASP). If a query uses a scope value, trace where that value comes from — it should originate from the DB or auth context, not from a request parameter that the user controls.

**Distinguish query types:**
- **Discovery queries** (listing available options): MUST filter by all scope dimensions
- **Validation queries** (checking an already-assigned record by ID): scope filter is usually NOT needed — the record was already validated at assignment time
- **Verification before classifying**: When you find a query without a scope filter, don't immediately flag it as a bug. Check if it's a lookup-by-ID for an already-assigned record. Ask yourself: "Could a user exploit this to access data from another scope?" If not, it's not a bug.

**API-level security:**
- Check if scope parameters in query strings are validated by middleware (e.g., TenantValidationMiddleware)
- Check if scope parameters in request bodies (POST/PUT DTOs) are validated — middleware often only checks query params, not body. This is a common gap.
- Verify that authorization policies (CanManageConfig, RequiresScope) are applied to all CRUD endpoints
- System admin bypass is expected — but non-admin users must not access cross-scope data

**CRUD operations:**
- Create: does the DTO accept scope fields? Is the user authorized to create in that scope?
- Update: are scope fields immutable (MapperIgnoreTarget) or intentionally editable? If editable, is the new value validated?
- Delete: is the delete scoped to the user's authorized scope?

**Cross-tenant contamination (OWASP):**
- Cache keys must include tenant identifier — a cache entry for tenant A must not be served to tenant B
- Background jobs and message queues must carry tenant context — a job enqueued by tenant A must not process data for tenant B
- File storage paths must be tenant-scoped — uploads from tenant A must not be accessible to tenant B
- Logging must not leak sensitive data from other tenants

### 12. Runtime Simulation

Don't just read the code structurally — simulate what happens at runtime.

- **Page load**: What happens when the component initializes? Are all dependencies resolved? Do async calls race?
- **Direct refresh (F5)**: Does the page work when accessed directly via URL, not just via navigation?
- **Selection change**: When a dropdown changes, do dependent fields update correctly?
- **Submit**: Does the form submit all expected data? Does the API receive what the form sends?
- **For frontend frameworks**: verify that every referenced component/type is registered (not just imported). A Formly type in a config that isn't registered will silently fail. A custom component used in a template that isn't declared will crash.
- **For reactive frameworks (Angular signals, React hooks)**: verify that computed values depend on reactive sources. Reading `formControl.value` in an Angular `computed()` is NOT reactive — it needs a signal.
- **Error paths**: What happens when the API returns 400, 403, 404, 500? Does the UI handle it gracefully or crash silently?

### 13. Performance

Performance issues are invisible until production load hits. Catch them in review.

**Database queries:**
- **N+1 queries**: Look for loops that execute queries inside them. Each iteration hits the DB separately instead of batching. In EF Core, check for missing `.Include()` on navigation properties accessed in loops.
- **Missing indexes**: New query patterns (especially new `WHERE` clauses on scope fields like CompanyId) may need database indexes. If a new filter is added to a frequently-called query, flag it for index consideration.
- **Unbounded queries**: Any `ToListAsync()` without a `.Take()` or pagination on a potentially large table is a risk. Verify that list endpoints have `MaxTop` or similar limits.

**Memory and resources:**
- **Resource leaks**: Database connections, HTTP clients, file streams — all must be properly disposed. Look for missing `using` statements or `IDisposable` implementations without `Dispose()`.
- **Unbounded caches**: Any in-memory cache or dictionary that grows without limit or TTL will eventually exhaust memory. Verify eviction policies exist.
- **Large object allocation**: Loading entire files or large collections into memory when streaming would suffice.

**Async patterns:**
- **Sync over async**: Calling `.Result` or `.Wait()` on async methods blocks threads and can cause deadlocks. All I/O operations should be properly `await`ed.
- **Missing cancellation tokens**: Long-running operations should accept and honor `CancellationToken` to allow graceful cancellation.
- **Fire and forget**: `async void` methods (except event handlers) lose exceptions silently. Async methods should return `Task`.

**Frontend performance:**
- **Unnecessary re-renders**: In Angular, check for function calls in templates (they re-execute every change detection cycle). Use signals or pipes instead.
- **Large bundle imports**: Importing an entire library when only one function is needed (e.g., importing all of lodash instead of `lodash/get`).
- **Missing trackBy**: `*ngFor` / `@for` without `track` causes DOM recreation on every change.

### 14. Accessibility (Optional)

Run this check when the changes touch UI components, especially forms, navigation, or interactive elements. Based on WCAG 2.2 Level AA.

- **Keyboard navigation**: Can every interactive element be reached and operated with keyboard only? Is there a visible focus indicator?
- **Color contrast**: Text meets 4.5:1 contrast ratio against background. UI elements meet 3:1.
- **Alt text**: Every informational image has descriptive alt text. Decorative images have `alt=""`.
- **Form labels**: Every form input has an associated `<label>` or `aria-label`. Error messages are announced to screen readers.
- **Semantic HTML**: Use `<button>` not `<div onclick>`. Use `<nav>`, `<main>`, `<aside>` for landmarks. Headings follow logical order (h1 → h2 → h3, no skips).
- **ARIA**: Only use ARIA when native HTML semantics are insufficient. Verify `aria-live` regions for dynamic content updates.

---

## How to Report Findings

### Report Mode

Produce a table:

| # | Check | Status | Details |
|---|-------|--------|---------|
| 1 | End-to-End User Flow | ✅ / ⚠️ / ❌ | What was verified, what was found |
| ... | ... | ... | ... |

Use:
- ✅ = Verified, no issues found
- ⚠️ = Minor issue or observation (not a bug, but worth noting)
- ❌ = Bug found (describe it clearly)

For each ❌, classify severity:
- **Critical**: Data leakage, security bypass, data loss
- **High**: Functional bug visible to end users
- **Medium**: Inconsistency, missing validation, performance issue
- **Low**: Code quality, DRY violation, minor UX issue

### Fix Mode

For each bug found:
1. Describe the issue in one line
2. Apply the fix
3. Move to the next check

After all checks, run the build to confirm zero errors. Then provide a summary:
- Number of bugs found and fixed (by severity)
- Number of observations (non-bugs)
- Confirm build status

---

## Principles

**Verify before classifying**: Don't flag a code difference as a bug just because two similar code paths do different things. First check: is the difference intentional? Does the downstream usage justify it? If unsure, ask the user rather than filing a false positive.

**Functional over syntactic**: The most dangerous bugs are functionally correct code that's incomplete — a filter that checks 2 of 3 scope dimensions, a form that saves 9 of 10 fields, a validation that covers 5 of 6 enum values. These pass every linter and compiler. Only systematic enumeration catches them.

**Runtime over structure**: A struct that looks correct in a diff can fail at runtime if a dependency isn't registered, a type isn't configured, or a reactive chain is broken. Always ask: "What happens when this actually runs?"

**Performance is a feature**: A query that works correctly but generates N+1 database calls will bring down production. Review queries with the same rigor as business logic.

**Defense in depth**: Security is never a single layer. Even if middleware validates scope, the service layer should also filter. Even if the frontend hides a button, the backend should still authorize. Redundant checks are not waste — they're resilience.
