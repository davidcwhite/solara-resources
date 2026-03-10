---
name: solara-ipyvuetify-styling
description: "ipyvuetify components, skeleton loaders, spinners, custom CSS/animations, slots, theming, and web-native patterns that are tricky in Solara"
version: 1.0.0
---

# Solara ipyvuetify Styling & Web-Native Patterns

This skill covers the tricky web-native patterns in Solara that are hard to figure out from docs alone. All examples are copy-paste ready.

---

## Using ipyvuetify in Solara

Two ways to use ipyvuetify:

- **`import ipyvuetify as v`** — imperative widget API (use for components Solara doesn't wrap)
- **`import reacton.ipyvuetify as rv`** — Reacton/declarative wrapper (type-safe, works with Solara rendering)

**When to use which:** Prefer `rv` for components in Solara component trees; use `v` when you need imperative widget manipulation or access to Vuetify components not in Solara.

ipyvuetify wraps Vuetify 2 components. Refer to https://v2.vuetifyjs.com/ for all available components and their props.

---

## Skeleton Loading Effect

```python
import ipyvuetify as v
import solara

@solara.component
def SkeletonCard():
    loading, set_loading = solara.use_state(True)
    
    skeleton = v.SkeletonLoader(
        type="card-heading, list-item-three-line, actions",
        loading=loading,
    )
    solara.display(skeleton)

@solara.component
def SkeletonTable(rows: int = 5):
    """Skeleton that mimics a data table while loading."""
    skeleton_type = ", ".join(["table-heading"] + ["table-row"] * rows)
    solara.display(v.SkeletonLoader(type=skeleton_type, loading=True))
```

**Available skeleton types:** `article`, `card`, `card-heading`, `card-avatar`, `list-item`, `list-item-avatar`, `list-item-two-line`, `list-item-three-line`, `table-heading`, `table-thead`, `table-tbody`, `table-row`, `table-row-divider`, `table-tfoot`, `actions`, `button`, `chip`, `date-picker`, `date-picker-options`, `date-picker-days`, `heading`, `image`, `paragraph`, `sentences`, `text`

Combine types with commas: `type="card-heading, paragraph, actions"`

Use `boilerplate=True` for full-width skeleton with avatar.

---

## Spinning / Circular Progress

```python
import ipyvuetify as v
import solara

@solara.component
def LoadingOverlay(loading: bool):
    if loading:
        solara.display(v.ProgressCircular(
            indeterminate=True,
            color="primary",
            size=64,
            width=4,
        ))

@solara.component  
def LoadingWithText():
    with solara.Column(align="center"):
        solara.display(v.ProgressCircular(indeterminate=True, color="primary", size=48))
        solara.Text("Loading data...")
```

- Also available: `solara.SpinnerSolara(size="large")` for Solara's built-in spinner
- `solara.Progress(value=True)` for indeterminate linear progress, `solara.Progress(value=65)` for determinate

---

## Combining Loading States with use_task

```python
@solara.component
def DataPanel(query: str):
    result = solara.lab.use_task(lambda: expensive_query(query), dependencies=[query])
    
    if result.pending:
        solara.display(v.SkeletonLoader(type="table-heading, table-row@5", loading=True))
    elif result.error:
        solara.Error(f"Query failed: {result.error}")
    else:
        solara.DataFrame(result.value)
```

---

## Custom CSS with solara.Style

```python
import solara

@solara.component
def StyledPage():
    # Inline CSS string
    solara.Style("""
        .shimmer {
            background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
            background-size: 200% 100%;
            animation: shimmer 1.5s infinite;
        }
        @keyframes shimmer {
            0% { background-position: -200% 0; }
            100% { background-position: 200% 0; }
        }
        
        .fade-in {
            animation: fadeIn 0.3s ease-in;
        }
        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(10px); }
            to { opacity: 1; transform: translateY(0); }
        }
        
        .pulse {
            animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
        }
        @keyframes pulse {
            0%, 100% { opacity: 1; }
            50% { opacity: 0.5; }
        }
    """)
    # Or load from file: solara.Style(Path("assets/custom.css"))
```

---

## Global CSS via assets directory

Place `custom.css` in `assets/` directory (one level above `pages/`):

```
├── pages/
│   └── ...
├── assets/
│   ├── custom.css
│   ├── custom.js
│   └── theme.js
```

Files in `assets/` are automatically loaded by Solara server.

---

## Overriding Vuetify Styles

```css
/* Vuetify generates high-specificity selectors, so you need to match or beat them */

/* Use the component class + your custom class */
.v-btn.my-custom-btn {
    color: #4CAF50;
}

/* For background-color, Vuetify's specificity is very high -- use !important */
.my-custom-btn {
    background-color: #FF9800 !important;
}

/* Override card styles */
.v-card.dashboard-card {
    border-radius: 12px;
    transition: box-shadow 0.3s ease;
}
.v-card.dashboard-card:hover {
    box-shadow: 0 8px 25px rgba(0,0,0,0.15);
}
```

---

## Styling Props — The Underscore Convention

```python
import ipyvuetify as v

# style_ (not style) to avoid Python keyword collision
card = v.Card(
    style_="border-radius: 12px; max-width: 400px;",
    class_="ma-2 elevation-3",  # class_ (not class)
    outlined=True,
    children=[
        v.CardTitle(children=["My Card"]),
        v.CardText(children=["Content here"]),
    ]
)

# For HTML-specific attrs not in Vuetify, use attributes dict
v.Btn(
    class_="ma-2",
    attributes={"download": True, "data-testid": "export-btn"},
    children=["Download"]
)

# Solara components use classes (list) and style (str or dict)
solara.Button("Click", classes=["my-custom-btn"])
solara.Column(style={"padding": "16px", "gap": "8px"})
```

---

## Vuetify CSS Utility Classes

```python
# Margin/Padding: ma-N, pa-N, mx-N, my-N, ml-N, mt-N, etc. (N = 0-16, each unit = 4px)
v.Card(class_="ma-4 pa-3")  # margin-all: 16px, padding-all: 12px

# Display: d-flex, d-none, d-block
# Flex: justify-center, align-center, flex-column, flex-wrap
v.Row(class_="d-flex justify-center align-center")

# Text: text-center, text-h4, text-body-1, font-weight-bold
v.CardTitle(class_="text-h5 font-weight-bold", children=["Title"])

# Colors: primary--text, error--text, grey--text, white--text
v.Icon(class_="primary--text", children=["mdi-check"])

# Elevation: elevation-N (N = 0-24)
v.Card(class_="elevation-4")

# Responsive display: d-none d-md-block (hidden below md breakpoint)
```

---

## Slots and Scoped Slots

```python
import ipyvuetify as v

# Named slot -- e.g., "no-data" slot on v-select
select = v.Select(
    label="Choose item",
    items=[],
    v_slots=[{
        'name': 'no-data',
        'children': [
            v.ListItem(children=[
                v.ListItemTitle(children=['No matching items found'])
            ])
        ]
    }]
)

# Scoped slot -- e.g., tooltip activator
tooltip = v.Tooltip(bottom=True, v_slots=[{
    'name': 'activator',
    'variable': 'tooltip',
    'children': v.Btn(
        v_on='tooltip.on',
        color='primary',
        children=['Hover me']
    ),
}], children=['This is the tooltip text'])

# Icon button with tooltip
icon_with_tooltip = v.Tooltip(bottom=True, v_slots=[{
    'name': 'activator',
    'variable': 'x',
    'children': v.Btn(
        v_on='x.on',
        icon=True,
        children=[v.Icon(children=['mdi-information'])]
    ),
}], children=['More information'])
```

---

## Event Handling

```python
import ipyvuetify as v
import reacton.ipyvuetify as rv
import solara

# Reacton style (preferred in Solara components)
@solara.component
def ClickCounter():
    count, set_count = solara.use_state(0)
    btn = rv.Btn(children=[f"Clicked {count} times"])
    rv.use_event(btn, 'click', lambda *_: set_count(count + 1))
    return btn

# Imperative style (for standalone ipyvuetify widgets)
btn = v.Btn(color='primary', children=['Click me'])
btn.on_event('click', lambda widget, event, data: print('clicked'))

# Event modifiers
icon = v.Icon(children=['mdi-close'])
icon.on_event('click.stop', lambda *_: print('icon only, no bubble'))
```

---

## Two-Way Binding (.sync pattern)

```python
import ipyvuetify as v

# Vuetify's .sync becomes on_event('update:propName')
drawer = v.NavigationDrawer(
    mini_variant=True,
    permanent=True,
    children=[v.ListItem(children=[v.ListItemTitle(children=["Item"])])]
)

def handle_mini_update(widget, event, data):
    drawer.mini_variant = data

drawer.on_event('update:miniVariant', handle_mini_update)
```

---

## Responsive Grid Layout

```python
import ipyvuetify as v

# 2-column on desktop, 1-column on mobile
layout = v.Container(fluid=True, children=[
    v.Row(children=[
        v.Col(cols=12, md=6, children=[
            v.Card(outlined=True, children=[
                v.CardTitle(children=["Chart"]),
                v.CardText(children=["Plotly chart here"])
            ])
        ]),
        v.Col(cols=12, md=6, children=[
            v.Card(outlined=True, children=[
                v.CardTitle(children=["Table"]),
                v.CardText(children=["Data table here"])
            ])
        ]),
    ])
])

# Breakpoints: xs (<600px), sm (600-960), md (960-1264), lg (1264-1904), xl (>1904)
```

---

## Theming

```javascript
// assets/theme.js -- Vuetify theme configuration
module.exports = {
    dark: false,
    themes: {
        light: {
            primary: '#1976D2',
            secondary: '#424242',
            accent: '#82B1FF',
            error: '#FF5252',
            info: '#2196F3',
            success: '#4CAF50',
            warning: '#FFC107',
        },
        dark: {
            primary: '#2196F3',
            secondary: '#424242',
            accent: '#FF4081',
        }
    }
}
```

- Toggle dark mode: `solara.lab.ThemeToggle()`
- Detect current theme: `dark_effective = solara.lab.use_dark_effective()`

---

## Common Gotchas

1. Use `style_` and `class_` (with underscore) on ipyvuetify widgets, NOT `style` or `class`
2. Solara components use `classes=["my-class"]` (list) and `style={"key": "value"}` (dict or string)
3. `v_slots` takes a list of dicts, not a single dict
4. When overriding Vuetify background colors in CSS, you almost always need `!important`
5. `solara.display(widget)` is needed to render raw ipyvuetify widgets inside Solara components
6. For icons, use Material Design Icons format: `mdi-icon-name` (browse at https://materialdesignicons.com/)
7. Vuetify 2 docs (not v3) apply: https://v2.vuetifyjs.com/
8. `v.Html(tag="div", class_="my-class", children=[...])` for arbitrary HTML elements with Vuetify styling
