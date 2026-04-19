---
name: cc-project-pipeline
description: Trigger only when the user's message explicitly contains the exact skill name `cc-project-pipeline` or directly says `使用 cc-project-pipeline`. Never auto-trigger from task size, project wording, analysis scope, planning, review, handoff, or delivery language alone. When explicitly invoked, run a staged project workflow with strict evidence, decision, safety, and delivery gates.
---

# CC Project Pipeline

Use this skill only when explicitly invoked by name. If the user did not explicitly say `cc-project-pipeline`, do not use this skill even if the task obviously looks like a big project.

## Core Workflow

```text
intake -> plan -> data validation -> execute -> review -> fix if needed -> final check -> pre-delivery gate
```

If the user asks only for planning or review, apply the same rigor but skip inapplicable stages.

Stages are not strictly one-way. `data validation` failures loop back to `plan`; discoveries during `execute` can trigger re-planning. See **Re-Planning**.

At `intake`, identify the project domain and activate corresponding skills via **Domain Routing**. Domain skills are then called as needed during `execute`.

## Non-Negotiables

- Never end silently. If blocked, output the blocker, up to 3 key questions, and the safest next step.
- Ask questions in plain text by default. Do not rely on interactive question tools; they may fail in CLI, VS Code, or CodePilot.
- Resolve ambiguity yourself first by reading provided materials, local project files, prior handoffs, available docs, formulas, and domain skills.
- Ask the user only after self-resolution fails and the missing answer materially changes the result, execution path, or delivery safety.
- When self-resolution fails on something that materially affects the result, stop and ask — do not guess, assume a default, or bury it as an untested assumption and continue. An incorrect silent assumption is worse than pausing to ask.
- For planning-only work, continue with an evidence-based conditional plan after listing unresolved blocking questions.
- If file creation is not allowed or tools are disabled, output all sections inline instead of creating handoff files.
- Apply rigor proportionally. Self-evident facts and obvious conclusions do not need full evidence chains or formal analysis — state them directly and move on. Reserve deep analysis for where it changes the outcome.
- If a task risks exceeding the session time limit, break it into segments and hand off cleanly between sessions. Never rush, skip steps, or lower quality to fit within one session. A correct partial delivery with a clear handoff is better than a complete but shallow one.

## Red Lines

- Do not delete or modify existing files. Overwrite, move, rename, truncate, in-place format, and metadata changes count as modification.
- Create new local output only when the user asks for it or the workflow explicitly allows new artifacts. Never overwrite an existing file.
- Do not write to databases, warehouses, dashboards, BI tools, production systems, or connected data sources. Read-only inspection and SELECT-only queries only.
- Do not send messages, emails, comments, mentions, tasks, or notifications to anyone. If a recipient is named, draft text locally for the user.
- Treat Feishu/Lark docs, sheets, wikis, tasks, and messages as read-only unless the user gives a one-action exception for a specific new document creation.
- Do not present a single recommendation when a key threshold, definition, or decision rule is still unresolved and materially changes the conclusion.
- Do not collapse multiple plausible standards or parallel result branches into one "final answer" unless evidence clearly supports that collapse.

## Audience Modeling (intake)

At intake, identify and record:

- **Decision maker**: who will act on this deliverable.
- **Decision**: what specific action they will take or not take based on the result.
- **Technical depth**: what level of detail they can absorb.
- **Known priors** (if available): what they already believe, so the deliverable can confirm or challenge with evidence.

Deliverables must be structured for the identified audience. For formal deliverables, if the decision maker or decision is unknown after self-resolution, flag as a blocking question. For exploratory tasks (e.g., "帮我看看 X 情况"), infer the audience on a best-effort basis and proceed — do not block on unknown decision maker.

## Assumption Registry (all stages)

Decisions are explicit choices between alternatives; assumptions are premises taken as given. Both must be tracked.

Each assumption entry:

