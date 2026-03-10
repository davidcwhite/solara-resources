---
name: solara-multipage
description: "Guide for structuring multipage Solara apps: routing, layouts, AppLayout, Sidebar, and dynamic pages"
version: 1.0.0
---

# Solara Multipage Applications

Actionable guidance for structuring multipage Solara applications with routing, layouts, AppLayout, Sidebar, and dynamic pages.

---

## Directory-Based Routing (Simple)

Create a directory with numbered Python files. Each file must define a `Page` component. Solara auto-generates routes from filenames.

```
my_app/
├── 01-dashboard.py
├── 02-explorer.py
└── 03-settings.py
```

Run with:

```bash
solara run ./my_app
```

- File `01-dashboard.py` becomes `/` (first = default)
- `02-explorer.py` becomes `/explorer`
- `03-settings.py` becomes `/settings`

**Filename conventions:** strip numeric prefix, replace `-`/`_` with `-` in URL, capitalize for title.

---

## Package-Based Routing (Recommended for Larger Apps)

Use `solara create portal my_app_name` to scaffold. Structure:

```
my_app/
├── pyproject.toml
└── my_app/
    ├── __init__.py
    ├── components/         # Reusable components
    │   ├── __init__.py
    │   ├── header.py
    │   └── layout.py
    ├── data.py             # Shared state / data stores
    └── pages/              # Route definitions
        ├── __init__.py     # Can define Layout component
        ├── dashboard.py    # /dashboard or / if first
        ├── explorer/
        │   ├── __init__.py # /explorer
        │   └── detail.py   # /explorer/detail
        └── settings.py     # /settings
```

Run with:

```bash
solara run my_app.pages
```

---

## Layout Component

Define a `Layout` component in `pages/__init__.py` to wrap all pages. If not defined, Solara provides a default with sidebar navigation.

```python
# pages/__init__.py
import solara

@solara.component
def Layout(children=[]):
    with solara.AppLayout(title="My Data Platform", navigation=True):
        with solara.Sidebar():
            solara.Button("Dashboard", icon_name="mdi-view-dashboard", href="/")
            solara.Button("Explorer", icon_name="mdi-magnify", href="/explorer")
            solara.Button("Reports", icon_name="mdi-file-chart", href="/reports")
        solara.Column(children=children)
```

---

## AppLayout Component

`solara.AppLayout(title, toolbar_dark, navigation, children)` provides the shell: app bar + optional sidebar + content area.

- Set `navigation=True` to show auto-generated navigation
- Use with `solara.AppBar()` and `solara.AppBarTitle()` to customize the top bar

---

## Page Components

Each page file defines a `Page` component (capital P).

```python
# pages/dashboard.py
import solara

@solara.component
def Page():
    solara.Markdown("# Dashboard")
    # ... page content
```

---

## Dynamic Routes

Add a parameter to `Page` to accept URL segments.

```python
# pages/detail.py
@solara.component
def Page(name: str = "default"):
    solara.Markdown(f"Viewing: {name}")
    # URL /detail/my-item passes name="my-item"
```

---

## Programmatic Routes (Single File)

For smaller apps, define routes explicitly in a single file:

```python
import solara

@solara.component
def Home():
    solara.Markdown("Home page")

@solara.component
def About():
    solara.Markdown("About page")

routes = [
    solara.Route(path="/", component=Home, label="Home"),
    solara.Route(path="about", component=About, label="About"),
]
```

---

## Navigation

- `solara.Link(path_or_route, children)` — client-side navigation
- `solara.Button(..., href="/path")` — button that navigates
- `use_router()` and `use_route()` hooks for programmatic navigation

```python
router = solara.use_router()
# router.push("/explorer")  # navigate programmatically
```

---

## Sidebar Patterns

Use `use_route()` to highlight the active route in the sidebar:

```python
@solara.component
def Layout(children=[]):
    route, routes = solara.use_route()
    with solara.AppLayout(title="App"):
        with solara.Sidebar():
            for r in routes:
                with solara.Link(r):
                    solara.Button(
                        r.label,
                        text=True,
                        color="primary" if r == route else None
                    )
        solara.Column(children=children)
```

---

## Key Takeaways

- **For production apps**, use the package-based structure
- Define `Layout` in `pages/__init__.py` for custom chrome
- Every page file needs a `Page` component
- Use `solara.AppLayout` as the outer shell
- Use `solara.Sidebar()` inside AppLayout for sidebar content
