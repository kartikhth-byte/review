# Spec-Defect Grammar v0.2

**Status:** Locked for the `nicky2tymze/estimated-to-measured` collaboration. Section 9 of the pre-registration co-signed 2026-06-26. This file is sent to the rig operator along with the predictions file. Not editable after receipt.

**Revision history:**
- **v0** (initial) — 30 entries extracted from review experiment findings.
- **v0.1** — 7 entries refined after pilot revealed too-vague injection rules. Added Appendix A "Rig requirements" enumerating baseline prerequisites and context dependencies.
- **v0.2** — Extended to 40 entries (added G-031 through G-040). Added a detection-criterion field to every entry per the pre-reg's Section 3 contract. Locked metadata header to the pre-reg's co-signed values. Predictions extracted to a separate sealed file (`tier_predictions_v1.md`); the per-entry "Expected catch" matrix remains here for reference and audit but is **not** the grading signal — only detection criteria are.
- **v0.2.1** (this version, pre-receipt refinement) — Disjunction sweep on detection criteria: every "OR" or "/" alternation rewritten as a single literal-match clause per the metadata's "one criterion per defect" rule (affected G-003, G-004, G-005, G-008, G-011, G-012, G-015, G-017, G-021, G-024, G-027, G-032, G-034, G-038, G-039, G-040). Standardized the "any one of X, Y, Z satisfies" phrasing across G-005, G-008, G-011, G-015, G-017, and G-034 so the scoring rule is identical in shape wherever the criterion accepts a partial match. Added worked plausibility-band examples for G-038 (role-name drift) and G-039 (field-name drift) in Appendix A, parallel to G-001's table. Added floor-entry notes to G-008 and G-034 acknowledging the "spec is silent on default behavior" framing, what their measured catch rates actually inform, and explicitly pairing them as the two operational-lifecycle floor entries. Added paired-surface design notes to G-022 and G-025 explaining that G-002/G-022 and G-007/G-025 are deliberate same-defect-different-surface pairs measuring reviewer context-sensitivity.

---

## Locked metadata (matches `PreRegistration_SpecDefect_Collaboration.md`)

- **Set size:** 40 injected spec defects across six grammar categories (~7 per category), inside the pre-reg's 30-50 range.
- **Review depth tiers:** `bare-prompt` / `scope-only directed` / `comprehensive directed (with failure-mode list)` / `specialist-passes + synthesis` / `multi-model`.
- **Designated weak reviewer (baseline):** Haiku 4.5 on the literal bare prompt `Review this spec.`
- **K (review generations per configuration):** at least 40.
- **Detection-criterion format:** declarative literal-match — `[review output must] {name / identify / flag} {specific object} {as missing / wrong / unenforceable / contradicting}`. One criterion per defect.
- **Predictions location:** `tier_predictions_v1.md` (separate file, sealed at receipt per Section 3 of the pre-reg). The "Expected catch" matrix below each entry mirrors that file for audit but is not the grading signal.

---

## Purpose

A structured catalog of spec-level defect types. Each entry carries:

- An **injection rule** that produces a known mutant when applied to a clean spec.
- A **detection criterion** stating what a review output must contain to count as having caught that defect (the grading signal, authored before any review is run).
- An **expected-catch matrix** predicting catch rates across the five review tiers (mirrored to the sealed predictions file; not used in grading).
- A **rig requirement** where the defect depends on specific baseline structure or external grounding.

---

## Catch-likelihood scale

For each defect, the expected-catch matrix uses five tiers:

- **L1** — `bare-prompt`
- **L2** — `scope-only directed`
- **L3** — `comprehensive directed (with failure-mode list)`
- **L4** — `specialist-passes + synthesis`
- **L5** — `multi-model`

Catch-likelihood ratings:

- **always** (~≥95% kill rate)
- **usually** (~70-90%)
- **sometimes** (~30-60%)
- **rarely** (~10-25%)
- **never** (<10%)

These ratings are predictions, sealed in the separate predictions file. Disagreement between predicted and measured catch is itself a result per the pre-reg's Section 8.

---

## Schema realism (8 entries)

### G-001 — FK to nonexistent table

**Description:** A foreign-key declaration points to a table that isn't created in any migration or referenced elsewhere in the spec/codebase.

**Injection:** Pick any FK reference in the clean spec. Replace the target table name with one not declared in the codebase's migration directory. The substitute should be plausible — neither obviously wrong nor an edit-distance-1 typo. Aim for: a singular/plural flip (`customers` → `customer_profiles`), a semantically adjacent domain term (`recruiters` instead of `profiles + user_roles`), or a near-synonym. Avoid both obvious gibberish (`xyz_table`) and trivial typos (`customeers`).

**Rig requirement:** The rig must have access to the codebase's migration directory (or an equivalent declared-table list) to verify the substitute name is genuinely absent. Without that grounding, this defect is structurally invisible — the mutant looks identical to a valid spec.

**Detection criterion:** review output must name the foreign-key reference and identify the target table as absent from the schema/migrations.

**Expected catch:** L1 sometimes / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-002 — Polymorphic FK (entity_id + entity_type)

**Description:** A single ID column intended to FK against different tables based on a sibling discriminator column. Postgres cannot enforce this natively.

**Injection:** In a logging or junction-style table, declare a single `entity_id UUID` column plus a sibling `entity_type` enum column. Annotate `entity_id` as "FK to the relevant entity" without specifying enforcement mechanism. Do not break it out into separate per-entity FK columns.

