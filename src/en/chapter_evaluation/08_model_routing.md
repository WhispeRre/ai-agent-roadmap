# 17.8 Model Routing Evaluation

> **Goal**: Understand the core problem of model routing, master cost-quality trade-off analysis, and implement evaluation methods for intelligent routers in multi-model Agent systems.

---

## Why Model Routing Matters

Not every Agent task requires the strongest model. Simple questions can often be handled by a small model, while complex reasoning, planning, and multi-tool tasks may require a larger model.

**Model routing** dynamically chooses the most suitable model for each task, balancing cost, latency, and quality.

### The Cost Reality

| Model | Input Price / 1M tokens | Output Price / 1M tokens | Reasoning Ability | Speed |
|------|--------------------------|---------------------------|-------------------|-------|
| gpt-4.1 | $2.00 | $8.00 | Strong | Medium |
| gpt-4.1-mini | $0.40 | $1.60 | Medium | Fast |
| gpt-4.1-nano | $0.10 | $0.40 | Basic | Very fast |

If an Agent handles 10,000 requests per day:

- **All on gpt-4.1**: about $100/day, or $3,000/month.
- **Intelligent routing, 70% small model + 30% large model**: about $40/day, or $1,200/month.
- **Savings**: about $1,800/month, or $21,600/year.

> 💡 **Key insight**: In production, most requests are simple tasks such as FAQ, format conversion, and information extraction. Large models should be reserved for the tasks that truly need them.

---

## When to Use Large vs Small Models

```text
Incoming task
    │
    ├─ Task classification
    │   ├── Simple: facts, format conversion, extraction → small model
    │   ├── Medium: short reasoning, simple tools → medium model
    │   └── Complex: planning, creativity, constraints → large model
    │
    ├─ Risk assessment
    │   ├── Low risk → small model acceptable
    │   └── High risk → prefer stronger model or human review
    │
    └─ Budget constraints
        ├── Flexible → optimize for quality
        └── Tight → optimize for cost with fallback
```

### Task Complexity Criteria

| Dimension | Simple | Medium | Complex |
|----------|--------|--------|---------|
| Reasoning steps | 1 | 2–3 | 4+ |
| Tool calls | None | 1–2 | 3+ |
| Input length | < 500 tokens | 500–2000 tokens | 2000+ tokens |
| Output format | Fixed | Semi-structured | Open-ended |
| Error tolerance | High | Medium | Low |
| Examples | Intent classification, keyword extraction | RAG QA, simple tool use | Complex planning, multi-turn workflows |

---

## Cost-Quality Trade-Off Analysis

A model router should not only minimize cost. It should maximize expected value:

```python
from dataclasses import dataclass


@dataclass
class ModelProfile:
    name: str
    input_cost_per_1m: float
    output_cost_per_1m: float
    avg_latency_ms: float
    quality_score: float


def estimate_request_cost(model: ModelProfile, input_tokens: int, output_tokens: int) -> float:
    return (
        input_tokens / 1_000_000 * model.input_cost_per_1m
        + output_tokens / 1_000_000 * model.output_cost_per_1m
    )


def cost_quality_ratio(model: ModelProfile, input_tokens: int, output_tokens: int) -> float:
    cost = estimate_request_cost(model, input_tokens, output_tokens)
    return model.quality_score / max(cost, 1e-9)
```

A cheap model is not always better. If it fails often and requires retries or human correction, its true cost may be higher than expected.

---

## A Simple Rule-Based Router

A rule-based router is easy to understand and often a good starting point.

```python
class RuleBasedModelRouter:
    def route(self, query: str, metadata: dict) -> str:
        input_tokens = metadata.get("input_tokens", len(query.split()) * 1.3)
        risk_level = metadata.get("risk_level", "low")
        requires_tools = metadata.get("requires_tools", False)
        user_tier = metadata.get("user_tier", "standard")

        if risk_level == "high":
            return "gpt-4.1"
        if input_tokens > 2000:
            return "gpt-4.1"
        if requires_tools:
            return "gpt-4.1-mini"
        if user_tier == "premium":
            return "gpt-4.1-mini"
        return "gpt-4.1-nano"
```

Rule-based routing is transparent, but it can be too rigid. As traffic grows, collect logs and train a classifier to predict the cheapest model that can still solve the task.

---

