---
name: deep-review
description: >
  Deep functional audit — a SECOND PASS that complements standard code reviews.
  Catches what syntax checks and standard reviews miss: contract-vs-implementation mismatches,
  cross-layer data flow gaps, incomplete domain coverage, multi-tenant security violations (OWASP),
  concurrency/retry/idempotency bugs, amplified inherited tech debt, N+1 queries,
  DTO/Form/Entity misalignment, and runtime failures.
  Covers .NET/C#, Angular/React, and any backend+frontend stack.
  Also supports a Spec/Design Review sub-mode for reviewing design documents BEFORE implementation —
  the cheapest moment to catch contract errors and concurrency gaps.
  MUST be invoked as a complementary second check AFTER or ALONGSIDE any other code review skill or agent.
  Trigger whenever the user asks for "review", "code review", "deep review", "check my changes",
  "verify the implementation", "review di sicurezza", "controlla il codice", "performance review",
  "spec review", "design review", "review del design", "review documento", "contract review",
  "design adherence check", or after completing a significant implementation task.
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

### Spec/Design Review sub-mode

If the user asks to review a spec, design document, or feature brief BEFORE implementation (triggers: "spec review", "design review", "review del design", "review documento", "contract review"), switch the checklist focus:

- **Target artifact**: the document, not the code. No `git diff`, no build output, no runtime.
- **Mandatory checks**: CP0 Contract Adherence, CP2 Domain Completeness, CP4 Business Invariants, CP11 Security (reasoned against the described data flows), CP12 Runtime Simulation (concurrency/retry/idempotency reasoned from the design), CP15 Amplified Tech Debt, CP16 Framework Due Diligence.
- **Skip or adapt**: CP5 Build, CP8 i18n keys, CP14 Accessibility are N/A unless mockups are included. CP10 DTO alignment becomes "schema alignment — flag fields mentioned in prose but missing from the described DTOs/entities, and vice versa."
- **Output**: same Report-mode table, but the Details column cites the paragraph or section of the spec, not file line numbers.
- Always include Fix-mode as "propose spec edits" — do not touch code.

## Before You Start

1. **Identify the scope**: What changed? Use `git diff` or the conversation context to understand which files, entities, and layers were modified.
2. **Map the data flow**: For each change, trace the path: Entity → Repository → Service → Controller → DTO → Frontend Model → Form Component → UI Template. This map guides every checkpoint below.
3. **Identify all actors**: Who interacts with this code? (admin, end-user, API consumer, background job). Each actor's flow must be verified.
4. **Work on diffs, not whole files**: Focus your review on the changed code and its immediate context. Reviewing entire files leads to noise and missed issues in the actual changes.
5. **Find the framework equivalent**: If the change introduces a custom implementation that integrates with a framework (a custom Formly type, a custom Angular Material component, a custom EF Core interceptor, a custom ASP.NET middleware, etc.), find the closest built-in equivalent and compare them property by property. The gap between "what the built-in provides" and "what the custom provides" is where the invisible bugs hide. Read the framework's source or config file for the equivalent built-in, then ask: "What does the built-in declare that we don't?" This step is mandatory before Check 12 whenever a custom framework integration is involved.

---

## The Checklist

Execute ALL checks in order. Do not skip any, even if it seems like there are no issues. For each check, document what you verified and what you found.

CP0 runs first and gates everything else. CP1–CP13 are always mandatory. CP14 (Accessibility) is optional. CP15–CP16 are mandatory on features that change concurrency, retry paths, message flows, or that inherit known limitations from the existing code.

### 0. Contract Adherence Check

Run this BEFORE any other check. For every invariant, goal, or promise declared in the spec, feature brief, PR description, or user-facing copy, enumerate the failure modes of the implementation.