**Detection criterion:** review output must identify the `entity_id + entity_type` pattern as unenforceable by the database's foreign-key constraints.

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-003 — Array-of-FK column

**Description:** A multi-value relationship modeled as an array column intended to hold FK references. Postgres cannot enforce FK inside arrays.

**Injection:** In a many-to-many relationship, replace what should be a junction table with an array column on one side (e.g., `required_country_ids UUID[]`). Annotate as "array of FKs to countries." Do not introduce the corresponding junction table.

**Detection criterion:** review output must identify the array column as unenforceable for FK references at the database level.

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-004 — Partial uniqueness expressed as plain UNIQUE

**Description:** A conditional uniqueness rule (e.g., "unique among non-archived rows") is declared as a plain UNIQUE constraint that would also block re-use of values on archived/deleted rows.

**Injection:** State in prose that a column is "unique among non-archived prospects" or "unique while active." In the schema declaration, use `UNIQUE(column)` without a partial-index expression like `WHERE archived_at IS NULL`.

**Detection criterion:** review output must identify the `UNIQUE` constraint as wrong for the stated conditional-uniqueness rule and name a partial unique index as the required mechanism.

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-005 — Soft-delete silent on uniqueness/FK interaction

**Description:** A `deleted_at` flag is added without specifying how it interacts with the table's UNIQUE constraints or with FK references from other tables.

**Injection:** Add a soft-delete flag (`deleted_at timestamptz`) to a table that has a UNIQUE constraint on another column. Don't state whether soft-deleted rows release the unique value, or what happens to FK references targeting a soft-deleted row.

**Detection criterion:** review output must flag the soft-delete pattern as missing specification on at least one of `UNIQUE` interaction or FK interaction (catching either dimension is sufficient; flagging both is not required).

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-006 — Case-sensitive uniqueness on email/identifier

**Description:** An email or username field has a UNIQUE constraint without normalization (citext, lower(), or stored normalized column), allowing `alice@x.com` and `Alice@X.com` to coexist.

**Injection:** Declare an `email` or `username` column as `UNIQUE` without mentioning case-insensitive comparison. Don't include `citext`, a lower-cased generated column, or a functional unique index on `lower(email)`.

**Detection criterion:** review output must identify the email/identifier `UNIQUE` constraint as missing case-insensitive normalization (`citext`, `lower()`, or equivalent).

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-007 — Schema field referenced but not declared

**Description:** A column referenced in another section (constraint, query, or formula) isn't in the schema's field-list table.

**Injection:** Reference a column in one section (e.g., `deleted_at IS NULL` inside a uniqueness predicate or `first_contacted_at` inside a metric formula) without adding the column to the schema field-list table for that entity.

**Detection criterion:** review output must name the referenced column and identify it as absent from the schema field-list for the relevant entity.

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-008 — FK cascade behavior unspecified

**Description:** An FK is declared without explicit `ON DELETE` or `ON UPDATE` behavior, defaulting to `NO ACTION`, which is rarely the intended semantics.

**Injection:** Declare a foreign-key relationship in the schema without stating cascade behavior. Don't mention what happens to dependent rows when the parent is deleted.

**Detection criterion:** review output must identify the FK declaration as missing any explicit cascade-behavior specification. Naming any one of `ON DELETE`, `ON UPDATE`, or "cascade behavior" as the missing element satisfies the criterion.

**Floor-entry note:** This is a deliberately low-bar defect. Most clean specs are silent on cascade behavior; the line between "defect" and "default state of the spec genre" is thin. The expected-catch matrix concedes near-uncatchable rates at lower tiers, by design. Treat measured catch on G-008 as a calibration signal for whether the review tier engages with default-state operational specifications, not as a measure of structural defect-detection quality. G-008 pairs with G-034 (audit retention undefined) as the two operational-lifecycle floor entries in the grammar; their measured catch rates are most informative when read together.

**Expected catch:** L1 rarely / L2 rarely / L3 sometimes / L4 usually / L5 usually

---

## State machine correctness (7 entries)

### G-009 — State with no documented exit

**Description:** A non-terminal state has no outgoing transitions for realistic workflows (e.g., approved candidate who ghosts; user account that needs recovery from a hold).

**Injection:** Add a new state to the existing state machine that has at least one incoming transition and zero outgoing transitions, where the state's name implies an intermediate stage requiring recovery — `suspended`, `pending-review`, `on-hold`, `blocked`, `quarantined`. Do not list the new state in the spec's explicit terminal-states list. The defect is the absence of any defined exit combined with the absence of terminal-state classification.

**Rig requirement:** The clean spec must include both an enumerated transition table AND an explicit terminal-states list. Without an explicit terminal-states list, the absence of exits is ambiguous (could be intentional terminal-state declaration vs. defect), and the mutant is structurally indistinguishable from a valid spec.

**Detection criterion:** review output must name the state and identify it as missing an outgoing transition AND not classified as terminal.

**Expected catch:** L1 sometimes / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-010 — Backward transition not enumerated

**Description:** Real workflows require returning to an earlier state (reopen, recall, retry, recover-from-error) but the spec lists only forward transitions.

**Injection:** From the existing transition table, remove all backward transitions. A backward transition is one where the target state appears earlier in the forward happy-path sequence than the source state. Leave forward transitions and any terminal states intact.

**Rig requirement:** The clean spec must contain at least one backward transition in its baseline state machine. If the baseline already has only forward transitions, this category cannot be measured against that spec — pick a different baseline, or note that this defect is already present in the spec and skip the injection for this run.

