# EvalLake — Backlog

Structured GitHub Issues across 4 milestones. Sequenced by dependency and risk: foundation first, then analytics, then CI/CD integration, then the product layer that makes it all legible.

Labels used: `type:design` `type:backend` `type:frontend` `type:devex` `type:integration` `stream:ai` `stream:data` `stream:infra` `stream:product` `milestone:1–4` `priority:p0–p2`

---

## Milestone 1 — Foundation: Ingestion & Storage (Sprint 1–2)

> **Goal:** Any eval result can be submitted and reliably stored. No quality analytics yet — just a durable, queryable ledger.

---

### Issue #1 — Define eval result schema
**Labels:** `type:design` `stream:data` `milestone:1` `priority:p0`

**Why first:** Everything else depends on this. A bad schema now means a migration later.

**Description:**
Define the canonical JSON schema for an eval result record. Must cover three run types: golden-set, scheduled, and online canary.

Required fields: `run_id`, `model_id`, `model_version`, `prompt_version`, `golden_set_version`, `run_type`, `test_case_id`, `faithfulness`, `relevance`, `context_recall`, `overall_score`, `input_hash` (not raw input — hashed for PHI safety), `timestamp_utc`, `triggered_by` (`ci|scheduled|manual`), `pr_ref` (nullable), `metadata` (object).

**Acceptance criteria:**
- Schema documented in `docs/schema.md` with field descriptions and allowed values
- JSON Schema file (`schemas/eval_result.json`) checked in and validated
- Reviewed with an ML engineer and a compliance contact before merging
- Covers all three run types with no ambiguous nullable fields

---

### Issue #2 — Design lakehouse table structure
**Labels:** `type:design` `stream:data` `milestone:1` `priority:p0`

**Why this sprint:** The table design determines what queries are fast and cheap forever. Do this before writing a single byte.

**Description:**
Design the Delta / Iceberg table layout for eval results. Define partitioning, clustering, retention, and the aggregation tables that power the dashboard.

Tables needed:
- `eval_results.runs_raw` — partitioned by `dt` (date) and `model_id`; append-only; 180d retention
- `eval_results.runs_summary` — one row per run; quality score + gate decision; forever retention
- `eval_results.quality_trends` — daily aggregate per model × prompt version; powers the trend chart
- `eval_results.gate_decisions` — every gate decision with outcome, thresholds applied, and override status

**Acceptance criteria:**
- ERD documented in `docs/schema.md`
- Partition strategy documented with rationale (read pattern justification)
- Retention policy specified per table
- Reviewed with a data engineer

---

### Issue #3 — Build eval result ingestion API
**Labels:** `type:backend` `stream:data` `milestone:1` `priority:p0`

**Description:**
`POST /v1/runs` — accepts an eval result payload (or a batch of results), validates against the schema from #1, runs the PII scrubber (#4), and writes to the lakehouse.

**Acceptance criteria:**
- Authenticated with workspace API key (Bearer token)
- Returns `{ run_id, status: "accepted", ingested_at }` on success
- Validates payload against JSON Schema; returns 422 with field-level errors on failure
- PII scrubber middleware fires before any write
- p95 latency < 200 ms for single-result payload
- Integration test with a real lakehouse write (not mocked)

---

### Issue #4 — PII scrubber middleware
**Labels:** `type:backend` `stream:ai` `milestone:1` `priority:p0`

**Why p0:** HIPAA/PHIPA compliance. PHI in eval payloads is a critical failure mode. This ships before the API goes live.

**Description:**
Middleware layer that intercepts every incoming eval payload and detects / redacts PHI (patient names, DOBs, MRNs, phone numbers, email addresses) before persistence. Two-pass approach: regex patterns for structured PHI (MRN format, DOB patterns) + lightweight NER model for unstructured clinical text.

**Acceptance criteria:**
- Tested against a synthetic dataset of 200 clinical text examples with known PHI
- Precision ≥ 99% (false PHI removal is acceptable; missed PHI is not)
- Redacted fields replaced with `[REDACTED:<type>]` tokens, not deleted
- Redaction events written to audit trail with field path, redaction type, and timestamp
- Adds < 50 ms to ingestion latency at p95

---

## Milestone 2 — Core Analytics (Sprint 3–4)

> **Goal:** Quality trends are queryable and visible. The dashboard shows something real.

---

### Issue #5 — Compute quality score per run
**Labels:** `type:backend` `stream:ai` `milestone:2` `priority:p0`

**Description:**
After ingestion, compute the aggregate quality score for a run: configurable weighted average of faithfulness, relevance, and context recall. Write the result to `runs_summary`.