- Statement.
- Type: `definitional` (e.g., "active user = opened app in last 7 days"), `data` (e.g., "events table has complete data since launch"), `domain` (e.g., "users inactive 7+ days rarely return"), `technical` (e.g., "CLI can handle this query volume").
- Validation status: `validated`, `plausible`, `untested`, `invalidated`.
- Validation method: how it was or can be checked.
- Impact if wrong: what conclusions would change.

Rules:
- Validate `untested` assumptions before they affect high-impact conclusions.
- Unvalidated assumptions that would change the core recommendation must be flagged in the delivery gate.
- Update the registry throughout execution as assumptions are confirmed, disproven, or newly discovered.

## Data Validation Gate (between plan and execute)

Mandatory input health check before execution. Do not proceed until critical checks pass.

Check categories:

1. **Coverage**: inputs cover the planned scope. No unexpected gaps.
2. **Structure**: inputs have expected schema, format, types, and fields.
3. **Scale sanity**: key quantities are within expected ranges (e.g., DAU vs. historical average, revenue sign and magnitude, row counts vs. prior periods, API response volume). Set the acceptable deviation threshold and explain why it's reasonable. Flag values outside it.
4. **Maturity/Completeness**: if the task depends on a time window or accumulation period, verify sufficient data has accrued.
5. **Integrity**: cross-references between sources are consistent (e.g., totals match across tables, join keys align, aggregated and detail-level numbers reconcile). Sample joins show no unexpected row multiplication or loss.
6. **Known issues**: check for project-specific gotchas recorded in memory or handoff docs.

Criticality:
- **Critical** (failure → loop back to plan): Coverage, Structure, Maturity. These mean the analysis would be built on wrong or missing inputs.
- **Non-critical** (log as caveat, proceed): Scale sanity, Integrity, Known issues — when the deviation is minor and documented.
- Criticality is context-dependent. A check that is normally non-critical can become critical if the deviation is large enough to invalidate the analysis. When making this judgment, explain the reasoning.

All thresholds and pass/fail boundaries must be explainable: backed by data, logical reasoning, or both. The justification form is flexible — historical baselines, structural properties, domain context, user-provided criteria are all valid as references, but none is mandatory. Never use an arbitrary cutoff without stating why it was chosen.

Output: `data_validation_report` with pass/fail per check, the threshold used and its justification, and gate decision (`proceed` / `loop back to plan` / `proceed with caveats`).

## Automated Assertions (execute)

Embed programmatic checks during execution rather than relying solely on review-stage inspection.

Standard assertion types:

- **Completeness**: mutually exclusive categories sum to the expected total (e.g., user segments covering total DAU, budget line items summing to total budget, category proportions summing to 100%). Define the acceptable rounding tolerance and explain why.
- **Conservation**: counts before and after transformations are consistent (e.g., user counts through a join pipeline, revenue before/after filtering, row counts through ETL steps). Define the acceptable loss/gain threshold based on what the transformation does and explain the reasoning.
- **Range**: key outputs fall within plausible bounds (e.g., conversion rates between 0–1, ARPDAU within historical range, cost estimates within order-of-magnitude of benchmarks, prediction accuracy within domain norms). State bounds and their source — data, logic, or domain context.
- **Null/missing**: critical fields have no unexpected nulls after joins or transformations.
- **Uniqueness**: where deduplication is expected, verify no duplicates remain.

Implementation: express assertions as executable checks in the appropriate language for the task (SQL checks, Python assert/validation, shell diff, automated tests, etc.). Log each assertion's pass/fail in the execution log. If an assertion fails, stop that branch and investigate — do not silently skip.

## Decision Rigor (all stages)

For every important choice, record:

- Decision. Evidence. Resolution strategy used. Alternatives considered. Reason for choice. Remaining uncertainty. Confidence. What would change the decision.

Do not use unsupported claims as conclusions. Distinguish:

- `direct`: directly observed in data/source material.
- `derived`: calculated from a defined formula or logic.
- `proxy`: inferred from indirect evidence.
- `unsupported`: not backed by available evidence.

Recommendations must follow: `evidence -> interpretation -> recommendation -> expected effect -> risk -> validation`.

Causal language requires causal evidence. Otherwise use "associated with", "suggests", or "hypothesis".

