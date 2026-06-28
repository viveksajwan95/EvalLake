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

##  Opportunity

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

The idea for EvalLake came directly from my experience at pareIT.

I built our LLM evaluation framework from the ground up, including golden datasets, offline evaluation metrics, regression gates in CI/CD, and online canary testing. While the evaluation pipeline worked, I kept running into the same problem. Every evaluation was a snapshot in time. Results lived in scripts or logs, disappeared after each run, and gave us no clear picture of how quality was changing over time.

As a product manager, I wanted to know whether the prompt changes we made during a sprint improved the product or introduced new issues. There was no simple way to answer that question. We often had to wait for user interviews or production feedback before we understood the impact of a release.

That's why I started building EvalLake. The challenge isn't calculating metrics like Accuracy or ROUGE-L. Those are straightforward. The real challenge is creating a system that stores every evaluation in a way that makes quality measurable over time, easy to compare across models and prompts, and accessible to everyone involved in shipping AI products.

EvalLake brings those pieces together. It gives engineering, product, and compliance teams a shared view of AI quality, making evaluation data part of the release process instead of an isolated engineering task. That intersection of AI evaluation, data infrastructure, and product experience is where I've spent my time, and it's the problem I care most about solving.

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
