# PRD 08: Interactive ECharts in Solara Chat via PandasAI

## Problem Statement

PandasAI 2.3's code generation pipeline generates matplotlib code by default. When a user asks "Plot revenue by region," the LLM produces Python code that calls `matplotlib.pyplot`, saves a static PNG to disk, and returns the file path. The Solara chat interface then renders this PNG as a flat image -- no zoom, no hover, no interactivity.

We want the chat to render **interactive ECharts charts** directly inside chat messages, using Solara's built-in `FigureEcharts` component. This means PandasAI's code generation pipeline must produce ECharts option dicts instead of matplotlib code, and the Solara chat must render those dicts as live, interactive charts.

## How PandasAI's Code Generation Pipeline Works

Understanding the pipeline is essential to knowing where to intercept it.

```
1. User calls agent.chat("Plot revenue by region")
       │
       ▼
2. PandasAI builds a prompt containing:
   - The user query
   - DataFrame schema (column names, types, sample rows)
   - Agent description (if provided)
   - Conversation history
   - Available skills
       │
       ▼
3. LLM generates Python code, e.g.:
   │  import matplotlib.pyplot as plt
   │  plt.bar(df["region"], df["revenue"])
   │  plt.savefig("exports/charts/chart.png")
   │  result = {"type": "plot", "value": "exports/charts/chart.png"}
       │
       ▼
4. PandasAI executes the code in a sandbox
   - Only whitelisted modules allowed (matplotlib, pandas, numpy, etc.)
   - The code must set a `result` variable
       │
       ▼
5. ResponseParser processes the result
   - format_plot(result)   → for type "plot"
   - format_dataframe(result) → for type "dataframe"
   - format_response(result)  → for type "text" / "number"
       │
       ▼
6. Parsed result returned to caller
```

**Key integration points** where we can redirect to ECharts:
- **Step 2**: The Agent `description` field injects instructions into the prompt
- **Step 3**: The LLM can generate ECharts option dicts instead of matplotlib code
- **Step 4**: `custom_whitelisted_dependencies` controls which modules are allowed
- **Step 5**: Custom `ResponseParser` intercepts and tags the response type

## Approach: Making PandasAI Generate ECharts Option Dicts

The LLM doesn't need to import any library to generate an ECharts chart. An ECharts chart is just a **Python dictionary** -- the LLM generates a dict literal that conforms to the ECharts option spec. No special imports, no sandbox concerns, no extra dependencies.

This is the critical advantage: while Plotly requires whitelisting `plotly` in PandasAI's sandbox and importing the library, **ECharts option dicts are plain Python dicts** that PandasAI can generate and return without any extra dependencies.

### Step 1: Agent Description

PandasAI's `Agent` class accepts a `description` parameter that gets injected into every prompt sent to the LLM. This is the primary mechanism to instruct the LLM on chart format.

```python
ECHARTS_AGENT_DESCRIPTION = """You are a data analysis agent. When asked to create 
any visualization, chart, or plot, you MUST return an Apache ECharts option 
dictionary instead of using matplotlib or any other plotting library.

The result for chart requests must be set as:
    result = {"type": "plot", "value": <echarts_option_dict>}

where <echarts_option_dict> is a Python dictionary following the Apache ECharts 
option specification (https://echarts.apache.org/en/option.html).

Rules for generating ECharts options:
1. Do NOT import matplotlib, seaborn, plotly, or any plotting library
2. The option dict must include at minimum: series, and typically xAxis/yAxis
3. Always include a "tooltip" with "trigger" set appropriately
4. Always include a descriptive "title" with "text" set to a chart title
5. Use these chart types as appropriate:
   - Bar charts: {"type": "bar"} in series
   - Line charts: {"type": "line"} in series, add "smooth": True for curves
   - Pie/donut charts: {"type": "pie"} with "radius" for donut
   - Scatter plots: {"type": "scatter"} in series
   - Heatmaps: {"type": "heatmap"} in series
6. For category axes, use {"type": "category", "data": [...]}
7. For value axes, use {"type": "value"}
8. Format numbers and labels clearly
9. Include "dataZoom" for time series to allow user scrolling
10. For the result variable, the dict value should be the ECharts option ONLY -- 
    do not nest it further

Example for a bar chart:
    result = {"type": "plot", "value": {
        "title": {"text": "Revenue by Region"},
        "tooltip": {"trigger": "axis"},
        "xAxis": {"type": "category", "data": ["East", "West", "North", "South"]},
        "yAxis": {"type": "value"},
        "series": [{"type": "bar", "data": [5000, 3200, 4100, 2800]}]
    }}
"""
```