## Sensitivity Analysis (execute, for high-stakes recommendations)

For any recommendation where a wrong call would be costly or hard to reverse — budget allocation, product changes, pricing, architecture decisions, go/no-go gates, resource allocation, campaign strategy, or other irreversible outcomes — sensitivity analysis is **required**:

- Show how the recommendation changes if key inputs vary across domain-appropriate ranges.
- If the recommendation involves a threshold, show the input range under which it holds vs. flips.
- If the recommendation depends on a scope or time boundary, show how results differ with reasonable shifts.

What counts as "high-stakes" is determined at intake based on the audience model and the decision being supported. Delivering a high-stakes recommendation without sensitivity analysis is `P1`.

## Visualization Standards (execute, when deliverable includes charts or tables for external audience)

When a deliverable contains charts, graphs, or formatted comparison tables intended for a decision maker:

- **Scale integrity**: Y-axis starts at zero for bar charts comparing magnitudes. If truncated, annotate clearly why and show the full-range version alongside.
- **Comparison fairness**: series being compared use the same time window, same denominator, and same unit. If normalization differs, label it.
- **Legend and label completeness**: every axis, series, and data point must be labeled. No unlabeled lines or mystery colors.
- **No visual exaggeration**: area/bubble charts must scale by the correct dimension (area, not radius). Dual-axis charts must justify why two scales are needed.
- **Table deviation format**: comparison columns show actual numerical deviations (`+26%`, `-$1.2k`), not symbolic markers. Empty cells must be explained.

These standards apply to external-facing deliverables. Internal working charts during exploration do not require full compliance but should not be misleading.

## Mandatory Review Checkpoints (review)

The following 5 dimensions must be explicitly checked during every review. Each failed check must be classified as P0/P1/P2 and resolved before delivery.

### 1. Logic Consistency

Every conclusion must follow from its premises. Check for:

- **Counting errors**: numbers cited in narrative match the data (e.g., "split into N types" must equal the actual count).
- **Constructive circularity**: features used as clustering/model inputs will trivially differ across groups — this is an algorithm result, not a discovery. Deliverables must distinguish *定义性特征* (input features, tautologically different) from *衍生发现* (metrics NOT used as inputs, independently validating the grouping). Presenting a clustering-input difference as a "finding" without this label is P1.
- **Causal claims**: correlation-based evidence cannot support "X causes Y" language. Use "associated with", "suggests", or "hypothesis" unless causal design is present.
- **Direction consistency**: if a metric is described as "highest" or "lowest", verify it actually is by checking the comparison table.

### 2. Data Interpretability

