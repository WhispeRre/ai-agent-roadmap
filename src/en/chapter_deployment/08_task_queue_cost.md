# 19.8 Long-Running Task Queues and Cost Governance

> **Goal**: Learn how to manage long-running Agent tasks with queues such as Celery and Temporal, control token budgets, route between large and small models, and build cost monitoring and alerting.

---

## The Long-Running Task Problem

Many Agent tasks cannot be completed within a short HTTP request:

- researching dozens of web pages;
- analyzing large documents;
- generating a multi-file code patch;
- running data analysis and visualization;
- waiting for human approval;
- retrying tools after external service failures.

If these tasks run directly inside the API request, users may experience timeouts, duplicated work, and poor reliability.

The solution is to move long work into a task queue.

```text
Client request
    ↓
API creates task
    ↓
Queue stores task
    ↓
Worker executes Agent workflow
    ↓
State and progress are persisted
    ↓
Client polls or receives events
```

---

## Celery vs Temporal

| Feature | Celery | Temporal |
|--------|--------|----------|
| Mental model | Distributed task queue | Durable workflow engine |
| Best for | Simple async jobs | Complex long-running workflows |
| State management | Mostly external | Built-in durable workflow state |
| Retries | Task-level retries | Workflow and activity retries |
| Visibility | Basic monitoring | Rich workflow history |
| Human-in-the-loop | Custom implementation | Natural wait/resume pattern |

> 💡 **Recommendation**: If your Agent is a simple "request → execute → return" service, Celery is enough. If your Agent involves multi-step workflows, conditional branches, human approval, and subtask orchestration, Temporal is much stronger.

---

## Celery Pattern

```python
from celery import Celery

app = Celery(
    "agent_tasks",
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/1",
)


@app.task(bind=True, autoretry_for=(Exception,), retry_backoff=True, max_retries=3)
def run_agent_task(self, task_id: str, user_input: str) -> dict:
    self.update_state(state="PROGRESS", meta={"step": "planning"})
    plan = plan_task(user_input)

    self.update_state(state="PROGRESS", meta={"step": "executing"})
    result = execute_plan(plan)

    self.update_state(state="PROGRESS", meta={"step": "finalizing"})
    return {"task_id": task_id, "result": result}
```

Celery works well for background jobs, batch summarization, embedding pipelines, and simple Agent tasks.

---

## Temporal Pattern

Temporal treats a long Agent task as a durable workflow. If a worker crashes, the workflow can resume from history.

```python
from temporalio import workflow, activity


@activity.defn
async def run_retrieval(query: str) -> list[str]:
    return ["doc1", "doc2"]


@activity.defn
async def generate_answer(query: str, docs: list[str]) -> str:
    return "final answer"


@workflow.defn
class AgentWorkflow:
    @workflow.run
    async def run(self, query: str) -> str:
        docs = await workflow.execute_activity(
            run_retrieval,
            query,
            start_to_close_timeout=30,
        )
        answer = await workflow.execute_activity(
            generate_answer,
            query,
            docs,
            start_to_close_timeout=60,
        )
        return answer
```

Temporal is especially valuable when you need retries, long waits, audit history, human approval, and exactly-once workflow semantics.

---

## Progress Reporting

Long tasks need progress updates:

```python
@dataclass
class TaskProgress:
    task_id: str
    status: str
    current_step: str
    percent: float
    message: str
```

Expose progress through:

- polling endpoint;
- WebSocket;
- Server-Sent Events;
- notification callback;
- workflow dashboard.

Good progress reporting reduces user anxiety and prevents duplicate submissions.

---

## Token Budget Control

Agents can easily spend too many tokens through repeated tool calls, long contexts, or retries.

```python
class TokenBudget:
    def __init__(self, max_tokens: int):
        self.max_tokens = max_tokens
        self.used_tokens = 0

    def consume(self, tokens: int) -> None:
        if self.used_tokens + tokens > self.max_tokens:
            raise RuntimeError("Token budget exceeded")
        self.used_tokens += tokens

    @property
    def remaining(self) -> int:
        return self.max_tokens - self.used_tokens
```

Budget should be tracked by:

- request;
- user;
- organization;
- workflow;
- model provider;
- time window.

---

## Cost Governance

Cost governance combines policies, routing, and monitoring.

```text
Before execution:
  estimate cost → choose model → enforce budget

During execution:
  track tokens → stop if budget exceeded → downgrade or ask user

After execution:
  log cost → aggregate by user/category/model → alert on anomalies
```

```python
class CostPolicy:
    def choose_model(self, task: dict, budget_remaining: float) -> str:
        if budget_remaining < 0.01:
            return "small-model"
        if task.get("risk") == "high" or task.get("complexity") == "high":
            return "strong-model"
        return "medium-model"
```

---

## Monitoring and Alerts

Track cost and queue health together:

| Metric | Alert Example |
|-------|---------------|
| Queue length | Queue length > 100 for 10 minutes |
| Task latency | p95 task time > 5 minutes |
| Token cost | Daily spend exceeds budget |
| Retry count | Retry spike indicates external outage |
| Failure rate | Failed tasks > 5% |
| Model distribution | Large-model usage unexpectedly increases |

---

## Best Practices

- Do not run long Agent tasks inside synchronous HTTP requests.
- Use Celery for simple background tasks and Temporal for durable workflows.
- Persist progress and intermediate state.
- Make tasks idempotent so retries are safe.
- Enforce token and cost budgets before and during execution.
- Route simple subtasks to cheaper models.
- Add alerts for cost spikes, retry storms, and queue backlogs.

---

## Chapter Takeaways

- Long-running Agent tasks should be handled by queues or workflow engines.
- Celery is simple and practical; Temporal is stronger for durable multi-step workflows.
- Progress reporting is part of the user experience.
- Token budgets and cost policies prevent runaway Agent loops.
- Cost governance requires routing, monitoring, budgets, and alerts.

---

*Previous: [19.7 Kubernetes Orchestration and Serverless GPU](./07_k8s_serverless.md)*
