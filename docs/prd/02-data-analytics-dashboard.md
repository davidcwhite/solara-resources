# PRD 02: Data Analytics Dashboard

## Overview

The dashboard is the landing page of the Solara + PandasAI data platform. It presents interactive visualizations and data tables with cross-filter coordination. Users can slice data using filters (dropdowns, sliders, date pickers) and see all charts update reactively. The dashboard serves as the primary entry point for data exploration, providing at-a-glance KPIs, trend analysis, and drill-down capabilities through coordinated interactions.

---

## User Stories

1. **As a data analyst**, I want to view key metrics (revenue, record count, averages) at the top of the dashboard so I can quickly assess the current state of the data without scrolling through charts.

2. **As a data analyst**, I want to filter data by category, date range, and numeric ranges using dropdowns, date pickers, and sliders so I can focus on specific segments of the dataset.

3. **As a data analyst**, I want to click on a bar or segment in one chart and see all other charts update to reflect that selection so I can drill down interactively without manually adjusting multiple filters.

4. **As a business user**, I want to see revenue by category in a bar chart, trends over time in a line chart, and distribution in a pie/donut chart so I can understand the data from multiple perspectives.

5. **As a data analyst**, I want to view the underlying filtered data in a sortable, paginated table so I can inspect individual records and verify chart aggregations.

6. **As a user**, I want to see skeleton loaders while data is loading so I know the app is responsive and content is on its way, rather than staring at a blank screen.

7. **As a data analyst**, I want to export the currently filtered dataset as CSV from the data table card so I can share or further analyze the data in Excel or other tools.

---

## Functional Requirements

### FR-01: Dashboard Layout

- Responsive grid layout using `solara.ColumnsResponsive` or `v.Row`/`v.Col`
- 2 columns on desktop, 1 column on mobile
- Each visualization lives in a `solara.Card` with title and optional actions
- Top section: filter bar with `solara.Select`, `solara.SliderInt`, `solara.lab.InputDate`

### FR-02: Interactive Charts

- Bar chart (revenue by category) using `solara.FigurePlotly`
- Line chart (trends over time) using Plotly
- Pie/donut chart (distribution) using Plotly
- Charts update reactively when filters change
- All chart config derived from filtered DataFrame using `use_memo`

### FR-03: Cross-Filter Coordination

- Use `solara.CrossFilterDataFrame` and `use_cross_filter` to link filters across charts
- Selecting a category in one chart filters data in all others
- Filter state managed via `solara.reactive()`

### FR-04: Data Table

- `solara.DataFrame` with pagination (`items_per_page=20`)
- Shows the currently filtered dataset
- Columns sortable

### FR-05: Summary Cards

- KPI cards at top: total revenue, record count, average value
- Cards show computed values from filtered data using `use_memo`
- Display with `v.Card` + `v.CardTitle` + `v.CardText` with large number styling

### FR-06: Loading States

- Show `v.SkeletonLoader` while data loads (type="card-heading, image" for chart cards, type="table-heading, table-row@5" for table)
- Use `solara.lab.use_task()` to manage data loading
- Transition from skeleton to content with fade-in CSS animation

### FR-07: Export

- `solara.FileDownload` button to export filtered data as CSV
- Located in the data table card's `CardActions`

---

## Technical Approach

| Aspect | Approach |
|--------|----------|
| **Page** | `pages/dashboard.py` — single page component |
| **Filter state** | Module-level `solara.reactive()` variables |
| **Data loading** | `solara.lab.use_task()` returning filtered DataFrame |
| **Charts** | Reusable components in `components/charts.py` |
| **Chart inputs** | Each chart component accepts a DataFrame and renders a Plotly figure |
| **Memoization** | Plotly figure creation memoized with `use_memo` keyed on filtered DataFrame |

---

## Implementation Details

### State Flow: Filters → Data → Charts

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Module-level reactive state                                                │
│  category_filter = solara.reactive(None)                                    │
│  date_start = solara.reactive(None)                                         │
│  date_end = solara.reactive(None)                                           │
│  value_range = solara.reactive((0, 100))                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  use_task() or use_cross_filter()                                           │
│  - Loads raw data (DataService)                                             │
│  - Applies filters → filtered_df                                            │
│  - Returns CrossFilterDataFrame or plain DataFrame                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  use_memo(key=filtered_df)                                                  │
│  - KPI values (sum, count, mean)                                           │
│  - Plotly figure for each chart                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Render: KPI cards | Filter bar | Chart cards | Data table                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Filter Bar Component

