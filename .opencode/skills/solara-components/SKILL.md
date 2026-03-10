---
name: solara-components
description: "Reference for building Solara UI components: component model, hooks, built-in widgets, and composition patterns"
version: 1.0.0
---

# Solara Components Skill

A practical reference for building Solara UI components. Solara is a Python web framework built on ipywidgets and ipyvuetify. **Components use context managers (the `with` statement) to define parent-child relationships**—they do not return JSX or children directly.

---

## Component Basics

- Use the `@solara.component` decorator to define functional components.
- Components are functions that **return nothing**—they render by using context managers or calling child components.
- Use `with Component():` to nest children; children are defined by what you render inside the `with` block.

```python
import solara

@solara.component
def Greeting(name: str):
    with solara.Column():
        solara.Markdown(f"# Hello, {name}!")
        solara.Button("Click me", on_click=lambda: print("Clicked"))

@solara.component
def Page():
    with solara.Column(gap="1rem"):
        Greeting("World")
        Greeting("Solara")
```

---

## Hooks

### `solara.use_state(initial)`

Returns `(value, setter)`. Changing state via the setter triggers a re-render.

```python
@solara.component
def Counter():
    count, set_count = solara.use_state(0)
    with solara.Row():
        solara.Button(f"Count: {count}", on_click=lambda: set_count(count + 1))
        # Prefer functional updates to avoid stale closures:
        solara.Button("Reset", on_click=lambda: set_count(lambda prev: 0))
```

### `solara.use_effect(fn, dependencies)`

Runs side effects after render. Return a cleanup function to run before re-execution or on unmount. Use for attaching event handlers, subscriptions, or other non-declarative code. `dependencies` triggers re-run when changed; empty list `[]` runs once.

```python
@solara.component
def SearchWithEffect():
    query, set_query = solara.use_state("")
    def log_search():
        if query:
            print(f"Searching: {query}")
        def cleanup():
            print("Cleanup")
        return cleanup
    solara.use_effect(log_search, [query])
    solara.InputText("Search", value=query, on_value=set_query)
```

### `solara.use_memo(fn, dependencies)`

Memoizes expensive computations. Re-runs only when dependencies change.

```python
@solara.component
def ExpensiveList(items: list):
    sorted_items = solara.use_memo(
        lambda: sorted(items, key=lambda x: x["name"]),
        [items]
    )
    with solara.Column():
        for item in sorted_items:
            solara.Markdown(f"- {item['name']}")
```

### `solara.use_previous(value)`

Returns the value from the previous render (or current value on first render).

```python
@solara.component
def DiffDisplay():
    value, set_value = solara.use_state(0)
    prev = solara.use_previous(value)
    solara.Markdown(f"Current: {value}, Previous: {prev}")
    solara.Button("Increment", on_click=lambda: set_value(value + 1))
```

### Rules of Hooks

- Call hooks only at the **top level** of a component (not inside conditionals or loops).
- Call hooks in the **same order** every render.

---

## Built-in Input Components

```python
# Button
solara.Button(label="Submit", on_click=handler, icon_name="mdi-send", color="primary", disabled=False)

# Text input
solara.InputText(label="Name", value=text, on_value=set_text)

# Select dropdown
solara.Select(label="Country", value=selected, values=["US", "UK", "DE"], on_value=set_selected)

# Sliders
solara.SliderInt(label="Age", value=age, min=0, max=120, on_value=set_age)
solara.SliderFloat(label="Score", value=score, min=0, max=1, on_value=set_score)
solara.SliderRangeInt(label="Range", value=range_val, min=0, max=100, on_value=set_range_val)  # (min, max) tuple
solara.SliderValue(label="Fruit", value=choice, values=["apple", "banana", "cherry"], on_value=set_choice)  # categorical

# Checkbox and Switch
solara.Checkbox(label="Agree", value=checked, on_value=set_checked)
solara.Switch(label="Enable", value=enabled, on_value=set_enabled)

# File drop
solara.FileDrop(on_file=lambda f: handle_file(f), label="Drop files here")

# Toggle buttons (single or multiple selection)
solara.ToggleButtonsSingle(value=choice, values=["A", "B", "C"], on_value=set_choice)
solara.ToggleButtonsMultiple(value=choices, values=["X", "Y", "Z"], on_value=set_choices)
```

---

## Layout Components

Use `with Component():` to nest children. Layout components are **context managers**.

