# PRD 03: Conversational Data Explorer

## Overview

The Conversational Data Explorer is a chat-based interface where users type natural language questions about their data. PandasAI 2.3's `Agent` class translates queries into pandas operations. Results render as text, tables, or charts inline in the chat. This page enables both technical and non-technical users to explore datasets through plain English, without writing code or constructing filters manually.

---

## User Stories

1. **As a business user**, I want to ask questions like "What are the top 10 products by revenue?" in plain English so I can get answers without learning SQL or pandas.

2. **As a data analyst**, I want to see my query results as formatted tables when the answer is tabular data, so I can inspect the underlying records and verify the analysis.

3. **As a business user**, I want to ask for charts (e.g., "Plot monthly revenue trends") and see the visualization rendered inline in the chat so I can explore trends visually without switching to another tool.

4. **As a data analyst**, I want to switch between different datasets (e.g., sales vs. customers) via a dropdown so I can explore multiple data sources in the same session.

5. **As a user**, I want to see a loading indicator while my query is processing so I know the system is working and not frozen.

6. **As a user**, I want clear, friendly error messages when something goes wrong (API failure, invalid query) so I can retry or rephrase without seeing raw Python tracebacks.

7. **As a data analyst**, I want to click example query chips (e.g., "Show top 10 by revenue") to quickly run common analyses without typing.

---

## Functional Requirements

### FR-01: Chat Interface

- Built with `solara.lab.ChatBox`, `solara.lab.ChatInput`, `solara.lab.ChatMessage`
- Message history maintained in component state
- User messages on the right, assistant responses on the left
- Chat input at bottom with send button

### FR-02: PandasAI Integration

- PandasAI `Agent` initialized with the active dataset(s)
- User message passed to `agent.chat(message)`
- Response parsed and displayed:
  - Text responses rendered as `solara.Markdown`
  - DataFrame responses rendered as `solara.DataFrame`
  - Chart responses (file paths) rendered as `solara.Image`
- Agent instance managed via `solara.reactive()` and initialized in `on_kernel_start`

### FR-03: Dataset Selection

- Sidebar or top bar with dataset selector (`solara.Select`)
- When dataset changes, PandasAI Agent is re-initialized
- Available datasets loaded from configured data sources

### FR-04: Loading States During Query

- Show typing indicator / `v.ProgressCircular` while PandasAI processes
- Use `use_task()` or threading to prevent blocking the UI
- Disable chat input while processing

### FR-05: Error Handling

- LLM API failures: show `solara.Error()` inline in chat
- Invalid query results: show warning message with retry suggestion
- Network errors: graceful fallback message
- Never expose raw Python tracebacks to the user

### FR-06: Conversation History

- Messages stored as list of dicts: `{"role": "user"|"assistant", "content": str, "type": "text"|"dataframe"|"chart"}`
- Clear conversation button
- History persists within the session (not across sessions)

### FR-07: Query Suggestions

- Show example queries as clickable chips above the chat input
- e.g., "Show top 10 by revenue", "Plot monthly trends", "What's the average?"
- Clicking a chip sends that query

---

## Technical Approach

| Aspect | Approach |
|--------|----------|
| **Page** | `pages/explorer.py` — single page component |
| **Chat state** | `solara.use_state([])` for message list |
| **Agent state** | `solara.reactive()` for Agent instance; re-init on dataset change |
| **Query execution** | `solara.lab.use_task()` or background thread to avoid blocking UI |
| **AIService** | `services/ai_service.py` — wraps PandasAI Agent, parses response types |
| **Initialization** | `solara.on_kernel_start()` to create Agent with default dataset |

---

## Implementation Details

### Component Hierarchy

```
Page (explorer.py)
├── AppLayout / Column
│   ├── Sidebar or Top Bar
│   │   └── solara.Select (dataset selector)
│   ├── Main Content
│   │   ├── solara.lab.ChatBox
│   │   │   ├── For each message:
│   │   │   │   └── solara.lab.ChatMessage
│   │   │   │       └── MessageContent (text | dataframe | chart)
│   │   │   └── [If loading] TypingIndicator (v.ProgressCircular)
│   │   ├── QuerySuggestions (chips)
│   │   └── solara.lab.ChatInput
│   └── Clear Conversation Button
```

