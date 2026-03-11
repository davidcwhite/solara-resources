# PRD 08: Interactive ECharts in Solara Chat via PandasAI

## Core Concept

PandasAI's code generation pipeline currently produces matplotlib code that saves static PNGs. We redirect it to produce **ECharts option dicts** -- plain Python dictionaries -- which Solara's built-in `FigureEcharts` component renders as interactive charts directly inside chat message bubbles.

The key insight: **the dynamic part is the data, not the component.** You don't need to dynamically generate Solara components. `FigureEcharts` is already the right component -- the pipeline just needs to produce the option dict, and the chat renderer conditionally mounts `FigureEcharts` when a message payload contains one.

An ECharts option dict is native Python -- no imports, no sandbox concerns, no extra packages. This is the critical advantage over Plotly, which requires whitelisting and installing a separate library.

## Pipeline Flow

```
1. agent.chat("Plot revenue by region")
       │
2. PandasAI builds prompt (query + DataFrame schema + Agent description)
   │  Example bank retrieves top-k similar query-code pairs via semantic search
   │  and injects them as few-shot context alongside the agent description
       │
3. LLM generates Python code that builds an ECharts option dict
   │  (no matplotlib, no imports -- just a dict literal)
   │  result = {"type": "plot", "value": {echarts option...}}
       │
4. PandasAI executes code in sandbox, extracts `result`
       │
5. SolaraResponseParser.format_plot() detects ECharts dict, tags as "echarts"
       │
6. AIService returns {"type": "echarts", "value": dict, "query": str}
       │
7. Chat renderer: ChatMessage → ChatMessageContent → FigureEcharts(option=dict)
       │
8. Interactive chart in browser (hover, zoom, click)
```

## Step 1: Agent Description

The `description` parameter on PandasAI's `Agent` class is injected into every LLM prompt. This steers code generation away from matplotlib toward ECharts option dicts.

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

## Step 2: ResponseParser

Detects ECharts option dicts via key sniffing, sanitizes non-JSON-serializable types (numpy, pandas), and tags the response so the frontend knows to render `FigureEcharts`.

```python
# services/response_parser.py
from pandasai.responses.response_parser import ResponseParser
import numpy as np
import pandas as pd
import logging

logger = logging.getLogger(__name__)

ECHARTS_KEYS = {"series", "xAxis", "yAxis", "tooltip", "title", "legend",
                "radar", "geo", "parallel", "calendar", "dataset"}


def _is_echarts_option(value) -> bool:
    return isinstance(value, dict) and bool(ECHARTS_KEYS & set(value.keys()))


def _sanitize(obj):
    """Convert numpy/pandas types to JSON-serializable Python types."""
    if isinstance(obj, (np.integer,)):
        return int(obj)
    if isinstance(obj, (np.floating,)):
        return float(obj)
    if isinstance(obj, np.ndarray):
        return obj.tolist()
    if isinstance(obj, pd.Timestamp):
        return obj.isoformat()
    if isinstance(obj, dict):
        return {k: _sanitize(v) for k, v in obj.items()}
    if isinstance(obj, (list, tuple)):
        return [_sanitize(i) for i in obj]
    return obj


class SolaraResponseParser(ResponseParser):
    def __init__(self, context) -> None:
        super().__init__(context)

    def format_plot(self, result):
        value = result.get("value") if isinstance(result, dict) else result

        if _is_echarts_option(value):
            return {"type": "echarts", "value": _sanitize(value)}

        if isinstance(value, str) and value.endswith((".png", ".jpg", ".svg")):
            logger.warning("LLM generated matplotlib; falling back to image")
            return {"type": "image", "value": value}

        return {"type": "plot", "value": value}

    def format_dataframe(self, result):
        value = result.get("value", result) if isinstance(result, dict) else result
        if isinstance(value, pd.DataFrame):
            return {"type": "dataframe", "value": value}
        return {"type": "text", "value": str(value)}

    def format_response(self, result):
        value = result.get("value", result) if isinstance(result, dict) else result
        return {"type": "text", "value": str(value)}
```