```python
# Vertical stack
with solara.Column(children=[], gap="1rem", align="stretch"):
    solara.Markdown("Child 1")
    solara.Button("Child 2")

# Horizontal stack
with solara.Row(children=[], gap="0.5rem", justify="center"):
    solara.Button("Left")
    solara.Button("Right")

# Card with actions
with solara.Card(title="My Card"):
    solara.Markdown("Card content")
    with solara.CardActions():
        solara.Button("Action 1")
        solara.Button("Action 2")

# Column grid with proportional widths
solara.Columns([1, 2, 1])  # 1:2:1 ratio

# Responsive columns
solara.ColumnsResponsive(default=4, small=1, medium=2, large=4)

# Fixed grid
solara.GridFixed(columns=3)
solara.GridDraggable(columns=3)

# Sidebar (inside AppLayout)
with solara.Sidebar():
    solara.Markdown("Sidebar content")

# Collapsible section
with solara.Details(summary="Click to expand"):
    solara.Markdown("Hidden content")
```

---

## Output/Display Components

```python
solara.Markdown("# Hello\n\nThis is **markdown**.")
solara.HTML(tag="div", attributes={"class": "custom"}, children=["Raw HTML"])
solara.Image(url="https://example.com/image.png")  # or data=bytes for inline
solara.FileDownload(data=csv_bytes, filename="export.csv", label="Download CSV")
solara.Tooltip(tooltip_text="Hover hint", children=[solara.Button("Hover me")])
```

---

## Status Components

```python
solara.SpinnerSolara(size="large")  # Loading spinner
solara.Progress(value=0.5, color="primary")  # 0–1, or value=True for indeterminate
solara.Error(label="Something went wrong", icon=True)
solara.Warning(label="Be careful")
solara.Info(label="FYI")
solara.Success(label="Done!")
```

---

## Data Display Components

```python
import pandas as pd

solara.DataFrame(df, items_per_page=10)  # Interactive paginated table
solara.PivotTable(df, rows=["region"], columns=["year"], values=["sales"])
```

---

## Visualization Components

```python
import plotly.express as px
import altair as alt

# Plotly
fig = px.scatter(df, x="x", y="y")
solara.FigurePlotly(fig)

# Altair
chart = alt.Chart(df).mark_point().encode(x="x", y="y")
solara.FigureAltair(chart)

# Matplotlib
import matplotlib.pyplot as plt
fig, ax = plt.subplots()
ax.plot([1, 2, 3], [1, 4, 9])
solara.FigureMatplotlib(figure=fig)
```

---

## Lab Components (`solara.lab`)

```python
import solara.lab

# Chat UI
with solara.lab.ChatBox():
    solara.lab.ChatMessage(user=True, children=[solara.Markdown("Hello")])
    solara.lab.ChatMessage(user=False, children=[solara.Markdown("Hi!")])
solara.lab.ChatInput(send_callback=lambda msg: handle_send(msg))

# Confirmation dialog
solara.lab.ConfirmationDialog(
    open=show_dialog,
    on_ok=lambda: confirm(),
    on_cancel=lambda: set_show_dialog(False),
    title="Confirm",
    content="Are you sure?"
)

# Tabs
with solara.lab.Tabs():
    with solara.lab.Tab("Tab 1"):
        solara.Markdown("Content 1")
    with solara.lab.Tab("Tab 2"):
        solara.Markdown("Content 2")

# Menu
with solara.lab.Menu(activator=solara.Button("Open menu")):
    solara.Button("Option A")
    solara.Button("Option B")

# Date/Time inputs
solara.lab.InputDate(value=date_val, on_value=set_date_val)
solara.lab.InputTime(value=time_val, on_value=set_time_val)

# Theme toggle
solara.lab.ThemeToggle()

# Async task with pending/error/result states
task = solara.lab.use_task(fetch_data, dependencies=[url])
if task.pending:
    solara.SpinnerSolara()
elif task.error:
    solara.Error(str(task.error))
else:
    solara.Markdown(str(task.result))
```

---

## Composition Pattern

A realistic example combining state, memoization, layout, and data components:

```python
import solara
import pandas as pd

@solara.component
def DataCard(title: str, df: pd.DataFrame):
    filter_text, set_filter_text = solara.use_state("")

    filtered = solara.use_memo(
        lambda: df[df["name"].str.contains(filter_text, case=False)] if filter_text else df,
        [filter_text, df]
    )

    with solara.Card(title):
        solara.InputText("Filter", value=filter_text, on_value=set_filter_text)
        solara.DataFrame(filtered, items_per_page=10)
        with solara.CardActions():
            solara.FileDownload(
                filtered.to_csv(index=False),
                filename="filtered_data.csv",
                label="Export CSV"
            )
```

---

## Key Takeaways

1. **Context managers**: Use `with solara.Column():`, `with solara.Card():`, etc. to define parent-child structure. Components do not return children.
2. **State**: `use_state` for reactive state; `use_memo` for derived values; `use_effect` for side effects.
3. **Controlled inputs**: Pass `value` and `on_value` (or `on_click`) to keep inputs in sync with state.
4. **Hooks rules**: Top level only, same order every render.