### Step 2: Custom ResponseParser

Create a custom `ResponseParser` that detects ECharts option dicts and tags them with a `"echarts"` type so the Solara frontend knows to render them with `FigureEcharts`.

```python
# services/response_parser.py
from pandasai.responses.response_parser import ResponseParser
import logging

logger = logging.getLogger(__name__)


def _is_echarts_option(value) -> bool:
    """Detect whether a value is an ECharts option dict."""
    if not isinstance(value, dict):
        return False
    echarts_keys = {"series", "xAxis", "yAxis", "tooltip", "title", "legend",
                    "radar", "geo", "parallel", "calendar", "dataset"}
    return bool(echarts_keys & set(value.keys()))


class SolaraResponseParser(ResponseParser):
    def __init__(self, context) -> None:
        super().__init__(context)

    def format_plot(self, result):
        value = result.get("value") if isinstance(result, dict) else result

        if _is_echarts_option(value):
            return {"type": "echarts", "value": value}

        # Fallback: matplotlib saved a file path
        if isinstance(value, str) and value.endswith((".png", ".jpg", ".svg")):
            logger.warning("LLM generated matplotlib instead of ECharts; falling back to image")
            return {"type": "image", "value": value}

        return {"type": "plot", "value": value}

    def format_dataframe(self, result):
        import pandas as pd
        value = result.get("value", result) if isinstance(result, dict) else result
        if isinstance(value, pd.DataFrame):
            return {"type": "dataframe", "value": value}
        return {"type": "text", "value": str(value)}

    def format_response(self, result):
        value = result.get("value", result) if isinstance(result, dict) else result
        return {"type": "text", "value": str(value)}
```

### Step 3: AIService Wrapper

Encapsulate the Agent configuration with ECharts prompt and response parser.

```python
# services/ai_service.py
from pandasai import Agent
from services.response_parser import SolaraResponseParser, _is_echarts_option
import pandas as pd
import logging

logger = logging.getLogger(__name__)

ECHARTS_AGENT_DESCRIPTION = """..."""  # Full description from Step 1 above


class AIService:
    def __init__(self, df: pd.DataFrame, llm):
        self.agent = Agent(
            df,
            config={
                "llm": llm,
                "response_parser": SolaraResponseParser,
                "save_charts": False,
                "open_charts": False,
                "verbose": False,
            },
            description=ECHARTS_AGENT_DESCRIPTION,
        )

    def query(self, message: str) -> dict:
        """
        Returns a normalized response:
        {
            "type": "echarts" | "dataframe" | "text" | "image" | "error",
            "value": dict | pd.DataFrame | str,
            "query": str,
        }
        """
        try:
            raw = self.agent.chat(message)

            # ResponseParser already tagged the result
            if isinstance(raw, dict) and "type" in raw:
                raw["query"] = message
                return raw

            # Direct ECharts dict returned
            if _is_echarts_option(raw):
                return {"type": "echarts", "value": raw, "query": message}

            # DataFrame
            if isinstance(raw, pd.DataFrame):
                return {"type": "dataframe", "value": raw, "query": message}

            # Fallback to text
            return {"type": "text", "value": str(raw), "query": message}

        except Exception as e:
            logger.exception("PandasAI query failed")
            return {"type": "error", "value": str(e), "query": message}

    def update_data(self, df: pd.DataFrame):
        llm = self.agent._config.get("llm") if hasattr(self.agent, '_config') else None
        self.__init__(df, llm)
```

## Rendering ECharts Inside Solara Chat Messages

Solara's `ChatMessage` component accepts **any Solara component** as children. This means `FigureEcharts` can be placed directly inside a message bubble.

### Message Data Model

Each message in the chat is stored as a dict:

```python
{
    "role": "user" | "assistant",
    "type": "text" | "echarts" | "dataframe" | "image" | "error",
    "content": str | dict | pd.DataFrame,  # ECharts option dict for type "echarts"
    "query": str,  # Original user query (for assistant messages)
}
```

### ChatMessageContent Component

Renders the appropriate component based on message type. For ECharts messages, it renders `FigureEcharts` with the option dict.

```python
# components/chat.py
import solara
import pandas as pd


@solara.component
def ChatMessageContent(message: dict):
    msg_type = message.get("type", "text")
    content = message.get("content")

    if msg_type == "echarts" and isinstance(content, dict):
        EChartsChartMessage(option=content)
    elif msg_type == "dataframe" and isinstance(content, pd.DataFrame):
        solara.DataFrame(content, items_per_page=10)
    elif msg_type == "image" and isinstance(content, str):
        solara.Image(content)
    elif msg_type == "error":
        solara.Error(str(content))
    else:
        solara.Markdown(str(content))


@solara.component
def EChartsChartMessage(option: dict):
    """Renders an interactive ECharts chart inside a chat message with
    theme-aware styling and a fixed height container."""
    dark = solara.lab.use_dark_effective()

    themed_option = solara.use_memo(lambda: _apply_theme(option, dark), [option, dark])

    solara.FigureEcharts(
        option=themed_option,
        responsive=True,
        attributes={"style": "width: 100%; height: 400px; min-width: 300px;"},
    )


def _apply_theme(option: dict, dark: bool) -> dict:
    """Non-destructively apply dark/light theme colors to an ECharts option."""
    themed = {**option}

    text_color = "#ccc" if dark else "#333"
    bg_color = "transparent"

    themed["backgroundColor"] = bg_color

    if "title" in themed:
        themed["title"] = {**themed.get("title", {}), "textStyle": {"color": text_color}}

    if "legend" in themed:
        themed["legend"] = {**themed.get("legend", {}), "textStyle": {"color": text_color}}

    for axis_key in ("xAxis", "yAxis"):
        if axis_key in themed:
            axis = themed[axis_key]
            if isinstance(axis, dict):
                themed[axis_key] = {
                    **axis,
                    "axisLabel": {**axis.get("axisLabel", {}), "color": text_color},
                    "axisLine": {"lineStyle": {"color": "#555" if dark else "#ccc"}},
                }
            elif isinstance(axis, list):
                themed[axis_key] = [
                    {**a, "axisLabel": {**a.get("axisLabel", {}), "color": text_color}}
                    for a in axis
                ]

    return themed
```

### Chat Page Component

The full chat page wiring messages, input, and the AIService together.

```python
# pages/explorer.py
import solara
from services.ai_service import AIService
from components.chat import ChatMessageContent
import ipyvuetify as v

ai_service = solara.reactive(None)
messages = solara.reactive([])
processing = solara.reactive(False)


@solara.component
def Page():
    # Initialize service (simplified -- real version loads data from config)
    def init():
        import pandas as pd
        df = pd.read_csv("data/sample.csv")
        from config import get_llm
        ai_service.value = AIService(df, get_llm())
    solara.use_effect(init, [])

    with solara.Column(style={"height": "calc(100vh - 130px)"}):
        ChatArea()
        QuerySuggestions()
        ChatInputBar()


@solara.component
def ChatArea():
    with solara.lab.ChatBox():
        for msg in messages.value:
            is_user = msg["role"] == "user"

            with solara.lab.ChatMessage(
                user=is_user,
                name="You" if is_user else "Assistant",
                avatar="mdi-account" if is_user else "mdi-robot",
                notch=True,
            ):
                ChatMessageContent(message=msg)

        # Typing indicator while processing
        if processing.value:
            with solara.lab.ChatMessage(
                user=False,
                name="Assistant",
                avatar="mdi-robot",
            ):
                solara.display(v.ProgressCircular(
                    indeterminate=True, size=24, width=2, color="primary"
                ))
                solara.Text("Analyzing your data...")


@solara.component
def ChatInputBar():
    def send(user_message: str):
        if not user_message.strip() or not ai_service.value:
            return

        # Add user message
        messages.value = [
            *messages.value,
            {"role": "user", "type": "text", "content": user_message},
        ]

        processing.value = True

        # Run PandasAI query
        result = ai_service.value.query(user_message)

        # Add assistant response
        messages.value = [
            *messages.value,
            {
                "role": "assistant",
                "type": result["type"],
                "content": result["value"],
                "query": user_message,
            },
        ]

        processing.value = False

    solara.lab.ChatInput(
        send_callback=send,
        disabled=processing.value,
    )


@solara.component
def QuerySuggestions():
    """Clickable example queries shown above the input."""
    suggestions = [
        "Show top 10 by revenue",
        "Plot monthly revenue trend",
        "Show distribution by category",
        "Compare regions as a bar chart",
    ]

    if not messages.value:
        with solara.Row(gap="8px", style={"flex-wrap": "wrap", "padding": "8px"}):
            for suggestion in suggestions:
                def on_click(s=suggestion):
                    # Trigger the same flow as typing
                    ChatInputBar  # re-render handled by reactive
                solara.Button(
                    suggestion,
                    outlined=True,
                    small=True,
                    on_click=lambda s=suggestion: send_suggestion(s),
                )


def send_suggestion(text: str):
    """Send a suggestion query through the same pipeline."""
    if not ai_service.value:
        return
    messages.value = [
        *messages.value,
        {"role": "user", "type": "text", "content": text},
    ]
    processing.value = True
    result = ai_service.value.query(text)
    messages.value = [
        *messages.value,
        {"role": "assistant", "type": result["type"], "content": result["value"], "query": text},
    ]
    processing.value = False
```

