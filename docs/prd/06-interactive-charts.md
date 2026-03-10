# PRD 06: Interactive Charts -- Replacing Matplotlib in PandasAI

## Problem Statement

PandasAI 2.3 generates charts via its code generation pipeline using **matplotlib** by default. The LLM generates Python code that calls `matplotlib.pyplot` to create static PNG images saved to disk. These charts are non-interactive -- users cannot zoom, pan, hover for values, or click data points.

The current flow is:

1. User asks a natural language question (e.g., "Plot revenue by region")
2. PandasAI's LLM generates Python code using `matplotlib.pyplot`
3. Code executes, calls `plt.savefig()` to save a PNG to `exports/charts/`
4. The saved file path is returned as the response
5. The Solara app renders the static PNG via `solara.Image`

We need interactive charts that users can explore -- zoom, pan, hover, select -- rendered natively in the Solara UI.

## Goal

Replace PandasAI's matplotlib chart output with interactive charts rendered via Solara's native visualization components, while keeping PandasAI's natural language query interface intact.

## Technology Evaluation: Plotly vs ECharts

Solara has **first-class native components for both** Plotly and ECharts. Both are fully supported, interactive, and production-ready in Solara.

### solara.FigurePlotly

```python
solara.FigurePlotly(figure)  # accepts plotly.graph_objects.Figure
```

- Wraps Plotly's `FigureWidget` for reactive rendering
- Requires `plotly` package as a dependency
- Full Python API via `plotly.express` and `plotly.graph_objects`

### solara.FigureEcharts

```python
solara.FigureEcharts(
    option=dict,           # Standard ECharts option configuration
    on_click=callback,     # Click event handler
    on_mouseover=callback, # Hover event handler
    on_mouseout=callback,  # Mouse-out event handler
    maps=dict,             # Geographic map data
    responsive=True,       # Auto-resize with container
    attributes={"style": "height: 400px;"},
)
```

- Built into Solara core -- no extra dependency required
- Takes a standard ECharts option dict (same format as the ECharts JS docs)
- Built-in click/hover/mouseout event callbacks
- Responsive resizing support
- Supports geographic maps via `maps` parameter
- Solara docs note: "we do not support a Python API to create the figure data" -- use pyecharts or construct dicts directly

### Comparison

| Criteria | Plotly | ECharts |
|----------|--------|---------|
| **Solara component** | `solara.FigurePlotly` (first-class) | `solara.FigureEcharts` (first-class, built-in) |
| **Extra dependency** | Requires `plotly` package | None -- included with Solara |
| **Python API** | Full Python API (`plotly.express`, `plotly.graph_objects`) | No Python API in Solara; option dicts or pyecharts |
| **PandasAI support** | Listed as optional dependency (`pandasai[plotly]`) | Not in PandasAI's dependency list |
| **LLM code generation** | LLM generates `plotly.express` Python code | LLM generates ECharts JSON option dict or pyecharts code |
| **Performance** | Good; can lag with very large datasets | Superior; optimized for large datasets and real-time updates |
| **Interactive features** | Zoom, pan, hover, lasso select, export toolbar | Zoom, hover, click events, data zoom, brush, tooltip |
| **Event callbacks** | Via FigureWidget trace events | Native `on_click`, `on_mouseover`, `on_mouseout` in Solara |
| **Static image export** | `fig.to_image()` via Kaleido (simple, one-line) | Requires headless browser (pyppeteer/selenium) -- complex |
| **Chart types** | Extensive: statistical, scientific, financial, 3D, maps | Extensive: statistical, geographic, 3D, gauge, treemap, sankey |
| **Theming** | Templates: `plotly_dark`, `plotly_white`, etc. | Rich built-in themes; custom themes via option dict |
| **pandas integration** | `pd.options.plotting.backend = "plotly"` | No pandas backend |
| **Geographic/maps** | Mapbox integration | Built-in map support via `maps` parameter |

