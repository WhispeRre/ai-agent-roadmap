# 5.6 Plan-and-Execute and Test-Time Compute Scaling

> **Goal**: Master the architecture and implementation of the Plan-and-Execute pattern, and understand how test-time compute scaling changes Agent reasoning.

---

## From ReAct to Plan-and-Execute

The ReAct pattern lets an Agent "think while acting": each step mixes reasoning, action, and observation. This is flexible, but complex tasks expose two problems:

1. **Short-sightedness**: the Agent focuses on the next action and may lose the global goal.
2. **Context growth**: every step adds thought, action, and observation to the context.

**Plan-and-Execute** decouples planning from execution:

```text
ReAct:
Think 1 → Act 1 → Observe 1 → Think 2 → Act 2 → Observe 2 → ...

Plan-and-Execute:
Planner creates a complete plan → Executor runs steps → Replan when reality diverges
```

This pattern is especially useful for long tasks, tool-heavy workflows, and production Agents that need auditability.

---

## Core Architecture

Plan-and-Execute usually has two roles:

- **Planner**: decomposes the task into ordered steps with expected outcomes.
- **Executor**: performs each step, records results, and reports failures.

```python
from openai import OpenAI
import json

client = OpenAI()


class PlanAndExecuteAgent:
    def __init__(self, model: str = "gpt-4.1", max_replans: int = 3):
        self.model = model
        self.max_replans = max_replans

    def run(self, task: str) -> str:
        plan = self._plan(task)
        executed_steps = []
        replan_count = 0

        step_index = 0
        while step_index < len(plan):
            step = plan[step_index]
            result = self._execute_step(step, executed_steps)
            executed_steps.append({"step": step, "result": result})

            if not result.get("success", False):
                if replan_count >= self.max_replans:
                    return self._finalize(task, executed_steps)
                plan = self._replan(task, plan, executed_steps)
                replan_count += 1
                step_index = 0
                continue

            step_index += 1

        return self._finalize(task, executed_steps)
```

---

## Planning Prompt

A good planner should produce structured, executable steps:

```python
def build_planning_prompt(task: str) -> str:
    return f"""
You are a task planner for an AI Agent.

Task:
{task}

Create a concise plan. Each step must include:
- description
- required tool or capability
- expected result
- risk level

Return JSON:
[
  {{"description": "...", "tool": "...", "expected_result": "...", "risk": "low|medium|high"}}
]
"""
```

Structured plans are easier to inspect, execute, revise, and test.

---

## Replanning

Replanning is what makes Plan-and-Execute robust. The Agent should replan when:

- a tool fails;
- an assumption is proven wrong;
- new information changes the task;
- the plan violates a constraint;
- the user changes the goal.

```python
class PlanAndExecuteAgent(PlanAndExecuteAgent):
    def _replan(self, task: str, old_plan: list[dict], executed_steps: list[dict]) -> list[dict]:
        prompt = f"""
The original task is:
{task}

Original plan:
{json.dumps(old_plan, indent=2)}

Executed steps and observations:
{json.dumps(executed_steps, indent=2)}

Create a revised plan for the remaining work. Avoid repeating failed actions unless you explain why they should work now.
Return JSON only.
"""
        response = client.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}],
        )
        return json.loads(response.choices[0].message.content)
```

---

## Test-Time Compute Scaling

Test-time compute scaling means spending more inference-time computation to get better answers. For Agents, this can be done by:

- generating multiple plans and selecting the best one;
- using a verifier to score plans;
- running self-reflection before execution;
- exploring alternative tool paths;
- using a stronger model only for planning or verification.

```python
def choose_best_plan(task: str, planner, verifier, n: int = 5) -> list[dict]:
    candidates = [planner.plan(task) for _ in range(n)]
    scored = []
    for plan in candidates:
        score = verifier.score(task, plan)
        scored.append((score, plan))
    scored.sort(key=lambda item: item[0], reverse=True)
    return scored[0][1]
```

The trade-off is cost and latency. Test-time compute should be used selectively for high-value or high-risk tasks.

---

## When to Use Plan-and-Execute

| Scenario | Recommendation |
|---------|----------------|
| Simple Q&A | ReAct or direct answer is enough |
| Multi-step research | Plan-and-Execute works well |
| Long coding task | Plan, execute, test, and replan |
| Risky operations | Require explicit plan approval |
| Unclear user intent | Ask clarification before planning |

---

## Production Best Practices

- Keep plans short enough to execute and inspect.
- Store the plan, step results, and replanning reasons in the trace.
- Use different models for planning, execution, and verification when cost matters.
- Add guardrails before executing high-risk steps.
- Replan only when necessary; excessive replanning can create loops.
- Evaluate plan quality separately from final task success.

---

## Chapter Takeaways

- Plan-and-Execute separates global planning from local execution.
- It reduces short-sighted behavior and improves traceability for complex tasks.
- Replanning handles tool failures and changing assumptions.
- Test-time compute scaling improves quality by generating, verifying, or comparing multiple reasoning paths.
- Production systems should balance planning quality against latency and cost.

---

*Previous: [5.5 Practice: Research Agent](./05_practice_research_agent.md)*