## Handling Click Events on Chat Charts

ECharts charts in the chat can be interactive beyond just hover. The `on_click` callback receives the clicked data point, enabling drill-down or follow-up queries.

```python
@solara.component
def EChartsChartMessage(option: dict):
    dark = solara.lab.use_dark_effective()
    themed_option = solara.use_memo(lambda: _apply_theme(option, dark), [option, dark])

    def on_chart_click(event_data):
        """When user clicks a bar/point, send a follow-up query about that item."""
        name = event_data.get("name", "")
        if name and ai_service.value:
            follow_up = f"Tell me more about {name}"
            send_suggestion(follow_up)

    solara.FigureEcharts(
        option=themed_option,
        on_click=on_chart_click,
        responsive=True,
        attributes={"style": "width: 100%; height: 400px; min-width: 300px;"},
    )
```

## What the LLM-Generated Code Looks Like

When the Agent processes a chart request, the LLM generates plain Python code that builds a dict. No imports needed:

```python
# Example code generated by the LLM for "Plot revenue by region as a bar chart"
# PandasAI provides `dfs` as the list of dataframes

df = dfs[0]
regions = df["region"].tolist()
revenue = df["revenue"].tolist()

result = {
    "type": "plot",
    "value": {
        "title": {"text": "Revenue by Region"},
        "tooltip": {"trigger": "axis", "axisPointer": {"type": "shadow"}},
        "xAxis": {"type": "category", "data": regions},
        "yAxis": {"type": "value", "name": "Revenue ($)"},
        "series": [{
            "type": "bar",
            "data": revenue,
            "itemStyle": {"borderRadius": [4, 4, 0, 0]},
        }],
    }
}
```

```python
# Example for "Show distribution of sales by category as a pie chart"

df = dfs[0]
categories = df.groupby("category")["sales"].sum().reset_index()
pie_data = [{"name": row["category"], "value": int(row["sales"])} 
            for _, row in categories.iterrows()]

result = {
    "type": "plot",
    "value": {
        "title": {"text": "Sales Distribution by Category"},
        "tooltip": {"trigger": "item", "formatter": "{b}: {c} ({d}%)"},
        "series": [{
            "type": "pie",
            "radius": ["40%", "70%"],
            "data": pie_data,
            "emphasis": {"itemStyle": {"shadowBlur": 10, "shadowColor": "rgba(0,0,0,0.5)"}},
        }],
    }
}
```

