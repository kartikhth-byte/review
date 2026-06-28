# Spec Review Playbook

The playbook is the operational companion to `spec-review-with-ai-agents.md`. The article explains why structured spec review with AI agents works. The playbook is what you return to on Monday morning to actually run one.

**Scope.** This playbook is for software spec review on any stack. Engine-specific examples (Postgres, Supabase, SQL) appear throughout as illustrations of how to be concrete, not as items to copy. The worked example is fictional. The seven baseline specialists cover system-level concerns; domain projects extend the baseline rather than living within it.

---

## Design Principles

1. **General to software spec review.** Specialist scaffolding (focus / scope / out-of-scope / failure-mode list / severity rubric / output format) is engine- and domain-agnostic.
2. **Adaptable to project size.** Depth ladder L1-L5 mapped to concrete project sizes, with explicit recipes per tier.
3. **Adaptable to domain.** The seven baseline specialists are a system-level starting point. The playbook teaches how to write domain specialists (translation, trading, medical, real-time, etc.) when the baseline isn't enough.
4. **Engine specificity shown as worked example, not as prescription.** Engine-specific examples are tagged `[engine-specific example: X]` (Postgres, Supabase, SQL, etc.) so readers on other stacks see them as illustrations of how to be concrete, not as items to copy.
5. **Worked example is generic and fictional.** The locked example is "team admin invites a new member" because that workflow has enough auth, schema, audit, and cross-section surface area for the method to be visible.

---

## 1. Pre-Flight Check

Before running a review, freeze the spec at a candidate version and confirm the inputs the reviewer will need. A review against a half-edited spec or an incomplete codebase context produces noisy findings the reviewer can't ground. The list of inputs is short:

- Spec finalized to a candidate version (no half-edited sections).
- Codebase paths identified (which directories the specialist should ground in).
- Risk tier decided (informs depth ladder choice; see sections 2, 2a, and 2b).
- Output destination chosen (where findings get logged).
- Prior review findings if this is a re-review cycle (optional).

### Artifacts a full L4 or L5 cycle produces

A review isn't just a set of prompts. It's a process that produces named outputs. Without an explicit artifact list, teams run the prompts but lose the process. One L4 or L5 cycle should produce all of:

- **Stage 1 raw findings.** One file per specialist, listed by section reference, severity, and description.
- **Synthesis findings.** A file listing interaction defects across specialists, with the underlying specialist findings cited.
- **Stage 2 triage table.** Each Stage 1 finding marked Real Defect, Overreach, or Demoted-with-reason. The Real Defect column becomes the patch input.
- **Stage 3 buildability gaps.** Additional gaps the spec needs to address before implementation, beyond what Stage 2 surfaced.
- **Patch list.** Combined Stage 2 Real Defects plus Stage 3 buildability gaps, sorted by severity.
- **Re-review checklist for cycle 2.** Which specialists to re-run, which sections to re-check, what each patch was attempting to fix.
- **Failure-mode-list updates.** Any new failure modes surfaced during this review that should be added to the list, with the layer they belong to (engine, architecture, category, or project; see section 10).


For L1 to L3 reviews, collapse the artifact set. A bare-prompt or single-prompt review doesn't justify seven separate artifacts. Produce one combined findings doc: a list of issues with section references and severities, plus any failure-mode-list updates if you spotted a new pattern. The full artifact set kicks in at L4 because that's where the multi-stage process actually produces distinguishable outputs.

Treat each artifact as a deliverable. Without them, the process degrades into ad-hoc prompting that can't be re-run, audited, or improved.

## 2. Sizing Decision (Quick Heuristic)

Pick a depth tier by matching the review to the cost of getting the spec wrong. This is the back-of-envelope version. For anything non-trivial, run the scoring rubric in section 2a as a check.

- **Solo side project, low blast radius → L1-L2.** Bare or scope-only directed prompt. No specialists. No synthesis. 15 minutes.
- **Small team, internal tools → L2-L3.** Comprehensive directed prompt with named dimensions. Optional one-pass synthesis. About an hour.
- **Medium project, external users → L3-L4.** Three to four specialists chosen from the seven based on blast radius. Synthesis step required. Two to four hours.
- **Large project, regulated, or migration-touching → L4-L5.** All applicable specialists, multi-model if budget allows, mandatory synthesis, mandatory re-review after patches. Four to eight hours.

## 2a. Scoring Rubric for Depth-Tier Selection

The size heuristic in section 2 catches the easy cases. For anything in the middle, run this rubric and let the score pick the tier. The rubric trumps the size heuristic when they conflict, because risk and team size are independent: a small internal tool handling regulated data is a higher-stakes review than a large project doing cosmetic work.

Each spec is scored 1 to 3 on seven axes:
    - **Data sensitivity:** 1 (cosmetic only) / 2 (internal business data) / 3 (PII, financial, health, or regulated).
    - **Reversibility:** 1 (easily reversed) / 2 (recoverable with effort) / 3 (irreversible: data loss, public exposure, sent communications).
    - **User impact:** 1 (single user or single-team) / 2 (department or customer subset) / 3 (all users or external customers).
    - **Migration risk:** 1 (no schema change) / 2 (additive schema change) / 3 (data migration, backfill, or destructive change).
    - **Security boundary:** 1 (no auth/permission changes) / 2 (read-permission changes) / 3 (write-permission, identity, or session changes).
    - **Compliance exposure:** 1 (none) / 2 (internal policy) / 3 (external regulation: HIPAA, GDPR, SOX, PCI, maritime safety, etc.).
    - **Implementation ambiguity:** 1 (well-bounded, prior art exists) / 2 (some open questions) / 3 (lots of judgment calls, novel pattern).

    Total range: 7-21. Tier mapping:
    - **7-10 → L1-L2.** Bare or scope-only directed prompt. The risk is too low to justify specialist passes.
    - **11-14 → L3.** Comprehensive directed prompt with failure-mode list. One reviewer, structured.
    - **15-18 → L4.** Full three-stage cycle. Multiple specialists, synthesis, steel-man, implementation-engineer pass.
    - **19-21 → L5.** Multi-model three-stage cycle. Specialists spread across model families.

    The rubric trumps the size heuristic in cases of conflict. A "small team internal tool" handling regulated health data starts at 11 from data-sensitivity and compliance alone, and can easily reach 15+ if it also touches permissions, migration, or irreversible workflows. At that point it gets L4 review even though its size profile suggests L3. Conversely, a "large project" with cosmetic-only changes and no migration scores in the 7-10 range and gets L1-L2 review.

    **The rubric is a forcing function, not a risk calculator.** Don't treat 14 vs 15 as objectively different tiers; it's still judgment. The two rules that override the raw score:
    - If any single axis is existential (an axis score of 3 on Security Boundary, Reversibility, or Compliance Exposure where a defect would cause material harm), manually promote to the next tier.
    - A payment-auth change might score only midrange if scoped narrowly, but still deserves L4 because the security/compliance blast radius is high. Score is a starting point; judgment finishes the decision.

## 2b. What L3 and L4 Mean in Terms of Prompts

A common point of confusion: the rubric maps to a tier, but the tier description spans both "one prompt" and "many prompts" depending on how you read it. The clean distinction:

- **L3 is one comprehensive prompt** that uses the relevant specialist sections as named dimensions, with failure-mode lists embedded. One invocation, one combined output.
- **L4 is separate specialist prompts** run independently, in parallel, with synthesis afterward, then steel-man, then implementation engineer. Many invocations, structured outputs.

A reader whose spec scored 13 (clearly L3) should run one prompt with multiple specialist sections, not three or four specialist prompts. A reader whose spec scored 15 (clearly L4) should run the full three-stage cycle with separate invocations. The middle case (around 14) is a judgment call. Default to L4 if the spec touches more than two specialists' focus areas *and at least one touch is load-bearing*. A spec that touches schema, workflow, and metrics only in passing (single read-only reference each, no new constraints, no new derived calculations) is still L3 with one combined prompt. A spec that touches schema with a new FK, workflow with a new exception path, and metrics with a new formula is L4 even if its rubric score is 13 or 14.

## 3. Specialist Selection Decision Tree

For an L4 review, decide which of the seven baseline specialists apply. Don't run all seven by default — running specialists whose defect class can't actually hurt the spec is noise that buries the real findings. Walk the tree:

- Does your spec touch persistent data? → **Schema Realism.**
- Does it have status fields, lifecycle states, or workflow stages? → **State Machine.**
- Does it have user-facing workflows or use cases? → **Workflow / Use Case.**
- Does it have identity, sessions, or permission boundaries? → **Auth.**
- Does it have logging, compliance, or audit requirements? → **Audit.**
- Does it have dashboards, derived numbers, or reporting? → **Metrics.**
- Default to **Cross-Section** when two or more specialists apply. Skip Cross-Section only when the spec is short and all affected sections are local (no cross-referencing of fields, rules, or states between sections).

## 4. The Seven Baseline Specialist Templates

The full templates live in Appendix A. Before reading them, two things to understand.