```python
# components/filter_bar.py or inline in dashboard.py

import solara
from solara.lab import InputDate

@solara.component
def FilterBar():
    category_filter = solara.use_reactive(solara.reactive(None))  # or from module
    date_start = solara.use_reactive(solara.reactive(None))
    date_end = solara.use_reactive(solara.reactive(None))
    value_min, value_max = solara.use_reactive(solara.reactive((0, 100)))

    with solara.Card("Filters", margin=0):
        with solara.Row(gap="1rem", wrap=True):
            # Category dropdown
            solara.Select(
                label="Category",
                values=["All", "Electronics", "Clothing", "Food", "Other"],
                value=category_filter.value,
                on_value=category_filter.set,
            )
            # Date range
            solara.lab.InputDate(
                label="Start Date",
                value=date_start.value,
                on_value=date_start.set,
            )
            solara.lab.InputDate(
                label="End Date",
                value=date_end.value,
                on_value=date_end.set,
            )
            # Numeric range slider
            solara.SliderInt(
                label="Value Range",
                value=value_min.value,
                min=0,
                max=100,
                on_value=lambda v: value_min.set((v, value_max.value)),
            )
```

### Chart Card Component

```python
# components/charts.py

import solara
from solara.lab import use_task
import plotly.express as px
import pandas as pd

@solara.component
def ChartCard(
    title: str,
    df: pd.DataFrame,
    chart_type: str,  # "bar" | "line" | "pie"
    x: str = None,
    y: str = None,
    color: str = None,
):
    loading = solara.use_reactive(False)

    def make_figure():
        if df is None or df.empty:
            return {}
        if chart_type == "bar":
            fig = px.bar(df, x=x, y=y, color=color)
        elif chart_type == "line":
            fig = px.line(df, x=x, y=y)
        elif chart_type == "pie":
            fig = px.pie(df, names=x, values=y, hole=0.4)  # donut
        fig.update_layout(margin=dict(l=20, r=20, t=30, b=20))
        return fig

    figure = solara.use_memo(make_figure, [df])

    with solara.Card(title=title, margin=0):
        if loading.value:
            v.SkeletonLoader(type_="card-heading, image", boilerplate=True)
        else:
            solara.FigurePlotly(figure)
```

### Page Composition

```python
# pages/dashboard.py

import solara
from solara.lab import use_task, InputDate
import pandas as pd

# Module-level reactive state
category_filter = solara.reactive(None)
date_start = solara.reactive(None)
date_end = solara.reactive(None)
value_range = solara.reactive((0, 100))

@solara.component
def Page():
    # Data loading with use_task
    task = use_task(load_filtered_data)
    filtered_df = task.result if task.finished else None
    loading = not task.finished

    # KPI memoization
    kpis = solara.use_memo(
        lambda: compute_kpis(filtered_df) if filtered_df is not None else {},
        [filtered_df]
    )

    # Layout: responsive 2-col desktop, 1-col mobile
    with solara.ColumnsResponsive(12, large=6) as main:
        # Top: KPI cards
        with solara.Row():
            KPICard("Total Revenue", kpis.get("revenue", 0))
            KPICard("Record Count", kpis.get("count", 0))
            KPICard("Average Value", kpis.get("avg", 0))

        # Filter bar
        FilterBar()

        # Charts (2 columns on large screens)
        with main:
            if loading:
                SkeletonChartCard()
            else:
                ChartCard("Revenue by Category", filtered_df, "bar", x="category", y="revenue")

        with main:
            if loading:
                SkeletonChartCard()
            else:
                ChartCard("Trends Over Time", filtered_df, "line", x="date", y="revenue")

        with main:
            if loading:
                SkeletonChartCard()
            else:
                ChartCard("Distribution", filtered_df, "pie", x="category", y="revenue")

        # Data table (full width)
        with solara.Column(12):
            DataTableCard(filtered_df, loading)
```

### Cross-Filter Integration