## Evaluation Dataset for Routing

To evaluate a router, build a dataset where each task includes:

```python
@dataclass
class RoutingEvalCase:
    id: str
    query: str
    category: str
    difficulty: str
    risk_level: str
    expected_min_model: str
    input_tokens: int
    expected_output_tokens: int
    quality_threshold: float = 0.8
```

The key label is not always "which model is best". More often, it is **the minimum model that meets the quality threshold**.

---

## Router Evaluation Metrics

| Metric | Meaning |
|-------|---------|
| Quality pass rate | Percentage of tasks whose answer meets the threshold |
| Average cost | Mean cost per request |
| Average latency | Mean response time |
| Over-routing rate | Percentage of tasks sent to a model stronger than necessary |
| Under-routing rate | Percentage of tasks sent to a model too weak to pass |
| Fallback rate | Percentage of tasks requiring retry on a stronger model |

```python
def evaluate_router(cases: list[RoutingEvalCase], router, model_runner) -> dict:
    total_cost = 0.0
    pass_count = 0
    under_routed = 0
    over_routed = 0

    for case in cases:
        model_name = router.route(case.query, case.__dict__)
        result = model_runner.run(model_name, case.query)
        total_cost += result.cost

        if result.quality_score >= case.quality_threshold:
            pass_count += 1
        else:
            under_routed += 1

        if model_rank(model_name) > model_rank(case.expected_min_model):
            over_routed += 1

    total = len(cases)
    return {
        "quality_pass_rate": pass_count / total if total else 0.0,
        "avg_cost": total_cost / total if total else 0.0,
        "under_routing_rate": under_routed / total if total else 0.0,
        "over_routing_rate": over_routed / total if total else 0.0,
    }


def model_rank(model_name: str) -> int:
    ranks = {"gpt-4.1-nano": 1, "gpt-4.1-mini": 2, "gpt-4.1": 3}
    return ranks[model_name]
```

---

## Fallback Routing

A robust router should support fallback. If a small model is uncertain or fails, retry with a stronger model.

```python
class FallbackRouter:
    def __init__(self, primary_router, confidence_threshold: float = 0.7):
        self.primary_router = primary_router
        self.confidence_threshold = confidence_threshold

    def run(self, query: str, metadata: dict, model_runner):
        model_name = self.primary_router.route(query, metadata)
        result = model_runner.run(model_name, query)

        if result.confidence < self.confidence_threshold or result.error:
            stronger = self._next_stronger_model(model_name)
            if stronger:
                fallback = model_runner.run(stronger, query)
                fallback.metadata["fallback_from"] = model_name
                return fallback

        return result

    def _next_stronger_model(self, model_name: str) -> str | None:
        order = ["gpt-4.1-nano", "gpt-4.1-mini", "gpt-4.1"]
        index = order.index(model_name)
        return order[index + 1] if index + 1 < len(order) else None
```

Fallback increases cost but improves reliability. Track fallback rate carefully: a high fallback rate means the primary routing policy is too aggressive.

---

## Production Monitoring

Model routing should be monitored continuously:

- cost per category;
- latency by selected model;
- quality pass rate by model and task type;
- fallback rate;
- user escalation or complaint rate;
- drift in task distribution.

```text
Routing Dashboard
├── Traffic distribution by model
├── Cost trend by category
├── Quality pass rate
├── Fallback rate
├── Over-routing / under-routing estimates
└── Top failed routing patterns
```

---

## Best Practices

- Start with transparent rules before training a learned router.
- Route by task complexity, risk, input length, tool requirements, and user tier.
- Evaluate the router by both quality and cost.
- Track under-routing more seriously than over-routing; quality failures are often more expensive than extra tokens.
- Use fallback for uncertain cases.
- Re-evaluate routing rules whenever models, prices, or task distributions change.

---

## Chapter Takeaways

- Model routing chooses the cheapest model that can still meet the quality requirement.
- The main trade-off is not just cost vs quality, but cost, quality, latency, and risk.
- Rule-based routers are a good starting point; learned routers can improve with traffic data.
- Evaluation should measure pass rate, cost, latency, over-routing, under-routing, and fallback rate.
- Production routing needs continuous monitoring because model prices, capabilities, and user tasks change over time.

---

*Previous: [17.7 A/B Testing and Regression Test Automation](./07_ab_testing.md)*