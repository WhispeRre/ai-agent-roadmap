# 4.7 Practice: MemGPT / Letta Memory Architecture

> **Goal**: Implement a production-oriented layered-memory Agent based on the core ideas of MemGPT, and understand how the Letta framework turns these ideas into an engineering system.

---

## From Paper to Engineering

Section 4.6 introduced the core idea of MemGPT: treat the LLM context window like operating-system memory. The Agent keeps critical information in a small always-visible context, stores less urgent information in external memory, and learns to edit or retrieve memory when needed.

The engineering version of MemGPT was later renamed **Letta**. Letta provides a complete framework for Agent memory management, but understanding the underlying design is still essential when you need custom behavior in production.

---

## Layered Memory Architecture

A practical MemGPT-style Agent usually has three memory layers:

1. **Core Memory**: always included in the prompt; stores stable user profile, preferences, and active goals.
2. **Working Memory**: short-term information related to the current task or conversation.
3. **Archive Memory**: external persistent storage; searched on demand.

```python
from openai import OpenAI
import json
import time

client = OpenAI()


class LayeredMemoryAgent:
    """A layered-memory Agent inspired by MemGPT.

    Memory layers:
    1. Core Memory: always in context; stores the most important facts.
    2. Working Memory: short-term task context.
    3. Archive Memory: external storage retrieved on demand.
    """

    def __init__(self, model: str = "gpt-4.1"):
        self.model = model
        self.core_memory = {
            "user_name": "",
            "preferences": [],
            "key_facts": [],
            "active_goals": [],
        }
        self.working_memory = []
        self.max_working_items = 10
        self.archive_memory = []
        self.conversation = []
```

The key design principle is that **not all information deserves the same memory level**. Stable, frequently used facts belong in core memory; temporary task details belong in working memory; long-tail historical facts belong in archive memory.

---

## Building Messages With Memory

```python
class LayeredMemoryAgent(LayeredMemoryAgent):
    def _build_messages(self, user_input: str) -> list[dict]:
        core_memory_text = json.dumps(self.core_memory, ensure_ascii=False, indent=2)
        working_memory_text = json.dumps(self.working_memory[-self.max_working_items:], ensure_ascii=False, indent=2)

        system_prompt = f"""
You are a helpful Agent with a layered memory system.

Core Memory, always visible:
{core_memory_text}

Working Memory, current task context:
{working_memory_text}

Memory policy:
- Save stable user preferences to core memory.
- Save temporary task details to working memory.
- Move old but potentially useful information to archive memory.
- Retrieve archive memory when the current question depends on past context.
"""

        messages = [{"role": "system", "content": system_prompt}]
        messages.extend(self.conversation[-8:])
        messages.append({"role": "user", "content": user_input})
        return messages
```

This prompt makes the memory structure explicit. The model does not need to guess where memory lives; it sees the boundaries and can use tools to update memory.

---

## Memory Editing Tools

A MemGPT-style Agent should be able to edit memory through tools rather than by silently changing hidden state.

```python
class LayeredMemoryAgent(LayeredMemoryAgent):
    def _get_memory_tools(self) -> list[dict]:
        return [
            {
                "type": "function",
                "function": {
                    "name": "update_core_memory",
                    "description": "Update stable user profile, preferences, key facts, or active goals.",
                    "parameters": {
                        "type": "object",
                        "properties": {
                            "field": {
                                "type": "string",
                                "enum": ["user_name", "preferences", "key_facts", "active_goals"],
                            },
                            "value": {"type": "string"},
                            "operation": {"type": "string", "enum": ["set", "append", "remove"]},
                        },
                        "required": ["field", "value", "operation"],
                    },
                },
            },
            {
                "type": "function",
                "function": {
                    "name": "archive_memory",
                    "description": "Store information in long-term archive memory.",
                    "parameters": {
                        "type": "object",
                        "properties": {
                            "content": {"type": "string"},
                            "tags": {"type": "array", "items": {"type": "string"}},
                        },
                        "required": ["content"],
                    },
                },
            },
            {
                "type": "function",
                "function": {
                    "name": "search_archive_memory",
                    "description": "Search long-term archive memory for relevant past information.",
                    "parameters": {
                        "type": "object",
                        "properties": {"query": {"type": "string"}, "top_k": {"type": "integer"}},
                        "required": ["query"],
                    },
                },
            },
        ]
```