**Detection criterion:** review output must flag the missing backward transition for the named workflow (reopen / recall / retry / recover-from-error).

**Expected catch:** L1 sometimes / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-011 — Concurrent-edit model unspecified

**Description:** Multiple actors can write to the same row but the spec doesn't define optimistic locking, version columns, or last-write-wins semantics.

**Injection:** Describe a multi-actor workflow (e.g., "any recruiter can update a prospect's status") without adding a `version` column, mentioning optimistic concurrency, or stating last-write-wins as the explicit policy.

**Detection criterion:** review output must identify the multi-actor workflow as missing concurrency-control specification. Naming any one of optimistic locking, version columns, or LWW as the missing mechanism satisfies the criterion; the criterion does not require the reviewer to enumerate all three.

**Expected catch:** L1 rarely / L2 sometimes / L3 sometimes / L4 usually / L5 usually

---

### G-012 — Reopen semantics conflate two distinct operations

**Description:** "Reopen" is described as one transition but the source state can be reached via two structurally different mechanisms (an archive overlay flag vs. a substantive status enum value), with no rule distinguishing which post-reopen behavior applies.

**Injection:** In a transition table where reopen applies to BOTH archived rows AND rejected (or other terminal-status) rows, replace the two distinct reopen rules (`archived → restore prior status` and `rejected → open`) with a single generic rule like "reopen → open" that doesn't distinguish the source. Remove any conditional language ("if source was archived...", "if source was rejected...").

**Rig requirement:** The clean spec must have a state machine that includes BOTH an archive flag pattern AND a substantive terminal status (e.g., rejected), with reopen transitions distinctly defined for each. If the baseline only handles one path, the conflation can't be cleanly injected — there's nothing to conflate.

**Detection criterion:** review output must identify the reopen transition as missing a rule that distinguishes the two source states (archived vs rejected).

**Expected catch:** L1 rarely / L2 sometimes / L3 sometimes / L4 usually / L5 usually

---

### G-013 — Derived field claimed as both stored and computed

**Description:** One section says a field is "auto-decremented" (implying a stored counter); another section says it's "derived from a count" (never stored). The spec contradicts itself on the same field.

**Injection:** In one section, define a field as "derived from `count(...)`, never stored." In another section, describe behavior as "X auto-decrements field Y on event Z." Don't reconcile.

**Detection criterion:** review output must identify the field as contradicting between its stored and computed definitions in the named sections.

**Expected catch:** L1 sometimes / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-014 — Terminal state has undocumented exit needed

**Description:** A state is declared terminal but real workflows require transitions out of it (joined candidate who fails to actually board; hired employee who quits in week one; cancelled subscription that needs reactivation within grace period).

**Injection:** From a transition table that has at least one documented exit from a terminal state (modeling a real recovery flow like `joined → withdrawn` for no-show or `hired → terminated` for early quit), remove that exit transition. Leave the state marked terminal in the explicit terminal-states list. The defect is the absence of the legitimate recovery case combined with the silence about whether it's intentionally unsupported.

**Rig requirement:** The clean spec must contain at least one terminal state with a documented recovery exit. Without that, this defect cannot be injected by removal — and a spec that genuinely treats all terminal states as truly terminal is not a defect, just a different design choice that this category doesn't apply to.

**Detection criterion:** review output must name the terminal state and identify the missing recovery exit for the implied real-world workflow.

**Expected catch:** L1 rarely / L2 sometimes / L3 sometimes / L4 usually / L5 usually

---

### G-031 — Parallel state fields with unspecified precedence

**Description:** An entity has two parallel lifecycle indicators (e.g., a `status` enum AND an `archived_at` timestamp, OR a `status` AND a `deleted_at` flag). The spec doesn't state which takes precedence when they disagree (e.g., `status = 'open'` but `archived_at IS NOT NULL`).

**Injection:** Define both a substantive `status` enum and an overlay flag (`archived_at`, `deleted_at`, or `suspended_at`). In separate sections, define behaviors that read from one or the other but never both. Don't state which wins when the two disagree, and don't add a constraint preventing the disagreement.

**Rig requirement:** The clean spec must establish that both fields are independently writable from different code paths. If only one path writes both, the precedence question is implicit but limited; the defect is sharper when independent writers can produce divergent states.

**Detection criterion:** review output must identify the two parallel state fields and flag the missing precedence rule when they disagree.

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

## Auth / identity / session (7 entries)

### G-015 — Auth gate predicate missing edge-case states

**Description:** A magic-link or login gate checks the primary substantive status but omits other state flags that should also block access (deleted_at, claimed_at, archived_at, bounce_flag).

**Injection:** Define an applicant sign-in gate predicate as `status = 'approved'` only. Elsewhere in the spec, introduce flags like `archived_at`, `deleted_at`, `bounce_flag`, or `claimed_at` that should also gate access. Don't include them in the predicate.

**Detection criterion:** review output must identify the auth-gate predicate as missing at least one of the relevant state flags. Naming any one of `archived_at`, `deleted_at`, `bounce_flag`, or `claimed_at` as the missing element satisfies the criterion; the reviewer is not required to enumerate all four.

**Expected catch:** L1 sometimes / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-016 — Post-claim identity change doesn't sync auth provider

**Description:** Editing a user's email after claim updates the app DB but doesn't sync the change to the auth provider (e.g., Supabase `auth.users`).

**Injection:** Define an "edit applicant email" RPC that updates `prospects.email` and `profiles.email`. Don't mention `supabase.auth.admin.updateUserById` or any equivalent call to sync the auth provider's user record.