### State Flow: User Message → AIService → Rendered Response

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  User types message or clicks suggestion chip                                │
│  → Append {"role": "user", "content": msg, "type": "text"} to messages       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  use_task(query_fn, dependencies=[message]) or threading                     │
│  - Set loading = True, disable ChatInput                                     │
│  - Call AIService.query(message)                                            │
│  - AIService uses Agent.chat(message)                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  AIService parses response:                                                  │
│  - isinstance(response, pd.DataFrame) → {"type": "dataframe", "content": df} │
│  - isinstance(response, str) and path exists → {"type": "chart", ...}       │
│  - else → {"type": "text", "content": str(response)}                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Append assistant message to messages                                        │
│  Render: MessageContent switches on type → Markdown | DataFrame | Image       │
│  Set loading = False, re-enable ChatInput                                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ChatExplorer Page Component

```python
# pages/explorer.py

import solara
from solara.lab import ChatBox, ChatInput, ChatMessage, use_task
import ipyvuetify as v
import pandas as pd
from services.ai_service import AIService

# Module-level reactive state
active_dataset = solara.reactive(None)
ai_agent = solara.reactive(None)  # AIService instance

# Placeholders: load_dataset() and get_llm_config() from DataService/Settings
available_datasets = ["sales", "customers", "products"]  # From config

def initialize_agent():
    """Called on kernel start and when dataset changes."""
    df = load_dataset(active_dataset.value)
    llm_config = get_llm_config()  # From settings/env
    ai_agent.value = AIService(df, llm_config)

solara.on_kernel_start(initialize_agent)

@solara.component
def Page():
    messages, set_messages = solara.use_state([])
    loading, set_loading = solara.use_state(False)

    # Re-initialize agent when dataset changes
    solara.use_effect(initialize_agent, [active_dataset.value])

    def send_message(content: str):
        if not content.strip() or loading:
            return
        # Append user message
        set_messages(lambda m: m + [{"role": "user", "content": content, "type": "text"}])
        set_loading(True)

        def run_query():
            try:
                result = ai_agent.value.query(content)
                set_messages(lambda m: m + [{"role": "assistant", **result}])
            except Exception as e:
                set_messages(lambda m: m + [{
                    "role": "assistant",
                    "type": "error",
                    "content": str(e)
                }])
            finally:
                set_loading(False)

        # Run in thread to avoid blocking UI
        import threading
        threading.Thread(target=run_query, daemon=True).start()

    def clear_conversation():
        set_messages([])

    with solara.Column():
        # Dataset selector
        with solara.Row():
            solara.Select(
                label="Dataset",
                value=active_dataset.value,
                values=available_datasets,
                on_value=lambda val: setattr(active_dataset, "value", val),
            )
            solara.Button("Clear", on_click=clear_conversation, icon_name="mdi-delete")

        # Chat area
        with ChatBox():
            for msg in messages:
                with ChatMessage(user=(msg["role"] == "user")):
                    MessageContent(msg)
            if loading:
                solara.display(v.ProgressCircular(indeterminate=True, size=32))

        # Query suggestions
        QuerySuggestions(on_select=send_message)

        # Chat input
        ChatInput(
            send_callback=send_message,
            disabled=loading or ai_agent.value is None,
        )
```

### AIService Wrapper

```python
# services/ai_service.py

import pandas as pd
from pandasai import Agent
from pandasai.llm import OpenAI  # or other LLM

class AIService:
    """Wraps PandasAI Agent and normalizes responses for Solara display."""

    def __init__(self, df: pd.DataFrame | list[pd.DataFrame], llm_config: dict):
        llm = llm_config.get("llm") or OpenAI(api_token=..., model="gpt-4")
        self.agent = Agent(df, config={"llm": llm})
        self.chart_export_dir = llm_config.get("chart_export_dir", "exports/charts")

    def query(self, message: str) -> dict:
        """
        Execute chat and return normalized response.
        Returns: {"type": "text"|"dataframe"|"chart", "content": ...}
        """
        response = self.agent.chat(message)

        # PandasAI 2.3: response can be str, DataFrame, or chart path
        if isinstance(response, pd.DataFrame):
            return {"type": "dataframe", "content": response}

        # Chart: PandasAI may return path or save to exports/charts/
        if isinstance(response, str):
            import os
            # Check if it's a file path (PandasAI v2 saves charts to disk)
            if os.path.isfile(response):
                return {"type": "chart", "content": response}
            # Check default export dir for recently created chart
            chart_path = self._find_latest_chart()
            if chart_path:
                return {"type": "chart", "content": chart_path}

        # Text or number: convert to string
        return {"type": "text", "content": str(response)}

    def _find_latest_chart(self) -> str | None:
        """Find most recently created chart in export dir."""
        import os
        import glob
        if not os.path.isdir(self.chart_export_dir):
            return None
        files = glob.glob(os.path.join(self.chart_export_dir, "*.png"))
        return max(files, key=os.path.getmtime) if files else None
```

### Message Rendering Logic

