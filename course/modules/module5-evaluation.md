# Module 5: Evaluating Your Agent System

**Duration**: 1.5-2 hours
**Prerequisites**: Module 4 (Production Wrapper), MLflow running
**Example Code**: Built during exercise (see `exercises/exercise4_evaluation.md`)
**Exercise**: `exercises/exercise4_evaluation.md`

---

## Learning Objectives

- Understand why agent evaluation requires measurement at multiple levels
- Apply the compound reliability formula to predict and diagnose system accuracy
- Build a reusable evaluation harness for the Stardew Vision pipeline
- Track evaluation results in MLflow to enable objective before/after comparison
- Use this harness to assess the impact of future changes (prompts, fine-tuning, model swaps)

---

## Why Evaluate Before You Integrate?

You're about to wire the VLMOrchestrator into a web API (Module 6). Once that happens, failures are user-facing. You need a baseline now - before integration - so that:

1. You know what "working" actually means, numerically
2. Any regression from further changes is immediately visible
3. When you fine-tune later, you can prove it helped (or didn't)

Without this, you're flying blind. "It seems to work" is not a measurement.

This also connects to a broader principle: **you cannot evaluate multi-agent systems purely by watching them run**. The compound reliability problem means individually-correct steps can produce a failing system. You need to measure each level separately.

---

## The Compound Reliability Formula

From production research on agentic systems (Rajan, 2026), applicable equally to single-agent pipelines with multiple steps:

```
overall_accuracy = per_step_accuracy ^ number_of_steps
```

Your Stardew Vision pipeline has three sequential steps:

```
Step 1                    Step 2                    Step 3
[VLM classifies      →   [VLM selects &        →   [Tool extracts
 screen type]             calls correct tool]        fields correctly]
```

If each step runs at 90% accuracy:

```
0.90 × 0.90 × 0.90 = 0.729  →  72.9% end-to-end accuracy
```

Nearly 1 in 4 screenshots returns wrong data - and that's before you've noticed anything is wrong.

```
┌─────────────────────────────────────────────────────────────┐
│              Compound Reliability by Step Count             │
│                                                             │
│  Per-step  │  2 steps  │  3 steps  │  5 steps  │ 10 steps  │
│  ──────────┼───────────┼───────────┼───────────┼────────── │
│  99%       │  98.0%    │  97.0%    │  95.1%    │  90.4%    │
│  95%       │  90.2%    │  85.7%    │  77.4%    │  59.9%    │
│  90%       │  81.0%    │  72.9%    │  59.0%    │  34.9%    │
│  85%       │  72.3%    │  61.4%    │  44.4%    │  19.7%    │
└─────────────────────────────────────────────────────────────┘
```

The formula gives you two tools:
- **Prediction**: given per-step measurements, what should end-to-end accuracy be?
- **Diagnostic**: if observed end-to-end is significantly lower than predicted, there is a compounding interaction between steps to investigate

**Key design principle**: keep your chains short. Every step you add multiplies the failure rate.

---

## What to Measure

Three levels. Measure all three - they diagnose different problems.

### Level 1: Step Accuracy

Does each step succeed in isolation?

| Step | What "success" means | How to measure |
|---|---|---|
| Screen classification | VLM correctly identifies `pierre_shop` vs other types | Inspect agent reasoning trace |
| Tool selection | VLM calls `crop_pierres_detail_panel`, not a wrong tool | Check `agent.logs` or tool call records |
| Field extraction | Tool returns output with all expected fields present | Compare structure against schema |

### Level 2: Field Accuracy

For each extracted field, what fraction of examples return the correct value?

```python
# Example field accuracy breakdown
{
    "name":              0.94,   # item name - high accuracy
    "price_per_unit":    0.87,   # price - occasional OCR-style errors
    "quantity_selected": 0.91,   # quantity - usually correct
    "total_cost":        0.83    # calculated field - most errors here
}
```

This tells you *which fields* to target when writing fine-tuning examples or adjusting prompts. Binary pass/fail at the example level hides this signal.

### Level 3: End-to-End Accuracy

Did the full pipeline return a result that exactly matches ground truth?

This is your headline number. Report it as: "The pipeline returns correct output on X% of screenshots."

### Operational Metrics

| Metric | Why it matters |
|---|---|
| Cost per workflow (tokens) | Catches cost regressions when you change models or add steps |
| Latency (ms) | Catches performance regressions; also a proxy for retry loops |
| Error rate | Unhandled exceptions that bypass the accuracy calculation |

---

## Building Your Test Set

You do not need a large dataset to start. You need a *representative* one.

**Minimum viable test set**: 20-30 labeled examples
- 15-20 normal cases (clear Pierre's shop screenshots, varied items)
- 3-5 edge cases (unusual prices, long item names, discounted items)
- 2-3 failure cases (wrong screen type, blurry image, partially occluded panel)

**Format**: A single JSON file at `tests/fixtures/eval_set.json`

```json
[
  {
    "id": "pierre_001",
    "screenshot": "tests/fixtures/pierre_shop_001.png",
    "screen_type": "pierre_shop",
    "expected_tool": "crop_pierres_detail_panel",
    "expected_output": {
      "name": "Parsnip Seeds",
      "price_per_unit": 20,
      "quantity_selected": 5,
      "total_cost": 100
    }
  },
  {
    "id": "wrong_screen_001",
    "screenshot": "tests/fixtures/inventory_screen_001.png",
    "screen_type": "inventory",
    "expected_tool": null,
    "expected_output": null,
    "expected_behavior": "graceful_rejection"
  }
]
```

**How to collect labels**: Run your pipeline on a screenshot, review the output manually, correct any errors, save to `eval_set.json`. Spend 5 minutes per screenshot. Do this incrementally - add examples as you encounter new failure modes.

**Important**: Set aside 20% of your examples as a held-out test set. Never use those for prompting examples or fine-tuning. They are your honest accuracy estimate.

---

## Code Walkthrough

The following code is what you will build during the exercise. Read through it here first to understand the structure, then implement it yourself.

### Part 1: The Evaluation Harness

```python
"""
Module 5: Evaluation harness for VLMOrchestrator.
Run this before and after any change to measure impact.
"""
import json
import time
import mlflow
from pathlib import Path
from collections import defaultdict
from stardew_vision.models.vlm_wrapper import VLMOrchestrator


def load_eval_set(path: str) -> list[dict]:
    """Load labeled examples from JSON file."""
    with open(path) as f:
        return json.load(f)


def evaluate_pipeline(
    orchestrator: VLMOrchestrator,
    eval_set: list[dict],
) -> dict:
    """
    Run full evaluation across all labeled examples.

    Returns dict with accuracy metrics, field scores, and operational metrics.
    """
    results = {
        "total": len(eval_set),
        "e2e_correct": 0,
        "field_scores": defaultdict(list),
        "latencies_ms": [],
        "errors": []
    }

    for example in eval_set:
        if example["expected_output"] is None:
            continue  # Failure-case examples handled separately

        start = time.time()
        try:
            extracted = orchestrator.analyze_screenshot(example["screenshot"])
            latency_ms = (time.time() - start) * 1000
            results["latencies_ms"].append(latency_ms)

            all_correct = True
            for field, expected in example["expected_output"].items():
                correct = extracted.get(field) == expected
                results["field_scores"][field].append(correct)
                if not correct:
                    all_correct = False

            if all_correct:
                results["e2e_correct"] += 1

        except Exception as e:
            results["errors"].append({"id": example["id"], "error": str(e)})

    n = results["total"] - len(results["errors"])
    results["e2e_accuracy"] = results["e2e_correct"] / n if n > 0 else 0.0
    results["error_rate"] = len(results["errors"]) / results["total"]
    results["avg_latency_ms"] = (
        sum(results["latencies_ms"]) / len(results["latencies_ms"])
        if results["latencies_ms"] else 0.0
    )
    results["field_accuracy"] = {
        field: sum(scores) / len(scores)
        for field, scores in results["field_scores"].items()
    }
    return results
```

### Part 2: Logging to MLflow

```python
def log_to_mlflow(results: dict, run_name: str):
    """Log evaluation results as an MLflow run for comparison."""
    with mlflow.start_run(run_name=run_name):
        mlflow.log_metric("e2e_accuracy", results["e2e_accuracy"])
        mlflow.log_metric("error_rate", results["error_rate"])
        mlflow.log_metric("avg_latency_ms", results["avg_latency_ms"])

        for field, score in results["field_accuracy"].items():
            mlflow.log_metric(f"field_accuracy_{field}", score)

        mlflow.log_dict(results, "full_eval_results.json")

    print(f"End-to-end accuracy: {results['e2e_accuracy']:.1%}")
    print(f"Error rate:          {results['error_rate']:.1%}")
    print(f"Avg latency:         {results['avg_latency_ms']:.0f}ms")
    print()
    print("Field accuracy breakdown:")
    for field, score in results["field_accuracy"].items():
        bar = "#" * int(score * 20)
        print(f"  {field:20s} {score:.1%}  |{bar:<20}|")
```

### Part 3: The Compound Reliability Check

```python
def compound_reliability_check(
    step_accuracies: list[float],
    observed_e2e: float
) -> None:
    """
    Compare predicted end-to-end accuracy against observed.
    A large gap signals a compounding interaction between steps.
    """
    predicted = 1.0
    for acc in step_accuracies:
        predicted *= acc

    gap = abs(predicted - observed_e2e)
    print(f"Predicted e2e (compound formula): {predicted:.1%}")
    print(f"Observed e2e:                     {observed_e2e:.1%}")
    if gap > 0.05:
        print(f"WARNING: {gap:.1%} gap - investigate step interactions")
    else:
        print("Gap within expected range.")
```

---

## Hands-On Checkpoints

### Checkpoint 1: Create your eval set structure

Create `tests/fixtures/eval_set.json` with at least 3 entries (placeholder paths are fine for now - structure matters more than data at this stage).

Verify it loads:
```bash
python -c "
import json
with open('tests/fixtures/eval_set.json') as f:
    data = json.load(f)
print(f'Loaded {len(data)} examples')
print('Fields:', list(data[0].keys()))
"
```

### Checkpoint 2: Run your evaluation harness

With at least a few real labeled screenshots:
```bash
python src/evaluate.py  # or wherever you place your implementation
```

Expected output:
- Per-field accuracy table
- End-to-end accuracy percentage
- MLflow run logged

### Checkpoint 3: Confirm your baseline in MLflow

```bash
mlflow ui --port 5000
```

Open `http://localhost:5000` and verify your eval run appears with all metrics. Label this run something meaningful (e.g., `baseline-pre-fastapi`). This is your reference point for everything that follows.

---

## Key Patterns

**Always run eval before AND after a change.** A single number without a comparison is useless.

**Field accuracy is more useful than pass/fail.** If `total_cost` is consistently wrong but `name` is correct, your problem is arithmetic or formatting, not understanding.

**Start small, grow incrementally.** 20 well-labeled examples beats 200 mediocre ones. Add examples whenever you encounter a new failure mode.

**Predicted vs. observed end-to-end is a diagnostic signal.** A large gap means there is a step interaction causing cascading errors you haven't seen individually.

---

## Common Pitfalls

**Pitfall**: Evaluating only on screenshots the model already handles well.
**Fix**: Include edge cases and failure cases from the start.

**Pitfall**: Using your eval set examples as prompting examples or fine-tuning data.
**Fix**: Keep a held-out portion you never touch for training. That's your honest accuracy estimate.

**Pitfall**: Running eval once and treating it as permanent truth.
**Fix**: Re-run eval every time you change the prompt, model, or tool implementation.

---

## Connection to Fine-Tuning

When you fine-tune Qwen2.5-VL later in the course:

1. Run your eval harness now → **save this MLflow run as your baseline**
2. Collect fine-tuning examples that target your lowest-scoring fields
3. Fine-tune the model
4. Run your eval harness again with the same eval set
5. Compare MLflow runs: did field accuracy improve? Did anything regress?

The eval harness you build in this module is the instrument you will use to measure all future improvements. Build it carefully - it outlives any individual model or prompt.

---

## Pre-Deployment Checklist

Before wiring the orchestrator into FastAPI (Module 6):

```
[ ] eval_set.json created with >= 20 labeled examples
[ ] End-to-end accuracy > 80% on eval set
[ ] All critical fields (name, price_per_unit) > 90% accuracy
[ ] Error rate < 5%
[ ] MLflow baseline run saved and labeled
[ ] Compound reliability prediction matches observed accuracy within 5%
```

If you cannot check all boxes, the issue is in Module 3 or 4. Fix it before integrating.

---

## Key Takeaways

1. End-to-end accuracy is your headline number, but field accuracy tells you where to improve
2. The compound reliability formula predicts system behavior from step-level measurements
3. MLflow run comparison is how you objectively assess whether a change helped
4. Your test set is an asset - maintain and grow it over time
5. Evaluation infrastructure outlives any individual model or prompt - build it once, use it always

---

## Additional Resources

- Exercise: `exercises/exercise4_evaluation.md` - Build the harness from scratch
- Research basis: `resources/TheMulti-Agent Trap _ TowardsDataScience.pdf` - Source of compound reliability math
- Related: `docs/best-practices-2026.md` - Observability patterns
- Next: `module6-fastapi-integration.md` - Integrate with the web app