### Recommendation: Plotly for PandasAI-Generated Charts

**For PandasAI integration specifically**, Plotly is the better choice for these reasons:

1. **PandasAI code execution pipeline**: PandasAI generates and executes Python code. The LLM naturally generates `import plotly.express as px; fig = px.bar(df, x="col", y="val")` -- this is idiomatic Python that PandasAI can execute and return as a `go.Figure` object. For ECharts, the LLM would need to generate a Python dict matching the ECharts JSON schema, which is less natural in a Python code generation context.

2. **PandasAI already supports Plotly**: It's a listed optional dependency (`pip install pandasai[plotly]`). ECharts/pyecharts would need to be added via `custom_whitelisted_dependencies` and is untested with PandasAI's sandbox.

3. **Static export for PowerPoint (PRD 07)**: Plotly's `fig.to_image()` via Kaleido is a one-line call returning PNG bytes. ECharts static export requires a headless browser (pyppeteer, selenium, or phantomjs), adding significant deployment complexity.

4. **Object-based response**: PandasAI's `ResponseParser` can intercept a `go.Figure` Python object directly. An ECharts option dict would need to be returned as a raw Python dict and type-checked differently.

**However, ECharts is the better choice for manually-built dashboard charts** (PRD 02) where we control the data pipeline directly, because:
- No extra dependency (built into Solara)
- Better performance with large datasets
- Native click/hover event callbacks in the Solara API
- Rich built-in chart types for dashboards (gauges, treemaps, sankey diagrams)

### Dual-Library Strategy

Use **both** libraries, each where it fits best:

| Context | Library | Reason |
|---------|---------|--------|
| PandasAI chat-generated charts | Plotly | Natural fit for LLM code generation pipeline |
| Dashboard page (static charts) | ECharts | Built-in, performant, rich event callbacks |
| Report page charts | Either | Plotly if re-using PandasAI output; ECharts if building custom |
| PowerPoint export | Plotly | Simple `to_image()` via Kaleido |

## User Stories

1. As a data analyst, I want charts generated by PandasAI to be interactive so I can hover over data points to see exact values.
2. As a business user, I want to zoom into specific regions of a chart to examine trends in detail.
3. As a data analyst, I want generated charts to appear inline in the chat interface, not as separate image files.
4. As a user, I want chart styling to match the app's dark/light theme.
5. As a developer, I want a clean integration point between PandasAI and Plotly that doesn't require forking PandasAI.

## Functional Requirements

### FR-01: Instruct PandasAI to Generate Plotly Code

PandasAI's code generation is driven by its LLM prompt. The LLM decides which library to use based on the prompt instructions and whitelisted dependencies.

**Approach**: Use three PandasAI configuration mechanisms together:

1. **Install Plotly dependency**: `pip install pandasai[plotly]`
2. **Whitelist Plotly**: Add `"plotly"` to `custom_whitelisted_dependencies` so PandasAI's sandbox allows Plotly imports
3. **Custom prompt override**: Use PandasAI's `custom_prompts` config to instruct the LLM to use `plotly.express` instead of `matplotlib`

```python
from pandasai import Agent

PLOTLY_INSTRUCTION = """
When generating visualization code, ALWAYS use plotly.express (imported as px) 
instead of matplotlib. Return the plotly figure object as the result.
Do NOT use plt.savefig() or plt.show(). Instead, assign the figure to a 
variable called `result` as: result = {"type": "plot", "value": fig}
where fig is a plotly.express or plotly.graph_objects Figure.
"""

agent = Agent(
    df,
    config={
        "llm": llm,
        "custom_whitelisted_dependencies": ["plotly"],
        "save_charts": False,
        "open_charts": False,
    }
)
```

### FR-02: Custom ResponseParser for Plotly Figures

Create a custom `ResponseParser` that intercepts chart responses and returns Plotly figure objects instead of file paths.

