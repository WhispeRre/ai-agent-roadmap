# 17.6 Agent-Specific Evaluation Frameworks

> **Goal**: Master frontier methods for evaluating Agents, including the Agent-as-Judge paradigm, trajectory evaluation, task benchmarks such as τ-bench / OSWorld / SWE-bench, and production-grade Agent metrics.

---

## From LLM-as-Judge to Agent-as-Judge

In earlier evaluation chapters, we introduced **LLM-as-Judge**: using one LLM to score the output quality of another model. But Agents are different from ordinary chat models. An Agent calls tools, performs multi-step operations, interacts with an environment, and may recover from failures.

Therefore, evaluating only the final answer is not enough. We need to evaluate the **entire behavior trajectory**.

This is the core idea of **Agent-as-Judge**: use an Agent, not just a single LLM call, to inspect and evaluate another Agent's full execution process.

### LLM-as-Judge vs Agent-as-Judge

| Dimension | LLM-as-Judge | Agent-as-Judge |
|----------|--------------|----------------|
| Evaluation target | Single text output | Full execution trajectory |
| Evaluation method | One-shot scoring | Step-by-step review + verification |
| Context | Input and final answer | Tool calls, intermediate states, recovery behavior |
| Depth | Semantic quality | Decision quality + execution efficiency + error handling |
| Cost | Lower | Higher |
| Consistency | Usually higher | More complex, requires calibration |

```python
from dataclasses import dataclass, field
from enum import Enum


class TrajectoryAspect(Enum):
    """Aspects for evaluating an Agent trajectory."""
    GOAL_ACHIEVEMENT = "goal_achievement"
    TOOL_SELECTION = "tool_selection"
    TOOL_USAGE = "tool_usage"
    ERROR_RECOVERY = "error_recovery"
    EFFICIENCY = "efficiency"
    REASONING_QUALITY = "reasoning_quality"


@dataclass
class AgentTrace:
    """A complete Agent execution trace."""
    task_id: str
    user_query: str
    steps: list[dict] = field(default_factory=list)
    final_output: str = ""
    success: bool = False
    total_tokens: int = 0
    total_time: float = 0.0


@dataclass
class TraceEvaluation:
    """Evaluation result for one trajectory aspect."""
    task_id: str
    aspect: TrajectoryAspect
    score: float
    reasoning: str
    evidence: list[str] = field(default_factory=list)
```

---

## Agent-as-Judge Methodology

A complete Agent-as-Judge workflow has three stages:

1. **Trace collection**: record every step of the evaluated Agent.
2. **Step-by-step review**: inspect tool choices, parameters, observations, and recovery behavior.
3. **Holistic judgment**: aggregate aspect scores into an overall evaluation.

```python
class AgentTrajectoryJudge:
    def __init__(self, llm):
        self.llm = llm

    def evaluate_trace(self, trace: AgentTrace) -> list[TraceEvaluation]:
        results = []
        for aspect in TrajectoryAspect:
            results.append(self._evaluate_aspect(trace, aspect))
        return results

    def _evaluate_aspect(self, trace: AgentTrace, aspect: TrajectoryAspect) -> TraceEvaluation:
        prompt = f"""
You are an expert Agent evaluator.

Task:
{trace.user_query}

Final output:
{trace.final_output}

Execution trace:
{trace.steps}

Evaluate the following aspect: {aspect.value}
Return a score from 0.0 to 1.0, reasoning, and concrete evidence from the trace.
"""
        response = self.llm.invoke(prompt)
        parsed = self._parse_response(response)
        return TraceEvaluation(
            task_id=trace.task_id,
            aspect=aspect,
            score=parsed["score"],
            reasoning=parsed["reasoning"],
            evidence=parsed.get("evidence", []),
        )

    def _parse_response(self, response) -> dict:
        # In production, enforce JSON schema or structured output.
        return response
```

The key is **evidence-based scoring**. A good judge should not simply say "the Agent did well". It should point to specific steps, tool calls, observations, and mistakes.

---

## Trajectory-Level Metrics

Agent evaluation should combine final-result metrics with process metrics.