**Detection criterion:** review output must identify the email-edit RPC as missing the auth-provider sync call (`supabase.auth.admin.updateUserById` or equivalent).

**Expected catch:** L1 sometimes / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-017 — Session not revoked on access-revoking state change

**Description:** When the user's prospect/account is archived, rejected, soft-deleted, or has its identity rebound, the spec doesn't include a session-revocation step. The old session remains valid for its normal lifetime.

**Injection:** Describe archive, soft-delete, or rebind workflows without mentioning `supabase.auth.admin.signOut`, token revocation, or session invalidation.

**Detection criterion:** review output must identify the access-revoking workflow (the specific archive, soft-delete, or rebind action injected) as missing session-revocation. Naming any one of the three workflow types injected satisfies the criterion; the reviewer is not required to enumerate all three.

**Expected catch:** L1 sometimes / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-018 — Identity coupling not enforced (no UNIQUE on binding)

**Description:** Two app-level records could be bound to the same external identity (e.g., two `prospects` rows could end up with the same `auth.users.id`) but no UNIQUE constraint prevents it.

**Injection:** Add an `applicant_user_id` column on a table that references `auth.users`. Don't include a UNIQUE constraint (or partial unique index on non-deleted rows).

**Detection criterion:** review output must identify the `applicant_user_id` column as missing a `UNIQUE` constraint (or partial unique index).

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-019 — Role permission matrix gap

**Description:** A feature mentioned in the spec body has no row in the access-control matrix. The required permission is implied but not stated.

**Injection:** Add a new feature in the body (e.g., "Master Recruiter can transfer requirement ownership"). Don't add a corresponding row in the role-permission matrix and don't enumerate the permission name in the seed list.

**Detection criterion:** review output must name the feature and identify it as missing from the role-permission matrix.

**Expected catch:** L1 sometimes / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-032 — Role hierarchy inheritance unspecified

**Description:** Roles are organized in an implied hierarchy (Master Recruiter > Recruiter > Coordinator) but the spec doesn't state whether higher roles inherit lower roles' permissions automatically. The matrix lists permissions per role explicitly, but an unstated inheritance assumption leads to surprises.

**Injection:** Define a role hierarchy in prose (e.g., "Master Recruiter is the most permissive role; Coordinator is the least"). In the permission matrix, list at least one permission for `Recruiter` that is NOT listed for `Master Recruiter`. Don't state explicitly whether the matrix is exhaustive (Master loses that permission) or whether hierarchy inheritance applies (Master gets everything Recruiter has).

**Rig requirement:** The clean spec must have at least three roles arranged in an implied hierarchy and a permission matrix that enumerates permissions per role.

**Detection criterion:** review output must identify the role hierarchy as missing an inheritance rule that resolves whether higher roles inherit lower roles' permissions.

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-033 — Bulk operation auth re-check unspecified

**Description:** A per-row operation has a documented permission check ("a recruiter can archive a prospect they own"). The spec also defines a bulk variant ("Master Recruiter can bulk-archive selected prospects") without stating whether the bulk path re-checks per-row permissions or bypasses them.

**Injection:** Define a per-row action with explicit permission semantics (e.g., "recruiter can only archive prospects they created"). Elsewhere, define a bulk variant that operates on a list of IDs ("bulk-archive accepts up to 100 prospect IDs"). Don't state whether the bulk variant re-checks per-row authorization or relies solely on the bulk-permission alone.

**Detection criterion:** review output must identify the bulk operation as missing specification on per-row authorization re-check.

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

## Audit / observability (7 entries)

### G-020 — Audit actor non-nullable but system events have no actor

**Description:** The audit-log schema requires `actor_user_id NOT NULL` but the event-type list includes events fired by webhooks, scheduled jobs, or applicant-initiated actions where there's no recruiter actor.

**Injection:** Declare `actor_user_id UUID NOT NULL` (or implicitly via FK with no nullable annotation). Then list event types including `webhook_received`, `bounce_received`, `claimed`, or `scheduled_cleanup`.

**Detection criterion:** review output must identify the audit `actor_user_id NOT NULL` constraint as contradicting the system-event (webhook / cron) event types.

**Expected catch:** L1 sometimes / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-021 — Audit event-type overlap (one operation fires multiple types)

**Description:** A single operation triggers multiple audit event types (e.g., `field_edited` and `email_changed` both fire on email edit) without an exclusion rule. Audit counts will double-count.

**Injection:** In the event-type list, include both a generic event (`field_edited`) and a specific event (`email_changed`) that both apply when the same field is changed. Don't include a rule like "specific event replaces generic event."

**Detection criterion:** review output must identify the two event types as overlapping (firing on the same operation without an explicit exclusion rule).

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-022 — Polymorphic audit_log.entity_id

**Description:** The audit log uses a single `entity_id` column that can reference any of several entity tables with no enforceable FK. Same shape as G-002 but in the audit log specifically.

**Paired-surface design note:** G-002 and G-022 are deliberately the same defect class applied at two surface locations (application schema vs audit-log schema). The pair tests *context-sensitivity*: does the reviewer catch a polymorphic-FK defect equally well when it appears in an application-domain table (G-002) versus when it appears in an audit-log table (G-022)? Per-tier catch-rate divergence between the two is a signal worth reporting; convergence confirms the reviewer's failure-mode list generalizes across surface locations.

**Injection:** In the audit_log schema, declare `entity_id UUID` and `entity_type enum`, annotated as "FK to the relevant entity." Don't break out separate per-entity nullable FK columns with a check constraint.