```python
from pandasai.responses.response_parser import ResponseParser
import plotly.graph_objects as go

class SolaraResponseParser(ResponseParser):
    def __init__(self, context) -> None:
        super().__init__(context)

    def format_plot(self, result):
        """
        Intercept plot results from PandasAI.
        
        If the result contains a Plotly figure object, return it directly.
        If it contains a matplotlib figure or file path, convert to Plotly
        or return the path as fallback.
        """
        value = result.get("value") if isinstance(result, dict) else result
        
        if isinstance(value, go.Figure):
            return {"type": "plotly", "value": value}
        
        # Fallback: if matplotlib was used despite instructions,
        # return the saved image path
        if isinstance(value, str) and value.endswith((".png", ".jpg")):
            return {"type": "image", "value": value}
        
        return {"type": "plot", "value": value}

    def format_dataframe(self, result):
        return {"type": "dataframe", "value": result.get("value", result)}

    def format_response(self, result):
        return {"type": "text", "value": str(result.get("value", result))}
```

Configure the Agent with this parser:

```python
agent = Agent(
    df,
    config={
        "llm": llm,
        "response_parser": SolaraResponseParser,
        "custom_whitelisted_dependencies": ["plotly"],
        "save_charts": False,
        "open_charts": False,
    }
)
```

### FR-03: AIService Wrapper

Encapsulate the PandasAI Agent configuration in a service class that standardizes responses.

```python
# services/ai_service.py
from pandasai import Agent
import plotly.graph_objects as go

class AIService:
    def __init__(self, df, llm_config):
        self.agent = Agent(
            df,
            config={
                "llm": llm_config["llm"],
                "response_parser": SolaraResponseParser,
                "custom_whitelisted_dependencies": ["plotly"],
                "save_charts": False,
                "open_charts": False,
            }
        )

    def query(self, message: str) -> dict:
        """
        Returns a normalized response dict:
        {
            "type": "text" | "dataframe" | "plotly" | "image" | "error",
            "value": str | pd.DataFrame | go.Figure | str(path),
        }
        """
        try:
            result = self.agent.chat(message)
            if isinstance(result, dict) and "type" in result:
                return result
            if isinstance(result, go.Figure):
                return {"type": "plotly", "value": result}
            if hasattr(result, "to_dict"):  # DataFrame
                return {"type": "dataframe", "value": result}
            return {"type": "text", "value": str(result)}
        except Exception as e:
            return {"type": "error", "value": str(e)}

    def update_data(self, df):
        """Re-initialize agent with new data."""
        self.agent = Agent(
            df,
            config=self.agent._config
        )
```

### FR-04: Rendering Plotly Charts in Solara

Render the Plotly figure objects inline using `solara.FigurePlotly`.

```python
import solara
import plotly.graph_objects as go

@solara.component
def ChatMessageContent(message: dict):
    msg_type = message["type"]
    value = message["value"]
    
    if msg_type == "plotly":
        solara.FigurePlotly(value)
    elif msg_type == "dataframe":
        solara.DataFrame(value, items_per_page=10)
    elif msg_type == "image":
        solara.Image(value)
    elif msg_type == "error":
        solara.Error(value)
    else:
        solara.Markdown(str(value))
```

### FR-05: Theme-Aware Chart Styling

Apply Plotly templates that match the app's current light/dark theme.

```python
import solara
import plotly.io as pio
import plotly.graph_objects as go

@solara.component
def ThemedChart(fig: go.Figure):
    dark = solara.lab.use_dark_effective()
    
    themed_fig = solara.use_memo(
        lambda: fig.update_layout(
            template="plotly_dark" if dark else "plotly_white",
            paper_bgcolor="rgba(0,0,0,0)",
            plot_bgcolor="rgba(0,0,0,0)",
        ),
        [fig, dark]
    )
    
    solara.FigurePlotly(themed_fig)
```

### FR-06: Fallback for Matplotlib Output

Despite prompt instructions, the LLM may occasionally generate matplotlib code. Handle this gracefully:

1. If `format_plot` receives a file path (PNG), render via `solara.Image` as fallback
2. Log a warning for monitoring
3. Optionally: post-process matplotlib figures by converting to Plotly using `plotly.tools.mpl_to_plotly()` (experimental, limited support)

### FR-07: Prompt Engineering for Reliable Plotly Output

The LLM prompt must be carefully crafted to consistently produce Plotly code. Key instructions:

```python
CHART_SYSTEM_PROMPT = """
When asked to create a visualization or plot:
1. ALWAYS use plotly.express (import as px) or plotly.graph_objects (import as go)
2. NEVER use matplotlib, seaborn, or any other plotting library
3. Store the figure in a variable called `fig`
4. Use appropriate chart types:
   - Bar charts: px.bar()
   - Line charts: px.line()
   - Scatter plots: px.scatter()
   - Pie charts: px.pie()
   - Histograms: px.histogram()
   - Heatmaps: px.imshow()
   - Box plots: px.box()
5. Always add descriptive title, axis labels, and hover data
6. Set the result as: result = {"type": "plot", "value": fig}
"""
```

## Technical Architecture

```
User Query ("Plot revenue by region")
    │
    ▼
┌─────────────────────────────────┐
│  AIService.query(message)       │
│                                 │
│  ┌───────────────────────────┐  │
│  │ PandasAI Agent            │  │
│  │  - LLM generates code     │  │
│  │  - Uses plotly.express     │  │
│  │  - Whitelisted: plotly     │  │
│  │  - Prompt: "use plotly"    │  │
│  └───────────┬───────────────┘  │
│              │                  │
│  ┌───────────▼───────────────┐  │
│  │ SolaraResponseParser      │  │
│  │  - format_plot()          │  │
│  │  - Returns go.Figure      │  │
│  └───────────┬───────────────┘  │
│              │                  │
│  Returns: {"type": "plotly",    │
│            "value": go.Figure}  │
└──────────────┬──────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  Solara Component                │
│                                  │
│  if type == "plotly":            │
│      solara.FigurePlotly(fig)    │
│  elif type == "dataframe":       │
│      solara.DataFrame(df)        │
│  else:                           │
│      solara.Markdown(text)       │
│                                  │
│  Interactive chart in browser:   │
│  zoom, pan, hover, lasso select  │
└──────────────────────────────────┘
```

## FR-08: ECharts for Dashboard Charts

For the dashboard page (PRD 02), use `solara.FigureEcharts` for manually-built charts where we control the data pipeline.

```python
import solara

@solara.component
def RevenueBarChart(categories: list, values: list):
    dark = solara.lab.use_dark_effective()
    
    option = {
        "backgroundColor": "transparent",
        "tooltip": {"trigger": "axis"},
        "xAxis": {
            "type": "category",
            "data": categories,
            "axisLabel": {"color": "#aaa" if dark else "#333"},
        },
        "yAxis": {
            "type": "value",
            "axisLabel": {"color": "#aaa" if dark else "#333"},
        },
        "series": [{
            "type": "bar",
            "data": values,
            "itemStyle": {"borderRadius": [4, 4, 0, 0]},
        }],
    }
    
    def on_bar_click(data):
        # data contains: name, value, dataIndex, seriesName, etc.
        print(f"Clicked: {data['name']} = {data['value']}")
    
    solara.FigureEcharts(option=option, on_click=on_bar_click, responsive=True)

@solara.component
def TrendLineChart(dates: list, series_data: dict):
    option = {
        "tooltip": {"trigger": "axis"},
        "legend": {"data": list(series_data.keys())},
        "xAxis": {"type": "category", "data": dates},
        "yAxis": {"type": "value"},
        "dataZoom": [{"type": "slider", "start": 0, "end": 100}],
        "series": [
            {"name": name, "type": "line", "data": values, "smooth": True}
            for name, values in series_data.items()
        ],
    }
    solara.FigureEcharts(option=option, responsive=True)

@solara.component
def DistributionPieChart(data: list):
    """data: list of {"name": str, "value": number}"""
    option = {
        "tooltip": {"trigger": "item", "formatter": "{b}: {c} ({d}%)"},
        "series": [{
            "type": "pie",
            "radius": ["40%", "70%"],  # Donut chart
            "data": data,
            "emphasis": {
                "itemStyle": {
                    "shadowBlur": 10,
                    "shadowOffsetX": 0,
                    "shadowColor": "rgba(0, 0, 0, 0.5)",
                }
            },
        }],
    }
    solara.FigureEcharts(option=option, responsive=True)
```