```python
# Example for "Show monthly revenue trend over time"

df = dfs[0]
df["month"] = pd.to_datetime(df["date"]).dt.strftime("%Y-%m")
monthly = df.groupby("month")["revenue"].sum().reset_index()

result = {
    "type": "plot",
    "value": {
        "title": {"text": "Monthly Revenue Trend"},
        "tooltip": {"trigger": "axis"},
        "xAxis": {"type": "category", "data": monthly["month"].tolist()},
        "yAxis": {"type": "value", "name": "Revenue"},
        "dataZoom": [{"type": "slider", "start": 0, "end": 100}],
        "series": [{
            "type": "line",
            "data": monthly["revenue"].tolist(),
            "smooth": True,
            "areaStyle": {"opacity": 0.3},
        }],
    }
}
```

## Why This Works Without Extra Dependencies

This is the key architectural advantage of the ECharts approach:

| Concern | Plotly approach | ECharts approach |
|---------|----------------|------------------|
| **LLM output** | Python code importing `plotly.express` | Python dict literal (no imports) |
| **Sandbox whitelist** | Must add `"plotly"` to `custom_whitelisted_dependencies` | No extra whitelist needed -- dicts are native Python |
| **Package install** | `pip install pandasai[plotly]` + `plotly` | Nothing -- ECharts is built into Solara |
| **Serialization** | `go.Figure` objects must serialize over WebSocket | Plain dicts serialize trivially as JSON |
| **Solara component** | `FigurePlotly` (requires `plotly` package) | `FigureEcharts` (built into Solara, zero dependencies) |

The LLM produces a **plain Python dict**. PandasAI executes it with no special imports. The dict travels through the ResponseParser, through the WebSocket, and arrives at `FigureEcharts` as-is.

## Error Handling

### Invalid ECharts Options

The LLM may generate a malformed option dict. Wrap rendering in error handling:

```python
@solara.component
def EChartsChartMessage(option: dict):
    dark = solara.lab.use_dark_effective()
    themed_option = solara.use_memo(lambda: _apply_theme(option, dark), [option, dark])

    if not option.get("series"):
        solara.Warning("Chart generated but missing series data. Try rephrasing your question.")
        return

    try:
        solara.FigureEcharts(
            option=themed_option,
            responsive=True,
            attributes={"style": "width: 100%; height: 400px; min-width: 300px;"},
        )
    except Exception as e:
        solara.Error(f"Failed to render chart: {e}")
        with solara.Details("Raw chart data"):
            solara.Markdown(f"```json\n{json.dumps(option, indent=2, default=str)}\n```")
```

### LLM Falls Back to Matplotlib

Despite the prompt, the LLM may occasionally revert to matplotlib. The `SolaraResponseParser.format_plot()` detects file paths and tags them as `"image"` type. The `ChatMessageContent` component renders these as `solara.Image(path)` -- degraded but not broken.

### Non-Serializable Values

ECharts options must be JSON-serializable. If the LLM puts numpy arrays or pandas objects in the dict, convert them:

```python
import json
import numpy as np
import pandas as pd

def sanitize_echarts_option(option: dict) -> dict:
    """Ensure all values in an ECharts option dict are JSON-serializable."""
    def convert(obj):
        if isinstance(obj, (np.integer,)):
            return int(obj)
        if isinstance(obj, (np.floating,)):
            return float(obj)
        if isinstance(obj, np.ndarray):
            return obj.tolist()
        if isinstance(obj, pd.Timestamp):
            return obj.isoformat()
        if isinstance(obj, dict):
            return {k: convert(v) for k, v in obj.items()}
        if isinstance(obj, (list, tuple)):
            return [convert(i) for i in obj]
        return obj
    return convert(option)
```

Call this in the ResponseParser:

```python
def format_plot(self, result):
    value = result.get("value") if isinstance(result, dict) else result
    if _is_echarts_option(value):
        return {"type": "echarts", "value": sanitize_echarts_option(value)}
    # ... fallbacks
```

## Sizing Charts in Chat Messages

Chat messages have constrained width. The ECharts container needs explicit sizing:

```python
solara.FigureEcharts(
    option=themed_option,
    responsive=True,
    attributes={
        "style": "width: 100%; height: 400px; min-width: 300px;",
        "class": "echarts-chat-chart",
    },
)
```

Add CSS to handle the chat context:

```css
/* assets/custom.css */
.echarts-chat-chart {
    border-radius: 8px;
    margin: 8px 0;
}

/* Prevent chat message from constraining chart width */
.solara-chat-message .echarts-chat-chart {
    min-width: 300px;
    max-width: 100%;
}
```