- **Mean-of-ratio trap (Jensen's inequality)**: mean(A/B) × mean(B) ≠ mean(A). Group-level ratio averages cannot be multiplied by group-level absolute averages to recover totals. If a deliverable presents both ratios and absolute values, add a note that they are not composable. Failing to flag this when both appear is P1.
- **Aggregation level**: don't mix user-level and cohort-level metrics without explicit annotation. A cohort average can mask bimodal distributions.
- **Deviation format**: comparison columns must show actual numerical deviations (`+26%`, `-40%`), not symbolic markers (▲▼★). Empty deviation cells ("—") require explanation or must be filled with the actual value.
- **Transformation disclosure**: if features were transformed (log, scaling, normalization), explain *why* in the methodology. If omitting the transform caused a known failure (e.g., degenerate clustering), state it.

### 3. Decision Point Logic

- **Evidence chain**: every recommendation must trace: `evidence → interpretation → recommendation → expected effect → risk → validation`. A recommendation without this chain is P1.
- **Oversimplification check**: broad conclusions (e.g., "geography doesn't matter") must include relevant caveats when known confounders exist (e.g., eCPM varies 5-10x across countries). Omitting a material caveat on a conclusion that drives a recommendation is P1.
- **Threshold justification**: any cutoff used for a decision (K=4, >=2 games, "high value" tier boundary) must state why that value was chosen and what would change at adjacent values.

### 4. Data Accuracy

- **Cross-validation**: key numbers cited in narrative text must match the corresponding table cells. Spot-check at least 3 numbers per section against source data.
- **Arithmetic**: deviation percentages must be arithmetically correct: `(group - global) / global × 100`.
- **Summation**: category percentages must sum to expected totals (e.g., user segment shares sum to 100%, revenue contributions sum to 100%). Define rounding tolerance.
- **Cross-table consistency**: if the same metric appears in a detail table and a summary table, values must match. Discrepancies are P1.

### 5. Caliber (口径) Accuracy

- **Definition stability**: the same metric name must mean the same calculation everywhere in the deliverable. If "收入" means D0-D7 ad revenue in one section, it cannot silently include D8-D30 in another.
- **Source alignment**: metric definitions must match the upstream data source. If the data uses `d0_d7_ad_revenue`, the report's "D0-D7 收入" must map to exactly that field.
- **Cross-section coherence**: when a segment profile cites a number and the comparison table cites the same metric for the same segment, values must be identical.
- **Version drift**: in multi-iteration projects (V1→V2→V3), any definition change between versions must be called out in the version delta. Silently changing a definition is P0.

### Review output format

For each dimension, state: `PASS` (no issues), `PASS_WITH_CAVEATS` (minor issues documented), or `FAIL` (issues must be fixed). List specific findings under each failed dimension with severity and fix action.

Severity-to-verdict mapping: any P0 or P1 finding within a dimension → that dimension is `FAIL`. Only P2 findings → `PASS_WITH_CAVEATS`. No findings → `PASS`.

### Self-review before delivery (non-negotiable)

The 5-dimension review is NOT optional and NOT something the user should have to ask for. Before presenting ANY final output to the user:

1. **Run the full 5-dimension review internally** — logic consistency, data interpretability, decision point logic, data accuracy, caliber accuracy. Do this silently as part of the workflow, not as a separate user-visible step.
2. **Fix every issue found.** If a fix requires a judgment call or information you don't have, ask the user — do not guess or leave it unresolved.
3. **Only present the deliverable once all 5 dimensions are PASS.** No exceptions. No "here's the output, I noticed a few issues" — fix first, deliver second.
4. **If you cannot achieve all-PASS** (e.g., source data is ambiguous, a definition is genuinely unclear), escalate the specific blocker to the user before delivery. Do not deliver with known FAIL items.

This applies to every deliverable: reports, analyses, Feishu documents, handoff artifacts, summary tables. The user should never receive output that would fail any of these 5 checks. The 5 dimensions must always be checked, but depth is proportional to deliverable complexity — a simple data extract may pass all 5 in seconds; a full analytical report requires thorough inspection.

## Missing Information (all stages)

Before asking the user, try to resolve from:

- Provided materials (files, screenshots, links).
- Local project files, prior reports, handoff docs, existing artifacts.
- Relevant domain skills and conventions.
- Read-only external sources when allowed and auth is valid.
- Logic and explainability strategies: deriving from available fields, checking metadata, comparing historical patterns, scenario branches.

Resolution order:

1. Source evidence: find explicit definition in requirements, files, docs, metadata.
2. Logical derivation: derive from available information, state the derivation.
3. Consistency check: compare against historical precedent and conventions.
4. Parallel analysis: keep multiple valid interpretations separate, show how conclusions differ.
5. Exclusion: mark unjustifiable choices as "not used for final decision".
6. Ask user: only if unresolved and would materially change result or delivery safety.

When multiple plausible standards exist and evidence does not favor one, present parallel standards rather than silently picking one.

## Tooling and Skill Acquisition (all stages)

This skill may proactively look for relevant local skills, official docs, or open-source tools when they would materially improve quality, speed, validation strength, or reproducibility for the current project.

Default order:

1. Use existing project files, local tools, and already-available skills first.
2. If they are insufficient or clearly inefficient, search read-only sources for better options:
   - local installed skills
   - official documentation
   - reputable GitHub repositories / package docs
3. Prefer adopting an existing well-maintained tool or skill over rebuilding the same capability from scratch.

Allowed proactive actions:

- Search for relevant GitHub projects, libraries, CLI tools, notebooks, templates, or skills.
- Read documentation to evaluate whether a tool fits the task.
- Use an already-installed local skill when it clearly matches the subtask.
- Install a new tool/package/skill **only** when all of the following are true:
  - it materially helps complete the current project;
  - it is from an official or reputable source;
  - it can be installed in an isolated, non-destructive way;
  - it does not require modifying existing project files, production systems, databases, dashboards, or user data;
  - it does not require sending messages, creating external side effects, or broad new permissions;
  - the install/use decision, version, and reason are logged in the execution notes.
- Actual installation happens at execute stage, not during intake or planning. Intake and planning may evaluate and recommend tools but must not install.

Not allowed without explicit user approval:

- System-wide installs.
- Changes to existing project dependency files, lockfiles, manifests, or environment configuration.
- Any install that writes into an existing repo/environment in a way that changes current behavior.
- Auto-running untrusted scripts fetched from the internet.
- Tools that require write access to Feishu/Lark, databases, dashboards, production systems, or external messaging surfaces.

Safe default for installs:

- Prefer isolated local environments, disposable temp directories, or user-scoped tool locations that do not modify the project.
- If isolation is not possible, do not install; continue with the best available local approach or ask the user.

Decision rule:

- Search freely when useful.
- Install only when the expected gain is meaningful and the blast radius is low.
- If a tool is merely "nice to have", skip installation and proceed without it.
- Log tool selection as a decision: what was considered, what was chosen, what was rejected and why.
- If an installed tool fails or causes issues, remove/clean up and fall back to the best available local approach. Do not let a broken tool block the project.

## Re-Planning (data validation and execute)

When evidence **materially** conflicts with the plan — meaning it would change the methodology, scope, key definition, or core conclusion:

1. Stop execution on the affected branch.
2. Log the conflict as a **re-plan trigger**: what was assumed, what was found, why it matters.
3. Create a revised plan version (`_v2`, `_v3`) — do not silently overwrite.
4. Present the revised plan to the user for approval before resuming.

Cosmetic discrepancies or minor parameter adjustments within the same approach are logged but do not trigger re-planning.

Re-planning is expected in complex projects, not a failure.

## Version Management (multi-iteration projects)

When a project iterates (V1 → V2 → V3):

- Each iteration has its own plan and execution artifacts with a version suffix.
- Each new iteration starts with a **version delta**: what changed, why, expected impact on results.
- Prior version artifacts are preserved as evidence trail.
- Decision log links cross-version: if V2 changes a definition V1 used, V2's entry references V1's.

What warrants a new version: change in methodology, scope, key definition, or data source. Parameter tuning within the same approach is logged in the execution log, not a new version.

## Stage Outputs

File naming convention: `01_intake.md`, `02_plan.md`, `02b_data_validation_report.md`, `03_execution_log.md`, `04_review.md`, `05_fix_log.md`, `06_final_check.md`, `07_pre_delivery_gate.md`. For versioned iterations, append suffix: `02_plan_v2.md`. Corrected deliverables use a descriptive suffix rather than overwriting: `report_v2_fixed.md`.

1. `Intake`: background, goal, **audience model**, known inputs, definitions, **assumption registry** (initial), **source coverage log** (what was searched, what was found, what was unreachable, what was not searched), self-resolved decisions, blocking questions (max 3), deliverables, risks.
2. `Plan`: scope, non-goals, inputs, **source coverage summary** (what intake already checked, what still needs checking), steps, **time/effort estimate per step**, validation steps, **assertions to embed**, decision policy, red-line compliance, stop conditions. If V2+: **version delta**.
3. `Data Validation`: **data_validation_report** (per-check pass/fail). Gate decision.
4. `Execute`: evidence log, decision log, **source coverage log** (updated — what additional sources were searched during execution), **assumption registry updates**, **assertion results**, outputs, **tooling log** (if any tool/skill was searched, adopted, installed, or rejected: source, version, install method, isolation, reason, fallback/cleanup), **re-plan triggers** if any.
5. `Review`: findings first `P0/P1/P2`; check requirement match, logic, decision rigor, red lines, **assumption coverage**, **source coverage adequacy**, **assertion pass rate**, **sensitivity completeness**, **mandatory review checkpoints** (logic consistency, data interpretability, decision point logic, data accuracy, caliber accuracy — all 5 dimensions must show explicit PASS/FAIL).
6. `Fix` (if needed): per issue — problem description, fix applied, verification result, residual risk. If fix changes core methodology, treat as re-plan trigger instead.
7. `Final check`: outputs exist and non-empty, conclusions trace to evidence, caveats explicit, **untested high-impact assumptions flagged**, **visualization accuracy**.
8. `Pre-delivery gate`: `READY_TO_DELIVER`, `READY_WITH_CAVEATS`, `NOT_READY`, or `INSUFFICIENT_DATA`.

Severity:

- `P0` (fatal): red-line violation, fake success, empty output, blocking decision presented as settled, parallel standards collapsed without evidence, **data validation skipped or failed without re-plan**, **invalidated assumption used for core conclusion**, **assertion failure silently skipped**, **caliber definition silently changed between versions**.
- `P1` (serious, repairable): weak evidence, missing validation, unsupported interpretation, **high-stakes recommendation without sensitivity analysis**, **misleading visualization in external deliverables**, **untested high-impact assumption not flagged**, **constructive circularity not labeled**, **mean-of-ratio composability not flagged when both ratios and absolutes are presented**, **material caveat omitted from a conclusion that drives a recommendation**, **cross-table number mismatch**.
- `P2` (quality only): clarity, structure, formatting issues that don't change the conclusion.

Delivery gate:

- `READY_TO_DELIVER` is an internal status, not execution permission. After reaching any gate verdict, **stop and present the verdict and deliverables to the user**. Do not auto-execute any external delivery action (creating Feishu docs, sending reports, publishing dashboards) without explicit user approval in that moment.
- Any `P0` → `NOT_READY`.
- Any mandatory review checkpoint dimension at `FAIL` → `NOT_READY`. All 5 dimensions must be `PASS` or `PASS_WITH_CAVEATS` before delivery.
- Unresolved key decision without parallel branches → `NOT_READY`.
- `READY_WITH_CAVEATS`: remaining issues non-fatal, don't change core recommendation.
- `INSUFFICIENT_DATA`: inputs fundamentally cannot answer the question — missing dimensions, insufficient sample, required data not yet available. This is a valid deliverable. Must explain what inputs are needed and when they might be available. Boundary: fixable analysis errors → `NOT_READY`; inadequate inputs regardless of analysis quality → `INSUFFICIENT_DATA`.

## Cross-Session Handoff

Each stage output must be self-contained: a new CC session should be able to continue from any stage without prior conversation context. This means:

- All referenced inputs identified by path, not just by name.
- All decisions made so far with rationale, not just conclusions.
- Current source coverage state: what has been searched, what remains unsearched, what was unreachable, and the impact on confidence.
- Current state of assumption registry and assertion results.
- Clear statement of what's done vs. what remains.

See `references/stage-prompts.md` for ready-made handoff prompt templates.

## Domain Routing

When the project involves a recognized domain, use the corresponding skill. This list is non-exhaustive:

- Feishu/Lark docs: `feishu-auth-guard`; default read-only.
- Scattered docs/chats/meetings: `work-intake-from-feishu`.
- Game levels/funnels: `game-level-analysis`.
- Game product data (BigQuery): `game-bq-analysis`.
- Level funnel / pass-fail mining: `level-funnel-mining`.
- Data insight → decision report: `data-insight-reporting`.
- ADLTV/LTV prediction: `adltv-prediction-review`.
- Quantitative report review (UA efficiency, ROA, etc.): `quantitative-report-review`.
- Feishu report delivery: `report-to-feishu-verified`.
- Agent/tooling issues: `agent-tooling-ops`.

## References

Do not read reference files in planning-only dry runs, experiments, or when tools are disabled.

Only read `references/stage-prompts.md` when the user explicitly asks for separate CC handoff prompt templates.
