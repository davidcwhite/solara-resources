# ECharts in Solara Chat: Rendering Cheatsheet

PandasAI generates an ECharts option dict (plain Python dict). `solara.FigureEcharts` renders it inside `solara.lab.ChatMessage`. That's the whole pattern.

## Pipeline

```
agent.chat(query)
  → LLM generates code that assigns result = {"type": "plot", "value": {echarts option dict}}
  → PandasAI executes in sandbox, extracts result
  → ResponseParser detects ECharts dict, sanitizes numpy/pandas types, tags as "echarts"
  → AIService returns {"type": "echarts", "value": dict, "query": str}
  → ChatMessageContent dispatches to EChartsChartMessage
  → FigureEcharts(option=dict) renders inside ChatMessage
```

## ResponseParser

Detects ECharts dicts by key presence. Sanitizes numpy/pandas types to JSON-serializable Python. Falls back gracefully on matplotlib output.

```python
from pandasai.responses.response_parser import ResponseParser
import numpy as np
import pandas as pd

ECHARTS_KEYS = {"series", "xAxis", "yAxis", "tooltip", "title", "legend",
                "radar", "geo", "parallel", "calendar", "dataset"}


def _is_echarts_option(value) -> bool:
    return isinstance(value, dict) and bool(ECHARTS_KEYS & set(value.keys()))


def _sanitize(obj):
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

## Chat Rendering

### Message shape

```python
{"role": "user"|"assistant", "type": "echarts"|"text"|"dataframe"|"image"|"error",
 "content": dict|str|pd.DataFrame, "query": str}
```

### Components

```python
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
    dark = solara.lab.use_dark_effective()
    themed = solara.use_memo(lambda: _apply_theme(option, dark), [option, dark])

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
    pass  # {name, value, dataIndex, seriesName, ...}


def _apply_theme(option: dict, dark: bool) -> dict:
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

### Wiring

```python
with solara.lab.ChatBox():
    for msg in messages.value:
        with solara.lab.ChatMessage(
            user=(msg["role"] == "user"),
            name="You" if msg["role"] == "user" else "Assistant",
        ):
            ChatMessageContent(message=msg)
```

## Gotchas

| Problem | Cause | Fix |
|---------|-------|-----|
| Blank chart, no error | Malformed option dict (missing `series`, wrong data shape) | Validate `option.get("series")` before rendering |
| Chart collapses to 0px | No explicit height on `FigureEcharts` | `attributes={"style": "height: 400px;"}` (not `style=`) |
| Crash on render | numpy/pandas types in option dict | `_sanitize()` converts to native Python types |
| Chart appears before data ready | Incremental state mutation | Append complete message dict to reactive list in one assignment |
| LLM ignores ECharts instructions | No few-shot examples for chart type | Add example bank entry with query + code for that chart pattern |
| matplotlib output instead of dict | LLM fallback | ResponseParser tags as `"image"`, renders `solara.Image(path)` |

## Example Bank Entry Format

Each entry stored in the example bank is a `(query, code)` pair. The code runs in PandasAI's sandbox with `dfs` available. Output is always `result = {"type": "plot", "value": {option dict}}`.

**Bar:**
```python
# query: "Plot revenue by region"
df = dfs[0]
result = {"type": "plot", "value": {
    "title": {"text": "Revenue by Region"},
    "tooltip": {"trigger": "axis"},
    "xAxis": {"type": "category", "data": df["region"].tolist()},
    "yAxis": {"type": "value"},
    "series": [{"type": "bar", "data": df["revenue"].tolist()}],
}}
```

**Stacked bar** -- `"stack"` groups series:
```python
# query: "Show quarterly revenue by product line, stacked"
df = dfs[0]
quarters = df["quarter"].unique().tolist()
result = {"type": "plot", "value": {
    "tooltip": {"trigger": "axis"}, "legend": {},
    "xAxis": {"type": "category", "data": quarters},
    "yAxis": {"type": "value"},
    "series": [
        {"name": name, "type": "bar", "stack": "total",
         "data": grp.set_index("quarter").reindex(quarters)["revenue"].fillna(0).tolist()}
        for name, grp in df.groupby("product_line")
    ],
}}
```

**Dual Y-axis** -- `yAxis` as list, `yAxisIndex` on series:
```python
# query: "Revenue bars with profit margin line on secondary axis"
df = dfs[0]
result = {"type": "plot", "value": {
    "tooltip": {"trigger": "axis"}, "legend": {},
    "xAxis": {"type": "category", "data": df["month"].tolist()},
    "yAxis": [
        {"type": "value", "name": "Revenue ($)", "position": "left"},
        {"type": "value", "name": "Margin (%)", "position": "right", "max": 100},
    ],
    "series": [
        {"name": "Revenue", "type": "bar", "yAxisIndex": 0, "data": df["revenue"].tolist()},
        {"name": "Margin", "type": "line", "yAxisIndex": 1, "smooth": True,
         "data": df["margin_pct"].tolist()},
    ],
}}
```

**Box plot** -- pre-computed `[min, Q1, median, Q3, max]`:
```python
# query: "Salary distribution by department"
df = dfs[0]
cats, data = [], []
for dept, grp in df.groupby("department"):
    s = grp["salary"]
    cats.append(dept)
    data.append([float(s.min()), float(s.quantile(.25)), float(s.median()),
                 float(s.quantile(.75)), float(s.max())])
result = {"type": "plot", "value": {
    "tooltip": {"trigger": "item"},
    "xAxis": {"type": "category", "data": cats},
    "yAxis": {"type": "value"},
    "series": [{"type": "boxplot", "data": data}],
}}
```

**Heatmap** -- `[x, y, value]` triples + `visualMap`:
```python
# query: "Sales heatmap by day and hour"
df = dfs[0]
days = ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"]
hours = [str(h) for h in range(24)]
pivot = df.pivot_table(index="day_of_week", columns="hour", values="sales", aggfunc="sum", fill_value=0)
data = [[j, i, int(pivot.loc[day, j]) if day in pivot.index else 0]
        for i, day in enumerate(days) for j in range(24)]
result = {"type": "plot", "value": {
    "tooltip": {"position": "top"},
    "xAxis": {"type": "category", "data": hours},
    "yAxis": {"type": "category", "data": days},
    "visualMap": {"min": 0, "max": max(d[2] for d in data), "calculable": True},
    "series": [{"type": "heatmap", "data": data}],
}}
```
