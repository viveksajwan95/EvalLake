# EvalLake — Continuous LLM Quality Intelligence Platform

**Engineering Assessment Submission — Vivek Sajwan**

> **Streams: AI × Data** — an evals pipeline backed by a lakehouse, surfaced as a quality signal in every release.

---

## What's in this repo

| File | What it is |
|------|-----------|
| [`PRODUCT_BRIEF.md`](./PRODUCT_BRIEF.md) | Problem, personas, JTBD, success metrics, v1 scope |
| [`index.html`](./index.html) | Click-through prototype — open in any browser, no server needed |
| [`BACKLOG.md`](./BACKLOG.md) | 15 GitHub issues across 4 milestones |
| `README.md` | This file |

---

## The opportunity in one paragraph

Most teams building LLM products still treat evaluations as one-off tasks. They run a Python script before a release, review the results, and move on. Those results are rarely stored in a way that makes them useful later. There is no shared baseline, no history of model quality, and no easy way to tell whether a change improved or hurt performance.

When a prompt update reduces accuracy or increases hallucinations, the problem often goes unnoticed until users report it.

EvalLake changes that by treating evaluation results as structured data instead of temporary output. Every evaluation is stored in a lakehouse, creating a complete history of model and prompt performance. From there, teams track quality trends, detect regressions, compare versions, and understand exactly which prompt or model change led to a shift in performance. Engineers see quality checks in their CI/CD pipeline, while product and compliance teams get dashboards and reports that make AI quality easy to understand and act on.

---

## How to view the prototype

Open `index.html` in any modern browser. No server, no dependencies.

**Navigate through:**
1. **Overview** — quality score cards, 30-day trend chart, live alert, failing categories
2. **Eval Runs** — full run history; click any row to drill into the run
3. **Run Detail** — gate decision banner, eval metric cards (Accuracy / ROUGE-L / BERTScore), Hallucination Rate, golden set results per test case, lineage + CI context
4. **Quality Trends** — multi-model chart, prompt change impact table
5. **Regression Gates** — threshold configuration, GitHub Actions integration snippet, gate decision history

Click the ⚠ alert on the Overview to jump straight into a regression investigation.

---

## Why AI × Data, in my own words

I built an eval harness at pareIT from scratch — golden sets, offline metrics, regression gates in CI/CD, online canaries. The pain I felt every day: eval results were ephemeral, quality trends were invisible, and the PM had no way to understand whether the sprint's prompt changes made things better or worse without waiting for the next round of user interviews.

EvalLake is the platform I wish I had. The hard part is not the metrics computation — any engineer can write a script that computes Accuracy or ROUGE-L. The hard part is the lakehouse design that makes historical quality *queryable*, and the product layer that makes it *legible* to non-ML teammates. That intersection is where I have genuine experience and genuine conviction.

---

## Streams

| Stream | Contribution |
|--------|-------------|
| **AI** | Eval harness, industry-standard metrics (Accuracy, ROUGE-L, BERTScore, Hallucination Rate), golden set versioning, regression detection, online canary instrumentation |
| **Data** | Lakehouse ingestion (Delta/Iceberg), partitioned table design, quality trend aggregation, prompt-change lineage, audit trail |
| **Product** | Dashboard, run detail UX, CI gate UX, onboarding flow |
| **Infra** | GitHub Actions integration, gate webhooks, PagerDuty/Slack alerting |

---

## Timebox

~3 hours.
AI usage: Claude (Anthropic) used for draft review and copy editing.