**The failure-mode list inside each template is not a static prompt.** It's a maintained asset that accumulates value over time through the four-layer model (engine, architecture, category, project; see section 10). Treat each template's `COMMON FAILURE MODES` section as a starter, and assume you'll extend it as you accumulate review experience. Templates that don't get updated lose calibration. Templates that do get updated compound in value.

Each template provides three things:

- A generic skeleton with seven labeled sections: focus, scope, out-of-scope, context to load, failure modes, severity rubric, output format, plus a `BEFORE FINALIZING` self-check.
- A starter failure-mode list with items explicitly tagged `[universal]` or `[engine-specific example: Postgres/Supabase/SQL/etc.]`.
- A "calibrate to your stack" instruction prompting the reader to extend the failure-mode list with their own engine's quirks.

The seven baseline specialists are: Schema Realism, State Machine Correctness, Workflow / Use Case, Auth, Identity, and Session Boundary, Audit and Observability, Metrics and Derivation Correctness, and Cross-Section Consistency.

### Severity rubric: global scale, per-specialist anchors

Every template uses the same five severity levels (Blocker, Critical, High, Medium, Low) but the *meaning* of each level differs by focus area. A finding's severity is calibrated against its specialist's anchor, not a generic scale. The per-specialist Blocker definitions:

- **Schema Blocker:** migration cannot run (FK to nonexistent table, syntactically invalid constraint).
- **State Machine Blocker:** main workflow path cannot complete; required state has no entry or exit.
- **Workflow Blocker:** the user-facing main success scenario cannot complete as specified.
- **Auth Blocker:** unauthorized access is possible; the gate is missing or bypassable.
- **Audit Blocker:** a required compliance event cannot fire as specified.
- **Metrics Blocker:** the metric cannot be computed from the data available (references nonexistent fields).
- **Cross-Section Blocker:** two sections directly contradict each other in a way the implementer cannot resolve.

Each template in Appendix A carries this anchored definition in its own `SEVERITY RUBRIC` section.

## 5. The Three-Stage Cycle Structure

A single review cycle has three stages. The spec doesn't change between them. Patches happen only at the end of the cycle. Section 5 of `spec-review-with-ai-agents.md` explains the rationale; what follows is the operational form.

- **Stage 1: parallel specialist pass with stance variety.** All applicable specialists run together, each with its preferred stance for its focus area. Each one's output is a list of candidate findings within its own focus area.
- **Stage 2: single steel-man pass over Stage 1 findings plus the spec.** Triages real defects from adversarial overreach. Focus-agnostic.
- **Stage 3: single implementation engineer pass over the spec plus Stage 2 surviving findings** (treated as committed patches). Identifies buildability gaps that would still block implementation even after Stage 2's findings are addressed. Focus-agnostic.
- **End of cycle.** Patch all Stage 2 surviving plus Stage 3 buildability findings in one batch. No patching between stages.

### Stage 1 stance × focus matrix

Mixing stances at Stage 1 matters. Running every specialist adversarial produces uniformly grumpy output. The recommended pairings:

- Schema: adversarial (Stage 1 default)
- State Machine: adversarial
- Workflow / Use Case: adversarial; switch to devil's advocate for long-iterated specs
- Auth: red team
- Audit: adversarial; red team when compliance-sensitive
- Metrics: adversarial
- Cross-Section: demanding-structural

Steel-man and Implementation engineer are whole stages, not Stage 1 stances. They don't appear in the matrix.

## 6. The Synthesis Prompt

Synthesis runs once after Stage 1, before Stage 2. Its job is to surface compound defects that no single specialist can see: cases where two or more findings combine into a third defect that neither describes alone.

**When to run synthesis:**

- **At L4 and L5: always.** The full three-stage cycle includes synthesis by definition.
- **At L3: when findings span two or more dimensions, or any finding is High severity or above.** A single-prompt L3 review on a narrowly-scoped spec usually doesn't need synthesis. A single-prompt L3 review that surfaces findings across schema, auth, and metrics — or any Critical-or-Blocker finding — almost always needs synthesis to check for compound defects.
- **At L1 and L2: not applicable.** These tiers don't run multiple specialists in the first place.

### The synthesis contract

- **Input:** all findings from Stage 1 specialists (baseline plus any domain specialists).
- **Task:** identify interaction defects only. Cases where two or more findings compound.
- **Explicit non-task:** do not include new in-category findings in the synthesis output. If synthesis notices that a specialist appears to have missed something within that specialist's focus area, route it back to the relevant specialist (a narrow re-run) or log it for a follow-up. Don't fold it into the synthesis report, because synthesis is reporting compound defects, not individual ones.
- **Output:** ranked interaction defects with the underlying specialist findings cited.

The worked example in Appendix B includes a concrete synthesis output (B.5) where three findings — Schema, Auth, and Audit — combine into a "silent dual-identity" defect that no individual specialist could see.

## 7. Writing Your Own Domain Specialist

The baseline seven cover system-level concerns: schema, state, workflow, auth, audit, metrics, cross-section. Domain projects have additional defect classes the baseline doesn't know about. A translation tool needs a Linguistic Correctness specialist; a trading system needs Settlement and Reconciliation specialists; a medical system needs Clinical Logic and Patient Safety; a payments system needs Idempotency and PCI Compliance. The playbook is designed to be extended, not lived within.

### Guardrail: when to actually create one

Don't create a domain specialist for every dimension your project touches. The cost of an additional specialist isn't the prompt; it's the cognitive overhead of triaging another finding stream. Create a domain specialist only when the domain has failure modes that are *both* (a) high-cost if missed, AND (b) not already adequately covered by the baseline seven. Otherwise every project ends up with ten reviewers and the review becomes noise.

A good test: name three concrete failure modes your proposed specialist would catch that no baseline specialist would. If you can't name three, fold the concern into an existing specialist's failure-mode list instead.

### Examples of domain specialists

Each of the families below meets the three-failure-mode test:

- **Translation / NLP:** Linguistic Correctness, Data Governance / Residency, Model Behavior.
- **Trading:** Settlement, Reconciliation, Risk Boundary.
- **Medical:** Clinical Logic, Patient Safety, Regulatory Compliance.
- **Payments:** Idempotency, Reconciliation, Compliance (PCI, etc.).
- **Real-time / control:** Latency Budget, Backpressure, Failure-Mode Containment.
- **ML pipelines:** Training/Inference Drift, Data Lineage, Reproducibility.

### Meta-template for writing one

Once you've decided a domain specialist is warranted, the steps are:

1. Name the defect class your project loses money, users, or trust over.
2. Identify three to five concrete failure patterns in that class.
3. Translate those into a failure-mode list with examples.
4. Define what this specialist does NOT cover (the defer-to lines).
5. Decide the severity rubric: what makes a finding Blocker vs Critical vs High vs Medium in this domain.
6. Write the prompt using the scaffolding from section 4.

### 7.1 Worked example: a Linguistic Correctness specialist for a translation product

Suppose your project is a B2B translation product that converts customer-facing communications (emails, support tickets, marketing copy) between languages. The baseline seven specialists won't catch the bugs that matter most for this product: schema is correct, auth is correct, audit is correct, and the metric formulas check out. The thing that breaks the product is the translation itself producing output that is wrong in ways the source-language reviewer can't catch. Brand damage, user trust loss, occasionally legal exposure.

Walking the six steps from earlier in this section:

**Step 1. Name the defect class.** Linguistic Correctness: translation output that is undefined, contradictory, or silent on cases where source-language semantics don't map cleanly to the target language. High-cost (brand damage, regulatory exposure), and not covered by the baseline seven.

**Step 2. Concrete failure patterns.** Four that recur in the translation domain:

- **Register flip.** Source uses formal register; output is informal, or vice versa. Example: an English email translated to French without specifying whether `tu` or `vous` is appropriate. If the system defaults to `tu`, the output becomes instantly informal in a corporate context.
- **Named-entity corruption.** Proper nouns get translated when they shouldn't. "Apple Inc." becomes "Manzana Inc." Brand names, product names, place names, person names.
- **Gender-pronoun mismatch.** Source uses a gendered noun the target language treats differently. English "the doctor said she..." translated to Spanish defaults to masculine `el doctor` unless gender context is preserved end-to-end.
- **Idiom mistranslation.** Source idiom translates literally. "Hit the ground running" rendered word-for-word produces nonsense in most target languages.

**Step 3. Translate into a failure-mode list.** Each pattern above becomes one bullet in the prompt's `COMMON FAILURE MODES` section, with concrete examples (see the mini-template below).

**Step 4. Define what's out of scope.** Several concerns sit next to translation but belong elsewhere. Without explicit defer-to lines, the Linguistic Correctness reviewer drifts into them:

- Translation job persistence and API integration: defer to Schema (where translation jobs are stored) and Auth (who can request a translation).
- Translation latency: defer to a Latency Budget specialist if your project has one; otherwise to Metrics.
- Audit of which translations were performed by which user: defer to Audit.
- Cost per translation: defer to Metrics.

**Step 5. Severity rubric.** Anchored to the failure class, not a generic scale:

- **Blocker:** output is offensive, factually wrong (corrupted number, date, or name), or legally inaccurate (legal disclaimer altered in translation).
- **Critical:** register flip in formality-sensitive context; gender mismatch in personal address; brand-name corruption.
- **High:** idiom mistranslation producing awkward but understandable output.
- **Medium / Low:** stylistic inconsistency that a native speaker would notice.