Default weights: faithfulness 0.45 · relevance 0.35 · context recall 0.20 (higher weight on faithfulness for clinical use cases where hallucination is highest risk).

**Acceptance criteria:**
- Weights configurable per workspace via settings API
- Handles partial runs (not all golden set cases present — score computed over present cases only, flagged as partial)
- Score breakdown available at test-case level in `runs_raw`
- Score recomputation triggered if weights change (backfill job)

---

### Issue #6 — Regression detection algorithm
**Labels:** `type:backend` `stream:data` `milestone:2` `priority:p0`

**Description:**
Compare a new run's scores against the rolling 7-day average for the same `model_id` × `prompt_version` combination. Classify each metric and the overall score as PASS / WARN / BLOCK based on workspace-configured thresholds. Emit a `gate_decision` event and write to `gate_decisions` table.

**Acceptance criteria:**
- Configurable warn and block thresholds per metric, per workspace
- Comparison baseline is rolling 7d average (not just previous single run — prevents flaky baselines)
- If fewer than 3 runs exist in the baseline window, gate defaults to WARN (not BLOCK) with a "insufficient baseline" flag
- Gate decision event emitted synchronously; async notification (Slack/PagerDuty) sent within 30s
- Override path: 2 LGTM approvals from `evallake/approvers` group unblocks the gate; override written to audit trail

---

### Issue #7 — Prompt change lineage tracker
**Labels:** `type:backend` `stream:ai` `milestone:2` `priority:p1`

**Description:**
When a run is submitted with a `prompt_version` not seen before for a given `model_id`, treat it as a new prompt release. Compute and store the quality delta vs the previous prompt version for every metric, for every model that ran both versions.

**Acceptance criteria:**
- Prompt change impact queryable: given `prompt_version`, return delta per metric per model
- Impact visible in the Quality Trends screen's "Prompt Change Impact" table
- Prompt change markers rendered on trend charts at the correct x-axis position
- Handles model-specific prompt versions (a prompt that only some models have run)

---

### Issue #8 — Quality dashboard: Overview + Trends screens
**Labels:** `type:frontend` `stream:product` `milestone:2` `priority:p0`

**Description:**
Build the Overview and Quality Trends screens. This is the first thing a PM or ML engineer sees. It needs to load fast and tell the story without requiring configuration.

Overview: quality score stat card, regression rate, canary status, 30-day trend chart (gpt-4o default), recent runs table, top failing category breakdown.
Quality Trends: model selector chips, multi-model trend chart with prompt change markers, prompt change impact table.

**Acceptance criteria:**
- Loads in < 2s on a standard laptop
- Overview alert banner fires automatically when any model has a WARN gate in the last 24h
- Model selector chips toggle trend lines without a full page reload
- Prompt change markers clickable — clicking opens the Prompt Change Impact row
- Empty state for new workspaces: friendly call to action to trigger first run

---

## Milestone 3 — CI/CD Integration (Sprint 5–6)

> **Goal:** Quality gates are running automatically on every PR. Engineers don't have to think about evals.

---

### Issue #9 — GitHub Actions gate action
**Labels:** `type:devex` `stream:infra` `milestone:3` `priority:p0`

**Description:**
Publish `evallake/gate-action@v1` — a GitHub Action that triggers an eval run on PR open/sync, polls for the gate decision (up to configurable timeout, default 8 min), and posts the result as a GitHub check status.

Check states: `success` (PASS), `neutral` (WARN — PR proceeds, alert fires), `failure` (BLOCK — PR cannot merge without override).

**Acceptance criteria:**
- Action inputs: `api-key`, `model`, `golden-set`, `fail-on` (block | warn | never), `timeout-minutes`
- Works with any model configured in the workspace — not hardcoded
- Async mode: `fail-on: never` posts a comment with the quality score and gate decision, never blocks the PR
- Step summary in the GitHub Actions UI shows quality score, gate decision, and link to the full run in EvalLake
- End-to-end test in a test repo runs green before publishing to GitHub Marketplace

---

### Issue #10 — Gate override workflow (2-LGTM)
**Labels:** `type:backend` `stream:infra` `milestone:3` `priority:p1`

**Description:**
When a gate blocks a PR (BLOCK decision), allow authorized team members to override it with 2 LGTM approvals. The override lifts the GitHub check block and writes a full audit record.

**Acceptance criteria:**
- Override requires exactly 2 approvals from the `evallake/approvers` group (configurable per workspace)
- The author of the blocked PR cannot be one of the 2 approvers
- Audit record includes: run_id, override_by (both approvers), override_reason (free text, required), timestamp
- Override does not suppress the WARN/BLOCK indicator in the EvalLake dashboard — it is visible and searchable
- Override rate surfaced as a metric on the Regression Gates screen (a rising override rate is a signal that thresholds need tuning)

