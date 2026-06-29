# EvalLake — Product Brief

## TL;DR

Teams shipping LLM products lack a continuous, shared view of AI quality. Eval results live in disposable scripts, regressions reach production undetected, and product teams have no quality signal without reading code. EvalLake is a continuous quality intelligence platform: an evals pipeline backed by a lakehouse that makes AI quality a first-class release signal — legible to engineers, PMs, and compliance teams alike.

---

## Problem

### Context

LLM products ship quality bugs differently from traditional software. There is no stack trace when Accuracy regresses. A prompt change that improves ICD-10 coding can silently spike the Hallucination Rate on medication dosage extraction. The feedback loop is slow: a regression ships on Tuesday and surfaces in support tickets by Friday.

Current state at most teams:
- Evals run in a one-off Python script before each release, if at all.
- Results are not persisted — the next run overwrites any baseline.
- No one can answer "did quality improve or degrade this sprint?" without re-running everything.
- Product managers and compliance teams are entirely dependent on engineers to interpret quality.

### Who is affected

**ML Engineers** — run evals manually, have no historical baseline, rebuild context from scratch every sprint.

**AI Product Managers** — learn about regressions from user complaints, not dashboards. Can't own the quality story in release planning.

**Platform / DevOps** — want AI quality gates to behave like test coverage gates: automatic, blocking, auditable. Nothing exists to plug in.

**Compliance / Legal** — need a chain of custody: which model, which prompt, which eval run produced which output. Currently untraceable.

---

## Opportunity

**Streams: AI × Data**

The key insight is simple: eval results are just data. If you treat them as structured records and land them in a lakehouse with a consistent schema, you immediately unlock everything that data infrastructure gives you: historical trends, model comparisons, regression detection, prompt-change attribution, audit trails.

The product layer on top of that lakehouse is what makes the signal legible — to engineers running CI, to PMs reading dashboards, and to compliance teams pulling audit reports.

This is a genuine cross-stream opportunity. The AI stream produces the quality signal; the data stream preserves and organizes it; the product stream makes it actionable; the infra stream embeds it in the release process.

---

## Competitive Positioning

Existing tools solve adjacent problems but not this one.

**Weights & Biases (W&B)** is the standard for ML experiment tracking — strong for training runs, hyperparameter sweeps, and model checkpoints. It was not designed for production LLM eval pipelines: there is no concept of a prompt-version gate, no CI hook that blocks a PR on a quality regression, and no persistent data layer a compliance team can query without Python.

**LangSmith** (LangChain) is strong at LLM observability and chain debugging. Its eval capabilities are improving but remain tied to the LangChain ecosystem — it is a tracing tool that acquired eval features, not an eval platform with a data infrastructure layer underneath. There is no lakehouse, no historical trend query, and no audit trail in the sense a HIPAA-governed team needs.

**Braintrust** and **Promptfoo** offer lightweight eval runners useful to individual engineers. They are not designed for multi-team, multi-model quality governance: results are ephemeral, there is no gate mechanic that integrates with a PR workflow, and they produce no queryable history.

EvalLake's differentiation is structural: **eval results are first-class data, not log output.** By landing every run into a partitioned lakehouse with a consistent schema, EvalLake makes quality trends queryable by any stakeholder, regression detection automatic and gate-enforced, and compliance audits self-serve — for teams that need quality to be a release-blocking signal, not a post-hoc report.

---

## Solution: EvalLake

EvalLake is a continuous LLM quality intelligence platform with four surfaces:

1. **Ingestion pipeline** — accept eval results (golden sets, RAG metrics, online canaries) via a REST API and persist them to a partitioned lakehouse table (Delta / Iceberg), with a PII scrubber layer before write.

2. **Quality analytics** — compute per-run quality scores (Accuracy, ROUGE-L, BERTScore, Hallucination Rate), detect regressions against configurable thresholds, and maintain a prompt-change lineage table that links every quality delta to a specific prompt version.

3. **Product dashboard** — trend charts, failing category breakdowns, model comparison, run detail views — designed to be readable by PMs and engineers without SQL.

4. **CI/CD regression gate** — a GitHub Actions integration that runs evals on PR open, evaluates the gate decision (PASS / WARN / BLOCK), and posts the result as a check status. Override requires 2 LGTM approvals, logged to the audit trail.

