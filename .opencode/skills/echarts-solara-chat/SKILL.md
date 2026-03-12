---
name: echarts-solara-chat
description: "Render PandasAI-generated ECharts inside Solara chat: pipeline, components, theming, gotchas, and example bank patterns"
version: 1.0.0
---

# ECharts in Solara Chat

Render interactive ECharts charts inside `solara.lab.ChatMessage` from PandasAI-generated code. The LLM produces a plain Python dict (ECharts option spec), `solara.FigureEcharts` renders it. No extra dependencies -- ECharts is built into Solara.

## Pipeline

```
agent.chat(query)
  → Example bank injects similar query-code pairs as few-shot context
  → LLM generates: result = {"type": "plot", "value": {echarts option dict}}
  → PandasAI executes in sandbox, extracts result
  → ResponseParser detects ECharts dict, sanitizes numpy/pandas types, tags "echarts"
  → ChatMessageContent dispatches to EChartsChartMessage
  → FigureEcharts(option=dict) inside ChatMessage
```

## ResponseParser

Detect via key sniffing. Sanitize numpy/pandas to JSON-serializable types. Fall back on matplotlib.

```python
from pandasai.responses.response_parser import ResponseParser
import numpy as np, pandas as pd

ECHARTS_KEYS = {"series", "xAxis", "yAxis", "tooltip", "title", "legend",
                "radar", "geo", "parallel", "calendar", "dataset"}

def _is_echarts_option(value) -> bool:
    return isinstance(value, dict) and bool(ECHARTS_KEYS & set(value.keys()))

def _sanitize(obj):
    if isinstance(obj, (np.integer,)):   return int(obj)
    if isinstance(obj, (np.floating,)):  return float(obj)
    if isinstance(obj, np.ndarray):      return obj.tolist()
    if isinstance(obj, pd.Timestamp):    return obj.isoformat()
    if isinstance(obj, dict):            return {k: _sanitize(v) for k, v in obj.items()}
    if isinstance(obj, (list, tuple)):   return [_sanitize(i) for i in obj]
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
        return {"type": "dataframe", "value": value} if isinstance(value, pd.DataFrame) else {"type": "text", "value": str(value)}

    def format_response(self, result):
        value = result.get("value", result) if isinstance(result, dict) else result
        return {"type": "text", "value": str(value)}
```

## Chat Components

Message shape: `{"role": "user"|"assistant", "type": "echarts"|"text"|"dataframe"|"image"|"error", "content": dict|str|DataFrame, "query": str}`

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
        option=themed, on_click=_on_chart_click, responsive=True,
        attributes={"style": "width: 100%; height: 400px; min-width: 300px;"},
    )

def _on_chart_click(event_data):
    pass  # {name, value, dataIndex, seriesName, ...}

def _apply_theme(option: dict, dark: bool) -> dict:
    themed = {**option, "backgroundColor": "transparent"}
    tc = "#ccc" if dark else "#333"
    lc = "#555" if dark else "#ccc"
    if "title" in themed:
        themed["title"] = {**themed["title"], "textStyle": {"color": tc}}
    if "legend" in themed:
        themed["legend"] = {**themed["legend"], "textStyle": {"color": tc}}
    for key in ("xAxis", "yAxis"):
        if key not in themed: continue
        axis = themed[key]
        if isinstance(axis, dict):
            themed[key] = {**axis, "axisLabel": {**axis.get("axisLabel", {}), "color": tc},
                           "axisLine": {"lineStyle": {"color": lc}}}
        elif isinstance(axis, list):
            themed[key] = [{**a, "axisLabel": {**a.get("axisLabel", {}), "color": tc}} for a in axis]
    return themed
```

Wire into the chat page:

```python
with solara.lab.ChatBox():
    for msg in messages.value:
        with solara.lab.ChatMessage(user=(msg["role"] == "user"),
                                    name="You" if msg["role"] == "user" else "Assistant"):
            ChatMessageContent(message=msg)
```

## Gotchas

| Problem | Fix |
|---------|-----|
| Blank chart, no error | Malformed option -- validate `option.get("series")` before render |
| Chart collapses to 0px | Set `attributes={"style": "height: 400px;"}` (not `style=`) |
| Crash on render | Unsanitized numpy/pandas types -- run `_sanitize()` |
| Chart before data ready | Append complete message dict to reactive list in one assignment |
| matplotlib fallback | ResponseParser tags as `"image"`, renders `solara.Image(path)` |

## Example Bank Patterns

Each example bank entry is `(query, code)`. Code runs in PandasAI sandbox with `dfs` available. Always output `result = {"type": "plot", "value": {option}}`.

### Bar

```python
result = {"type": "plot", "value": {
    "tooltip": {"trigger": "axis"},
    "xAxis": {"type": "category", "data": df["region"].tolist()},
    "yAxis": {"type": "value"},
    "series": [{"type": "bar", "data": df["revenue"].tolist()}],
}}
```

### Stacked Bar

`"stack"` groups series together:

```python
"series": [
    {"name": name, "type": "bar", "stack": "total",
     "data": grp["revenue"].tolist()}
    for name, grp in df.groupby("product_line")
]
```

### Dual Y-Axis (Bar + Line)

`yAxis` as list, `yAxisIndex` on each series:

```python
"yAxis": [
    {"type": "value", "name": "Revenue", "position": "left"},
    {"type": "value", "name": "Margin %", "position": "right", "max": 100},
],
"series": [
    {"name": "Revenue", "type": "bar", "yAxisIndex": 0, "data": revenue_list},
    {"name": "Margin", "type": "line", "yAxisIndex": 1, "data": margin_list},
]
```

`_apply_theme()` handles `yAxis` as both dict and list -- dual-axis themes correctly.

### Box Plot

Pre-compute `[min, Q1, median, Q3, max]` per category:

```python
"xAxis": {"type": "category", "data": department_names},
"series": [{"type": "boxplot", "data": [
    [float(s.min()), float(s.quantile(.25)), float(s.median()),
     float(s.quantile(.75)), float(s.max())]
    for _, s in groups
]}]
```

### Heatmap

`[xIndex, yIndex, value]` triples with `visualMap`:

```python
"xAxis": {"type": "category", "data": hour_labels},
"yAxis": {"type": "category", "data": day_labels},
"visualMap": {"min": 0, "max": max_val, "calculable": True},
"series": [{"type": "heatmap", "data": [[x, y, val], ...]}]
```

### Pie / Donut

```python
"series": [{"type": "pie", "radius": ["40%", "70%"],
            "data": [{"name": cat, "value": int(val)} for cat, val in pairs]}]
```

Use `"tooltip": {"trigger": "item", "formatter": "{b}: {c} ({d}%)"}` for pie charts.