---

### Issue #11 — Slack + PagerDuty alert integration
**Labels:** `type:integration` `stream:product` `milestone:3` `priority:p1`

**Description:**
Send alerts on WARN and BLOCK gate decisions to Slack and/or PagerDuty. Configuration per workspace; Slack channel and PD service key set in Settings.

**Acceptance criteria:**
- Slack message: run ID, model, quality score, delta vs baseline, top failing metric, link to run detail in EvalLake
- PagerDuty alert created on BLOCK only (not WARN) — respects severity semantics
- Mute/snooze available per model per workspace (useful when a model is known-flaky and a fix is in flight)
- Alert delivery confirmed within 30 seconds of gate decision emission
- Webhook retried up to 3x with exponential backoff on failure

---

## Milestone 4 — Product Layer (Sprint 7–8)

> **Goal:** Non-engineers can use EvalLake. Onboarding is smooth. The product is ready to share with customers.

---

### Issue #12 — Eval Runs list view + run detail
**Labels:** `type:frontend` `stream:product` `milestone:4` `priority:p0`

**Description:**
Eval Runs screen: full run history table with model, prompt version, quality score, gate decision, duration, triggered-by, timestamp. Filterable by model, date range, and gate status.

Run detail: gate decision banner (pass/warn/block), RAG metric cards with progress bars and threshold labels, golden set results table (sortable by delta vs previous run), lineage panel (lakehouse table, partition, golden set version, embeddings model), CI context panel (PR link, author, branch, gate decision).

**Acceptance criteria:**
- Runs table: pagination (50 rows/page), client-side column sorting, CSV export
- Filters preserve state on navigation (don't reset when you open a run and come back)
- Run detail gate banner changes color and copy depending on gate decision (PASS / WARN / BLOCK)
- Golden set results sortable by delta — failing tests float to the top by default
- "View in Lakehouse" button in lineage panel deeplinks to the correct table partition

---

### Issue #13 — Golden set management (read-only v1)
**Labels:** `type:frontend` `stream:product` `milestone:4` `priority:p1`

**Description:**
Allow team members to view and export the current golden set. Authoring is out of scope for v1 (manual upload via API). This screen is for inspection and audit.

**Acceptance criteria:**
- Shows all test cases: ID, category, last result (pass/warn/fail), last run score, date last run
- Filter by category and status
- Version history: can view previous golden set versions side-by-side
- CSV export
- Empty state: links to the API docs for uploading a golden set

---

### Issue #14 — Workspace onboarding flow
**Labels:** `type:frontend` `type:integration` `stream:product` `milestone:4` `priority:p0`

**Description:**
Guide new workspaces through activation: (1) copy the API key, (2) install the GitHub Action, (3) upload a golden set, (4) trigger the first eval run. Completable in under 20 minutes.

**Acceptance criteria:**
- 4-step wizard with visible progress; each step has a "done" check that auto-advances when detected (e.g. step 3 auto-advances when the first run is ingested)
- Time-to-first-gate tracked as the primary activation metric
- Contextual help tooltips on every threshold field in step 4
- Skippable steps (some workspaces will already have a golden set)
- In-app checklist persists in the sidebar until all 4 steps are complete

---

### Issue #15 — Workspace Settings: thresholds + integrations
**Labels:** `type:frontend` `stream:product` `milestone:4` `priority:p1`

**Description:**
Settings screen covering: gate threshold configuration (warn/block per metric, per model or global), Slack and PagerDuty integration setup, team member access management, golden set version management, and audit log viewer.

**Acceptance criteria:**
- Threshold changes require confirmation ("Changing the faithfulness block threshold affects X active gates")
- Slack integration: test-message button confirms delivery before saving
- Audit log: searchable by actor, action type, and date range; exportable as CSV
- Team access: role-based (Admin, Engineer, Viewer); Viewer cannot change thresholds or override gates

---

## Icebox (v2+)

- `#16` Self-serve golden set authoring with LLM-assisted test case generation
- `#17` LLM-as-judge scoring (RAGAS framework / custom judge model)
- `#18` Side-by-side model comparison view (run the same golden set across models in one click)
- `#19` Embedding drift detection (alert when retrieved chunks diverge from golden set expectations)
- `#20` HIPAA audit export — full chain-of-custody report per patient encounter
- `#21` FinOps integration — cost per eval run, cost per quality-point-gained
- `#22` Automated canary rollout (Kubernetes / Argo Rollouts integration)
- `#23` Multi-workspace org dashboard — quality roll-up across product lines