```python
# components/chat.py or inline in explorer.py

import solara
from solara.lab import ChatMessage
import pandas as pd

@solara.component
def MessageContent(msg: dict):
    """Render a single message based on type."""
    msg_type = msg.get("type", "text")
    content = msg.get("content", "")

    if msg_type == "error":
        solara.Error(label=content, icon=True)
        return

    if msg_type == "dataframe":
        if isinstance(content, pd.DataFrame) and not content.empty:
            solara.DataFrame(content, items_per_page=10)
        else:
            solara.Markdown("*No data to display.*")
        return

    if msg_type == "chart":
        # content is file path
        solara.Image(filename=content)  # or url= for served path
        return

    # Default: text
    solara.Markdown(str(content))
```

### Loading State Management

```python
# Option A: Threading (simpler, no async)
def send_message(content: str):
    set_loading(True)
    def run():
        try:
            result = ai_agent.value.query(content)
            # Use solara context to schedule UI update on main thread
            solara.context.get_current_context().add_delta(...)
        finally:
            set_loading(False)
    threading.Thread(target=run, daemon=True).start()

# Option B: use_task with sync wrapper (recommended for Solara)
def query_task():
    """Task function - runs in executor, returns result."""
    return ai_agent.value.query(pending_message.value)

# In component:
pending_message, set_pending = solara.use_state(None)
task = use_task(
    query_task,
    dependencies=[pending_message],
    trigger=pending_message is not None,
)
if task.pending:
    # Show ProgressCircular, disable input
    pass
elif task.finished and task.result:
    set_messages(lambda m: m + [{"role": "assistant", **task.result}])
    set_pending(None)
elif task.error:
    set_messages(lambda m: m + [{"role": "assistant", "type": "error", "content": str(task.error)}])
    set_pending(None)
```

### Query Suggestions Component

```python
# components/chat.py

@solara.component
def QuerySuggestions(on_select: callable):
    suggestions = [
        "Show top 10 by revenue",
        "Plot monthly trends",
        "What's the average?",
        "Count records by category",
        "Show me the data",
    ]
    with solara.Row(gap="0.5rem", wrap=True):
        for s in suggestions:
            solara.Button(
                label=s,
                on_click=lambda _, text=s: on_select(text),
                icon_name="mdi-lightbulb-outline",
                outlined=True,
            )
```

---

## Solara / PandasAI API Reference

| Component / API | Package | Purpose |
|-----------------|---------|---------|
| `solara.lab.ChatBox` | solara.lab | Container for chat messages |
| `solara.lab.ChatMessage` | solara.lab | Single message; `user=True` for right-aligned user msg |
| `solara.lab.ChatInput` | solara.lab | Input + send; `send_callback`, `disabled` |
| `solara.Markdown` | solara | Render markdown text |
| `solara.DataFrame` | solara | Paginated table; `items_per_page` |
| `solara.Image` | solara | `filename=` for local path, `url=` for URL |
| `solara.Error` | solara | Error display; `label`, `icon` |
| `solara.use_state(initial)` | solara | Local state; returns `(value, setter)` |
| `solara.reactive(initial)` | solara | Module-level reactive; `.value` |
| `solara.use_effect(fn, deps)` | solara | Side effect; run when deps change |
| `solara.lab.use_task(fn, deps)` | solara.lab | Async/sync task; `.pending`, `.result`, `.error` |
| `solara.on_kernel_start(fn)` | solara | Initialize on session start |
| `solara.Select` | solara | Dataset dropdown |
| `v.ProgressCircular` | ipyvuetify | `indeterminate=True`, `size=32` |
| `Agent(df, config)` | pandasai | `agent.chat(message)` → str/DataFrame/chart |
| `Agent([df1, df2])` | pandasai | Multiple DataFrames |

---

## Dependencies

- `solara` — reactive UI, ChatBox, ChatInput, ChatMessage
- `solara.lab` — `use_task`, Chat components
- `pandasai>=2.3,<3.0` — Agent, LLM config
- `pandas` — DataFrame handling
- `ipyvuetify` — `v.ProgressCircular` for loading

---

## Acceptance Criteria

- [ ] Chat interface displays user messages on the right, assistant on the left
- [ ] Dataset selector re-initializes PandasAI Agent when changed
- [ ] Text responses render as Markdown
- [ ] DataFrame responses render as paginated `solara.DataFrame`
- [ ] Chart responses render as `solara.Image` from file path
- [ ] Loading indicator (ProgressCircular) shows during query; input disabled
- [ ] API/network errors show `solara.Error` inline, no tracebacks
- [ ] Clear conversation button resets message history
- [ ] Query suggestion chips send the query when clicked
- [ ] Conversation history persists within session only
