# 17.7 A/B Testing and Regression Test Automation

> **Goal**: Master A/B testing and regression testing methods for Agents, build automated prompt-variant comparison workflows, and integrate evaluation into CI/CD.

---

## Why Agents Need A/B Testing

When you modify a prompt, adjust a tool description, change a model version, or add a new tool, how do you know the Agent actually became better?

Traditional software has unit tests. You change code, run tests, and get a clear pass/fail signal. Agent behavior is probabilistic and context-dependent, so "trying it once" is not enough. We need **statistically meaningful A/B testing**.

### Common Change Scenarios

| Change Type | Risk | Value of A/B Testing |
|------------|------|----------------------|
| System prompt change | Improves some scenarios but breaks others | Quantify quality differences across task categories |
| Model upgrade | New model may regress on specific tasks | Compare old and new models on the same dataset |
| Tool description change | May cause tool misuse | Measure tool-selection accuracy |
| Adding a new tool | May interfere with existing tool calls | Detect negative transfer |
| Temperature change | Affects creativity and stability | Balance diversity against reliability |

---

## Core Design of an Agent A/B Test

The core idea is simple: run **two Agent variants** on the **same evaluation set**, then compare their performance using deterministic metrics, judge scores, and statistical tests.

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional
import time


class TestStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"


@dataclass
class TestCase:
    id: str
    query: str
    expected_output: Optional[str] = None
    expected_tools: Optional[list[str]] = None
    category: str = "default"
    difficulty: str = "medium"
    metadata: dict = field(default_factory=dict)


@dataclass
class TestRun:
    case_id: str
    variant: str
    output: str
    tools_used: list[str] = field(default_factory=list)
    score: float = 0.0
    latency_ms: float = 0.0
    token_cost: float = 0.0
    error: str | None = None
```

---

## Running Two Agent Variants

```python
class AgentABRunner:
    def __init__(self, agent_a, agent_b, evaluator):
        self.agent_a = agent_a
        self.agent_b = agent_b
        self.evaluator = evaluator

    def run_case(self, case: TestCase, variant_name: str, agent) -> TestRun:
        start = time.time()
        try:
            result = agent.invoke(case.query)
            latency_ms = (time.time() - start) * 1000
            score = self.evaluator.score(case, result)
            return TestRun(
                case_id=case.id,
                variant=variant_name,
                output=result.get("output", ""),
                tools_used=result.get("tools_used", []),
                score=score,
                latency_ms=latency_ms,
                token_cost=result.get("cost", 0.0),
            )
        except Exception as exc:
            return TestRun(
                case_id=case.id,
                variant=variant_name,
                output="",
                error=str(exc),
            )

    def run_suite(self, cases: list[TestCase]) -> list[TestRun]:
        runs = []
        for case in cases:
            runs.append(self.run_case(case, "A", self.agent_a))
            runs.append(self.run_case(case, "B", self.agent_b))
        return runs
```

The most important rule is: **both variants must see the same inputs under the same evaluation conditions**. Otherwise, the comparison is not meaningful.

---

## Metrics to Compare

A/B tests for Agents should compare more than final answer quality.

| Metric | Meaning |
|-------|---------|
| Success rate | Percentage of tasks completed correctly |
| Average judge score | Mean quality score across cases |
| Tool accuracy | Whether required tools were selected correctly |
| Latency | Time to complete each task |
| Token cost | Cost per task and total suite cost |
| Error rate | Exceptions, invalid tool calls, or failed workflows |
| Regression count | Number of historical cases that got worse |

```python
def summarize_runs(runs: list[TestRun]) -> dict:
    by_variant: dict[str, list[TestRun]] = {}
    for run in runs:
        by_variant.setdefault(run.variant, []).append(run)

    summary = {}
    for variant, items in by_variant.items():
        valid = [item for item in items if item.error is None]
        summary[variant] = {
            "case_count": len(items),
            "success_rate": len(valid) / len(items) if items else 0.0,
            "avg_score": sum(item.score for item in valid) / len(valid) if valid else 0.0,
            "avg_latency_ms": sum(item.latency_ms for item in valid) / len(valid) if valid else 0.0,
            "total_cost": sum(item.token_cost for item in valid),
            "error_count": len(items) - len(valid),
        }
    return summary