| Metric | Question Answered | Example |
|--------|-------------------|---------|
| Goal completion | Did the Agent solve the user task? | Final task success rate |
| Tool accuracy | Did it choose the right tools? | Correct tool selection rate |
| Parameter correctness | Were tool arguments valid? | SQL query / API parameter accuracy |
| Recovery ability | Did it recover from failures? | Success after tool error |
| Efficiency | Did it waste steps or tokens? | Average steps per successful task |
| Safety | Did it violate policies? | Unsafe action count |

```python
def summarize_agent_metrics(traces: list[AgentTrace]) -> dict:
    total = len(traces)
    successful = [trace for trace in traces if trace.success]
    return {
        "success_rate": len(successful) / total if total else 0.0,
        "avg_steps": sum(len(trace.steps) for trace in traces) / total if total else 0.0,
        "avg_tokens": sum(trace.total_tokens for trace in traces) / total if total else 0.0,
        "avg_time": sum(trace.total_time for trace in traces) / total if total else 0.0,
    }
```

---

## Representative Agent Benchmarks

### τ-bench

τ-bench evaluates tool-using Agents in realistic business workflows such as airline booking and retail customer service. It emphasizes **policy compliance**, multi-turn interaction, and correct tool use under constraints.

### OSWorld

OSWorld evaluates Agents that operate real desktop environments. It is especially useful for Computer Use and GUI Agents because it tests whether the Agent can complete tasks by interacting with an operating system.

### SWE-bench

SWE-bench evaluates coding Agents on real GitHub issues. It checks whether an Agent can understand a bug report, edit code, run tests, and produce a patch that actually fixes the issue.

### GAIA

GAIA evaluates general-purpose assistants on tasks that require reasoning, tool use, web search, and multimodal understanding. It is useful for measuring broad Agent capability.

| Benchmark | Focus | Best For |
|----------|-------|----------|
| τ-bench | Tool use in business workflows | Customer-service and enterprise Agents |
| OSWorld | Desktop GUI operation | Computer Use Agents |
| SWE-bench | Real software engineering tasks | Coding Agents |
| GAIA | General multimodal tool use | General-purpose Agents |

---

## Building an Evaluation Dataset

A good Agent evaluation set should cover more than happy paths.

```python
@dataclass
class AgentEvalCase:
    id: str
    user_query: str
    expected_outcome: str
    required_tools: list[str] = field(default_factory=list)
    forbidden_tools: list[str] = field(default_factory=list)
    category: str = "default"
    difficulty: str = "medium"
    risk_level: str = "low"
    metadata: dict = field(default_factory=dict)
```

Recommended categories:

- **Normal tasks**: common workflows the Agent should complete reliably.
- **Edge cases**: ambiguous inputs, missing information, unusual formats.
- **Tool errors**: APIs fail, return empty results, or produce invalid data.
- **Safety cases**: prompt injection, dangerous commands, sensitive data requests.
- **Regression cases**: historical bugs that must never reappear.

---

## Production Evaluation Pipeline

A production Agent evaluation pipeline usually looks like this:

```text
Evaluation Dataset
      ↓
Run Agent Variants
      ↓
Collect Traces
      ↓
Compute Deterministic Metrics
      ↓
Run Agent-as-Judge / LLM-as-Judge
      ↓
Aggregate Scores and Compare Baselines
      ↓
Block Release or Generate Report
```

Key engineering recommendations:

- Store raw traces, not just scores.
- Version the evaluation dataset together with prompts and tool definitions.
- Separate deterministic checks from judge-based checks.
- Calibrate judge prompts against human-labeled samples.
- Track both quality and cost metrics.

---

## Chapter Takeaways

- Agent evaluation must inspect the full trajectory, not only the final answer.
- Agent-as-Judge evaluates tool selection, tool usage, recovery, efficiency, and reasoning quality.
- Benchmarks such as τ-bench, OSWorld, SWE-bench, and GAIA test different Agent capabilities.
- A production evaluation set should include normal cases, edge cases, tool failures, safety cases, and regressions.
- Reliable evaluation requires trace collection, deterministic metrics, judge calibration, and release gating.

---

*Previous: [17.5 Evaluation Best Practices](./05_observability.md)*  
*Next: [17.7 A/B Testing and Regression Test Automation](./07_ab_testing.md)*