**Gate webhook contract:** The gate action calls `POST /v1/gate/evaluate` with a JSON body of `{ run_id, workspace_id, model_id, prompt_version, pr_number }`. EvalLake responds synchronously (within the 8-minute timeout) with `{ decision: "pass"|"warn"|"block", quality_score, failing_metrics: [...], run_url }`. The GitHub check is set to `success` on pass/warn and `failure` on block. On WARN or BLOCK, EvalLake fires a webhook to the configured Slack channel (`POST` with an attachment block containing run_url, delta summary, and a one-click "View Run" deep link) and a PagerDuty event (`severity: warning|critical`) via the PagerDuty Events API v2. All webhook deliveries are logged to the audit trail with timestamp, payload hash, and HTTP response code — giving compliance teams a complete record of every quality signal fired.

---

## Data Model

Three core tables power EvalLake's lakehouse layer. The schema is designed for partition-efficient time-range queries and extensible metric types — adding a new metric requires no DDL change.

### `eval_runs`
Top-level record for each pipeline execution.

| Field | Type | Notes |
|-------|------|-------|
| `run_id` | `STRING` | UUID, PK — e.g. `EVL-2249` |
| `workspace_id` | `STRING` | Tenant identifier |
| `model_id` | `STRING` | e.g. `gpt-4o-2024-11-20` |
| `prompt_version` | `STRING` | Semver tag — e.g. `v2.3.1` |
| `trigger` | `ENUM` | `ci_pr` · `scheduled` · `manual` |
| `pr_number` | `INT` | GitHub PR ref (nullable) |
| `quality_score` | `FLOAT` | Composite weighted score, 0–100 |
| `gate_decision` | `ENUM` | `pass` · `warn` · `block` |
| `run_duration_s` | `INT` | Wall-clock seconds |
| `golden_set_version` | `STRING` | e.g. `golden-set-v4.2` |
| `created_at` | `TIMESTAMP` | **Partition key** (with `model_id`) |

### `test_cases`
One row per test case execution within a run.

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | `STRING` | UUID, PK |
| `run_id` | `STRING` | FK → `eval_runs` |
| `golden_case_id` | `STRING` | FK → golden set — e.g. `GT-0041` |
| `category` | `STRING` | e.g. `Medication Dosage` |
| `input_hash` | `STRING` | SHA-256 of prompt input — no PHI stored |
| `output_hash` | `STRING` | SHA-256 of model output |
| `status` | `ENUM` | `pass` · `warn` · `fail` |
| `created_at` | `TIMESTAMP` | |

### `metrics`
One row per metric per test case — flat design for extensibility.

| Field | Type | Notes |
|-------|------|-------|
| `metric_id` | `STRING` | UUID, PK |
| `case_id` | `STRING` | FK → `test_cases` |
| `run_id` | `STRING` | Denormalized FK → `eval_runs` (avoids join for dashboard queries) |
| `metric_name` | `STRING` | e.g. `accuracy` · `rouge_l` · `bert_score` · `hallucination_rate` |
| `value` | `FLOAT` | Metric value |
| `threshold_warn` | `FLOAT` | Warn threshold at time of run (snapshotted — thresholds can change) |
| `threshold_block` | `FLOAT` | Block threshold at time of run |
| `created_at` | `TIMESTAMP` | |

**Partition strategy:** `eval_runs` is partitioned by `dt` (date) × `model_id`. This keeps 30-day trend queries and sprint-over-sprint comparisons fast without full table scans. `test_cases` and `metrics` are co-located by date for join locality. Because `metric_name` is a free string, new metric types (e.g. `g_eval`, `human_preference_score`) land without schema migrations.

---

## Jobs to be Done

| User | Job | Outcome |
|------|-----|---------|
| ML Engineer | Catch an accuracy regression before the PR merges | Zero surprise quality bugs in production |
| AI PM | Know whether last sprint's prompt changes improved quality | Owns the quality story in release planning |
| Platform Lead | Enforce quality gates in CI the same way test coverage works | AI quality is a first-class build signal |
| Compliance | Trace which model and prompt produced any given output | Full chain of custody for HIPAA/PHIPA audits |

---

## Personas