**Step 6. The mini-template.**

```
You are a Linguistic Correctness reviewer for [TRANSLATION PRODUCT].
Your task is to review [SPEC PATH] and identify every place the
translation behavior is undefined, contradictory, or silent on cases
where source-language semantics don't map cleanly to the target.

SCOPE
- Register handling (formal vs informal, T-V distinction)
- Named-entity preservation rules (brands, products, places, people)
- Gender and pronoun handling when source and target differ
- Idiom translation policy (literal, equivalent, or flagged-for-review)
- Default behavior when source register or gender context is ambiguous

OUT OF SCOPE (flag and defer)
- Translation job persistence and API integration: defer to Schema/Auth
- Translation latency: defer to Latency Budget reviewer if present, else Metrics
- Audit of which translations were performed: defer to Audit
- Cost per translation: defer to Metrics

CONTEXT TO LOAD
1. The spec at [PATH]
2. Supported language list
3. Glossary or do-not-translate list (if defined)
4. Any tenant or customer-level register/style configuration

COMMON FAILURE MODES TO WATCH FOR
- Register handling unspecified; output defaults to one register without rationale or per-language override
- Named-entity preservation rules undefined; proper nouns at risk of translation
- Gender handling silent when source is less-gendered and target is more-gendered (or vice versa)
- Idiom policy undefined; literal-translation default leaks awkward output
- Default behavior when source register or gender is ambiguous unspecified
- [ADD YOUR PROJECT-SPECIFIC PATTERNS HERE — calibrate to your stack over time; place new patterns at the right layer per section 10]

SEVERITY RUBRIC
- Blocker: output offensive, factually wrong, or legally inaccurate
- Critical: register flip in formality-sensitive context; gender mismatch in personal address; brand-name corruption
- High: idiom mistranslation producing awkward output
- Medium / Low: stylistic inconsistency, minor word choice

OUTPUT FORMAT
For each finding:
- Section reference
- Severity
- Language pair affected (if specific)
- What is undefined or contradictory
- Suggested fix

BEFORE FINALIZING
- Each finding names a specific spec section
- Each identifies a language-pair scenario where the gap matters
- You haven't drifted into translation API integration or latency concerns
```

Notice what didn't happen in this walkthrough. We didn't ship a list of every linguistic concern that could exist. We named four high-cost recurring patterns, anchored severity to them, and stopped. As reviews accumulate, the failure-mode list extends through the same four-layer process used for any baseline specialist (see section 10).

## 8. Worked Example

The full walkthrough lives in Appendix B. It uses a fictional team-invitation spec because that workflow touches schema, state machine, workflow, auth, audit, metrics, and cross-section consistency all at once — enough surface for the method to be visible without inventing a contrived scenario. Appendix B covers: the spec under review, specialist selection, Stage 1 outputs from each applicable specialist, the synthesis pass, Stage 2 triage, Stage 3 buildability, the patch list, the cycle-2 re-review, and four bad-output samples paired with their good-output counterparts.

## 9. The Patch Loop and Cycle 2 Setup

Once the patch list is in hand, work it in severity order and run cycle 2. Cycle 2 is structurally different from cycle 1, and the difference is what makes the loop reliable.

### Triage rubric

- **Blocker:** fix now.
- **Critical:** fix before merge.
- **High:** fix, or defer with an explicit deferral note.
- **Medium / Low:** backlog.

### Cycle 2 setup

Cycle 2 has a narrower goal than cycle 1: confirm that patches closed the original defects, detect any new defects introduced by patches, and catch cross-section drift that patches frequently cause.

- **Focus areas:** re-run only the specialists that had findings prompting patches. Default to re-running Cross-Section as well; treat it as mandatory for non-trivial patches (those that touch more than one section, change any cross-referenced field, or revise a state machine). Skip Cross-Section only when the patch is a single localized fix that doesn't touch other sections.
- **Stance:** adversarial on the previously-patched areas (try to break the patches). Demanding-structural on Cross-Section.
- **Steel-man:** skip it for narrow patch verification — its job was done in cycle 1. Re-run it if cycle 2 produces new high-severity findings or if patches changed the design materially. In either case there's new adversarial output that needs triaging before edits.
- **Implementation engineer:** optional for non-trivial patches.

Cycle 2 sometimes surfaces things cycle 1 missed: not just patch bugs, but actual gaps that became visible only after the first round of fixes.

### Done condition

**Formal done condition:** a cycle returns no new Blocker or Critical findings.

**Judgment-call exception:** if a cycle returns three or more High findings clustered around the same defect class, consider one narrow re-check before stopping. The clustering signals an area of weakness the formal rule doesn't catch. Appendix B.9 shows this case in the worked example.

If you're still finding Blockers after cycle 2, the spec needs structural rework, not more patching.

The anti-pattern to avoid is chasing zero findings. The diminishing-returns curve is real, and a third or fourth cycle on a clean spec produces mostly noise. Stop when the formal rule is met.

## 10. Failure-Mode Lists and the Four-Layer Model

Of everything in this methodology, the failure-mode list is the only artifact that compounds across reviews. Specialists come and go. Stance choices change with the spec. Cycles vary by stakes. The failure-mode list gets more valuable with every review you run, but only if you maintain it deliberately.

### Four layers, most stable to most volatile

The list has structure. Treating it as a flat document loses the structure and makes it harder to extend over time.

- **Layer 1 (Engine).** Constraints of the database, runtime, or platform you build on, framed as spec-review failure modes. "Specs that declare polymorphic FK columns are wrong because Postgres can't enforce them." "Specs that rely on a plain `UNIQUE` constraint for soft-deleted records are wrong when the invariant only applies to active rows; use a partial unique index such as `WHERE deleted_at IS NULL`." "Specs that address Kubernetes pods by IP are wrong because pod IPs aren't stable." These change with the engine, not the project. Once in your list, they stay until you switch engines.
- **Layer 2 (Architecture).** Patterns of the system shape you've chosen. "Soft-delete patterns silently break uniqueness without partial indexes." "Event-sourced systems silently drop late-arriving events without an explicit reordering rule." "Edge-side API gateways mean rate limits must be enforced server-side, not client-side." Change when architecture changes.
- **Layer 3 (Category).** Recurring failure modes within a defect class, independent of project. "States with no documented exit." "Rate metrics with mismatched units." "Audit actor required but webhook events have no actor." These are what the baseline seven specialists hunt. Stable across projects of similar shape.
- **Layer 4 (Project).** Failure modes specific to your codebase. "The `profiles + user_roles` pattern instead of a single `recruiters` table." "The single-active-application trigger keys on `user_id`, so identity changes must preserve this binding." Facts about your project that future specs need to know.

When you start a new project, layers 1 through 3 transfer. Layer 4 rebuilds. Over time, layers 1 through 3 accumulate into a personal corpus more valuable than any single prompt.

### Operational guidance

- Keep the list in the repo, versioned alongside code.
- Tag the list before any formal review run (`v0.1`, `v0.2`, and so on); never let it drift mid-run.
- Treat the list as part of the codebase, not as a prompt-engineering scratchpad.
- When a review surfaces a new failure mode, decide its layer and add it to the right tier.
- Promote project-layer findings to category-layer when you see them across two or more projects.

This is the answer to "isn't this expensive to set up?" The setup compounds. The first L4 review for a new project takes real work because you're building the project layer from scratch. The fifth review on the same project is fast because most of the work is already in your list.

## 11. Customization Guidance

The playbook is designed to be customized. Over time, your version diverges from this one in three ways: the failure-mode list grows, specialists get added or retired, and prior-review findings get captured. The customization workflow:

- **Maintain the project-specific failure-mode list over time.** Section 10 covers the structure; the workflow around it is your own. Commit changes alongside spec changes. Reference the failure-mode file's tagged version in each review run so you can correlate findings to the list state that produced them.
- **Retire or upgrade specialists as the project matures.** A specialist whose failure-mode list hasn't changed in six months either is fully calibrated (rare) or has fallen out of relevance (common). Re-examine it.
- **Capture findings from past reviews into the playbook itself.** When a finding recurs across two or three reviews, it belongs in the failure-mode list, not the review report. Promote it.
- **Split a domain specialist out of the baseline when it grows too large.** A domain failure-mode list that exceeds eight to ten items inside an existing specialist is usually telling you to spin out a separate specialist. See section 7 for when to actually do that.

## 12. Operational Summary

The playbook has enough moving pieces that a compact summary is necessary for actual operational use. The rest of the playbook is reference material; the summary is the operational view, designed to be printed or pinned next to the workstation. The intent is a single page; whether it lands on one page in practice depends on how you render it, so don't treat one-page as a hard constraint.

The summary lives in Appendix C and covers six panels: the seven-axis scoring rubric with score-to-tier mapping, the specialist selection table, the three-stage cycle flow as a compact diagram, the per-specialist Blocker anchors, the artifact checklist, and the done condition.

---

## Appendix A: The seven baseline specialist templates

