# PRD 04: Reporting and Export

## Overview

A reporting page where users can run saved query templates with configurable parameters, view results in pivot tables, and export data as CSV or Excel. Provides structured, repeatable analysis workflows. Unlike the conversational Explorer, this page offers deterministic, template-driven reports that can be parameterized and re-run on demand—ideal for scheduled reports, stakeholder dashboards, and standardized analytics.

---

## User Stories

1. **As a business analyst**, I want to select a predefined report template (e.g., "Sales by Region") from a dropdown so I can quickly run standard reports without writing queries or remembering data structures.

2. **As a data analyst**, I want to adjust report parameters (year, minimum revenue, region, etc.) using intuitive controls (dropdowns, sliders, date pickers) so I can customize the report output without editing code.

3. **As a stakeholder**, I want to view report results in multiple formats—a data table, a pivot table for aggregation, and a summary chart—so I can analyze the data from different perspectives in a single view.

4. **As a business user**, I want to export report results as CSV or Excel with a single click so I can share data with colleagues or use it in external tools (Excel, Google Sheets).

5. **As a data analyst**, I want to see loading indicators and skeleton placeholders while a report runs so I know the system is working and when results will appear, especially for long-running queries.

6. **As a frequent report user**, I want to see a list of recently run reports with their parameters and timestamps so I can quickly re-run the same report without re-entering parameters.

---

## Functional Requirements

### FR-01: Report Templates

- Predefined report templates stored as Python dicts or JSON
- Each template defines: name, description, parameters (with types and defaults), and a query function
- Templates selectable via `solara.Select` dropdown
- Example templates: "Sales by Region", "Monthly Revenue Summary", "Top N Products"
- Template metadata displayed when selected (name, description) in `solara.Markdown` or card header

### FR-02: Parameter Input

- Dynamic parameter form generated from template definition
- Parameter types map to Solara input components:
  - `string` → `solara.InputText`
  - `int` → `solara.SliderInt`
  - `date` → `solara.lab.InputDate`
  - `choice` → `solara.Select`
  - `boolean` → `solara.Switch`
- Parameters update reactively; report regenerates on change
- Default values pre-populate the form when a template is selected

### FR-03: Results Display

- Results shown in tabbed view using `solara.lab.Tabs` / `solara.lab.Tab`:
  - **Tab 1: Data Table** — `solara.DataFrame` with pagination (`items_per_page=20`)
  - **Tab 2: Pivot Table** — `solara.PivotTable` for aggregation views (rows, columns, values configurable from data shape)
  - **Tab 3: Summary Chart** — Auto-generated Plotly figure based on data shape (bar for categorical, line for time series)
- Empty state when no report has been run or no data matches parameters

### FR-04: Export Capabilities

- CSV export via `solara.FileDownload` with `data=df.to_csv(index=False)`
- Excel export via `solara.FileDownload` with BytesIO buffer + openpyxl
- Export buttons in `solara.CardActions`
- Filename includes report name and timestamp (e.g., `Sales_by_Region_2026-03-10_14-30.csv`)

### FR-05: Loading States

- Skeleton loaders while report executes
- Progress indicator for long-running queries (`solara.Progress` with `value=True` for indeterminate)
- Disable "Run Report" button during execution
- Use `solara.lab.use_task()` for non-blocking report execution

### FR-06: Report History

- List of recently run reports in sidebar or collapsible section (`solara.Details`)
- Each entry shows: template name, parameters used, timestamp
- Click to re-run with same parameters
- History persisted in session (module-level `solara.reactive()` list, or `use_state`)

---

## Technical Approach

| Aspect | Approach |
|--------|----------|
| **Page** | `pages/reports.py` — single page component |
| **Templates** | `services/report_templates.py` — dict of template definitions |
| **Report execution** | `solara.lab.use_task()` for non-blocking UI |
| **Result caching** | `solara.use_memo()` keyed on template ID + parameters hash |
| **Export** | `solara.FileDownload` with data callback for on-demand file generation |

### Template Structure