- List every "always", "sempre", "guaranteed", "never", "invariant", "will" statement in the source document, the commit message, and the UI copy (toasts, confirmation dialogs, help text, docs)
- For each, ask: "What concrete scenarios could produce a committed state that violates this?" Network failure, process crash between writes, message bus unavailable, external service timeout, partial rollback, retry, concurrent actor
- If any scenario is plausible, the invariant is NOT guaranteed. Two options: (a) strengthen the implementation to the level of the promise, or (b) weaken the promise in the contract AND in all user-facing copy
- Do NOT accept "we log on failure" as sufficient when the contract says "always". The log is the admission that the contract is aspirational, not contractual
- Do NOT accept "this is how the existing code does it" as justification. A pattern that satisfied an old best-effort contract may violate a new always contract — see CP16 and the Principles section.
- Common smell: a feature goal declares "the user is always notified" but the implementation publishes the notification non-blocking with log-on-failure. Flag it.

### 1. End-to-End User Flow

Mentally simulate every actor interacting with the modified code.

- For every configuration an admin can make, ask: "What does the end user see? Is the behavior correct?"
- Follow data from the entry point (UI form) through persistence (DB) and back (read/display)
- Don't stop at backend logic: if backend adds a value to an internal list (e.g., `effectiveMandatoryIds`), verify that same data reaches the frontend for rendering. An internal check-list is not the same as what the frontend renders.
- **Test inverse operations**: if a field can be added, test removal. If a value can be set, test reset to null/empty.
- **Ask "what if this fails?"**: for every external call, async operation, or state change, consider the failure path. Is there error handling? Does the UI show a meaningful message?

**Mandatory scenarios for every mutating endpoint** (simulate each explicitly, do not merge them into one "happy path" walkthrough):

- First-time success path
- Duplicate request (client retries after timeout, browser refresh on the POST confirmation page, double-click on submit)
- Two concurrent actors mutating the same resource
- Mid-operation failure of any external dependency (DB write OK, message bus down; external API OK, DB rollback; outbox flush fails after commit)
- Authorized actor on a resource outside their scope
- Unauthorized actor
- Pre-existing inconsistent state (previous failure left orphaned rows — does the new endpoint cope?)

### 2. Domain Completeness

- List ALL possible values for every domain involved (enums, groups, types, states)
- Verify every value is handled in ALL code paths that use it
- If there's a set/list/map, compare it against the source of truth (DB, backend enum, i18n keys)
- Watch for partial handling: if 5 out of 6 enum values are handled, that's a bug, not a feature
- **Import/batch processing**: for every field that maps to a UI dropdown or fixed-value enum, verify that the import service enforces the same domain — not just that parsing succeeds. Locate the authoritative value list (frontend controller, backend enum, DB lookup table) and confirm the import validation accepts exactly that set. A field accepted as a free integer or free string when the UI restricts it to specific values is a domain gap that silently persists invalid data.

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
- **For custom framework integrations** (custom Formly types, custom form controls, custom directives): apply the "framework equivalent" principle from step 5 of "Before You Start". Compare the custom registration/declaration against a built-in equivalent. Common gaps: missing `wrappers` in Formly custom types (causes label/validation to disappear silently), missing `[formlyAttributes]` (loses id/aria/disabled propagation), missing `disabled` binding (field stays editable when the control is disabled), missing `ControlValueAccessor` implementation (value/touched/dirty state not propagated).
- **For reactive frameworks (Angular signals, React hooks)**: verify that computed values depend on reactive sources. Reading `formControl.value` in an Angular `computed()` is NOT reactive — it needs a signal.
- **Error paths**: What happens when the API returns 400, 403, 404, 500? Does the UI handle it gracefully or crash silently?

**Concurrency, Retry & Idempotency** (mandatory on every mutating endpoint — do not skip):