## Step 3: AIService

Wraps the Agent with the ECharts description and custom parser. Normalizes all responses into a consistent shape.

```python
# services/ai_service.py
from pandasai import Agent
from services.response_parser import SolaraResponseParser, _is_echarts_option
import pandas as pd
import logging

logger = logging.getLogger(__name__)


class AIService:
    def __init__(self, df: pd.DataFrame, llm):
        self.agent = Agent(
            df,
            config={
                "llm": llm,
                "response_parser": SolaraResponseParser,
                "save_charts": False,
                "open_charts": False,
            },
            description=ECHARTS_AGENT_DESCRIPTION,
        )

    def query(self, message: str) -> dict:
        """Returns {"type": "echarts"|"dataframe"|"text"|"image"|"error",
                    "value": dict|DataFrame|str, "query": str}"""
        try:
            raw = self.agent.chat(message)

            if isinstance(raw, dict) and "type" in raw:
                raw["query"] = message
                return raw
            if _is_echarts_option(raw):
                return {"type": "echarts", "value": raw, "query": message}
            if isinstance(raw, pd.DataFrame):
                return {"type": "dataframe", "value": raw, "query": message}
            return {"type": "text", "value": str(raw), "query": message}

        except Exception as e:
            logger.exception("PandasAI query failed")
            return {"type": "error", "value": str(e), "query": message}
```

## Step 4: Chat Rendering

Solara's `ChatMessage` accepts **any Solara component** as children via the `with` context manager. The message data model stores the ECharts option dict in the payload, and a dispatcher component conditionally renders `FigureEcharts` vs `Markdown` vs `DataFrame` based on type.

### Message data model

```python
{
    "role": "user" | "assistant",
    "type": "text" | "echarts" | "dataframe" | "image" | "error",
    "content": str | dict | pd.DataFrame,
    "query": str,  # original user question (assistant messages only)
}
```

### Components

```python
# components/chat.py
import solara
import pandas as pd
import json


@solara.component
def ChatMessageContent(message: dict):
    """Dispatches to the right renderer based on message type."""
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
    """Renders an interactive ECharts chart inside a chat message.
    Handles theming, validation, sizing, and click events."""
    dark = solara.lab.use_dark_effective()
    themed = solara.use_memo(lambda: _apply_theme(option, dark), [option, dark])

    # Malformed specs render a blank chart silently -- validate first
    if not option.get("series"):
        solara.Warning("Chart data missing. Try rephrasing your question.")
        return

    solara.FigureEcharts(
        option=themed,
        on_click=_on_chart_click,
        responsive=True,
        attributes={"style": "width: 100%; height: 400px; min-width: 300px;"},
    )


def _on_chart_click(event_data):
    """Receives {name, value, dataIndex, seriesName, ...} on click."""
    pass  # Wire to drill-down or follow-up query as needed


def _apply_theme(option: dict, dark: bool) -> dict:
    """Non-destructively overlay dark/light colors onto an ECharts option."""
    themed = {**option, "backgroundColor": "transparent"}
    text_color = "#ccc" if dark else "#333"
    line_color = "#555" if dark else "#ccc"

    if "title" in themed:
        themed["title"] = {**themed["title"], "textStyle": {"color": text_color}}
    if "legend" in themed:
        themed["legend"] = {**themed["legend"], "textStyle": {"color": text_color}}

    for key in ("xAxis", "yAxis"):
        if key not in themed:
            continue
        axis = themed[key]
        if isinstance(axis, dict):
            themed[key] = {**axis,
                "axisLabel": {**axis.get("axisLabel", {}), "color": text_color},
                "axisLine": {"lineStyle": {"color": line_color}}}
        elif isinstance(axis, list):
            themed[key] = [{**a,
                "axisLabel": {**a.get("axisLabel", {}), "color": text_color}}
                for a in axis]

    return themed
```

### Wiring into the chat page

```python
# In pages/explorer.py
with solara.lab.ChatBox():
    for msg in messages.value:
        with solara.lab.ChatMessage(
            user=(msg["role"] == "user"),
            name="You" if msg["role"] == "user" else "Assistant",
        ):
            ChatMessageContent(message=msg)
```

