# 22.5 Computer Use and GUI Agents

> **Goal**: Master the core architecture and implementation patterns of Computer Use Agents, and understand the latest progress in GUI automation for 2025–2026.

---

## From Conversation to Operation: The Final Form of Agents

Most multimodal Agents are still conversational: the user asks, the Agent answers. In real work, however, we often need an Agent to **operate software directly**: open a browser, search for information, fill in a spreadsheet, modify files in an IDE, or install software in an operating system.

A **Computer Use Agent** is designed for exactly this scenario. The Agent no longer only "speaks"; it also "acts". It observes screenshots, reasons about the visual interface, calculates click coordinates, and sends mouse and keyboard events just like a human user.

> 📄 **Milestones**:
> - **October 2024**: Anthropic released the Computer Use beta for Claude 3.5 Sonnet, enabling a mainstream LLM to directly operate a desktop computer.
> - **January 2025**: OpenAI released Operator, based on a Computer-Using Agent model for browser automation.
> - **March 2025**: Google released Mariner, enabling Gemini 2.0 to operate Chrome.
> - **2025–2026**: Open-source frameworks such as SWE-Agent, OpenHands, and OSAtlas continued to mature.

---

## The Core Computer Use Loop

Computer Use follows almost the same workflow as a human operating a computer:

```text
┌─────────────────────────────────────────────────┐
│                 Computer Use Loop                │
│                                                  │
│  User instruction: "Open a browser and search"   │
│       ↓                                          │
│  1. Take a screenshot                            │
│       ↓                                          │
│  2. Understand the screen with a vision model     │
│       ↓                                          │
│  3. Plan the next action: click/type/scroll       │
│       ↓                                          │
│  4. Execute mouse or keyboard operation           │
│       ↓                                          │
│  5. Return to step 1 until the task is complete   │
└─────────────────────────────────────────────────┘
```

This is still the **Perceive-Think-Act** loop: perception is the screenshot, thinking is visual reasoning and planning, and action is mouse/keyboard control.

---

## Anthropic Computer Use in Practice

Anthropic exposes Computer Use through a beta tool interface. A minimal setup defines a virtual computer, a shell tool, and a text editor tool:

```python
import anthropic

client = anthropic.Anthropic()

computer_tool = {
    "type": "computer_20241022",
    "name": "computer",
    "display_width_px": 1920,
    "display_height_px": 1080,
    "display_number": 1,
}

bash_tool = {
    "type": "bash_20241022",
    "name": "bash",
}

text_editor_tool = {
    "type": "text_editor_20241022",
    "name": "str_replace_based_editor",
}
```

A typical Agent loop repeatedly asks the model for the next action, executes the tool call, and feeds the observation back:

```python
def run_computer_use(task: str, max_steps: int = 20) -> list[dict]:
    """Run a Computer Use Agent for a given task."""
    messages = [{"role": "user", "content": task}]
    trajectory = []

    for step in range(max_steps):
        response = client.beta.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=2048,
            tools=[computer_tool, bash_tool, text_editor_tool],
            messages=messages,
            betas=["computer-use-2024-10-22"],
        )

        messages.append({"role": "assistant", "content": response.content})
        trajectory.append({"step": step, "response": response.content})

        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool_call(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result,
                })

        if not tool_results:
            break

        messages.append({"role": "user", "content": tool_results})

    return trajectory
```

The implementation detail hidden behind `execute_tool_call` is where most engineering work happens: screenshot capture, coordinate conversion, mouse movement, keyboard input, scrolling, and error recovery.

---

## Browser Automation: A Safer First Step

Full desktop control is powerful but risky. For production systems, it is usually better to start with browser-only automation because the environment is easier to sandbox.

```python
from playwright.sync_api import sync_playwright

class BrowserComputerUseAgent:
    """A browser-scoped Computer Use Agent."""

    def __init__(self):
        self.playwright = sync_playwright().start()
        self.browser = self.playwright.chromium.launch(headless=False)
        self.page = self.browser.new_page(viewport={"width": 1280, "height": 720})

    def observe(self) -> bytes:
        """Return a screenshot of the current browser state."""
        return self.page.screenshot(full_page=False)

    def click(self, x: int, y: int) -> None:
        self.page.mouse.click(x, y)

    def type_text(self, text: str) -> None:
        self.page.keyboard.type(text)

    def navigate(self, url: str) -> None:
        self.page.goto(url, wait_until="networkidle")

    def close(self) -> None:
        self.browser.close()
        self.playwright.stop()
```

