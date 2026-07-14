# 11.6 LangSmith Integration and Observability

> **Goal**: Master LangSmith's core capabilities, integrate tracing, evaluation, and prompt management into LangChain applications, and build production-grade observability for Agent systems.

---

## Why Observability Matters

A LangChain demo may work well in a notebook, but production Agents fail in more subtle ways:

- the wrong retriever result enters the context;
- the Agent chooses the wrong tool;
- a prompt change silently degrades quality;
- token cost grows without a clear reason;
- a user complaint cannot be reproduced.

**Observability** answers three questions:

1. What happened?
2. Why did it happen?
3. How can we prevent it from happening again?

LangSmith is designed for this full lifecycle: tracing, debugging, evaluation, dataset management, and prompt iteration.

---

## Core Concepts

| Concept | Meaning | Typical Use |
|--------|---------|-------------|
| Trace | A complete execution record | Debug one user request |
| Run | One operation inside a trace | Inspect LLM, retriever, tool, chain, or Agent step |
| Dataset | A collection of test examples | Regression testing and evaluation |
| Evaluator | A scoring function or judge | Measure quality, faithfulness, tool use |
| Prompt | Versioned prompt artifact | Compare prompt variants safely |

For Agents, traces are especially important because the final answer alone is not enough. You need to inspect tool calls, intermediate observations, and retries.

---

## Basic LangSmith Setup

```bash
export LANGCHAIN_TRACING_V2="true"
export LANGCHAIN_API_KEY="your-langsmith-api-key"
export LANGCHAIN_PROJECT="agent-learning-prod"
```

Once tracing is enabled, LangChain and LangGraph runs are automatically recorded.

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

llm = ChatOpenAI(model="gpt-4.1-mini")

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful customer support Agent."),
    ("human", "{question}"),
])

chain = prompt | llm

response = chain.invoke(
    {"question": "How do I reset my password?"},
    config={"metadata": {"user_tier": "free", "channel": "web"}},
)
```

The metadata attached to each run is useful for filtering traces by user tier, product area, traffic source, or experiment version.

---

## Tracing Agent Tool Calls

A production Agent trace should show every important decision:

```python
from langchain_core.tools import tool


@tool
def search_knowledge_base(query: str) -> str:
    """Search the internal knowledge base."""
    return f"Search results for: {query}"


@tool
def create_ticket(summary: str, priority: str = "medium") -> str:
    """Create a support ticket."""
    return f"Ticket created: {summary}, priority={priority}"
```

When these tools are used inside a LangChain or LangGraph Agent, LangSmith records:

- selected tool name;
- tool input arguments;
- tool output;
- latency and errors;
- parent-child relationship in the execution trace.

This makes it much easier to debug why an Agent called the wrong tool or passed invalid arguments.

---

## Evaluation Workflow

LangSmith can evaluate Agents using datasets and evaluators.

```python
from langsmith import Client

client = Client()

dataset = client.create_dataset(
    dataset_name="customer-support-regression",
    description="Regression cases for the customer support Agent",
)

client.create_example(
    inputs={"question": "I was charged twice. What should I do?"},
    outputs={"expected": "Explain refund process and create a billing ticket if needed."},
    dataset_id=dataset.id,
)
```

A simple evaluator can combine deterministic checks and LLM-as-Judge scoring:

```python
def contains_required_action(run, example) -> dict:
    output = run.outputs.get("output", "")
    expected = example.outputs.get("expected", "")
    score = 1 if "ticket" in output.lower() and "refund" in output.lower() else 0
    return {
        "key": "required_action",
        "score": score,
        "comment": f"Expected behavior: {expected}",
    }
```

---

## Prompt Versioning and Experiments

Prompt changes are one of the most common causes of Agent regressions. LangSmith helps manage prompt versions and compare variants.

Recommended workflow:

1. Save the baseline prompt.
2. Create a new prompt variant.
3. Run both prompts on the same dataset.
4. Compare quality, cost, latency, and tool behavior.
5. Promote only if the new version passes regression gates.

```text
Prompt v1 ─┐
           ├── Same evaluation dataset ──> Compare results ──> Ship or reject
Prompt v2 ─┘
```

---

## Production Observability Dashboard

A useful LangSmith dashboard should track:

- average latency by chain / Agent graph;
- token usage and estimated cost;
- error rate by tool;
- retrieval hit quality;
- evaluation score trend;
- top failed user intents;
- prompt and model version distribution.

```python
run_config = {
    "tags": ["prod", "customer-support", "rag-agent"],
    "metadata": {
        "prompt_version": "support-v3",
        "model": "gpt-4.1-mini",
        "retriever": "hybrid-v2",
    },
}
```

Tags and metadata are not decoration; they are how you later answer operational questions.

---

## Best Practices

- Enable tracing in staging before enabling it in production.
- Attach prompt version, model name, retriever version, and experiment ID to metadata.
- Store regression datasets for historical failures.
- Combine deterministic evaluators with LLM-as-Judge or Agent-as-Judge.
- Monitor both quality and cost, not just latency.
- Redact sensitive data before sending traces to any external observability system.
- Review failed traces regularly and convert them into evaluation cases.

---

## Chapter Takeaways

- LangSmith provides tracing, evaluation, prompt management, and experiment comparison for LangChain applications.
- Agent observability must include tool calls, intermediate states, errors, and costs.
- Datasets and evaluators turn debugging into repeatable regression testing.
- Prompt changes should be validated through experiments rather than intuition.
- Production observability requires metadata discipline and privacy governance.

---

*Previous: [11.5 Practice: Customer Service Agent](./05_practice_customer_service.md)*  
*Next: [11.7 LangChain Ecosystem 2026](./07_langchain_ecosystem_2026.md)*