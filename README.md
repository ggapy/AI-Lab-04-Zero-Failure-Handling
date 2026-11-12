# AI Lab 04 — Zero-Failure Handling in n8n

**A controlled, reproducible simulation of a customer-message classification evaluation pipeline that continues to generate a report when there are no API failures.**

> **SIMULATION ONLY:** This workflow makes **no OpenAI API calls**. The 15 predictions are copied from the expected labels, and the zero-failure outcome is intentionally simulated. The reported 100% accuracy is **not** evidence of real model accuracy or API reliability.

## Overview

AI Lab 03 evaluated a partial-failure scenario: 12 completed predictions and three simulated API failures. It exposed an important workflow-design issue: when a failure branch returns **zero items**, n8n may stop processing that branch, leaving a downstream Merge node without an input and preventing final reporting.

**AI Lab 04** addresses that empty-branch case while keeping evaluation accuracy, processing coverage, and recorded failures separate.

The workflow is built with **n8n Code nodes (JavaScript)**, a **Merge** node, and a **Convert to File** node. It uses a small, synthetic dataset of 15 customer messages classified into **Billing, Sales, Support, and Other**.

## The problem and the fix

| Behavior | Before the fix | After the fix |
|---|---|---|
| Zero-error simulation returns | Zero items (`[]`) | Zero items (`[]`) |
| Downstream failure branch | Stops by default | Continues with `Always Output Data` |
| Empty output interpretation | No summary produced | `Capture API Failures` emits a `no_api_failures` control marker |
| Failure summary | Missing | One summary item reporting **0** failures |
| Final report | Not generated | Generated successfully |

The fix has three parts:

1. **Simulate Zero API Failures** returns `[]` but has **Settings → Always Output Data** enabled. n8n supplies a placeholder output item.
2. **Capture API Failures** distinguishes genuine error records from the empty placeholder. In the absence of errors, it emits `{ "event_type": "no_api_failures" }`, which is **not** counted as a failure.
3. **API Failure Summary** ignores the control marker and emits **one** structured summary item with zero counts and empty lists. That allows **Merge Evaluation Results** to combine it with the classification evaluation branch.

## Workflow architecture

```text
Trigger
  └─ 15 Test Cases
      ├─ Simulate All Predictions
      │    └─ Evaluate Predictions
      │         └─ Category Metrics
      │              └─ Confusion Matrix
      │                   └─ Evaluation Report
      │                        └─ Evaluation Coverage ───────┐
      │                                                     │ (Input 2)
      └─ Simulate Zero API Failures                          │
           └─ Capture API Failures                           │
                └─ API Failure Summary ───────┐              │
                                              ▼              ▼
                                       Merge Evaluation Results
                                                │
                                       Unified Evaluation Report
                                                │
                                       Export Evaluation Report
```

The Merge node uses **Combine → By Position**. Each branch supplies **one summary item** in this tested scenario.

## Verified simulation results

The lab was executed in n8n and its exported Markdown report was inspected on **November 12, 2026**.

| Measurement | Observed result |
|---|---:|
| Expected test cases | 15 |
| Completed simulated predictions | 15 |
| Correct simulated predictions | 15 |
| Accuracy among completed predictions | **100%** |
| Evaluation coverage | **100%** |
| Missing predictions | **0** |
| Simulated API failures | **0** |
| Failures matching missing predictions | **0** |
| Unexplained missing predictions | **0** |
| Coverage status | `complete` |

All perfect classification scores are **by construction**: simulated predictions reuse the known expected labels. What this lab actually tests is **control flow, failure-summary handling, reconciliation, and report generation**.

## Files in this repository

| File | Purpose |
|---|---|
| [`AI_Lab_04_Zero_Failure_Handling_Public.json`](AI_Lab_04_Zero_Failure_Handling_Public.json) | Sanitized n8n workflow with 14 connected nodes; no OpenAI node or credential reference |
| [`AI_Lab_04_Zero_Failure_Simulation_2026-11-12.md`](AI_Lab_04_Zero_Failure_Simulation_2026-11-12.md) | Exported Markdown evaluation report from the tested run |
| `README.md` | Project explanation, architecture, results, and reproduction steps |

## How to reproduce

1. Import [`AI_Lab_04_Zero_Failure_Handling_Public.json`](AI_Lab_04_Zero_Failure_Handling_Public.json) into a **test** n8n environment.
2. Review all JavaScript Code nodes and the node connections before execution. The public JSON contains no OpenAI node or API credentials.
3. Verify that **Simulate Zero API Failures** has **Always Output Data** enabled and returns `[]`.
4. Run the manual **Trigger** using **Execute workflow**.
5. Inspect the **API Failure Summary** output: `total_api_failures: 0`, `affected_test_ids: []`, and `failure_details: []`.
6. Inspect **Unified Evaluation Report**: `expected_test_cases: 15`, `completed_predictions: 15`, `evaluation_coverage_percent: 100`, `reconciled_api_failures: 0`, and `unexplained_missing_predictions: 0`.
7. Open **Export Evaluation Report** and download its binary Markdown output.

The public workflow structure has been inspected, but this sanitized copy has **not** been independently re-imported and rerun; the documented results come from the original Lab 04 execution.

## What this lab demonstrates

- Handling a legitimate **zero-event** case without blocking the rest of a workflow.
- Separating **control markers** from actual error records.
- Producing an explicit **zero-count summary** so a two-branch Merge can complete.
- Tracking **classification accuracy** separately from **evaluation coverage**.
- Reconciling missing predictions with recorded failures before producing a report.
- Exporting a readable Markdown record for review and audit.

## Limitations and next steps

- This is a **zero-failure simulation**; it does not measure real service reliability, latency, throughput, retry behavior, or model quality.
- The dataset is small, manually designed, and not representative of production traffic.
- The public workflow deliberately omits the disconnected OpenAI node and environment-specific credential references/identifiers.
- The output still inherits some labels from Lab 03 (such as its report's workflow description). These labels are documentation artifacts, not evidence of a model invocation.
- A future regression test should use a **single generalized workflow** to run both zero-failure and partial-failure scenarios, including malformed error records and mixed outcomes.
- Production implementation should use real API responses, verified item-to-test-case mappings, independently labeled data, and monitoring/alerting.

## Related project

[AI Lab 03 — Classification Evaluation and Failure Simulation](https://github.com/ggapy/AI-Lab-03-Classification-Evaluation) tests the complementary case with **three simulated failures**, **12 completed predictions**, and **80% coverage**.

---

**Status:** Completed controlled zero-failure simulation. **Do not interpret the 100% simulated accuracy or coverage as live-model accuracy or production reliability.**