Browser automation has three advantages:

- **Lower risk**: the Agent cannot accidentally operate the whole operating system.
- **Better observability**: DOM snapshots, console logs, and network traces can be collected.
- **More deterministic recovery**: page reload, URL navigation, and selector-based fallback are available.

---

## Coordinate-Based vs DOM-Based Control

Computer Use systems usually combine two control modes:

| Mode | Input | Action | Strength | Weakness |
|------|-------|--------|----------|----------|
| Coordinate-based | Screenshot | Click at `(x, y)` | Works for any GUI | Brittle when layout changes |
| DOM/API-based | Structured page state | Click selector / call API | Stable and auditable | Only works where structure is available |

A robust production Agent should prefer structured actions when possible and fall back to coordinate actions when necessary.

```python
def click_button(page, label: str, fallback_xy: tuple[int, int] | None = None):
    """Prefer accessible selectors; fall back to coordinates."""
    try:
        page.get_by_role("button", name=label).click(timeout=2000)
        return "clicked_by_role"
    except Exception:
        if fallback_xy is None:
            raise
        x, y = fallback_xy
        page.mouse.click(x, y)
        return "clicked_by_coordinate"
```

---

## Safety Guardrails

Computer Use is dangerous because it can execute irreversible real-world actions. A production system should apply strict guardrails:

1. **Sandbox first**: run in a VM, container, browser profile, or restricted desktop.
2. **Permission boundaries**: block access to secrets, private files, payment pages, and production consoles.
3. **Human confirmation**: require approval before purchases, deletions, account changes, or external messages.
4. **Action allowlist**: restrict shell commands, domains, file paths, and tools.
5. **Full audit log**: store screenshots, planned actions, executed actions, and tool outputs.

```python
DANGEROUS_PATTERNS = [
    "rm -rf",
    "sudo",
    "curl | sh",
    "payment",
    "delete account",
    "transfer money",
]


def require_approval(action: str) -> bool:
    action_lower = action.lower()
    return any(pattern in action_lower for pattern in DANGEROUS_PATTERNS)
```

---

## Common Failure Modes

| Failure Mode | Symptom | Mitigation |
|-------------|---------|------------|
| Coordinate drift | Clicks land on the wrong element | Re-observe after every action; use accessibility tree when possible |
| Modal interruption | Popups block the planned action | Detect modal patterns and add close/confirm policies |
| Slow page loading | Agent acts before the page is ready | Wait for network idle, visible selectors, or stable screenshots |
| Infinite loop | Agent repeats the same failed action | Track repeated actions and force replanning |
| Hidden destructive action | Agent clicks a dangerous confirmation | Add human approval and action classification |

---

## Production Architecture

A practical Computer Use service usually consists of five layers:

```text
User Task
  ↓
Task Planner
  ↓
Visual Observer  ← screenshot / DOM / accessibility tree
  ↓
Action Policy    ← safety checks, approval gates, retry policy
  ↓
Execution Runtime ← browser, VM, desktop, or remote sandbox
```

The key design principle is: **never let the model directly execute high-risk operations**. The model proposes actions; the runtime validates, constrains, logs, and executes them.

---

## Chapter Takeaways

- Computer Use extends Agents from conversation to direct GUI operation.
- The core loop is still Perceive-Think-Act, but perception is screenshots and action is mouse/keyboard control.
- Browser automation is the safest first production target.
- Coordinate control should be combined with DOM/API control for robustness.
- Sandboxing, approval gates, and audit logs are mandatory for production Computer Use systems.

---

*Previous: [22.4 Practice: Multimodal Personal Assistant](./04_practice_multimodal_assistant.md)*  
*Next: [22.6 Video Understanding and Multimodal RAG](./06_video_and_multimodal_rag.md)*