## ECharts Example Bank Entries

The example bank (vector store with semantic search) already exists. At query time, the top-k most similar entries are retrieved and injected into the LLM prompt as few-shot context. This section catalogs the **code examples** for each chart type that should be stored in the bank.

Each entry is a `(query, code)` pair. The code runs in PandasAI's sandbox where `dfs` (the list of dataframes) is provided. No imports are needed -- the output is always a plain dict assigned to `result`.

`FigureEcharts` renders any valid ECharts option dict regardless of chart type -- the rendering pipeline is chart-type-agnostic. The diversity is entirely in the option dict structure.

### Bar Chart

```python
# query: "Plot revenue by region as a bar chart"
df = dfs[0]
result = {
    "type": "plot",
    "value": {
        "title": {"text": "Revenue by Region"},
        "tooltip": {"trigger": "axis", "axisPointer": {"type": "shadow"}},
        "xAxis": {"type": "category", "data": df["region"].tolist()},
        "yAxis": {"type": "value", "name": "Revenue ($)"},
        "series": [{"type": "bar", "data": df["revenue"].tolist(),
                    "itemStyle": {"borderRadius": [4, 4, 0, 0]}}],
    }
}
```

### Pie / Donut Chart

```python
# query: "Show distribution of sales by category as a pie chart"
df = dfs[0]
grouped = df.groupby("category")["sales"].sum().reset_index()
result = {
    "type": "plot",
    "value": {
        "title": {"text": "Sales by Category"},
        "tooltip": {"trigger": "item", "formatter": "{b}: {c} ({d}%)"},
        "series": [{"type": "pie", "radius": ["40%", "70%"],
                    "data": [{"name": r["category"], "value": int(r["sales"])}
                             for _, r in grouped.iterrows()]}],
    }
}
```

### Stacked Bar Chart

The `"stack"` key groups series into a single stacked column. All series sharing the same `stack` value are stacked together.

```python
# query: "Show quarterly revenue by product line as a stacked bar chart"
df = dfs[0]
quarters = df["quarter"].unique().tolist()
result = {
    "type": "plot",
    "value": {
        "title": {"text": "Revenue by Product Line (Stacked)"},
        "tooltip": {"trigger": "axis", "axisPointer": {"type": "shadow"}},
        "legend": {},
        "xAxis": {"type": "category", "data": quarters},
        "yAxis": {"type": "value", "name": "Revenue ($)"},
        "series": [
            {"name": name,
             "type": "bar",
             "stack": "total",
             "data": grp.set_index("quarter").reindex(quarters)["revenue"].fillna(0).tolist()}
            for name, grp in df.groupby("product_line")
        ],
    }
}
```

### Bar + Line Combo with Dual Y-Axis

Use `yAxis` as a **list** of two axis configs. Each series references its axis via `yAxisIndex`.

```python
# query: "Show monthly revenue as bars and profit margin as a line on a secondary axis"
df = dfs[0]
months = df["month"].tolist()
result = {
    "type": "plot",
    "value": {
        "title": {"text": "Revenue & Profit Margin"},
        "tooltip": {"trigger": "axis"},
        "legend": {},
        "xAxis": {"type": "category", "data": months},
        "yAxis": [
            {"type": "value", "name": "Revenue ($)", "position": "left"},
            {"type": "value", "name": "Margin (%)", "position": "right",
             "axisLabel": {"formatter": "{value}%"}, "max": 100},
        ],
        "series": [
            {"name": "Revenue", "type": "bar", "yAxisIndex": 0,
             "data": df["revenue"].tolist()},
            {"name": "Margin", "type": "line", "yAxisIndex": 1,
             "smooth": True, "data": df["margin_pct"].tolist()},
        ],
    }
}
```

Note: `_apply_theme()` already handles `yAxis` as both a dict and a list (see Chat Rendering section), so dual-axis charts theme correctly without changes.

### Box Plot