**ECharts advantages for dashboard use**:
- No extra dependency -- part of Solara core
- Built-in `dataZoom` for time series exploration
- Click events via `on_click` for drill-down navigation
- Better rendering performance with large datasets
- Rich chart types: gauge, treemap, sankey, heatmap, radar

## Dependencies

```toml
[project]
dependencies = [
    "solara>=1.35",
    "pandasai[plotly]>=2.3,<3.0",
    "plotly>=5.18",
    "kaleido>=1.0",          # For static export (PRD 07)
    # ECharts: no extra dependency needed (built into Solara)
]
```

## Implementation Plan

### Phase 1: PandasAI Plotly Integration (Chat/Explorer)
1. Install `pandasai[plotly]` and add `plotly` to whitelisted dependencies
2. Create `SolaraResponseParser` in `services/response_parser.py`
3. Update `AIService` to use the custom parser and Plotly prompt instructions
4. Update `ChatMessageContent` to render `go.Figure` via `FigurePlotly`

### Phase 2: Dashboard ECharts Components
5. Create reusable ECharts chart components in `components/charts.py` (bar, line, pie, etc.)
6. Wire dashboard charts to filtered reactive data
7. Implement click event handlers for drill-down interactions
8. Add ECharts `dataZoom` for time-series charts

### Phase 3: Polish
9. Add theme-aware styling for both Plotly (`ThemedChart`) and ECharts (dark option dicts)
10. Add matplotlib fallback handling in ResponseParser
11. Tune the LLM prompt for consistent Plotly output across chart types
12. Add chart toolbar customization (hide/show Plotly modebar buttons)

### Phase 4: Integration
13. Store generated Plotly figures in reactive state for reuse on dashboard
14. Allow pinning a chat-generated chart to the dashboard page
15. Ensure consistent visual style between ECharts dashboard charts and Plotly chat charts

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| LLM generates matplotlib despite prompt instructions | Static image fallback | Robust fallback in ResponseParser; iterate on prompt wording |
| Plotly figure serialization across WebSocket | Chart doesn't render | `FigurePlotly` handles serialization natively via FigureWidget |
| Large datasets make Plotly charts slow | Poor UX | Sample/aggregate data before charting; PandasAI typically does this; use ECharts for known-large datasets on dashboard |
| PandasAI sandbox blocks Plotly imports | Code execution fails | `custom_whitelisted_dependencies: ["plotly"]` resolves this |
| Visual inconsistency between Plotly and ECharts | Confusing UX | Align color palettes and dark/light theming across both libraries |
| ECharts static export needed for PPTX | Complex deployment | For dashboard PPTX export, convert ECharts data to Plotly figures for `to_image()`; or use pyppeteer as fallback |

## Acceptance Criteria

1. Natural language chart requests via PandasAI produce interactive Plotly charts (zoom, pan, hover)
2. Charts render inline in the Solara chat interface via `FigurePlotly`
3. Dashboard charts render via `FigureEcharts` with click event handling and data zoom
4. Both chart types respect the app's dark/light theme
5. If the LLM falls back to matplotlib, a static image is displayed (not a crash)
6. No matplotlib `plt.show()` pop-ups occur during chart generation
7. Chart generation latency is comparable to the existing matplotlib pipeline
8. ECharts dashboard charts resize responsively on window changes