## Static Image Export for PowerPoint

ECharts charts rendered in the browser cannot use Plotly's simple `to_image()` for PPTX export (see PRD 07). Two options:

**Option A: Convert ECharts data to Plotly at export time**

Since we have the raw data in the ECharts option dict, convert it to a Plotly figure only when PPTX export is requested:

```python
import plotly.graph_objects as go

def echarts_to_plotly(option: dict) -> go.Figure:
    """Convert an ECharts option dict to a Plotly figure for static export."""
    fig = go.Figure()

    for series in option.get("series", []):
        chart_type = series.get("type", "bar")
        data = series.get("data", [])
        name = series.get("name", "")

        x_data = option.get("xAxis", {}).get("data", list(range(len(data))))
        if isinstance(x_data, dict):
            x_data = x_data.get("data", list(range(len(data))))

        if chart_type == "bar":
            fig.add_trace(go.Bar(x=x_data, y=data, name=name))
        elif chart_type == "line":
            fig.add_trace(go.Scatter(x=x_data, y=data, mode="lines", name=name))
        elif chart_type == "scatter":
            fig.add_trace(go.Scatter(x=x_data, y=data, mode="markers", name=name))
        elif chart_type == "pie":
            labels = [d["name"] for d in data if isinstance(d, dict)]
            values = [d["value"] for d in data if isinstance(d, dict)]
            fig.add_trace(go.Pie(labels=labels, values=values, name=name))

    title = option.get("title", {}).get("text", "")
    fig.update_layout(title=title, template="plotly_white")
    return fig
```

Then in the PPTX export pipeline:

```python
def export_message_to_slide(builder, msg):
    if msg["type"] == "echarts":
        fig = echarts_to_plotly(msg["content"])
        builder.add_chart_slide(fig=fig, title=msg.get("query", "Chart"))
```

**Option B: Use pyppeteer for server-side ECharts rendering** (heavier, but pixel-perfect)

```python
# Requires: pip install pyppeteer snapshot-pyppeteer
# Renders ECharts option as PNG via headless Chrome
```

Option A is recommended for simplicity. Option B preserves exact ECharts styling but adds deployment complexity.

## Dependencies

```toml
[project]
dependencies = [
    "solara>=1.35",
    "pandasai>=2.3,<3.0",
    "pandas>=2.0",
    # ECharts: ZERO extra dependencies (built into Solara)
    # Plotly only needed if using PPTX export (PRD 07):
    "plotly>=5.18",
    "kaleido>=1.0",
]
```

## Implementation Plan

### Phase 1: Core Pipeline
1. Write `ECHARTS_AGENT_DESCRIPTION` prompt with ECharts generation instructions and examples
2. Implement `SolaraResponseParser` with `_is_echarts_option()` detection and `sanitize_echarts_option()`
3. Implement `AIService` wrapping the Agent with description and parser
4. Implement `EChartsChartMessage` component with theme handling

### Phase 2: Chat Integration
5. Implement `ChatMessageContent` dispatcher (echarts/dataframe/text/image/error)
6. Build the `ChatArea` page with `ChatBox`, `ChatMessage`, and `ChatInput`
7. Add processing indicator (spinning `v.ProgressCircular` in a ChatMessage)
8. Add query suggestion chips

### Phase 3: Interactivity
9. Add `on_click` handlers for drill-down follow-up queries
10. Add chart sizing CSS for chat message context
11. Test across chart types: bar, line, pie, scatter, heatmap

### Phase 4: Export
12. Implement `echarts_to_plotly()` converter for PPTX export
13. Wire export button into chat toolbar

## Acceptance Criteria

1. Asking "Plot revenue by region" in the chat produces an interactive ECharts bar chart inside the assistant's message bubble
2. Charts support hover (tooltips show values), zoom (via dataZoom on time series), and click events
3. Charts adapt to the app's dark/light theme
4. If the LLM falls back to matplotlib, a static image is rendered (not a crash)
5. ECharts option dicts with numpy/pandas types are sanitized before rendering
6. Charts resize responsively when the chat container width changes
7. No extra Python packages are required beyond Solara and PandasAI for chart rendering
8. PPTX export converts ECharts data to Plotly figures for static image generation