ECharts boxplot expects pre-computed five-number summaries: `[min, Q1, median, Q3, max]` per category.

```python
# query: "Show salary distribution by department as a box plot"
df = dfs[0]
categories = []
box_data = []
for dept, grp in df.groupby("department"):
    s = grp["salary"]
    categories.append(dept)
    box_data.append([
        float(s.min()), float(s.quantile(0.25)), float(s.median()),
        float(s.quantile(0.75)), float(s.max())
    ])
result = {
    "type": "plot",
    "value": {
        "title": {"text": "Salary Distribution by Department"},
        "tooltip": {"trigger": "item"},
        "xAxis": {"type": "category", "data": categories},
        "yAxis": {"type": "value", "name": "Salary ($)"},
        "series": [{"type": "boxplot", "data": box_data}],
    }
}
```

### Heatmap

Heatmap data is a list of `[xIndex, yIndex, value]` triples. Requires `visualMap` to map values to colors.

```python
# query: "Show a heatmap of sales by day of week and hour"
df = dfs[0]
days = ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"]
hours = [str(h) for h in range(24)]
pivot = df.pivot_table(index="day_of_week", columns="hour", values="sales",
                       aggfunc="sum", fill_value=0)
data = []
for i, day in enumerate(days):
    for j in range(24):
        data.append([j, i, int(pivot.loc[day, j]) if day in pivot.index else 0])
max_val = max(d[2] for d in data) if data else 1
result = {
    "type": "plot",
    "value": {
        "title": {"text": "Sales Heatmap: Day x Hour"},
        "tooltip": {"position": "top",
                    "formatter": "{c} sales"},
        "xAxis": {"type": "category", "data": hours, "splitArea": {"show": True}},
        "yAxis": {"type": "category", "data": days, "splitArea": {"show": True}},
        "visualMap": {"min": 0, "max": max_val, "calculable": True,
                      "orient": "horizontal", "left": "center", "bottom": 0},
        "series": [{"type": "heatmap", "data": data,
                    "label": {"show": False},
                    "emphasis": {"itemStyle": {"shadowBlur": 10}}}],
    }
}
```

### Authoring Guidelines for New Examples

When adding entries to the example bank:

1. **One example per chart pattern.** Each entry should demonstrate a single ECharts chart type or variation (e.g., stacked bar is separate from grouped bar).
2. **Keep data manipulation minimal.** The example bank teaches the LLM the ECharts option dict structure, not pandas. Use simple `groupby`, `pivot_table`, or column access.
3. **Always cast to native types.** Use `.tolist()`, `int()`, `float()` on any pandas/numpy values to avoid serialization failures downstream.
4. **Include tooltip and title.** Every example should set `tooltip` (with appropriate `trigger`) and `title` so the LLM learns to include them consistently.
5. **Match query phrasing to real user language.** The semantic search retrieves by similarity to the user's query, so phrase example queries naturally (e.g., "Show me a breakdown of..." not "Generate an ECharts option dict for...").

## Why This Works Without Extra Dependencies

| Concern | Plotly | ECharts |
|---------|--------|---------|
| **LLM output** | Python code importing `plotly.express` | Plain dict literal -- no imports |
| **Sandbox** | Must whitelist `plotly` | Nothing -- dicts are native Python |
| **Install** | `pip install pandasai[plotly]` + `plotly` | Nothing -- `FigureEcharts` is built into Solara |
| **Serialization** | `go.Figure` must serialize over WebSocket | Dicts serialize trivially as JSON |

## Practical Gotchas

**Blank chart, no error.** A malformed ECharts option (e.g., missing `series` or wrong data shape) renders a blank chart container -- it does not throw an exception. Always validate `option.get("series")` before rendering.

**Height collapse.** `FigureEcharts` inside a chat bubble will collapse to zero height without explicit dimensions. The `attributes` parameter (not `style`) controls this: `attributes={"style": "height: 400px;"}`. The Solara docs confirm this is the correct API -- the default is `attributes={"style": "height: 400px;"}`.