```python
# services/report_templates.py

TEMPLATES = {
    "sales_by_region": {
        "name": "Sales by Region",
        "description": "Revenue breakdown by geographic region",
        "parameters": [
            {"name": "year", "type": "choice", "values": [2024, 2025, 2026], "default": 2026},
            {"name": "min_revenue", "type": "int", "min": 0, "max": 100000, "default": 0},
        ],
        "query_fn": lambda df, params: df[
            (df["year"] == params["year"]) & (df["revenue"] >= params["min_revenue"])
        ].groupby("region").sum().reset_index(),
    },
    "monthly_revenue": {
        "name": "Monthly Revenue Summary",
        "description": "Aggregate revenue by month",
        "parameters": [
            {"name": "year", "type": "choice", "values": [2024, 2025, 2026], "default": 2026},
        ],
        "query_fn": lambda df, params: df[df["year"] == params["year"]].groupby("month").agg({"revenue": "sum"}).reset_index(),
    },
    "top_n_products": {
        "name": "Top N Products",
        "description": "Best-selling products by revenue",
        "parameters": [
            {"name": "n", "type": "int", "min": 1, "max": 50, "default": 10},
        ],
        "query_fn": lambda df, params: df.groupby("product").agg({"revenue": "sum"}).nlargest(params["n"], "revenue").reset_index(),
    },
}
```

---

## Implementation Details

### Component Hierarchy

```
ReportPage
├── solara.Column (main layout)
│   ├── solara.Row (header: title + template selector)
│   │   ├── solara.Markdown("Reporting & Export")
│   │   └── solara.Select (template dropdown)
│   │
│   ├── solara.Row (two-column layout)
│   │   ├── solara.Column (left: params + history)
│   │   │   ├── solara.Card ("Parameters")
│   │   │   │   └── ParameterForm (dynamic, generated from template)
│   │   │   ├── solara.Button ("Run Report", disabled when loading)
│   │   │   └── solara.Details ("Recent Reports")
│   │   │       └── ReportHistoryList
│   │   │
│   │   └── solara.Column (right: results)
│   │       └── solara.Card ("Results")
│   │           ├── [Loading] SkeletonLoader or solara.Progress
│   │           └── [Loaded] ResultsTabs
│   │               ├── solara.lab.Tabs
│   │               │   ├── Tab("Data Table") → solara.DataFrame
│   │               │   ├── Tab("Pivot Table") → solara.PivotTable
│   │               │   └── Tab("Chart") → solara.FigurePlotly
│   │               └── solara.CardActions
│   │                   ├── solara.FileDownload (CSV)
│   │                   └── solara.FileDownload (Excel)
```

### ReportPage Component

```python
# pages/reports.py

import solara
from solara.lab import use_task, InputDate, Tabs, Tab
import pandas as pd
from services.report_templates import TEMPLATES
from solara_data_platform.data import get_data  # or DataService

# Module-level reactive state
selected_template_id = solara.reactive(list(TEMPLATES.keys())[0] if TEMPLATES else None)
report_history = solara.reactive([])  # List of {template_id, params, timestamp}

@solara.component
def Page():
    template = TEMPLATES.get(selected_template_id.value, {})
    params = solara.use_reactive({})  # Current parameter values

    # Initialize params from template defaults when template changes
    def init_params():
        p = {}
        for param_def in template.get("parameters", []):
            p[param_def["name"]] = param_def.get("default")
        params.value = p

    solara.use_effect(init_params, [selected_template_id.value])

    # Run report when template or params change
    def run_report():
        df = get_data()  # Get base DataFrame from DataService
        if df is None or df.empty:
            return pd.DataFrame()
        query_fn = template.get("query_fn")
        if not query_fn:
            return df
        return query_fn(df, params.value)

    task = use_task(run_report, dependencies=[selected_template_id.value, params.value])
    result_df = task.result if task.finished and task.result is not None else None
    loading = not task.finished

    # Layout
    with solara.Column(gap="1rem"):
        with solara.Row(justify="space-between"):
            solara.Markdown("# Reporting & Export")
            solara.Select(
                label="Report Template",
                value=selected_template_id.value,
                values=list(TEMPLATES.keys()),
                on_value=selected_template_id.set,
            )

        with solara.Row(gap="2rem"):
            # Left: Parameters + History
            with solara.Column(classes=["col-12", "col-md-4"]):
                with solara.Card("Parameters", margin=0):
                    ParameterForm(template=template, params=params)
                solara.Button(
                    "Run Report",
                    on_click=lambda: None,  # Report auto-runs via use_task deps
                    disabled=loading,
                )
                with solara.Details(summary="Recent Reports", open=False):
                    ReportHistoryList(history=report_history, on_rerun=...)

            # Right: Results
            with solara.Column(classes=["col-12", "col-md-8"]):
                with solara.Card("Results", margin=0):
                    if loading:
                        SkeletonResultsCard()
                    else:
                        ResultsTabs(df=result_df, template_name=template.get("name", "Report"))
```

### Dynamic Parameter Form Generation

