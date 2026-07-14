# 10.8 Agent-Specific Fine-Tuning: Teaching Models to Use Tools

> 🔧 *"General SFT teaches a model how to speak; Agent SFT teaches it how to act in the right format; Agentic RL teaches it whether the action actually succeeds."*

---

## SFT and Agentic RL: Different Jobs

Supervised fine-tuning and reinforcement learning are often discussed together, but they solve different problems in Agent training.

> **SFT teaches the model what expert behavior looks like. Agentic RL teaches the model which behavior actually works in an environment.**

| Stage | Learns | Example |
|------|--------|---------|
| General SFT | Instruction following and response style | Answer questions politely |
| Agent SFT | Tool-call format and task trajectories | Call `search`, then `calculator`, then answer |
| Agentic RL | Outcome-driven optimization | Prefer actions that solve the task with fewer errors |

A practical training pipeline often starts with Agent SFT, then uses RL or preference optimization to improve success rate.

---

## Why General SFT Is Not Enough

A generally capable model may still fail as an Agent:

- **Hallucinated tools**: calling `search_google` when only `web_search` exists.
- **Inconsistent format**: mixing JSON, natural language, and function-call formats.
- **Invalid arguments**: passing wrong types or missing required fields.
- **Poor recovery**: failing after one tool error instead of retrying or replanning.
- **No environment awareness**: giving an answer without using required tools.

Agent fine-tuning focuses on trajectories, not only final answers.

---

## Agent Training Data Format

A useful Agent SFT example should include the full interaction:

```json
{
  "messages": [
    {"role": "system", "content": "You are a tool-using Agent."},
    {"role": "user", "content": "What is 17% of 286?"},
    {
      "role": "assistant",
      "tool_calls": [
        {
          "name": "calculator",
          "arguments": {"expression": "286 * 0.17"}
        }
      ]
    },
    {"role": "tool", "name": "calculator", "content": "48.62"},
    {"role": "assistant", "content": "17% of 286 is 48.62."}
  ]
}
```

The model learns when to call tools, which tool to call, how to format arguments, and how to use observations in the final answer.

---

## Dataset Categories

A balanced Agent SFT dataset should cover:

| Category | Purpose |
|---------|---------|
| Direct answer | Learn when tools are unnecessary |
| Single-tool tasks | Learn basic tool selection and arguments |
| Multi-tool tasks | Learn sequencing and state tracking |
| Tool failure cases | Learn recovery behavior |
| Clarification cases | Learn when to ask the user |
| Safety cases | Learn refusal and approval boundaries |
| Negative examples | Learn what not to do |

Do not train only on successful clean examples. Real Agents need messy trajectories, failures, and corrections.

---

## Fine-Tuning Objective

For SFT, the objective is next-token prediction on expert trajectories:

\[
\mathcal{L}_{\text{SFT}} = - \sum_t \log \pi_\theta(a_t \mid s_t)
\]

Where:

- \(s_t\) is the dialogue and environment state before an action;
- \(a_t\) is the expert action, such as a tool call or final response;
- \(\pi_\theta\) is the model policy.

For Agent SFT, the important detail is deciding which tokens to train on. In many pipelines, user and tool messages are context, while assistant messages and tool-call arguments are training targets.

---

## Quality Checks for Agent SFT Data

Before training, validate the dataset:

```python
REQUIRED_TOOL_FIELDS = {"name", "arguments"}


def validate_tool_call(tool_call: dict, available_tools: set[str]) -> list[str]:
    errors = []
    missing = REQUIRED_TOOL_FIELDS - set(tool_call)
    if missing:
        errors.append(f"Missing fields: {missing}")
    if tool_call.get("name") not in available_tools:
        errors.append(f"Unknown tool: {tool_call.get('name')}")
    if not isinstance(tool_call.get("arguments", {}), dict):
        errors.append("Tool arguments must be an object")
    return errors
```

Also check:

- schema validity;
- duplicated examples;
- leaking answers into user messages;
- inconsistent tool names;
- unsafe or private data;
- imbalance between easy and hard tasks.

---

## Training Strategy

Recommended progression:

1. **Collect expert trajectories** from human demonstrations, successful Agent runs, and curated synthetic tasks.
2. **Normalize tool schemas** so the model sees consistent tool names and argument formats.
3. **Filter low-quality trajectories** with validators and human review.
4. **Train with LoRA or full fine-tuning** depending on model size and budget.
5. **Evaluate on held-out Agent tasks**, not only language benchmarks.
6. **Use RL or preference optimization** to improve long-horizon success.

---

## Evaluation Metrics

Agent fine-tuning should be evaluated on behavior:

| Metric | Meaning |
|-------|---------|
| Tool selection accuracy | Did the model choose the right tool? |
| Argument accuracy | Were tool inputs valid and correct? |
| Format validity | Can the runtime parse the tool call? |
| Task success rate | Did the Agent solve the task? |
| Recovery rate | Did it recover from tool errors? |
| Step efficiency | Did it use unnecessary tools or loops? |
| Safety compliance | Did it avoid unsafe actions? |

---

## Best Practices

- Train on trajectories, not just final answers.
- Include both tool-use and no-tool examples.
- Validate every tool call against real tool schemas.
- Include tool failure and recovery examples.
- Keep train and evaluation tools versioned.
- Evaluate with full Agent runs, not only perplexity.
- Use RL or preference optimization after SFT to optimize long-horizon success.

---

## Chapter Takeaways

- General SFT is not enough for reliable tool-using Agents.
- Agent SFT teaches tool selection, argument formatting, sequencing, and recovery behavior.
- Good datasets include successful trajectories, failures, clarifications, and safety cases.
- Data validation is critical because schema errors directly become runtime failures.
- SFT teaches expert imitation; Agentic RL optimizes actual task success.

---

*Previous: [10.7 Latest Agentic-RL Research](./07_latest_research.md)*  
*Next: [10.9 Agentic Data Flywheel](./09_data_flywheel.md)*