**Detection criterion:** review output must identify the audit_log `entity_id + entity_type` pattern as unenforceable by FK constraints.

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-023 — Read auditing client-side only (bypassable)

**Description:** The spec says certain reads are audited (e.g., "viewing prospect detail writes `detail_viewed`") but doesn't specify a server-side enforcement path. A direct DB query bypasses the audit.

**Injection:** Declare that a sensitive read fires an audit event. Don't specify that the read must go through a `SECURITY DEFINER` RPC or edge function that writes the audit row before returning data.

**Detection criterion:** review output must identify the audit event as missing server-side enforcement (`SECURITY DEFINER` RPC or edge function).

**Expected catch:** L1 rarely / L2 sometimes / L3 sometimes / L4 usually / L5 usually

---

### G-024 — List view exposes sensitive data without audit

**Description:** A list view includes sensitive content (notes excerpts, PII fields) but list-view rendering is exempt from audit requirements. Bulk reads slip through.

**Injection:** Add a "notes" or "last comment" column to a list-view definition. Elsewhere in the spec, state that "list-view rendering is not audited" as a carve-out from the deep-read auditing rules.

**Detection criterion:** review output must identify the list-view carve-out as bypassing deep-read auditing rules for the named sensitive content.

**Expected catch:** L1 rarely / L2 sometimes / L3 sometimes / L4 usually / L5 usually

---

### G-034 — Audit retention policy undefined

**Description:** The spec defines extensive audit logging but doesn't state how long audit rows live, what gets pruned, or whether retention varies by event type (security events vs operational events). Compliance and storage assumptions collide silently.

**Injection:** Specify an audit-logging design with detailed event types and required fields. Don't define a retention policy (no max-age, no archival rule, no per-event-type lifetime). Don't reference a compliance regime that would dictate retention.

**Detection criterion:** review output must identify the audit-logging design as missing a retention policy. Naming any one of max-age, archival rule, per-event-type lifetime, or "retention" as the missing element satisfies the criterion.

**Floor-entry note:** Same shape as G-008. Most clean specs are silent on retention policy; the defect is "absence of a positively-stated rule" rather than a positively-wrong rule. The expected-catch matrix concedes low rates at lower tiers, by design. Measured catch on G-034 is informative about whether the review tier engages with operational lifecycle concerns at all, more than about structural defect-detection quality.

**Expected catch:** L1 rarely / L2 sometimes / L3 sometimes / L4 usually / L5 usually

---

### G-035 — Failed-action audit semantics undefined

**Description:** The spec specifies that operations are audited but doesn't state whether failed/rejected attempts also produce audit rows. A recruiter who tries to archive a prospect they don't own — does that attempt get logged for security monitoring? Often unstated.

**Injection:** Define audit logging for successful actions ("on prospect.archive, write `prospect_archived` event"). Define authorization rules that can reject the action. Don't state whether rejection produces an audit row, what fields are populated for a failed attempt, or whether failed-action events are distinguishable from successes.

**Detection criterion:** review output must identify the audit-logging design as missing specification on failed/rejected-action audit semantics.

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

## Metrics / derivation correctness (6 entries)

### G-025 — Metric formula references nonexistent field

**Description:** A KPI formula uses a field name that isn't in the schema for that entity.

**Paired-surface design note:** G-007 and G-025 are deliberately the same defect class (a field referenced but never declared in the schema field-list) applied at two surface locations (a constraint or query in G-007, a metric formula in G-025). The pair tests *context-sensitivity* across the schema/metrics boundary: does the reviewer catch a missing-field reference equally well in a `WHERE deleted_at IS NULL` predicate as in a `COUNT(first_contacted_at)` metric expression? Per-tier catch-rate divergence between G-007 and G-025 is a calibration signal for whether the reviewer's grounding extends to derived/metric content or stops at the schema layer.

**Injection:** In the dashboard or KPI section, define a metric using a field name (e.g., `first_contacted_at`, `last_login_at`) that doesn't appear in the schema field-list for the relevant table.

**Detection criterion:** review output must name the metric-formula field and identify it as absent from the schema field-list for the relevant entity.

**Expected catch:** L1 sometimes / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-026 — Rate metric has numerator/denominator unit mismatch

**Description:** A rate metric's numerator and denominator count fundamentally different things. The rate can exceed 100% or be otherwise meaningless.

**Injection:** Define a conversion rate as `COUNT(joined_junctions) / COUNT(DISTINCT prospects_contacted)`. Numerator counts junction rows; denominator counts distinct people. One person joining multiple requirements pushes the rate above 100%.

**Detection criterion:** review output must identify the rate metric as wrong due to numerator/denominator counting different units.

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-027 — "Today" semantics undefined under historical date range

**Description:** A metric labeled "today" (e.g., "Contacted today") doesn't specify whether it means current calendar day or the end of a selected historical window.

**Injection:** Define a daily metric using `date = today()`. Elsewhere, introduce a date-range picker that allows historical windows. Don't reconcile what "today" means when the user selects a range ending three months ago.

**Detection criterion:** review output must identify the "today" metric as undefined when the date-range picker selects a non-current window.

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-028 — Timezone unspecified in time-dependent metric

**Description:** Metrics involving day, today, month, or age don't specify which timezone defines the boundaries (server UTC, tenant-configured, user-local).

**Injection:** Define daily counts, ageing calculations, or SLA-day metrics without specifying tenant timezone or a global timezone setting. Don't mention "Asia/Kolkata," "UTC," or any normalization rule.

