# EvalLake — Product Brief

## Overview

Teams building LLM products often struggle to keep track of AI quality over time. Evaluation results end up scattered across scripts, regressions slip into production, and product teams have no clear way to understand model performance without digging through code. EvalLake brings everything together in one place. It continuously tracks AI quality through an evaluation pipeline backed by a lakehouse, giving engineers, product managers, and compliance teams a shared view of performance that they trust before every release.

---

## Problem

### Context

LLM products ship quality bugs differently from traditional software. There is no stack trace when faithfulness regresses. A prompt change that improves ICD-10 coding can silently degrade medication dosage extraction. The feedback loop is slow: a regression ships on Tuesday and surfaces in support tickets by Friday.

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

## Solution: EvalLake

EvalLake is a continuous LLM quality intelligence platform with four surfaces:

1. **Ingestion pipeline** — accept eval results (golden sets, RAG metrics, online canaries) via a REST API and persist them to a partitioned lakehouse table (Delta / Iceberg), with a PII scrubber layer before write.

2. **Quality analytics** — compute per-run quality scores (faithfulness, relevance, context recall), detect regressions against configurable thresholds, and maintain a prompt-change lineage table that links every quality delta to a specific prompt version.

3. **Product dashboard** — trend charts, failing category breakdowns, model comparison, run detail views — designed to be readable by PMs and engineers without SQL.

4. **CI/CD regression gate** — a GitHub Actions integration that runs evals on PR open, evaluates the gate decision (PASS / WARN / BLOCK), and posts the result as a check status. Override requires 2 LGTM approvals, logged to the audit trail.

---

## Jobs to be Done

| User | Job | Outcome |
|------|-----|---------|
| ML Engineer | Catch a faithfulness regression before the PR merges | Zero surprise quality bugs in production |
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

### North Star metric
**Quality-gated deploys per week** — the number of production deploys that passed (or were blocked by) an EvalLake gate. This measures whether quality gates are actually embedded in the release process, not just installed.

### Leading indicators

| Metric | Target (90d) | Rationale |
|--------|-------------|-----------|
| Time-to-first-gate (TTFG) | < 20 minutes from signup | Activation quality |
| Golden set coverage | ≥ 50 test cases per model | Depth of quality baseline |
| Mean time to regression detection | < 15 min from PR open | Speed of signal |
| Gate block rate | 3–8% of PRs | Too low = not meaningful; too high = false positives |
| Dashboard WAU | ≥ 80% of PM + ML org | Product adoption |
| Prompt change attribution | 100% of quality deltas linked to a prompt version | Lineage completeness |

### Guardrails (things we don't break)
- Eval run p95 latency < 5 minutes (if it's slower, engineers bypass it)
- Gate false positive rate < 5% (blocks a PR that was fine)
- Zero PHI persisted in eval payloads (HIPAA/PHIPA compliance)
- Audit trail completeness: 100% of gate overrides logged with approver identity

---

## v1 Scope (6 weeks, 2 engineers + PM)

### In scope
- Eval result ingestion API (`POST /v1/runs`) with schema validation and PII scrubber
- Lakehouse table design (partitioned by date × model)
- RAG metric computation (faithfulness, relevance, context recall)
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