Each template uses the six-section structure described in section 4 plus a `BEFORE FINALIZING` self-check at the bottom. Failure modes are tagged `[universal]` (applies to most projects) or `[engine-specific example: X]` (illustrative of a concrete engine; replace with your stack's equivalent). The `[ADD YOUR PROJECT-SPECIFIC PATTERNS HERE]` line is the calibration point: extend it as you accumulate reviews and the four-layer model (section 10) grows.

### A.1 Schema Realism

```
You are a schema realism reviewer for [PROJECT] on [ENGINE: Postgres / MySQL /
DynamoDB / etc.]. Your task is to review [SPEC PATH] and identify every place
the proposed schema is non-implementable, internally inconsistent at the DB
level, or makes silent assumptions the engine cannot enforce.

SCOPE
- FK references and whether their targets exist in the current schema
- Constraint expressibility (UNIQUE, CHECK, partial indexes, exclusion)
- RLS / row-level policy expressibility given existing roles
- Migration sequenceability
- Soft-delete, normalization, and generated-column assumptions
- Case sensitivity and collation behavior on identifier columns

OUT OF SCOPE (flag and defer)
- State machine correctness: defer to State Machine reviewer
- Auth/identity coupling: defer to Auth reviewer
- Audit semantics: defer to Audit reviewer
- Metric formula correctness: defer to Metrics reviewer
- Section-vs-section contradictions: defer to Cross-Section reviewer

CONTEXT TO LOAD
1. The spec at [PATH]
2. Current migrations: [MIGRATIONS PATH]
3. Row-level policies: search for `create policy` or equivalent in the repo

COMMON FAILURE MODES TO WATCH FOR
- [universal] FK references to tables not in current migrations
- [universal] "Unique among non-X rows" expressed as plain UNIQUE rather than a partial unique index
- [universal] Soft-delete patterns silent on uniqueness and FK behavior
- [universal] Case-sensitive uniqueness on email or identifier without normalization
- [universal] Schema field referenced elsewhere but not declared in the field-list
- [universal] FK cascade behavior unspecified (relies on engine default, rarely intended)
- [engine-specific example: Postgres] Polymorphic FK (entity_id + entity_type) cannot be enforced
- [engine-specific example: Postgres] Array-of-FK columns cannot be enforced inside arrays
- [engine-specific example: Postgres] UNIQUE permits multiple NULLs by default; use NULLS NOT DISTINCT (Postgres 15+) when the invariant requires at most one NULL
- [ADD YOUR PROJECT-SPECIFIC PATTERNS HERE — calibrate to your stack over time; place new patterns at the right layer per section 10]

SEVERITY RUBRIC
- Blocker: schema cannot be created (migration will fail at CREATE TABLE)
- Critical: schema can be created but violates a load-bearing invariant
- High: enforces no integrity for a structural relationship
- Medium / Low: gaps, defaults, naming inconsistencies

OUTPUT FORMAT
For each finding:
- Section reference (e.g., §5.1)
- Severity
- What the spec says
- Why it fails on this engine
- Suggested fix

BEFORE FINALIZING
- Each finding has a section reference
- Each identifies a specific engine mechanism
- You haven't drifted into other categories
```

### A.2 State Machine Correctness

```
You are a state machine reviewer for [PROJECT]. Your task is to review
[SPEC PATH] and identify every place the proposed state model is incomplete,
contradictory, or silent on workflows that real users will need.

SCOPE
- All transitions enumerated (forward and backward)
- Every state has a documented exit, or is explicitly marked terminal
- Reopen, restore, and recover semantics distinguish their source states
- Concurrent-edit model specified (optimistic locking, version columns, LWW)
- Status field vs overlay flag (archived_at, deleted_at) precedence
- Transition triggers (who or what fires the transition)

OUT OF SCOPE (flag and defer)
- Schema and storage of the status field: defer to Schema reviewer
- Auth predicates on transitions: defer to Auth reviewer
- Audit events emitted on transitions: defer to Audit reviewer
- Metric formulas over state counts: defer to Metrics reviewer
- State-name consistency across sections: defer to Cross-Section reviewer

CONTEXT TO LOAD
1. The spec at [PATH]
2. The transition table or state diagram (if present)
3. Status enum values from the schema or spec
4. The terminal-states list (if present)

COMMON FAILURE MODES TO WATCH FOR
- [universal] States with no documented exit AND not classified as terminal
- [universal] Backward transitions missing (reopen, recall, retry, recover-from-error)
- [universal] Terminal state with a documented real-world recovery exit left undocumented
- [universal] Reopen rule generic across two structurally different source states (archive overlay vs terminal status)
- [universal] Multi-actor workflow without optimistic locking, version column, or stated LWW
- [universal] Status enum and overlay flag both writable, with unspecified precedence when they disagree
- [universal] State machine permits a transition that contradicts the spec's narrative description
- [universal] Derived state field claimed as both stored and computed in different sections
- [engine-specific example: Postgres] CHECK constraint on status enum doesn't match the spec's enumerated states
- [ADD YOUR PROJECT-SPECIFIC PATTERNS HERE — calibrate to your stack over time; place new patterns at the right layer per section 10]

SEVERITY RUBRIC
- Blocker: main workflow path cannot complete; a required state has no entry or exit
- Critical: workflow completes but a documented real-world case has no path
- High: state machine is internally consistent but missing concurrency control
- Medium / Low: naming, missing examples, undocumented but inferrable transitions

OUTPUT FORMAT
For each finding:
- Section reference
- Severity
- What state or transition is affected
- Why the spec is incomplete or contradictory there
- Suggested fix

BEFORE FINALIZING
- Each finding names a specific state or transition
- Each identifies whether the issue is missing transition, undefined precedence, or contradictory rule
- You haven't drifted into other categories
```

### A.3 Workflow / Use Case

```
You are a workflow reviewer for [PROJECT]. Your task is to review [SPEC PATH]
and identify every place a user-facing workflow is incomplete, missing
exception paths, or silent on real-world cases an implementer will hit.

SCOPE
- Main success scenario enumerated step by step
- Extensions per step (what happens when this step fails or branches)
- Exception paths documented separately from the happy path
- Stakeholders and their interests named
- Preconditions and postconditions stated
- Pre-existing-user / pre-existing-state / conflict cases handled
- Multi-actor coordination (locks, conflicts, ordering)

OUT OF SCOPE (flag and defer)
- Schema of entities involved: defer to Schema reviewer
- Auth predicates per step: defer to Auth reviewer
- Audit events fired per step: defer to Audit reviewer
- Metric capture during the workflow: defer to Metrics reviewer
- Cross-section consistency of step references: defer to Cross-Section reviewer

CONTEXT TO LOAD
1. The spec at [PATH], especially workflow and use-case sections
2. Stakeholder list or role definitions
3. Documented entry points and preconditions

COMMON FAILURE MODES TO WATCH FOR
- [universal] Happy path enumerated; exception paths missing entirely
- [universal] Step that can fail has no documented fallback or retry behavior
- [universal] Workflow assumes a precondition that is never enforced anywhere in the spec
- [universal] Multi-actor workflow without conflict, ordering, or coordination rules
- [universal] Workflow touches an existing-user or existing-state case without specifying behavior
- [universal] Stakeholder named in the feature body has no defined interest or interaction point
- [universal] Workflow promises a postcondition the spec doesn't show how to verify
- [universal] Bulk operation with no defined behavior for partial success
- [universal] Workflow crosses a network or system boundary with no idempotency or retry semantics
- [universal] Implicit assumption that a previous step has already happened, without checking
- [ADD YOUR PROJECT-SPECIFIC PATTERNS HERE — calibrate to your stack over time; place new patterns at the right layer per section 10]

SEVERITY RUBRIC
- Blocker: the user-facing main success scenario cannot complete as specified
- Critical: main scenario completes but a documented exception case has no defined path
- High: workflow is complete but a key precondition is not enforced anywhere
- Medium / Low: missing stakeholder details, step-level naming, minor sequence ambiguity

OUTPUT FORMAT
For each finding:
- Section reference
- Severity
- Which workflow, which step
- What case is unhandled or what precondition is unenforced
- Suggested fix

BEFORE FINALIZING
- Each finding names a specific workflow and step
- Each identifies a concrete unhandled case, not a vague "could be more thorough"
- You haven't drifted into other categories
```

### A.4 Auth, Identity, and Session Boundary

```
You are an auth and identity reviewer for [PROJECT]. Your task is to review
[SPEC PATH] adversarially, from a red-team perspective, and identify every
place identity, authorization, or session lifecycle is undefined, bypassable,
or inconsistent with the underlying auth provider.

SCOPE
- Identity model: where user records live and how app records bind to them
- Auth gate predicates: what state flags actually need to allow access
- Identity-change propagation: does an email rebind or role change reach the auth provider
- Session lifecycle: when sessions get revoked, when they don't
- Permission matrix: every feature in the spec body has a matrix row
- Role hierarchy and inheritance rules stated explicitly
- Bulk operation authorization (per-row re-check or single bulk permission)

OUT OF SCOPE (flag and defer)
- Schema and storage of identity columns: defer to Schema reviewer
- State machine transitions: defer to State Machine reviewer
- Audit events for auth changes: defer to Audit reviewer
- Cross-section role-name consistency: defer to Cross-Section reviewer

CONTEXT TO LOAD
1. The spec at [PATH]
2. Auth provider integration documentation (Supabase, Cognito, Auth0, etc.)
3. Permission matrix or role table
4. Session management and token-revocation configuration

COMMON FAILURE MODES TO WATCH FOR
- [universal] Auth gate predicate omits state flags that should also block access (archived_at, deleted_at, bounce_flag, claimed_at)
- [universal] Identity change updates app DB but not the auth provider
- [universal] Sessions not revoked on access-revoking state changes (archive, role change, deletion)
- [universal] Identity binding (e.g., `applicant_user_id`) not specified as unique; flag the auth risk (one auth identity could end up bound to multiple app records) and defer the DB mechanism to Schema reviewer
- [universal] Feature in spec body has no row in the role-permission matrix
- [universal] Role hierarchy stated in prose but inheritance rule unspecified
- [universal] Bulk operation has no specification on per-row authorization re-check
- [universal] Permission check expressed only client-side or only on the read path
- [universal] Endpoint response shape differs for collision vs non-collision (enables enumeration)
- [engine-specific example: Supabase] Email-edit RPC missing `supabase.auth.admin.updateUserById` call
- [universal] Archive workflow missing provider-supported session revocation for the affected user. For Supabase specifically, verify the current SDK mechanism (e.g., admin session deletion vs token-lifetime expiry); the exact API surface has changed between versions, so don't hardcode an SDK call without checking current docs.
- [ADD YOUR PROJECT-SPECIFIC PATTERNS HERE — calibrate to your stack over time; place new patterns at the right layer per section 10]

SEVERITY RUBRIC
- Blocker: unauthorized access is possible; the gate is missing or bypassable
- Critical: gate exists but a documented state can bypass it
- High: gate is correct but session/identity sync to auth provider is missing
- Medium / Low: role-name inconsistencies, missing audit hooks (defer audit findings)

OUTPUT FORMAT
For each finding:
- Section reference
- Severity
- What authorization or identity contract is at risk
- Concrete bypass or sync failure scenario
- Suggested fix

BEFORE FINALIZING
- Each finding describes a specific bypass or sync gap, not a vague "auth concern"
- Each cites the auth provider call or constraint that is missing
- You haven't drifted into other categories
```

### A.5 Audit and Observability

```
You are an audit and observability reviewer for [PROJECT]. Your task is to
review [SPEC PATH] and identify every place audit logging, compliance event
capture, or read-side observability is incomplete, bypassable, or
self-contradictory.

SCOPE
- Event-type enumeration (all meaningful events captured)
- Actor handling for system-fired events (webhooks, cron) and for unauthenticated / pre-auth applicant-initiated events
- Event-type overlap (one operation firing multiple types) with explicit exclusion rules
- Read-event auditability (server-side enforcement path)
- Retention and lifecycle policy
- Failed-action audit semantics (does a rejected attempt produce an event)
- Audit row structure (entity polymorphism, FK enforceability)

OUT OF SCOPE (flag and defer)
- Schema of audit_log table itself: defer to Schema reviewer
- Auth predicates that gate audited operations: defer to Auth reviewer
- State transitions that fire audit events: defer to State Machine reviewer
- Cross-section consistency of event-type names: defer to Cross-Section reviewer
- Dashboard metrics over audit data: defer to Metrics reviewer

CONTEXT TO LOAD
1. The spec at [PATH]
2. audit_log schema (if it exists in current migrations)
3. Event-type enumeration / list

COMMON FAILURE MODES TO WATCH FOR
- [universal] `actor_user_id NOT NULL` but event list includes events fired by webhooks, cron, external systems, or unauthenticated / pre-auth applicant actions (e.g., `bounce_received`, `magic_link_requested` before claim)
- [universal] Generic event (`field_edited`) and specific event (`email_changed`) both fire on same operation without an exclusion rule
- [universal] Audit event points to entities through polymorphic IDs (entity_id + entity_type), making audit-trail integrity unenforceable; flag the audit risk and defer the schema mechanism to Schema reviewer
- [universal] Sensitive reads declared "audited" without specifying server-side enforcement (SECURITY DEFINER RPC, edge function)
- [universal] List-view rendering exempted from audit (bypasses deep-read audit rules)
- [universal] Audit retention policy undefined
- [universal] Failed-action audit semantics undefined (rejected operations may or may not log)
- [universal] Audit row insert and the audited operation not specified as atomic
- [engine-specific example: Postgres] Audit row insert outside the same transaction as the operation
- [ADD YOUR PROJECT-SPECIFIC PATTERNS HERE — calibrate to your stack over time; place new patterns at the right layer per section 10]

SEVERITY RUBRIC
- Blocker: a required compliance event cannot fire as specified
- Critical: event fires but actor is null or wrong, breaking auditability
- High: events are correct but retention or lifecycle is undefined
- Medium / Low: missing fields, naming inconsistencies

OUTPUT FORMAT
For each finding:
- Section reference
- Severity
- Which event or audit path is at risk
- Concrete failure mode (bypass, double-count, missing actor, etc.)
- Suggested fix

BEFORE FINALIZING
- Each finding names a specific event type or audit path
- Each identifies whether the issue is missing capture, wrong actor, double-fire, or unenforced read path
- You haven't drifted into other categories
```

### A.6 Metrics and Derivation Correctness

```
You are a metrics and derivation reviewer for [PROJECT]. Your task is to
review [SPEC PATH] and identify every place a metric formula, KPI, or derived
calculation references nonexistent data, mixes units, or has undefined edge
cases under realistic inputs.

SCOPE
- Field existence: every referenced column is in the schema
- Numerator/denominator unit consistency in rate metrics
- Time-window semantics (rolling, calendar, snapshot)
- Timezone declaration in time-dependent metrics
- Cohort timing anchors (created_at vs first_action vs claimed_at)
- Funnel step-skip semantics
- "Today" / "now" semantics under historical date range pickers
- Combining additive (counts) and non-additive (averages, rates) metrics

OUT OF SCOPE (flag and defer)
- Schema field declarations themselves: defer to Schema reviewer
- Auth gates on metric viewing: defer to Auth reviewer
- Audit of metric views: defer to Audit reviewer
- Cross-section consistency of metric names: defer to Cross-Section reviewer
- State machine implications of metric values: defer to State Machine reviewer

CONTEXT TO LOAD
1. The spec at [PATH], especially dashboard, KPI, and reporting sections
2. Schema field list for every entity the metrics reference
3. Timezone configuration or assumption (server, tenant, user-local)

COMMON FAILURE MODES TO WATCH FOR
- [universal] Formula references a field that is not present in the schema field-list
- [universal] Rate metric with numerator counting one unit and denominator counting a different one (per-row vs per-distinct-entity)
- [universal] "Today" semantics undefined under a historical date-range picker
- [universal] Timezone unspecified in any time-dependent metric (day, today, month, age, SLA-day)
- [universal] Cohort definition timing ambiguous (multiple candidate timestamp anchors)
- [universal] Funnel step-skip behavior undefined (does a skipped stage count as passed)
- [universal] Aggregate that averages percentages without weighting
- [universal] Metric combines additive and non-additive components without clarifying the aggregation rule
- [engine-specific example: SQL] Windowed funnel or latest-status metric lacks a deterministic ORDER BY tie-breaker when timestamps collide
- [ADD YOUR PROJECT-SPECIFIC PATTERNS HERE — calibrate to your stack over time; place new patterns at the right layer per section 10]

SEVERITY RUBRIC
- Blocker: the metric cannot be computed from the data available (references nonexistent fields)
- Critical: metric can be computed but produces a materially wrong number under realistic input
- High: metric is correct under one interpretation but the spec is ambiguous between two valid ones
- Medium / Low: labeling, naming, axis precision

OUTPUT FORMAT
For each finding:
- Section reference
- Severity
- Which metric or formula is affected
- Why it produces wrong or ambiguous output
- Suggested fix

BEFORE FINALIZING
- Each finding names a specific metric or formula
- Each identifies whether the issue is missing field, unit mismatch, undefined edge case, or timezone gap
- You haven't drifted into other categories
```

### A.7 Cross-Section Consistency

```
You are a cross-section consistency reviewer for [PROJECT]. Your task is to
review the entire spec at [SPEC PATH] (not just individual sections) and
identify contradictions, name drift, and references-without-definitions that
become visible only when sections are read against each other.

SCOPE
- Universal rules in one section vs feature mechanics in another that requires the rule not to hold
- Field-name drift across sections (`archived_at` vs `archive_date`)
- Role-name drift (Master Recruiter vs Master Admin vs Master)
- Fields referenced in one section but never declared anywhere
- State machine state names that differ between the transition table and the workflow narrative
- Two enforcement mechanisms (RLS, trigger, application validator) with unspecified precedence
- Same-entity definitions that drift between sections

OUT OF SCOPE (flag and defer)
- Within-section schema correctness: defer to Schema reviewer
- Within-section auth, state, audit, metrics correctness: defer to respective specialist
- The canonical first instance of a name; only flag the variants that drift from it

CONTEXT TO LOAD
1. The entire spec at [PATH] (this specialist needs whole-spec context, not just sections)
2. Any glossary or definitions section
3. The canonical role table and field-list table

COMMON FAILURE MODES TO WATCH FOR
- [universal] Universal-quantifier rule ("all X", "no X anywhere", "never X", "every X") in section A contradicted by section B's feature mechanism
- [universal] Field defined as multi-value in one section, consumed as scalar in another, without explicit reduction rule
- [universal] Role-name drift: same role referenced by different names in different sections, no equivalence statement
- [universal] Field-name drift: same column referenced by different names in different sections
- [universal] Field referenced in a constraint, formula, or workflow but never declared in the schema field-list
- [universal] Two enforcement mechanisms (RLS + trigger, schema constraint + application validator) with unspecified precedence
- [universal] State machine state names differ between the transition table and the workflow narrative
- [universal] Stakeholder named differently in the role section and the workflow section
- [ADD YOUR PROJECT-SPECIFIC PATTERNS HERE — calibrate to your stack over time; place new patterns at the right layer per section 10]

SEVERITY RUBRIC
- Blocker: two sections directly contradict each other in a way the implementer cannot resolve
- Critical: contradiction exists but the implementer could pick one interpretation, creating long-term inconsistency
- High: drift or naming inconsistency that doesn't block implementation but will accumulate as the spec evolves
- Medium / Low: minor cosmetic variation that doesn't affect implementation

OUTPUT FORMAT
For each finding:
- Section references for both sides of the contradiction or drift
- Severity
- What each section says
- Why they conflict or drift
- Suggested reconciling fix

BEFORE FINALIZING
- Each finding cites at least two section references
- Each identifies whether the issue is contradiction, drift, undefined reference, or precedence gap
- You haven't surfaced findings that should belong to a within-section specialist
```

### A.8 Synthesis (cross-specialist interaction defects)

```
You are a synthesis reviewer for [PROJECT]. You will receive the
findings from N Stage 1 specialist reviewers. Your task is to identify
interaction defects: cases where two or more findings, each small in
isolation, compound into a third defect that neither specialist could
have seen alone.

INPUT
A concatenated list of Stage 1 findings, one block per specialist,
each finding tagged with: specialist, section reference, severity,
description.

YOUR TASK
For each interaction you find, report:
- Which two-or-more Stage 1 findings combine.
- What the compound defect is (the thing no individual specialist saw).
- Severity of the compound defect (calibrated to the most severe
  consequence, not to the individual findings).
- Suggested fix that addresses the compound defect, not the individual
  findings in isolation.

EXPLICIT NON-TASK
- Do not include new in-category findings in the synthesis output. If
  you notice that a specialist appears to have missed something within
  its own focus area, do NOT fold it into synthesis. Route it back to
  the relevant specialist or log it for a narrow re-run.
- Do not re-report individual specialist findings unchanged. Synthesis
  is for compound defects only.
- Do not propose architectural redesigns. Identify the defect; the
  patch loop handles fixes.

OUTPUT FORMAT
For each compound defect:
- Compound finding name
- Severity
- Stage 1 findings combined (cite specialist + section per finding)
- Why the combination produces a worse defect than the sum
- Suggested fix

BEFORE FINALIZING
- Every reported finding cites two or more Stage 1 findings
- No reported finding is a single-specialist finding restated
- You haven't drifted into proposing redesigns
```

---

## Appendix B: Worked example — Team admin invites a new member

This walks through one review cycle plus a cycle-2 re-review on a deliberately-flawed fictional spec. It shows what good specialist outputs look like, what synthesis surfaces, how steel-man triages, what implementation-engineer adds, and what cycle 2 catches. It closes with bad-output samples alongside their corresponding good-output counterparts.

### B.1 The spec section under review

```
§4.2 Team membership and invitations

A team admin can invite a new member by email. The invitation is created
immediately on submission and an email is sent to the recipient.

Behavior:
- Submitting the invite creates a row in `invitations` with status = 'sent'.
- The recipient clicks the link in the email, which marks
  `invitations.status` = 'accepted' and adds them as a team member.
- The invitation has a `recipient_user_id` FK to `users` if the email
  matches an existing user, or null otherwise.

Constraints:
- Email must be unique per team. Only one outstanding invitation per
  email per team is allowed.
- Once accepted, the invitation is terminal.

Permissions:
- Only team admins can send invitations.
- Bulk-invite is supported for admins: up to 50 invitations in one request.

Metrics (see §9.3):
- Invitation acceptance rate: ratio of accepted invitations to sent
  invitations.
- Time-to-accept: average days between sent and accepted.

Audit (see §7.1):
- Audit event `invitation_sent` is logged with the admin's user_id as actor.
- Audit event `invitation_accepted` is logged.

Cross-references:
- §3.1 says users have a unique email across the system.
- §10.4 says admins can deactivate any member they invited.
```

### B.2 Pre-flight

Standard pre-flight. Spec is the section above (assume the cross-referenced sections §3.1, §7.1, §9.3, §10.4 are also part of the larger spec and visible to reviewers). Codebase paths: `migrations/` for schema, `lib/auth/` for auth provider integration, `lib/audit/` for audit emitters.

Risk-tier scoring against the seven-axis rubric:

| Axis | Score | Reason |
|---|---:|---|
| Data sensitivity | 2 | Internal team-membership data, not PII |
| Reversibility | 2 | Recoverable with effort (invitation can be revoked / member deactivated) |
| User impact | 2 | Affects team admins and invited members, customer subset |
| Migration risk | 2 | New tables and indexes, additive schema change |
| Security boundary | 3 | New write permission (invite + auto-join on accept) |
| Compliance exposure | 2 | Audit completeness matters for internal compliance |
| Implementation ambiguity | 2 | Several open questions (existing-user case, bulk semantics) |
| **Total** | **15** | → L4 review |

### B.3 Specialist selection

Walking the decision tree:
- Persistent data? Yes (`invitations` table). → Schema Realism.
- Status fields, lifecycle states? Yes (`sent → accepted`). → State Machine.
- User-facing workflow? Yes (admin invites, recipient accepts). → Workflow / Use Case.
- Identity, sessions, permissions? Yes (only admins invite, recipient becomes member). → Auth.
- Logging or audit? Yes (two audit events declared). → Audit.
- Dashboards or derived numbers? Yes (acceptance rate, time-to-accept). → Metrics.
- Six of the seven apply; Cross-Section is mandatory by section 3's rule.

All seven baseline specialists run at Stage 1.

### B.4 Stage 1 outputs

Each specialist returns findings against the spec. Compressed but representative.

**Schema Realism (adversarial stance):**

- §4.2 / Blocker: `recipient_user_id` FK to `users` — the migrations directory has no `users` table; the canonical user table is `profiles`. Migration will fail at CREATE TABLE time. **Fix:** point FK to `profiles(id)`.
- §4.2 / Critical: "Email must be unique per team" expressed in prose without a schema mechanism. Likely intent is a UNIQUE constraint on `(team_id, email_normalized)`, but the spec doesn't say where this is enforced. **Fix:** add explicit UNIQUE constraint and specify normalized column.
- §4.2 / High: "Only one outstanding invitation per email per team" is conditional uniqueness (unique only when status != 'accepted'). A plain UNIQUE would also block re-invites after a previous one was accepted. **Fix:** partial unique index `WHERE status != 'accepted'`.

**State Machine (adversarial stance):**

- §4.2 / Critical: Only two states enumerated (`sent`, `accepted`). Real workflows include `expired`, `cancelled`, `declined`, `bounced`. None defined. **Fix:** enumerate complete state set and transitions.
- §4.2 / High: No backward transitions. What if admin wants to revoke a sent invitation before acceptance? Spec is silent. **Fix:** add `sent → cancelled` transition or explicitly note "cancellation not supported in v1."
- §4.2 vs §10.4 / Critical: "Once accepted, the invitation is terminal" contradicts §10.4 which permits deactivation. What is the invitation status when the member is deactivated? Is re-invite supported? **Fix:** specify the deactivation-and-reinvite path explicitly.

**Workflow / Use Case (adversarial stance):**

- §4.2 / Critical: Happy path only. What happens when email matches an existing user on the same team? Different team? An existing user who declined a previous invitation? Three undefined cases. **Fix:** enumerate behavior per pre-existing-user case.
- §4.2 / High: Bulk invite of up to 50 — no defined behavior for partial success (e.g., 47 succeed, 3 fail validation). **Fix:** specify whether the request is atomic, best-effort, or returns per-row status.
- §4.2 / Medium: Recipient clicks link in email — what if the link is expired or the recipient no longer has access to that email? **Fix:** define link lifetime and a resend-link workflow (or explicitly defer to v2).

**Auth, Identity, Session (red team stance):**

- §4.2 / Critical: The spec stores `recipient_user_id` on `invitations` and is silent on whether the invite endpoint returns it (or anything else that reveals existence-of-user). If the endpoint exposes that field, response shape will reveal whether the email maps to an existing user, enabling enumeration by any team admin. The response contract is undefined, so enumeration risk is unresolved. **Fix:** define the invite endpoint response contract explicitly, with uniform shape regardless of FK match, plus rate limiting.
- §4.2 / High: "Only team admins can send invitations" — bulk-invite path is not explicit about per-row authorization re-check. Could a team admin bulk-invite to a team they don't admin if the bulk endpoint accepts team IDs per row? **Fix:** specify per-row authorization for bulk invites.
- §4.2 vs §3.1 / High: Identity binding once accepted is not specified as unique. If the same email is invited and accepted on two teams via different paths, are two `profiles` rows created or is the same row used? Flag the auth risk; defer DB mechanism to Schema reviewer.
- §10.4 / High: "Admins can deactivate any member they invited" creates a permanent history dependency on the inviter. What about admins who later changed role or left the team? Spec doesn't say.

**Audit and Observability (adversarial stance):**

- §4.2 / Critical: `invitation_accepted` event has no actor specified. The recipient is becoming an authenticated user during this event; before they click the link they are unauthenticated. Is the actor the new `profiles.id`, the admin who sent the invite, or null? **Fix:** specify actor handling for `invitation_accepted` explicitly.
- §4.2 / High: Missing events. Workflow implies states the spec does not name (cancelled, expired, bounced, deactivated-then-reinvited) but only two events are defined. **Fix:** enumerate audit events for every state transition.
- §4.2 / Medium: Bulk-invite generates 50 individual `invitation_sent` events with no aggregate event for the bulk operation. **Fix:** decide whether a `bulk_invitation_submitted` event is emitted; if not, document the rationale.
- §4.2 / Medium: Retention policy for invitation-related audit events undefined.

**Metrics and Derivation (adversarial stance):**

- §9.3 / Critical: "Acceptance rate: ratio of accepted to sent" — the denominator is ambiguous on two dimensions. First, does "sent" mean all-time sent, sent in a window, or sent within the same cohort as the numerator? Different choices produce materially different numbers. Second, does the denominator include invitations that were cancelled, expired, or bounced? Spec doesn't say.
- §9.3 / Critical: Both "accepted" and "sent" grow over time as new invitations enter the pipeline. The rate is non-stationary if computed cohort-free. **Fix:** specify cohort-by-sent-date with an explicit window definition.
- §9.3 / High: "Time-to-accept: average days between sent and accepted" — over what window? Timezone unspecified for the "day" boundary. **Fix:** specify the time window and tenant or UTC.

**Cross-Section Consistency (demanding-structural stance):**

- §3.1 vs §4.2 / High: §3.1 says users have unique email across the system. §4.2 says invitation email is unique per team. If user X is invited to teams A and B, two `invitations` rows with the same email can exist. This is consistent with §3.1 only if a user can be invited to multiple teams (which §10.4 implies). The spec doesn't explicitly resolve this.
- §4.2 vs §10.4 / Critical: §4.2 says invitation status is terminal at `accepted`. §10.4 says admins can deactivate members. Deactivation-and-reinvite path is undefined. (State Machine flagged the same gap from its angle; this is the cross-section view of the same defect.)
- §4.2 vs §7.1 / Medium: §7.1 (audit) presumably enumerates standard event fields. The invitation events declared in §4.2 may not match §7.1's event schema. Verify.
- §4.2 internal / Medium: "Email must be unique per team" — what defines a duplicate? Case-sensitive? Whitespace-sensitive? Spec doesn't say.

### B.5 Synthesis pass

The synthesis prompt receives all 24 Stage 1 findings and is asked to identify interaction defects.

**Synthesis finding (Critical) — compound defect across three specialists:**

- Schema Realism flagged: `recipient_user_id` FK points to `users` but `users` doesn't exist; canonical is `profiles`.
- Auth flagged: undefined response contract may reveal existence-of-user if `recipient_user_id` or equivalent match state is exposed.
- Audit flagged: `invitation_accepted` event has no actor specified; the recipient becomes authenticated mid-event.

**Together:** The spec implicitly assumes a `users` table that doesn't exist, *and* the invitation flow has no clear identity-becomes-user transition. Result: every implementation will guess at (a) the canonical user table to bind to, (b) how the audit actor is captured at the moment of acceptance, and (c) whether the FK-vs-null response signal is intentional or accidental. Three different implementations will produce three different security postures and three different audit trails. **Fix:** add an explicit identity-becomes-user section specifying (1) the canonical user table, (2) how the FK is populated at invite-time vs accept-time, (3) the actor at each audit event in the flow, (4) the response-shape contract that doesn't expose user existence.

### B.6 Stage 2 steel-man triage

For each Stage 1 finding (plus the synthesis finding), the steel-man pass attempts to defend the spec. Output is a triaged combined list.

**Findings that survive (Real Defects):** 22 of 24, plus the synthesis finding (one of the 22 has its severity reduced; see demotions below). No plausible defense available for the schema gaps, state-machine incompleteness, workflow pre-existing-user case, auth enumeration, audit actor for accept, metrics cohort/window/timezone ambiguity, cross-section §3.1-vs-§4.2 and §4.2-vs-§10.4. These all stand.

**Findings demoted in severity (still in patch list):**

- State Machine *"no backward transitions"* — defended as: cancellation may be intentionally not supported in v1, with the spec implicitly treating sent as immutable. Real point but not a defect at the original severity; demote from High to Medium with an explicit "not supported in v1" note required.

**Findings removed from list:**

- Workflow *"expired link"* — defended as: link generation is an implementation detail handled by the link library, not a spec-level question. Demote to "out of scope: defer to link generation." Drops from the list.
- Cross-Section *"email duplicate definition"* (case-sensitive? whitespace-sensitive?) — defended as: the spec implicitly inherits the system-wide normalization rule from §3.1. The reviewer was right that the rule is implicit, but it's not a contradiction; convert to a documentation suggestion rather than treating as a defect. Drops from the list.

**Stage 2 surviving real-defect count:** 22 (with 1 severity-demoted) + 1 synthesis finding = 23 entries going forward.

### B.7 Stage 3 implementation engineer pass

Given the surviving Stage 2 findings will be patched, what's still missing to actually build?

**Buildability gaps beyond Stage 2:**

- **Rate-limit threshold for the invite endpoint.** Auth flagged enumeration risk; Stage 2 confirmed; but the spec doesn't specify the threshold (per minute? per hour? per admin? per team?). Cannot build without a number.
- **Email send infrastructure contract.** Spec says "an email is sent." Through what mechanism? Synchronously in the request, queued, retried on failure? What's the contract between the invite endpoint and the email service?
- **Link generation contract.** The link in the email needs a token. Token format, expiry, single-use vs reusable, signed vs random — none specified. Note that the workflow-level expired-link recovery path was demoted to "out of scope: defer to link generation" in B.6, but the link-generation contract itself remains a Stage 3 buildability gap because no implementer can build the email without these decisions.
- **Audit event payload structure.** Even after enumerating events, the field set per event isn't specified (just event type and actor). Cannot build the audit row inserts without field definitions.
- **Bulk-invite response shape.** Stage 2 confirmed partial-success behavior must be defined. Stage 3 adds: even with partial-success defined, the response shape (per-row status array? aggregate counts? both?) is undefined.

### B.8 The patch list

Combined: 23 Stage 2 entries (22 surviving findings + 1 synthesis) deduplicated to 22 unique items (State Machine and Cross-Section flagged the terminal-vs-deactivation defect from two angles; one patch addresses both), plus 5 Stage 3 buildability gaps = **27 items total**.

Sorted by severity (severities carried through from B.4, with the no-backward demotion from B.6 applied):

- **Blocker (1):** Schema `users` FK target (migration will fail at CREATE TABLE).
- **Critical (9):** Schema email uniqueness mechanism; State Machine only-two-states; terminal-vs-deactivation (consolidated State Machine + Cross-Section); Workflow pre-existing-user case; Auth enumeration; Audit actor for `invitation_accepted`; Metrics cohort/window ambiguity; Metrics non-stationary rate; Synthesis identity-binding compound defect.
- **High (13):** Schema partial-uniqueness index; Workflow bulk partial-success; Auth bulk per-row re-check; Auth identity-binding uniqueness (flagged, schema mechanism deferred); Auth deactivation history dependency; Audit missing events; Metrics timezone; Cross-Section §3.1 vs §4.2; plus the 5 Stage 3 buildability gaps (rate-limit threshold, email send infrastructure contract, link generation contract, audit event payload structure, bulk-invite response shape).
- **Medium (4):** State Machine no-backward (demoted from High, with required "not supported in v1" note); Audit bulk audit event; Audit retention; Cross-Section §4.2 vs §7.1.

Author works through the list, edits the spec, commits patches, and runs Cycle 2.

### B.9 Cycle 2 walkthrough

Cycle 2 setup per section 9: re-run only specialists whose findings prompted patches (in this case all of them, since all seven had findings), plus Cross-Section by default because patches touched multiple sections.

**What cycle 2 catches:**

The author's patch to address the State Machine and Cross-Section findings introduced new states (`cancelled`, `expired`, `declined`, `bounced`). The new state list is good, but the patch added a `sent → declined` transition without specifying who triggers it (the recipient? the system on bounce?). The patch also reworded §10.4 to handle deactivation-and-reinvite, but the new wording references an `invitation_supersedes` link that isn't defined anywhere.

Cycle 2 specialists return:

- **State Machine (adversarial on patches):** New `sent → declined` transition has no documented trigger. Who fires this and when? **Severity: High.**
- **Cross-Section (demanding-structural):** New `invitation_supersedes` reference in §10.4 has no definition anywhere in the spec. Cross-section drift introduced by the patch. **Severity: High.**
- **Audit (adversarial):** New `invitation_cancelled` and `invitation_declined` events added but actor handling not specified for either. Same shape as the original `invitation_accepted` actor issue; the patch fixed one event, missed the others. **Severity: High.**
- **Steel-man (rerun because cycle 2 produced three High findings):** All three findings stand; no plausible defense.

**Formal stop condition is met** (no new Blocker or Critical findings — see section 9). The author chooses a third cycle anyway because the three High findings cluster around the same defect class (actor-handling) and are cheap to re-check; one more round of patches plus a narrow re-review is a lower-cost insurance bet than shipping with a clustered known-weak area. This is a judgment call within the formal rule, not an override of it.

### B.10 Bad-output samples paired with good outputs

What follows are deliberately-bad reviewer outputs with their corresponding good-output versions. Calibrates readers on what to look for in their own runs.

**Bad output 1, vague finding with no section reference:**

> The schema needs work. There are some FK issues that could cause problems during migration.

This is the AI equivalent of a sticky note. No section reference, no specific FK, no specific table, no concrete failure mode. A patch loop can't act on it.

**Good output 1** (the Schema specialist's actual finding from B.4):

> §4.2 / Blocker: `recipient_user_id` FK to `users` — the migrations directory has no `users` table; the canonical user table is `profiles`. Migration will fail at CREATE TABLE time. **Fix:** point FK to `profiles(id)`.

Same defect, but actionable: a reader can search the spec, find §4.2, find the FK, apply the fix.

**Bad output 2, ungrounded claim with hallucinated constraint:**

> §4.2 / Schema realism: The UNIQUE constraint on `(team_id, email)` doesn't account for case sensitivity. Postgres will treat "Alice@x.com" and "alice@x.com" as distinct.

The defect class is real, but the reviewer hallucinated the constraint declaration: the spec says "email must be unique per team" but doesn't declare a UNIQUE constraint in any code. The reviewer wrote as if the constraint already existed and critiqued its details. A reader searching the spec for `UNIQUE (team_id, email)` finds nothing and concludes the reviewer is confused.

**Good output 2:**

> §4.2 / Critical: "Email must be unique per team" expressed in prose without a schema mechanism. Likely intent is a UNIQUE constraint on `(team_id, email_normalized)`, but the spec doesn't say where this is enforced. **Fix:** add explicit UNIQUE constraint and specify normalized column.

Same insight, properly grounded: flags that the constraint is implied, suggests the missing mechanism, names the normalized column.

**Bad output 3, cosmetic findings dominating while structural defects are missed:**

> §4.2: The section uses "team admin" in one place and "admin" elsewhere; should be consistent.
> §4.2: "Bulk-invite is supported" is passive voice; consider "Admins can bulk-invite."
> §4.2: The bullet list under "Behavior:" should use periods.

None of these is wrong, but they're irrelevant compared to: there's no `users` table, the auth path enables enumeration, the audit event has no actor, the metric denominator is non-stationary. A reviewer that spends its output budget on cosmetic findings missed the actual defects. This is what happens when the reviewer is asked to "review" with no failure-mode list to anchor to.

**Good output 3:** any of the structural findings from B.4. The fix at the prompt level: the failure-mode list directs the reviewer to specific defect classes that aren't style.

**Bad output 4, "helpful" rewriting instead of identifying the defect:**

> §4.2 needs to address the case where an email matches an existing user. Here's a rewritten Behavior section:
>
> - On submission, the system checks if the email exists in `profiles`.
> - If yes, the invitation is bound to the existing `profiles.id` immediately.
> - If no, a placeholder profile is created with `is_invited = true`.
> - The invitation row is created with `recipient_user_id = profiles.id`.
> - ...

The reviewer "helpfully" wrote an alternative spec. Now the reader has to adjudicate between the original spec and the reviewer's draft. The defect identification work (what's missing, why it matters, what shape the fix should take) is buried under the draft. The reviewer also embedded several design decisions (`is_invited` flag, placeholder profiles) that weren't asked for and aren't necessarily what the author intended.

**Good output 4** (the Workflow specialist from B.4):

> §4.2 / Critical: Happy path only. What happens when email matches an existing user on the same team? Different team? An existing user who declined a previous invitation? Three undefined cases. **Fix:** enumerate behavior per pre-existing-user case.

Same insight, no competing draft: tells the author what's missing, gives the shape of the fix, leaves the design call to them.

---

## Appendix C: Operational Summary

A compact reference designed to be printed and pinned next to the workstation. The body of the playbook is reference material. This is the operational view. The intent is single-page; whether it fits on one page depends on rendering and font, so adjust layout as needed.

### 1. Score the spec

Score each axis 1 to 3; sum for the total.

| Axis | 1 | 2 | 3 |
|---|---|---|---|
| Data sensitivity | cosmetic only | internal business data | PII, financial, health, regulated |
| Reversibility | easily reversed | recoverable with effort | irreversible |
| User impact | single user or team | dept or subset | all users / external |
| Migration risk | no schema change | additive schema change | data migration / destructive |
| Security boundary | no auth changes | read-permission changes | write / identity / session |
| Compliance exposure | none | internal policy | external regulation |
| Implementation ambiguity | well-bounded | some open questions | novel pattern |

| Score | Tier | Prompt form |
|---|---|---|
| 7-10 | L1-L2 | bare or scope-only directed prompt |
| 11-14 | L3 | one comprehensive prompt with failure-mode list |
| 15-18 | L4 | full three-stage cycle, separate specialist prompts |
| 19-21 | L5 | multi-model three-stage cycle |

**Override:** if any single axis scores 3 on Security Boundary, Reversibility, or Compliance Exposure and a defect would cause material harm, manually promote one tier. The rubric is a forcing function, not a calculator.

### 2. Pick specialists (L4 and above)

| Signal in the spec | Specialist |
|---|---|
| Persistent data, schema changes | Schema Realism |
| Status fields, lifecycle states | State Machine Correctness |
| User-facing workflows or use cases | Workflow / Use Case |
| Identity, sessions, permissions | Auth, Identity, and Session Boundary |
| Logging, audit, compliance events | Audit and Observability |
| Dashboards, derived numbers, KPIs | Metrics and Derivation |
| Two or more above apply | + Cross-Section Consistency |

### 3. Run the cycle

```
Cycle 1
  Stage 1: applicable specialists in parallel
           (Schema: adversarial; Auth: red team;
            Cross-Section: demanding-structural; ...)
       ↓
  Synthesis: one prompt across all Stage 1 findings
             (compound defects only)
       ↓
  Stage 2: one steel-man pass over (spec + Stage 1 + synthesis)
           triages real defects vs adversarial overreach
       ↓
  Stage 3: one implementation-engineer pass over
           (spec + Stage 2 surviving findings)
           identifies remaining buildability gaps
       ↓
  Patch all (Stage 2 surviving + Stage 3 buildability)
       ↓
Cycle 2 (re-review the patched spec)
  Narrower scope: re-run specialists whose findings
                  prompted patches
  Default: Cross-Section; mandatory for non-trivial
           patches; skip only for single localized fixes
  Skip: steel-man, unless cycle 2 produces new
        high-severity findings
```

### 4. Severity Blocker anchors

Use these per-specialist anchors when assigning Blocker severity. Each anchored definition appears in its template's `SEVERITY RUBRIC` section.

- **Schema:** migration cannot run.
- **State Machine:** main workflow path cannot complete; required state has no entry or exit.
- **Workflow:** user-facing main success scenario cannot complete as specified.
- **Auth:** unauthorized access is possible; the gate is missing or bypassable.
- **Audit:** required compliance event cannot fire as specified.
- **Metrics:** metric cannot be computed from the data available.
- **Cross-Section:** two sections directly contradict in a way the implementer cannot resolve.

### 5. Artifact checklist

For an L4 or L5 cycle, produce all of:

- [ ] Stage 1 raw findings (one file per specialist)
- [ ] Synthesis findings
- [ ] Stage 2 triage table (Real Defect / Overreach / Demoted)
- [ ] Stage 3 buildability gaps
- [ ] Patch list (sorted by severity)
- [ ] Re-review checklist for cycle 2
- [ ] Failure-mode-list updates (with layer: engine / architecture / category / project)

For L1 to L3, collapse to one combined findings doc with section references and severities, plus failure-mode-list updates if you spotted a new pattern.

### 6. Done condition

**Formal done condition:** no new Blocker or Critical findings on the latest cycle. Stop unless clustered High findings justify one narrow re-check (see Appendix B.9 for the worked case).

If still finding Blockers after cycle 2, the spec needs structural rework, not more patching.

Anti-pattern: chasing zero findings. A third or fourth cycle on a clean spec produces mostly noise.