**Detection criterion:** review output must identify the time-dependent metric as missing timezone specification.

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-036 — Cohort-definition timing ambiguous

**Description:** A cohort metric (e.g., "month-over-month conversion of approved prospects") doesn't state which timestamp defines cohort membership: `created_at`, `first_contacted_at`, `approved_at`, or `claimed_at`. Each anchor produces materially different cohort sizes and conversion curves.

**Injection:** Define a cohort metric in the dashboard ("monthly cohort of prospects who eventually joined"). Reference at least three different timestamp columns elsewhere in the spec (`created_at`, `approved_at`, `claimed_at`). Don't state which timestamp defines cohort membership.

**Rig requirement:** The clean spec must define at least two distinct lifecycle timestamps that could each plausibly anchor a cohort.

**Detection criterion:** review output must identify the cohort metric as missing specification on which timestamp anchors cohort membership.

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-037 — Funnel-step skip semantics undefined

**Description:** A funnel report counts prospects passing through stages (`open → contacted → interviewed → offered → joined`). The spec doesn't say how to count a prospect who skips a stage (joined directly from contacted without an explicit interviewed event). Do they count as having passed every intermediate stage, or only the stages with explicit events?

**Injection:** Define a funnel metric over the standard stages. State that stage progression is recorded by audit events (`contacted_at`, `interviewed_at`, etc.). Allow the spec's transition table to permit direct transitions (e.g., `contacted → joined` without `interviewed`). Don't state how the funnel handles step-skipping.

**Rig requirement:** The transition table must permit at least one direct transition that skips an intermediate funnel stage.

**Detection criterion:** review output must identify the funnel metric as missing specification on step-skipping semantics.

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

## Cross-section consistency (5 entries)

### G-029 — Section-vs-section rule/feature contradiction

**Description:** One section declares a universal rule with a quantifier ("all X", "no X anywhere", "never X", "every X"). Another section introduces a feature whose mechanism requires the rule to not hold. Neither section acknowledges the conflict.

**Injection:** Identify two existing sections in the clean spec: (a) section A containing a universal-quantifier statement (search for "all", "no ... anywhere", "never", "every"), and (b) section B containing a feature whose described mechanism requires breaking that universal claim. Remove any "except", "exception", "carve-out", or other reconciling language from section A. Leave section B unchanged.

**Rig requirement:** This defect requires the clean spec to ALREADY contain both sides of a potential contradiction with explicit reconciliation language between them. The injection removes the reconciliation. If the baseline doesn't have a pre-existing reconciled-tension pair, this defect can't be cleanly injected by modification — synthesizing both sides from scratch would produce an unrealistically forced mutant that wouldn't measure realistic catch.

**Detection criterion:** review output must name section A's universal rule and section B's feature and identify them as contradicting.

**Expected catch:** L1 rarely / L2 rarely / L3 sometimes / L4 usually / L5 always

---

### G-030 — Array → scalar prefill type mismatch across sections

**Description:** One section defines a multi-value (array or junction-table) field. Another section defines a workflow that prefills, copies, or derives a scalar from it, without specifying the reduction rule (0/1/many handling).

**Injection:** Identify two existing sections in the clean spec: (a) section A with a multi-value field definition (`UUID[]`, `text[]`, or a documented many-to-many junction), and (b) section B with a workflow that copies, derives, or prefills a scalar value sourced from that field. Remove any explicit reduction rule (e.g., "if exactly one, prefill; if multiple, show picker; if none, leave blank") from section B. Leave both sections referring to each other through the field name.

**Rig requirement:** The clean spec must contain both a multi-value field definition and a cross-section workflow that consumes it as a scalar. If only one side exists in the baseline, this defect can't be injected by removal — only by synthesizing the missing side, which produces an unrealistic mutant that wouldn't measure realistic catch.

**Detection criterion:** review output must identify the cross-section workflow as missing the multi-value-to-scalar reduction rule (0/1/many handling).

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-038 — Role-name drift across sections

**Description:** Different sections refer to the same role by different names. One section uses "Master Recruiter", another uses "Master Admin", another uses "Master." Whether they are the same role is ambiguous on a careful read.

**Injection:** Identify a role mentioned in at least three sections of the clean spec. In one section, rename it to a near-synonym ("Master Admin" instead of "Master Recruiter"). In a second section, shorten it ("Master"). Don't add a glossary or a "synonymous with" statement. Leave the original name in the canonical role-definition section.

**Rig requirement:** The clean spec must reference at least one named role in three or more sections, so the drift can be applied to a meaningful number of mentions.

**Detection criterion:** review output must name the role and identify the differing references across sections as the same role lacking an equivalence statement.

**Expected catch:** L1 rarely / L2 rarely / L3 sometimes / L4 usually / L5 always

---

### G-039 — Field-name drift across sections

**Description:** Different sections refer to the same field by different names. The schema declares `archived_at`, a downstream section refers to `archive_date`, a metric formula references `archived_on`. Whether they're the same column is unclear.

**Injection:** Identify a field defined in the schema field-list and referenced in at least two other sections. In one referencing section, rename it to a near-synonym (`archived_at` → `archive_date`). In another, use a different suffix (`archived_at` → `archive_timestamp`). Leave the schema-defined name in the schema section.

**Rig requirement:** The clean spec must reference at least one schema field in two or more non-schema sections, so the drift can be applied to a meaningful number of references.

**Detection criterion:** review output must name the field and identify the differing references across sections as the same field lacking an equivalence statement.