```python
# components/report_parameter_form.py

import solara
from solara.lab import InputDate

@solara.component
def ParameterForm(template: dict, params):
    """Renders input controls based on template['parameters']."""
    param_defs = template.get("parameters", [])

    for param_def in param_defs:
        name = param_def["name"]
        param_type = param_def.get("type", "string")
        value = params.value.get(name, param_def.get("default"))

        def make_on_value(n=name, _pd=param_def):  # Default args capture loop vars
            def on_value(v):
                p = dict(params.value)
                p[n] = v
                params.value = p
            return on_value

        if param_type == "string":
            solara.InputText(
                label=name.replace("_", " ").title(),
                value=value or "",
                on_value=make_on_value(),
            )
        elif param_type == "int":
            solara.SliderInt(
                label=param_def.get("label", name.replace("_", " ").title()),
                value=value if value is not None else param_def.get("default", 0),
                min=param_def.get("min", 0),
                max=param_def.get("max", 100),
                on_value=make_on_value(),
            )
        elif param_type == "date":
            solara.lab.InputDate(
                label=param_def.get("label", name.replace("_", " ").title()),
                value=value,
                on_value=make_on_value(),
            )
        elif param_type == "choice":
            solara.Select(
                label=param_def.get("label", name.replace("_", " ").title()),
                value=value,
                values=param_def.get("values", []),
                on_value=make_on_value(),
            )
        elif param_type == "boolean":
            solara.Switch(
                label=param_def.get("label", name.replace("_", " ").title()),
                value=value if value is not None else param_def.get("default", False),
                on_value=make_on_value(),
            )
```

### Tabbed Results Display

```python
# components/report_results.py

import solara
from solara.lab import Tabs, Tab
import plotly.express as px
import pandas as pd

@solara.component
def ResultsTabs(df: pd.DataFrame, template_name: str):
    if df is None or df.empty:
        solara.Markdown("No data. Adjust parameters and run the report.")
        return

    # Auto-generate chart based on data shape
    figure = solara.use_memo(
        lambda: _make_summary_chart(df),
        [df]
    )

    # Infer pivot config from columns (first col = rows, second = columns, numeric = values)
    pivot_config = solara.use_memo(
        lambda: _infer_pivot_config(df),
        [df]
    )

    with solara.lab.Tabs():
        with solara.lab.Tab("Data Table"):
            solara.DataFrame(df, items_per_page=20)

        with solara.lab.Tab("Pivot Table"):
            if pivot_config:
                solara.PivotTable(
                    df,
                    rows=pivot_config.get("rows", [df.columns[0]]),
                    columns=pivot_config.get("columns", []),
                    values=pivot_config.get("values", [c for c in df.columns if df[c].dtype in ["int64", "float64"]] or [df.columns[-1]]),
                )
            else:
                solara.DataFrame(df, items_per_page=20)

        with solara.lab.Tab("Chart"):
            if figure:
                solara.FigurePlotly(figure)
            else:
                solara.Markdown("Chart not available for this data shape.")

    with solara.CardActions():
        ExportButtons(df=df, template_name=template_name)

def _make_summary_chart(df: pd.DataFrame):
    if df is None or df.empty or len(df.columns) < 2:
        return None
    # Bar chart if categorical x, line if datetime
    x_col = df.columns[0]
    y_col = df.columns[1] if len(df.columns) > 1 and df[df.columns[1]].dtype in ["int64", "float64"] else df.columns[-1]
    if pd.api.types.is_datetime64_any_dtype(df[x_col]):
        return px.line(df, x=x_col, y=y_col)
    return px.bar(df, x=x_col, y=y_col)

def _infer_pivot_config(df: pd.DataFrame):
    numeric_cols = [c for c in df.columns if df[c].dtype in ["int64", "float64"]]
    cat_cols = [c for c in df.columns if c not in numeric_cols]
    if len(cat_cols) >= 2 and numeric_cols:
        return {"rows": [cat_cols[0]], "columns": [cat_cols[1]], "values": numeric_cols[:1]}
    if len(cat_cols) >= 1 and numeric_cols:
        return {"rows": cat_cols, "columns": [], "values": numeric_cols[:1]}
    return None
```

### Export Functionality (CSV and Excel)