```

---

## Statistical Significance

A variant may look better simply because of randomness. Use statistical testing to decide whether the difference is reliable.

```python
from scipy import stats


def paired_t_test(scores_a: list[float], scores_b: list[float]) -> dict:
    """Compare paired scores for two Agent variants."""
    statistic, p_value = stats.ttest_rel(scores_b, scores_a)
    improvement = sum(b - a for a, b in zip(scores_a, scores_b)) / len(scores_a)
    return {
        "mean_improvement": improvement,
        "p_value": p_value,
        "significant": p_value < 0.05,
    }
```

For binary success/failure, use McNemar's test or bootstrap confidence intervals. For judge scores, paired tests are usually better than independent tests because both variants are evaluated on the same tasks.

---

## Regression Test Automation

A/B testing compares two variants. Regression testing ensures new changes do not break known important cases.

A good Agent regression set includes:

- historical bugs;
- high-value user workflows;
- safety and prompt-injection cases;
- tool failure scenarios;
- edge cases with ambiguous or incomplete input.

```python
class RegressionGate:
    def __init__(self, min_score: float = 0.8, max_error_rate: float = 0.02):
        self.min_score = min_score
        self.max_error_rate = max_error_rate

    def check(self, summary: dict) -> tuple[bool, list[str]]:
        reasons = []
        if summary["avg_score"] < self.min_score:
            reasons.append(f"Average score below threshold: {summary['avg_score']:.3f}")
        if summary["error_count"] / max(summary["case_count"], 1) > self.max_error_rate:
            reasons.append("Error rate exceeds threshold")
        return len(reasons) == 0, reasons
```

---

## CI/CD Integration

A practical CI workflow should run different evaluation depths for different situations:

| Trigger | Evaluation Depth | Purpose |
|--------|------------------|---------|
| Pull request | Small regression set | Fast feedback and release blocking |
| Nightly job | Full evaluation set | Detect broader regressions |
| Model upgrade | Full A/B test | Compare old and new model behavior |
| Prompt release | Category-level A/B test | Validate task-specific improvements |

```yaml
name: Agent Regression Evaluation
on: [pull_request]

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Run Agent regression tests
        run: python scripts/run_agent_eval.py --suite regression --fail-on-regression
```

---

## Report Structure

A useful A/B report should answer three questions:

1. **Is B better than A?** Include overall metrics and statistical significance.
2. **Where is B better or worse?** Break down results by category and difficulty.
3. **Is B worth the cost?** Compare quality gains against latency and token cost.

```text
A/B Test Report
├── Overall summary
├── Statistical significance
├── Category-level breakdown
├── Regression cases
├── Cost and latency comparison
└── Recommended decision: ship / hold / investigate
```

---

## Best Practices

- Use paired evaluation: both variants should run on the same cases.
- Keep evaluation datasets versioned.
- Separate deterministic checks from LLM/Agent judge checks.
- Track cost and latency alongside quality.
- Use category-level analysis instead of only a single average score.
- Add every production failure to the regression suite.
- Avoid shipping based on small, non-significant improvements.

---

## Chapter Takeaways

- Agent changes require A/B testing because behavior is probabilistic and context-dependent.
- Compare two variants on the same evaluation set and track quality, cost, latency, and errors.
- Statistical significance prevents shipping improvements caused by noise.
- Regression suites protect historical fixes and high-value workflows.
- CI/CD should include fast PR gates, deeper nightly evaluations, and full tests for model or prompt upgrades.

---

*Previous: [17.6 Agent-Specific Evaluation Frameworks](./06_agent_evaluation.md)*  
*Next: [17.8 Model Routing Evaluation](./08_model_routing.md)*