**Sarah — Senior ML Engineer @ pareIT**
Runs evals in a Jupyter notebook before each release. Spends 30 minutes every sprint manually comparing score printouts. Has no dashboard. When asked "did the last prompt change help?", she has to re-run everything. EvalLake lets her answer that question in 10 seconds.

**James — AI Product Manager**
PM for a clinical copilot. Discovers quality regressions from support tickets, not metrics. Wants to own a quality metric in the sprint review but has nothing to show that non-engineers can interpret. EvalLake gives him the dashboard and the vocabulary.

**Priya — Platform Lead**
Runs the GitHub Actions CI pipeline. Already blocks PRs on failing unit tests and coverage thresholds. Wants to add an AI quality gate with the same mechanics. Currently nothing exists to plug in. EvalLake is a one-line addition to the workflow file.

---

## Success Metrics

### Outcome metrics (what EvalLake is actually for)

These anchor EvalLake as a quality-safety tool, not just an analytics dashboard.

| Metric | Target | Why it's an outcome |
|--------|--------|---------------------|
| **Regression catch rate before production** | ≥ 90% of quality regressions caught by gate before merge to main | Directly measures the core safety promise |
| **Time to detect quality drift** | Median ≤ 15 min from a quality-degrading change to first WARN/BLOCK | Tightness of the feedback loop — slow detection means slow remediation |
| **Mean time to remediation (MTTR)** | ≤ 4 hours from BLOCK decision to PR fix or override | Gate is only valuable if teams act on it quickly |
| **Escaped regression rate** | < 1 quality regression per month reaches production undetected | The number that matters to compliance and clinical safety teams |

### North Star
**Quality-gated deploys per week** — the number of production deploys that passed or were blocked by an EvalLake gate. This measures whether quality gates are actually embedded in the release process, not just installed.

### Leading indicators

| Metric | Target (90d) | Rationale |
|--------|-------------|-----------|
| Time-to-first-gate (TTFG) | < 20 minutes from signup | Activation quality |
| Golden set coverage | ≥ 50 test cases per model | Depth of quality baseline |
| Gate block rate | 3–8% of PRs | Too low = not meaningful; too high = false positives |
| Dashboard WAU | ≥ 80% of PM + ML org | Product adoption |
| Prompt change attribution | 100% of quality deltas linked to a prompt version | Lineage completeness |

### Guardrails (things we don't break)
- Eval run p95 latency < 5 minutes (if it's slower, engineers bypass it)
- Gate false positive rate < 5% (blocks a PR that was healthy)
- Zero PHI persisted in eval payloads (HIPAA/PHIPA compliance)
- Audit trail completeness: 100% of gate overrides logged with approver identity

---

## v1 Scope (6 weeks, 2 engineers + PM)

### In scope
- Eval result ingestion API (`POST /v1/runs`) with schema validation and PII scrubber
- Lakehouse table design (partitioned by date × model)
- Eval metric computation (Accuracy, ROUGE-L, BERTScore, Hallucination Rate)
- Regression detection (configurable warn / block thresholds)
- Prompt change lineage table
- Product dashboard: Overview, Eval Runs, Run Detail, Quality Trends, Regression Gates
- GitHub Actions gate integration (`evallake/gate-action@v1`)
- Slack + PagerDuty alerting on WARN / BLOCK

### Out of scope (v2)
- Self-serve golden set authoring UI with LLM-assisted generation
- LLM-as-judge scoring (RAGAS / custom judge model)
- Multi-workspace / org-level quality roll-up
- Automated canary deployment (Kubernetes / Argo Rollouts integration)
- FinOps integration: cost-per-quality-point analytics

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Golden set becomes stale / unrepresentative | High | Medium | Version golden sets; alert when coverage decay detected |
| Eval latency causes engineers to bypass gate | Medium | High | Async gate mode; configurable timeout; p95 SLO alerting |
| PHI leaks into eval payloads | Low | Critical | PII scrubber middleware before every write; scrubber tested on synthetic clinical text |
| Teams override gate routinely, undermining it | Medium | Medium | Override requires 2 LGTM approvals; override rate surfaced on dashboard; trend alert if >20% of blocks overridden |
| False positives on gate block legitimate PRs | Medium | High | Tune thresholds with 30d baseline before enforcing; warn-before-block rollout |