- **Duplicate requests**: What happens if the client retries the same POST after a timeout, or the user refreshes the browser on a confirmation page, or double-clicks submit? Does the server create a duplicate row / duplicate message / duplicate side effect? Require an idempotency key, a natural-key dedup, or a uniqueness constraint on every mutating endpoint — document which one.
- **Parallel actors**: Two staff/users act on the same resource simultaneously. Does last-write-wins silently overwrite the first? Require optimistic concurrency (RowVersion, ETag, version column, `[ConcurrencyCheck]`) on any update that can race. Check that a 409 is surfaced to the UI, not swallowed.
- **Distributed consistency**: When an operation spans DB + message bus + external service, does a failure mid-way leave an inconsistent state? Prefer transactional outbox or equivalent (Wolverine, MassTransit, NServiceBus all ship one). Never fire-and-forget a message after a DB commit and assume both succeeded — that is an at-most-once guarantee on something the contract probably calls "always".
- **Retry semantics**: If the client retries, if the message bus redelivers, if an infrastructure layer replays — is the operation safe? Non-idempotent handlers under at-least-once delivery are a bug, not a trade-off.
- **Amplified exposure**: If the feature replaces a low-traffic internal call with a high-traffic HTTP endpoint, or adds a UI button that encourages retries, any race condition the old path tolerated will now fire routinely. Treat the new access pattern as a load generator. See CP15.

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

### 15. Amplified Tech Debt Check

For every known bug, limitation, or "out of scope" item the feature inherits from the existing code, ask: does the new feature amplify it?

- Does the new access pattern increase the frequency of the bug (more calls, more retries, more users touching it)?
- Does the new feature introduce concurrency where there was none (HTTP endpoint replacing internal call, multiple staff replacing a single background job, UI refresh replacing a scheduled sync)?
- Does the new retry path make a previously-rare race condition routine?
- Does a new actor (staff, API client, external system) reach the buggy code path on a schedule the original design never anticipated?
- If yes to any: the debt is no longer "inherited unchanged" — it is "inherited and amplified". Pull it into scope, or explicitly document that the PR ships an amplified risk and name the person accepting it
- Do NOT accept "pre-existing, out of scope" as a final answer without answering the amplification question in writing
- Common smell: a design brief says "duplicate-document bug is pre-existing tech debt" while the same brief adds an HTTP endpoint that is retry-prone and multi-actor. Flag it.

### 16. Framework Due Diligence

Whenever the design or the code accepts a compromise ("best-effort", "eventually consistent", "log and swallow", "fire and forget", "we'll handle retries later"), check whether the framework in use already provides a built-in primitive that eliminates the compromise.

- Search the framework docs / NuGet / npm for the capability BEFORE accepting the compromise. Cost of the check: two minutes. Cost of missing it: the whole compromise.
- .NET examples:
  - Wolverine: `PersistMessagesWithSqlServer() + UseEntityFrameworkCoreTransactions()` → transactional outbox for free
  - MassTransit / NServiceBus: built-in outbox and saga support
  - EF Core: `[ConcurrencyCheck]` / `RowVersion` → optimistic concurrency without a custom version column
  - Polly: retry with jitter, circuit breaker, hedging
  - ASP.NET Core: idempotency key middleware patterns, `ProblemDetails` for conflict responses
- Web / frontend examples: TanStack Query retry + dedup, Angular HttpInterceptor retry, browser `Request` with `Idempotency-Key` header
- If a primitive exists and the cost is negligible: use it. The compromise was an accidental decision, not an informed one
- Rule of thumb: any time the design says "we'll handle this in application code", confirm the framework doesn't already handle it. Flag every compromise that survived without this check.

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

**Contract adherence over pattern matching**: Replicating an existing pattern is not the same as adhering to the current contract. A log-on-failure publish may be fine for the old caller (whose contract said "best effort") and catastrophic for the new caller (whose contract says "always"). Before replicating a pattern, verify the new contract fits inside the pattern's guarantees. When it doesn't, either strengthen the implementation or weaken the contract — never ship the mismatch.

**Performance is a feature**: A query that works correctly but generates N+1 database calls will bring down production. Review queries with the same rigor as business logic.

**Defense in depth**: Security is never a single layer. Even if middleware validates scope, the service layer should also filter. Even if the frontend hides a button, the backend should still authorize. Redundant checks are not waste — they're resilience.

**Framework over diff**: Standard review reads the code written. Framework integration review reads the code written AND the code that should have been written (the built-in equivalent). The most dangerous gaps are invisible in a diff — a custom Formly type without `wrappers` looks correct, but silently loses label and validation rendering. A custom form control without `[disabled]` binding looks correct, but silently ignores the control's disabled state. Always ask: "What does the framework's built-in equivalent declare that we don't?" The answer is where the silent bugs live.