```python
# components/report_export.py

import solara
import pandas as pd
from io import BytesIO
from datetime import datetime

def _safe_filename(name: str) -> str:
    return "".join(c if c.isalnum() or c in " -_" else "_" for c in name).replace(" ", "_")

@solara.component
def ExportButtons(df: pd.DataFrame, template_name: str):
    if df is None or df.empty:
        return

    base_name = _safe_filename(template_name)
    timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M")
    csv_filename = f"{base_name}_{timestamp}.csv"
    excel_filename = f"{base_name}_{timestamp}.xlsx"

    # CSV: pass string directly to FileDownload
    csv_data = df.to_csv(index=False)

    # Excel: create BytesIO buffer with openpyxl
    def get_excel_bytes():
        buffer = BytesIO()
        with pd.ExcelWriter(buffer, engine="openpyxl") as writer:
            df.to_excel(writer, index=False, sheet_name="Report")
        buffer.seek(0)
        return buffer.getvalue()

    with solara.CardActions():
        solara.FileDownload(
            data=csv_data,
            filename=csv_filename,
            label="Export CSV",
        )
        solara.FileDownload(
            data=get_excel_bytes,
            filename=excel_filename,
            label="Export Excel",
        )
```

**Note:** `solara.FileDownload` accepts either a value or a callable for `data`. For Excel, passing a callable `get_excel_bytes` ensures the buffer is created on-demand when the user clicks download, avoiding unnecessary computation.

### Skeleton Loader for Results

```python
# components/loading.py (extend existing)

from ipyvuetify import v

@solara.component
def SkeletonResultsCard():
    with v.Card():
        v.SkeletonLoader(type_="card-heading, table-heading, table-row@5", boilerplate=True)
```

### Report History Component

```python
# components/report_history.py

import solara
from datetime import datetime

@solara.component
def ReportHistoryList(history, on_rerun):
    """List of recent reports; click to re-run with same params."""
    for i, entry in enumerate(reversed(history.value[-10:])):  # Last 10
        template_name = entry.get("template_name", "Report")
        params_str = ", ".join(f"{k}={v}" for k, v in entry.get("params", {}).items())
        ts = entry.get("timestamp", "")
        with solara.Row():
            solara.Markdown(f"**{template_name}** — {params_str} ({ts})")
            solara.Button("Re-run", on_click=lambda e=entry: on_rerun(e))
```

---

## Solara / ipyvuetify API Reference

| Component | Package | Purpose |
|-----------|---------|---------|
| `solara.Select` | solara | Template dropdown; `values`, `value`, `on_value` |
| `solara.InputText` | solara | Text parameter; `value`, `on_value` |
| `solara.SliderInt` | solara | Integer parameter; `value`, `min`, `max`, `on_value` |
| `solara.Switch` | solara | Boolean parameter; `value`, `on_value` |
| `solara.lab.InputDate` | solara.lab | Date parameter; `value`, `on_value` |
| `solara.lab.Tabs` / `solara.lab.Tab` | solara.lab | Tabbed results; `with Tabs():` then `with Tab("Label"):` |
| `solara.DataFrame` | solara | Paginated table; `items_per_page` |
| `solara.PivotTable` | solara | Pivot view; `rows`, `columns`, `values` |
| `solara.FigurePlotly` | solara | Plotly chart |
| `solara.FileDownload` | solara | Download button; `data` (value or callable), `filename`, `label` |
| `solara.Card` / `solara.CardActions` | solara | Card container and action buttons |
| `solara.Details` | solara | Collapsible section; `summary`, `open` |
| `solara.reactive()` | solara | Reactive state; `.value`, `.set()` |
| `solara.use_memo(fn, deps)` | solara | Memoize; recompute when deps change |
| `solara.lab.use_task(fn, dependencies)` | solara.lab | Async task; `.result`, `.finished`, `.error`, `.pending` |
| `solara.Progress(value=True)` | solara | Indeterminate progress bar |
| `v.SkeletonLoader` | ipyvuetify | Skeleton placeholder; `type_="card-heading, table-heading, table-row@5"` |

---

## Dependencies

- `solara` — reactive UI framework
- `solara.lab` — `use_task`, `InputDate`, `Tabs`, `Tab`
- `plotly` — chart generation
- `pandas` — DataFrame operations
- `openpyxl` — Excel export
- `ipyvuetify` — Vuetify components (v.SkeletonLoader)

---

## Acceptance Criteria

- [ ] Report template dropdown populates from `TEMPLATES`; selecting a template shows its description and parameter form
- [ ] Parameter form renders correct input types (InputText, SliderInt, InputDate, Select, Switch) from template definition
- [ ] Report executes via `use_task()` when template or parameters change; results update reactively
- [ ] Results display in three tabs: Data Table (paginated), Pivot Table, Summary Chart
- [ ] CSV export downloads correct data with report name and timestamp in filename
- [ ] Excel export downloads .xlsx file with openpyxl; filename includes report name and timestamp
- [ ] Skeleton loader or progress indicator shown during report execution; Run Report button disabled
- [ ] Report history lists recent runs; clicking Re-run re-applies same parameters and executes report
