# Exercise 4: Build the Evaluation Harness

**Objective**: Build a reusable evaluation harness for the VLMOrchestrator and establish a baseline MLflow run.

**Time**: 45-60 minutes

**Prerequisites**: Completed Modules 1-4, MLflow installed

---

## Overview

In Module 5 you read through the evaluation harness code. Now you will build it yourself. By the end of this exercise you will have:

- A labeled eval set in `tests/fixtures/eval_set.json`
- An `evaluate_pipeline()` function that measures accuracy at multiple levels
- A baseline MLflow run you can compare all future changes against

---

## Part 1: Create Your Eval Set

Create `tests/fixtures/eval_set.json` with labeled examples.

If you do not yet have real screenshots, use placeholder paths and fill in realistic expected values - the harness structure matters more than live data at this stage. Add real examples as you collect them.

### Your Task

Create the file with this structure. Include at least:
- 3 normal Pierre's shop examples
- 1 edge case (unusual price, long item name, or discounted item)
- 1 failure case (wrong screen type, expected_output is null)

```json
[
  {
    "id": "pierre_001",
    "screenshot": "tests/fixtures/pierre_shop_001.png",
    "screen_type": "pierre_shop",
    "expected_tool": "crop_pierres_detail_panel",
    "expected_output": {
      "name": "TODO: real item name",
      "price_per_unit": 0,
      "quantity_selected": 0,
      "total_cost": 0
    }
  }
]
```

### Verify

```bash
python -c "
import json
with open('tests/fixtures/eval_set.json') as f:
    data = json.load(f)
print(f'Loaded {len(data)} examples')
for ex in data:
    print(f'  {ex[\"id\"]}: expected_output={ex[\"expected_output\"] is not None}')
"
```

---

## Part 2: Implement `evaluate_pipeline()`

Create a file at `src/stardew_vision/eval/harness.py` (or wherever fits your project layout).

### Your Task

Implement the following functions. Use the code walkthrough in Module 5 as a reference, but type it yourself rather than copying - the act of writing it builds the understanding.

```python
"""Evaluation harness for VLMOrchestrator."""
import json
import time
from collections import defaultdict
from stardew_vision.models.vlm_wrapper import VLMOrchestrator


def load_eval_set(path: str) -> list[dict]:
    """Load labeled examples from JSON file."""
    # TODO: implement


def evaluate_pipeline(
    orchestrator: VLMOrchestrator,
    eval_set: list[dict],
) -> dict:
    """
    Run evaluation across all examples with expected_output.

    Returns:
        {
            "total": int,
            "e2e_correct": int,
            "e2e_accuracy": float,
            "error_rate": float,
            "avg_latency_ms": float,
            "field_accuracy": {field_name: float, ...},
            "errors": [{"id": str, "error": str}, ...]
        }
    """
    # TODO: implement


def print_results(results: dict) -> None:
    """Print a human-readable accuracy report."""
    # TODO: implement - include a field accuracy bar chart
```

### Hints

- Iterate over `eval_set`, skip examples where `expected_output is None`
- For each example, call `orchestrator.analyze_screenshot()` inside a try/except
- Track per-field correctness with `defaultdict(list)` - append `True` or `False` for each field
- Compute `e2e_correct` only when all fields match
- Compute averages at the end, not inside the loop

### Questions to answer as you implement

1. What happens if `extracted.get(field)` returns a float like `20.0` but expected is the int `20`? How will you handle type mismatches?

2. If an example raises an exception, should it count as `e2e_incorrect` or be excluded from the denominator? Why does this matter for your accuracy numbers?

<details>
<summary>Hints for the questions</summary>

1. You may need to normalize types before comparing - e.g., `int(extracted.get(field, 0)) == int(expected)` for numeric fields. Or store both as strings. Decide on a convention and document it.

2. Errors should be tracked separately and excluded from the denominator. If you include them as incorrect, your accuracy conflates "wrong answer" with "crashed" - these have different causes and different fixes.

</details>

---

## Part 3: Wire in MLflow

Add a function that logs an evaluation run to MLflow.

### Your Task

```python
import mlflow

def log_to_mlflow(results: dict, run_name: str) -> None:
    """
    Log evaluation results as an MLflow run.

    Logs:
    - e2e_accuracy, error_rate, avg_latency_ms as metrics
    - field_accuracy_{field} for each field as metrics
    - full results dict as artifact
    """
    # TODO: implement
```

Then write a `__main__` block that:
1. Loads the eval set from `tests/fixtures/eval_set.json`
2. Creates a `VLMOrchestrator` (with `enable_mlflow=False` to avoid nested runs)
3. Runs `evaluate_pipeline()`
4. Calls `print_results()`
5. Calls `log_to_mlflow()` with a run name you provide as a CLI argument

```python
if __name__ == "__main__":
    import sys
    run_name = sys.argv[1] if len(sys.argv) > 1 else "eval"
    # TODO: implement the main block
```

### Verify

```bash
# Run with a named baseline
python src/stardew_vision/eval/harness.py baseline-pre-fastapi

# Check it appears in MLflow
mlflow ui --port 5000
```

Expected: a run named `baseline-pre-fastapi` with e2e_accuracy, error_rate, avg_latency_ms, and field_accuracy_* metrics all logged.

---

## Part 4: The Compound Reliability Check

Add one more function:

```python
def compound_reliability_check(
    step_accuracies: list[float],
    observed_e2e: float
) -> None:
    """
    Compare predicted end-to-end accuracy (compound formula) against observed.
    Print a warning if the gap exceeds 5%.
    """
    # TODO: implement
```

Call it with placeholder step accuracies for now (you will measure these properly in later modules):

```python
# Placeholder - replace with real measurements as you collect them
compound_reliability_check(
    step_accuracies=[0.97, 0.93, 0.89],  # screen classify, tool select, field extract
    observed_e2e=results["e2e_accuracy"]
)
```

---

## Verification Checklist

```
[ ] eval_set.json exists with >= 5 entries (3 normal, 1 edge, 1 failure)
[ ] evaluate_pipeline() runs without errors
[ ] print_results() shows per-field accuracy breakdown
[ ] MLflow run logged and visible in mlflow ui
[ ] compound_reliability_check() prints predicted vs observed
[ ] Run is labeled "baseline-pre-fastapi" (or similar) for future reference
```

---

## Reflection Questions

1. Which field has the lowest accuracy in your results? What's the most likely cause?

2. If you change nothing and run the eval again, do you get exactly the same numbers? If not, why not - and does that matter?

3. The compound reliability formula assumes steps are independent. In your pipeline, are they truly independent? What happens if step 1 fails - does it affect step 2?

---

## Next Steps

- Keep `eval_set.json` as a living document - add examples as you encounter failures
- Run the harness before every significant change (prompt edits, model swaps, fine-tuning)
- When you reach fine-tuning, your baseline run here is what you compare against
- Continue to Module 6: FastAPI Integration