The tool interface creates an audit trail: every memory change has a structured reason and can be logged, reviewed, or rolled back.

---

## Implementing Memory Operations

```python
class LayeredMemoryAgent(LayeredMemoryAgent):
    def update_core_memory(self, field: str, value: str, operation: str = "append") -> str:
        if field not in self.core_memory:
            raise ValueError(f"Unknown core memory field: {field}")

        if operation == "set":
            self.core_memory[field] = value
        elif operation == "append":
            if not isinstance(self.core_memory[field], list):
                self.core_memory[field] = [self.core_memory[field]] if self.core_memory[field] else []
            if value not in self.core_memory[field]:
                self.core_memory[field].append(value)
        elif operation == "remove":
            if isinstance(self.core_memory[field], list) and value in self.core_memory[field]:
                self.core_memory[field].remove(value)

        return f"Updated core memory: {field}"

    def archive_memory_item(self, content: str, tags: list[str] | None = None) -> str:
        item = {
            "id": f"mem_{len(self.archive_memory) + 1}",
            "content": content,
            "tags": tags or [],
            "created_at": time.time(),
        }
        self.archive_memory.append(item)
        return item["id"]

    def search_archive_memory(self, query: str, top_k: int = 5) -> list[dict]:
        query_terms = set(query.lower().split())
        scored = []
        for item in self.archive_memory:
            score = len(query_terms & set(item["content"].lower().split()))
            score += len(query_terms & set(" ".join(item["tags"]).lower().split()))
            if score > 0:
                scored.append((score, item))
        scored.sort(key=lambda pair: pair[0], reverse=True)
        return [item for _, item in scored[:top_k]]
```

In production, `archive_memory` should usually be backed by a vector database, a relational database, or a hybrid search engine. The in-memory implementation above is only for understanding the control flow.

---

## Automatic Memory Management

The Agent can proactively decide whether a user message contains memory-worthy information.

```python
class LayeredMemoryAgent(LayeredMemoryAgent):
    def _auto_manage_memory(self, user_input: str) -> None:
        """A lightweight heuristic before invoking the model."""
        lower = user_input.lower()

        if "remember" in lower or "i prefer" in lower or "my name is" in lower:
            self.working_memory.append({
                "type": "memory_candidate",
                "content": user_input,
                "created_at": time.time(),
            })

        if len(self.working_memory) > self.max_working_items:
            old_items = self.working_memory[:-self.max_working_items]
            self.working_memory = self.working_memory[-self.max_working_items:]
            for item in old_items:
                self.archive_memory_item(
                    content=json.dumps(item, ensure_ascii=False),
                    tags=["auto_archived", "working_memory"],
                )
```

A production system can replace this heuristic with a classifier that decides:

- Should this be remembered?
- Which memory layer should it enter?
- Is it sensitive or private?
- Does it conflict with existing memory?
- Should the user confirm before saving it?

---

## Letta-Style Engineering Principles

Letta turns MemGPT's ideas into a full Agent runtime. The important engineering lessons are:

- **Memory is explicit state**: it should be inspectable, editable, and testable.
- **Memory operations are tools**: the model should call structured operations rather than mutate hidden state.
- **Context is managed like a scarce resource**: the Agent must decide what stays visible and what moves out.
- **Long-term memory needs retrieval**: archive memory is only useful if it can be searched accurately.
- **Memory requires governance**: user consent, privacy rules, and deletion workflows matter.

---

## Common Pitfalls

| Pitfall | Consequence | Fix |
|--------|-------------|-----|
| Saving everything | Memory becomes noisy and expensive | Add memory-worthiness filtering |
| No deletion path | Incorrect memories keep affecting answers | Support update, remove, and user review |
| Core memory too large | Prompt becomes bloated | Keep only stable, high-value facts in core memory |
| No conflict handling | Old facts contradict new facts | Track timestamps and ask for confirmation |
| No privacy policy | Sensitive information is stored silently | Require consent and redact secrets |

---

## Chapter Takeaways

- MemGPT's core idea is to manage LLM context like an operating-system memory hierarchy.
- Core memory, working memory, and archive memory serve different purposes.
- Memory editing should be exposed as structured tools for auditability.
- Letta demonstrates how these ideas become a production Agent runtime.
- Good memory systems need not only retrieval, but also governance, deletion, and conflict handling.

---

*Previous: [4.6 MemGPT: Managing LLM Context Like an Operating System](./06_paper_readings.md)*