**Expected catch:** L1 rarely / L2 sometimes / L3 usually / L4 always / L5 always

---

### G-040 — Constraint-source precedence unspecified

**Description:** Two enforcement mechanisms (e.g., RLS policy + trigger validation, or schema check constraint + application-layer validator) both apply to the same write. The spec doesn't state which runs first, what happens if they disagree, or whether one is the source of truth.

**Injection:** Define two enforcement mechanisms for the same write path: (a) an RLS policy or schema constraint, AND (b) a trigger, edge function, or application-layer validator. Have at least one constraint expressed in both mechanisms (e.g., "only the owner can update" appears in both the RLS policy and the trigger). Don't state precedence, conflict-resolution rules, or which is the source of truth.

**Rig requirement:** The clean spec must show at least one write path with two enforcement mechanisms, and at least one overlapping constraint between them.

**Detection criterion:** review output must identify the two enforcement mechanisms as missing a precedence rule for the overlapping constraint.

**Expected catch:** L1 rarely / L2 rarely / L3 sometimes / L4 usually / L5 always

---

## Summary distribution

| Category | Entries | IDs |
|---|---|---|
| Schema realism | 8 | G-001 to G-008 |
| State machine | 7 | G-009 to G-014, G-031 |
| Auth/identity | 7 | G-015 to G-019, G-032, G-033 |
| Audit/observability | 7 | G-020 to G-024, G-034, G-035 |
| Metrics/derivation | 6 | G-025 to G-028, G-036, G-037 |
| Cross-section | 5 | G-029, G-030, G-038, G-039, G-040 |
| **Total** | **40** | |

Average: 6.67 entries per category, inside the pre-reg's "~7 per category" target.

---

## Notes on use

**Freezing protocol.** This v0.2 file is frozen at co-sign of the pre-reg (2026-06-26). Modifications after receipt by the rig operator invalidate the result. Any genuine defect found in an injection rule or detection criterion is recorded as a dated amendment per Section 6 of the pre-reg, not silently fixed.

**Injection isolation.** Each injection should be applied to a clean spec independently. Do not inject multiple defects into the same mutated spec unless explicitly measuring interaction catch (a separate experiment shape).

**Per-category catch as the primary signal.** The aggregate catch rate is less informative than the per-category breakdown. With ~7 entries per category, category-level catch rates have meaningful statistical headroom (still tight, but reportable). A weak category surfaces a specific calibration gap in the failure-mode lists.

**Detection criteria are the grading signal.** The per-entry "Expected catch" matrix mirrors the predictions in `tier_predictions_v1.md` for audit, but it is **not** the grading signal. A defect is scored caught for a configuration if and only if that configuration's review output satisfies the entry's **Detection criterion**. Predictions are the hypothesis under test, scored against measurement after the fact per Section 8 of the pre-reg.

**Predictions are sealed at receipt.** The rig operator does not see the predictions file while authoring detection criteria or applying them. The predictions never enter the score. Disagreement between predicted and measured catch is itself a result.

**Project-specific patterns excluded.** This grammar is the generic + Postgres/Supabase + architecture layer. Project-specific patterns (maritime compliance certifications, the local `profiles + user_roles` structure) are not in this grammar.

**Scope of generalization.** Best fit: any Postgres + Supabase web application with state-machine workflows, audit logging, and dashboard metrics. For other engines (MySQL, SQL Server) or other architectures (event-sourced, distributed), the engine-specific defects in schema realism will need substitution.

**Known limitations of v0.2.1 (acknowledged, deferred to v0.3 if needed).** Two things a careful reviewer of this grammar would point out, neither of which is addressed in v0.2.1:

- **Category imbalance.** Schema has 8 entries; Cross-Section has 5. The pre-reg promises "~7 per category" and the 8/7/7/7/6/5 distribution averages 6.67, which honors the spec. But Cross-Section is arguably the most tier-discriminating category (it most cleanly separates L3 from L4), so under-weighting it by three entries vs Schema is a missed opportunity. A v0.3 rebalance to 7/7/7/7/6/6 (moving one Schema entry to Cross-Section or restructuring) would tighten the per-category statistical headroom for the category most likely to show real tier differentiation.
- **Coverage gaps.** Six defect classes the grammar could plausibly cover but currently doesn't: index/performance assumption defects (e.g., a query assumes an index that isn't declared), migration safety (e.g., `NOT NULL` column added to populated table without default), PII / data-retention classification (which fields are PII, what's the retention class), backfill semantics (new column added with no backfill rule), auth rate-limit / abuse-control specification, and nullability-mismatch (column declared `NOT NULL` but a workflow inserts without setting it). Adding these would expand the grammar beyond 40 entries and require a new pre-reg co-sign; deferring to v0.3.

---

## Appendix A: Rig requirements

Different defects in this grammar have different context dependencies. The rig must be configured to provide the inputs each defect category needs.

### Codebase grounding

Required for defects that depend on the spec referencing real-world artifacts:

- **G-001** (FK to nonexistent table) — needs access to the migration directory or an equivalent declared-table list.
- **G-007** (schema field referenced but not declared) — needs the ability to verify a column name is absent from a schema definition while still being referenced elsewhere.
- **G-025** (metric formula references nonexistent field) — same shape as G-007 for metric formulas.

Without this grounding, the mutant is structurally indistinguishable from a valid spec and the defect cannot be detected.

### Full-spec context

Required for all cross-section defects:

- **G-029** (section-vs-section contradiction)
- **G-030** (array → scalar prefill mismatch)
- **G-038** (role-name drift)
- **G-039** (field-name drift)
- **G-040** (constraint-source precedence)

The rig must process the entire spec, not just isolated sections, because the defect lives in the relationship between sections.

### Baseline prerequisites

Some injection rules require specific structure to exist in the baseline before the defect can be cleanly introduced. If a baseline lacks the prerequisite, the rig should either pick a different baseline or skip that defect for the run.

| Defect | Baseline must contain |
|---|---|
| G-009 | An explicit terminal-states list (so absence-of-exits is unambiguously a defect, not a terminal declaration). |
| G-010 | At least one backward transition (so it can be removed). |
| G-012 | Both an archive overlay flag AND a substantive terminal status, with distinct reopen transitions for each. |
| G-014 | At least one documented exit from a state that is also marked terminal. |
| G-031 | Two parallel state fields independently writable from different code paths. |
| G-029 | Two sections containing pre-existing reconciliation language between a universal rule and a feature that breaks it. |
| G-030 | A multi-value field definition AND a cross-section workflow that consumes it as a scalar. |
| G-032 | At least three roles arranged in an implied hierarchy and a permission matrix that enumerates per-role permissions. |
| G-036 | At least two distinct lifecycle timestamps that could each plausibly anchor a cohort. |
| G-037 | A funnel transition table that permits at least one direct transition skipping an intermediate stage. |
| G-038 | A named role referenced in three or more sections. |
| G-039 | A schema field referenced in two or more non-schema sections. |
| G-040 | At least one write path with two enforcement mechanisms and one overlapping constraint. |

### Baseline diversity

A single clean baseline cannot exercise all 40 defects, because the baseline prerequisites differ. For full coverage, the rig should either:

**Option A: One comprehensive baseline.** Construct a single clean baseline that includes all baseline prerequisites listed above. Apply all 40 injections to this one baseline.

**Option B: Multiple targeted baselines.** Use category-aligned baselines:
- Schema-heavy baseline (exercises G-001 through G-008, G-022, G-025).
- State-machine-heavy baseline (exercises G-009 through G-014, G-031).
- Auth-heavy baseline (exercises G-015 through G-019, G-032, G-033).
- Audit-heavy baseline (exercises G-020 through G-024, G-034, G-035).
- Metrics-heavy baseline (exercises G-025 through G-028, G-036, G-037).
- Cross-section baseline (exercises G-029, G-030, G-038, G-039, G-040).

Option A is simpler operationally. Option B isolates per-category catch rates more cleanly but multiplies the number of measurement runs.

### Plausibility band

Several injection rules require the substituted content to be plausible — neither obviously wrong nor edit-distance-1 to the real value. The rig should aim for substitutions that:

- Differ from the real value enough to be wrong, but
- Wouldn't raise immediate suspicion on a quick read.

Concrete examples for G-001:

| Real value | Substitute | Verdict |
|---|---|---|
| `customers` | `customer_profiles` | ✓ Plausible — could be a real table in a different domain. |
| `customers` | `customeers` | ✗ Too obvious — looks like a typo. |
| `customers` | `xyz_table` | ✗ Too obvious — looks like a placeholder. |
| `recruiters` | `profiles` | ✓ Plausible — semantically adjacent and structurally different. |

The same plausibility band applies to G-038 (role-name drift) and G-039 (field-name drift). Concrete substitution examples for each:

**G-038 (role-name drift):**

| Canonical role | Drifted reference | Verdict |
|---|---|---|
| `Master Recruiter` | `Master Admin` | ✓ Plausible — same role re-described, near-synonym. |
| `Master Recruiter` | `Master` | ✓ Plausible — shortened form, ambiguous on first read. |
| `Master Recruiter` | `Super Admin` | ✗ Too different — sounds like a separate role. |
| `Master Recruiter` | `MR` | ✗ Too obvious — looks like an abbreviation gloss, not a real drift. |
| `Coordinator` | `Coord` | ✗ Too obvious — abbreviation. |
| `Coordinator` | `Recruiting Coordinator` | ✓ Plausible — qualifying-modifier expansion. |

**G-039 (field-name drift):**

| Canonical field | Drifted reference | Verdict |
|---|---|---|
| `archived_at` | `archive_date` | ✓ Plausible — same semantics, different naming convention. |
| `archived_at` | `archive_timestamp` | ✓ Plausible — synonym with explicit type-suffix. |
| `archived_at` | `archived` | ✗ Too different — could be a boolean rather than a timestamp. |
| `archived_at` | `archivedAt` | ✗ Too obvious — just a case-style change. |
| `first_contacted_at` | `contacted_at` | ✓ Plausible — drops the qualifier; ambiguous on whether it's the same field. |
| `first_contacted_at` | `last_contacted_at` | ✗ Too different — names a different semantic field, not a drift. |

In all three (G-001, G-038, G-039), the operator's job is to pick a substitute that a careful reader would notice but a quick reader might miss. Trivial typos and obvious placeholders fail the band because they signal "defect" too loudly. Substantively different names also fail because they signal "different thing" rather than "same thing drifted."

### What the rig does NOT need

For completeness, these dependencies are NOT required by the grammar:

- A specific framework or language ecosystem beyond Postgres/Supabase for the schema-realism defects.
- Production data (the defects are static spec-level, not runtime).
- A reviewer pool of humans (the rig measures AI review processes; human reviewers are out of scope for this grammar).
- Any artifact beyond the spec itself plus the codebase context noted above.