**State timing.** The ECharts option dict must be stored in the reactive message list *before* the `ChatMessage` renders. Append the complete message (with the option dict in `content`) to the reactive list in one assignment, not incrementally.

**Non-serializable types.** The LLM-generated code operates on pandas DataFrames, so the option dict may contain `numpy.int64`, `numpy.float64`, `numpy.ndarray`, or `pd.Timestamp` values. These must be converted to native Python types before reaching `FigureEcharts`. The `_sanitize()` function in the ResponseParser handles this.

**Matplotlib fallback.** Despite prompt instructions, the LLM occasionally reverts to matplotlib. The ResponseParser detects file path strings (`.png`, `.jpg`) and tags them as `"image"` type. The chat renderer shows `solara.Image(path)` -- degraded but functional.

**Why not `solara.HTML` with raw ECharts JS.** You could inject ECharts via an iframe or `solara.HTML`, but this loses the ipywidget bidirectional event model -- no `on_click`, `on_mouseover`, or `on_mouseout` callbacks. `FigureEcharts` is a proper Solara/ipywidget component that preserves these.

## PPTX Export Bridge

ECharts charts live in the browser and have no Python-side `to_image()`. For PowerPoint export (PRD 07), convert the stored ECharts data to a Plotly figure at export time:

```python
import plotly.graph_objects as go

def echarts_to_plotly(option: dict) -> go.Figure:
    fig = go.Figure()
    for series in option.get("series", []):
        chart_type = series.get("type", "bar")
        data = series.get("data", [])
        name = series.get("name", "")
        x_data = option.get("xAxis", {})
        if isinstance(x_data, dict):
            x_data = x_data.get("data", list(range(len(data))))

        if chart_type == "bar":
            fig.add_trace(go.Bar(x=x_data, y=data, name=name))
        elif chart_type == "line":
            fig.add_trace(go.Scatter(x=x_data, y=data, mode="lines", name=name))
        elif chart_type == "pie":
            labels = [d["name"] for d in data if isinstance(d, dict)]
            values = [d["value"] for d in data if isinstance(d, dict)]
            fig.add_trace(go.Pie(labels=labels, values=values))

    title = option.get("title", {}).get("text", "")
    fig.update_layout(title=title, template="plotly_white")
    return fig
```

Then: `fig.to_image(format="png", scale=2)` via Kaleido for the PPTX slide.

## Dependencies

```toml
[project]
dependencies = [
    "solara>=1.35",
    "pandasai>=2.3,<3.0",
    "pandas>=2.0",
    # ECharts: zero extra deps (built into Solara)
    # Only needed for PPTX export (PRD 07):
    "plotly>=5.18",
    "kaleido>=1.0",
]
```

## Implementation Plan

1. Write `ECHARTS_AGENT_DESCRIPTION` prompt with examples
2. Implement `SolaraResponseParser` with detection + sanitization
3. Implement `AIService` wrapping Agent with description and parser
4. Implement `EChartsChartMessage` component (theming, validation, sizing, click)
5. Implement `ChatMessageContent` dispatcher
6. Wire into chat page with `ChatBox`/`ChatMessage`/`ChatInput`
7. Add processing indicator during PandasAI execution
8. Author example bank entries for each chart type (bar, pie, stacked bar, dual-axis, boxplot, heatmap) and seed into vector store
9. Test LLM output quality across chart types with and without few-shot examples; iterate on example phrasing
10. Implement `echarts_to_plotly()` bridge for PPTX export

## Acceptance Criteria

1. Chart requests produce interactive ECharts inside chat message bubbles
2. Charts support hover tooltips, zoom (dataZoom), and click events
3. Charts adapt to dark/light theme
4. Matplotlib fallback renders a static image (not a crash)
5. numpy/pandas types are sanitized before rendering
6. Charts resize responsively within the chat container
7. No extra Python packages required for chart rendering (only Solara + PandasAI)
8. Complex chart types (stacked bar, dual-axis, boxplot, heatmap) render correctly when guided by example bank entries
9. Dual-axis charts theme correctly (both axes pick up dark/light colors)
