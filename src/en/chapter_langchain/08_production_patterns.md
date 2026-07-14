# 11.8 LangChain Production Patterns

> **Goal**: Master the key engineering capabilities required to move LangChain applications from demos to production: streaming, async execution, error handling, caching, and concurrency control.

---

## From Demo to Production

A demo only needs to work once. A production Agent must work repeatedly, under load, with failures, changing inputs, and cost constraints.

Common gaps between demo and production:

| Area | Demo | Production |
|-----|------|------------|
| Output | Wait until full response | Stream tokens and intermediate states |
| Execution | Synchronous | Async, concurrent, cancellable |
| Errors | Print exception | Retry, fallback, user-friendly recovery |
| Cost | Ignored | Budget, cache, model routing |
| Observability | Logs | Traces, metrics, evaluation |
| State | In memory | Durable, auditable, resumable |

---

## Streaming Output

Streaming improves user experience because users see progress immediately.

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

llm = ChatOpenAI(model="gpt-4.1-mini", streaming=True)

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a concise assistant."),
    ("human", "{question}"),
])

chain = prompt | llm

for chunk in chain.stream({"question": "Explain RAG in three paragraphs."}):
    print(chunk.content, end="", flush=True)
```

For Agents, streaming should include not only final tokens but also meaningful progress events:

```text
Thinking about the task...
Searching knowledge base...
Reading 3 documents...
Calling calculator...
Generating final answer...
```

---

## Async Execution

Async execution is essential for APIs, chat servers, and high-concurrency workloads.

```python
import asyncio
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4.1-mini")


async def answer_question(question: str) -> str:
    response = await llm.ainvoke(question)
    return response.content


async def main():
    questions = [
        "What is ReAct?",
        "What is RAG?",
        "What is LangGraph?",
    ]
    answers = await asyncio.gather(*(answer_question(q) for q in questions))
    for answer in answers:
        print(answer)

asyncio.run(main())
```

Use async carefully: concurrency improves throughput, but it can also amplify rate-limit errors and cost spikes.

---

## Error Handling and Fallbacks

Production chains should expect failures:

- model API timeout;
- rate limit;
- retriever returns no documents;
- tool returns invalid output;
- output parser fails;
- downstream service is unavailable.

```python
from langchain_core.runnables import RunnableLambda


def fallback_answer(inputs: dict) -> str:
    return "Sorry, I cannot complete this request right now. Please try again later."

safe_chain = chain.with_fallbacks([
    RunnableLambda(fallback_answer)
])
```

For more advanced systems, use:

- retry with exponential backoff;
- fallback to a smaller or alternative model;
- fallback from tool action to human review;
- structured error messages for the frontend;
- trace logging for every failure.

---

## Caching

Caching reduces latency and cost for repeated or deterministic requests.

```python
from langchain.globals import set_llm_cache
from langchain_community.cache import InMemoryCache

set_llm_cache(InMemoryCache())

# Repeated identical requests can reuse cached responses.
response = chain.invoke({"question": "What is LangChain?"})
```

In production, prefer Redis or another external cache so multiple service replicas can share cached results.

Good cache candidates:

- FAQ answers;
- document summaries;
- embedding results;
- retrieved document chunks;
- classification outputs;
- tool schemas and static metadata.

Do not blindly cache personalized or sensitive responses.

---

## Concurrency and Rate Limits

A production Agent should protect both the model provider and your own backend services.

```python
import asyncio

class AsyncLimiter:
    def __init__(self, max_concurrency: int):
        self.semaphore = asyncio.Semaphore(max_concurrency)

    async def run(self, coro):
        async with self.semaphore:
            return await coro


limiter = AsyncLimiter(max_concurrency=5)
```

Apply concurrency limits at multiple levels:

- per user;
- per organization;
- per model provider;
- per tool or database;
- per deployment replica.

---

## Structured Outputs

Production systems often need JSON or typed objects, not free-form text.

```python
from pydantic import BaseModel, Field


class SupportIntent(BaseModel):
    intent: str = Field(description="User intent category")
    urgency: str = Field(description="low, medium, or high")
    needs_human: bool


structured_llm = llm.with_structured_output(SupportIntent)
result = structured_llm.invoke("I was charged twice and need help immediately.")
```

Structured output reduces parsing errors and makes downstream automation safer.

---

## Production Checklist

Before shipping a LangChain application, check:

- streaming is available for long responses;
- async execution is used in API servers;
- retries and fallbacks are defined;
- traces are enabled in staging and production;
- prompts, models, and retrievers are versioned;
- cost and latency are monitored;
- sensitive data is redacted in logs and traces;
- evaluation datasets cover key workflows;
- concurrency limits protect providers and tools.

---

## Chapter Takeaways

- Production LangChain applications require streaming, async execution, error handling, caching, and concurrency control.
- Streaming should expose meaningful progress, not only final tokens.
- Async improves throughput but must be paired with rate limiting.
- Caching can reduce cost significantly, but privacy and personalization must be considered.
- Structured outputs and observability make Agent systems safer and easier to operate.

---

*Previous: [11.7 LangChain Ecosystem 2026](./07_langchain_ecosystem_2026.md)*