```python
# Using solara.CrossFilterDataFrame and use_cross_filter

# use_cross_filter is from solara (api/hooks), not solara.lab
from solara import use_cross_filter

@solara.component
def Page():
    # Get cross-filtered DataFrame from shared data store
    # use_cross_filter(data_key, name="...", reducer=...) returns (df, set_filter)
    df, set_cross_filter = use_cross_filter(data_key="dashboard", name="main")

    # df is already filtered by chart selections
    # Apply additional UI filters on top
    filtered = apply_ui_filters(df, category_filter, date_start, date_end)

    # Charts receive filtered; selections propagate via set_cross_filter
    with solara.FigurePlotly(bar_figure) as fig:
        # Plotly click events -> set_cross_filter
        pass  # Solara handles via CrossFilterDataFrame integration
```

### Skeleton Loader Usage

```python
# components/loading.py

import solara
from ipyvuetify import v

@solara.component
def SkeletonChartCard():
    with v.Card():
        v.SkeletonLoader(type_="card-heading, image", boilerplate=True)

@solara.component
def SkeletonTableCard():
    with v.Card():
        v.SkeletonLoader(type_="table-heading, table-row@5", boilerplate=True)
```

CSS for fade-in transition (in `assets/custom.css`):

```css
.solara-FigurePlotly,
.v-card {
  animation: fadeIn 0.3s ease-in;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}
```

### Data Table with Export

```python
@solara.component
def DataTableCard(df: pd.DataFrame, loading: bool):
    with solara.Card("Data Table"):
        with solara.CardActions():
            solara.FileDownload(
                df.to_csv(index=False) if df is not None else "",
                filename="filtered_data.csv",
                label="Export CSV",
            )
        if loading:
            SkeletonTableCard()
        else:
            solara.DataFrame(df, items_per_page=20)
```

---

## Solara / ipyvuetify API Reference

| Component | Package | Purpose |
|-----------|---------|---------|
| `solara.ColumnsResponsive` | solara | Responsive grid: `ColumnsResponsive(12, large=6)` = 1 col mobile, 2 cols desktop |
| `solara.Card` | solara | Card container with `title`, `margin`, `CardActions` |
| `solara.Select` | solara | Dropdown; `values`, `value`, `on_value` |
| `solara.SliderInt` | solara | Integer slider; `value`, `min`, `max`, `on_value` |
| `solara.lab.InputDate` | solara.lab | Date picker; `value`, `on_value` |
| `solara.FigurePlotly` | solara | Renders Plotly `go.Figure` or dict |
| `solara.DataFrame` | solara | Paginated table; `items_per_page`, sortable columns |
| `solara.FileDownload` | solara | Download button; `data`, `filename`, `label` |
| `solara.reactive()` | solara | Reactive state; `.value`, `.set()` |
| `solara.use_memo(fn, deps)` | solara | Memoize; recompute when deps change |
| `solara.lab.use_task(fn)` | solara.lab | Async task; `.result`, `.finished`, `.error` |
| `solara.CrossFilterDataFrame` | solara | Cross-filter coordination |
| `solara.use_cross_filter()` | solara | Hook for cross-filtered DataFrame |
| `v.SkeletonLoader` | ipyvuetify | `type_="card-heading, image"` or `"table-heading, table-row@5"` |
| `v.Card`, `v.CardTitle`, `v.CardText` | ipyvuetify | KPI card structure |

---

## Dependencies

- `solara` — reactive UI framework
- `solara.lab` — `use_task`, `InputDate`
- `solara` — `use_cross_filter` (api/hooks)
- `plotly` — chart generation
- `pandas` — DataFrame operations
- `ipyvuetify` — Vuetify components (v.Card, v.SkeletonLoader)

---

## Acceptance Criteria

- [ ] Dashboard loads with skeleton placeholders, then fades in content
- [ ] Filter bar (Select, SliderInt, InputDate) updates all charts and table reactively
- [ ] Bar, line, and pie/donut charts render with Plotly and update on filter change
- [ ] Clicking a chart segment filters other charts (cross-filter)
- [ ] KPI cards show total revenue, record count, average value from filtered data
- [ ] Data table is paginated (20 rows), sortable, and shows filtered data
- [ ] Export CSV button downloads current filtered dataset
- [ ] Layout is 2 columns on desktop, 1 column on mobile
