# PRD 05: Architecture and Technology Stack

## Overview

This document defines cross-cutting architectural decisions, dependency management, configuration, deployment, testing, and state architecture for the Solara + PandasAI 2.3 data platform. It serves as the technical reference for implementation, ensuring consistency across all feature PRDs ([01](01-project-overview.md), [02](02-data-analytics-dashboard.md), [03](03-conversational-data-explorer.md), [04](04-reporting-and-export.md)).

---

## Technology Decisions

### UI Framework: Solara

| Aspect | Decision |
|--------|----------|
| **Why** | Pure Python, reactive components, built-in multipage routing, hot reload, production-ready |
| **Component Stack** | Built on Reacton + ipyvuetify (Vuetify 2 material design) |
| **Developer Experience** | No JavaScript/HTML required for most features |
| **Data Integration** | Built-in `DataFrame`, `PivotTable`, `FigurePlotly`, `FigureAltair`, `FigureMatplotlib` |

**Key Solara APIs:**

- `solara.component` — decorator for reactive components
- `solara.Page` — route-level page component
- `solara.routing` — `use_route()`, `use_pathname()` for navigation
- `solara.AppLayout` — shell with AppBar, Sidebar, Content
- `solara.Card`, `solara.Column`, `solara.Row`, `solara.ColumnsResponsive` — layout primitives

### NL Data Queries: PandasAI 2.3

| Aspect | Decision |
|--------|----------|
| **Why** | Natural language to pandas code, supports multiple LLMs, data connectors |
| **Primary Interface** | `Agent` class with `.chat()` method |
| **LLM Support** | OpenAI, Google PaLM, Azure OpenAI, HuggingFace, LangChain-wrapped models |
| **Data Connectors** | CSV, XLSX, PostgreSQL, MySQL, BigQuery, Databricks, Snowflake |

**Core API:**

```python
from pandasai import Agent

agent = Agent(df, config={"llm": llm})
response = agent.chat("Which category has the highest revenue?")
```

### Charting: Plotly

| Aspect | Decision |
|--------|----------|
| **Why** | Interactive charts, wide chart type support, works natively with Solara via `FigurePlotly` |
| **Quick Charts** | `plotly.express` (px.bar, px.line, px.pie, px.scatter) |
| **Custom Layouts** | `plotly.graph_objects` (go.Figure, go.Bar, go.Scatter) |

**Solara Integration:** Pass a `go.Figure` or dict to `solara.FigurePlotly(figure)`.

### Data Processing: pandas

| Aspect | Decision |
|--------|----------|
| **Why** | Industry standard, PandasAI operates on DataFrames, Solara's `DataFrame` component accepts pandas |
| **Version** | pandas >= 2.0 |

---

## Dependency Versions

Use `pyproject.toml` for dependency management:

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "solara-data-platform"
version = "0.1.0"
requires-python = ">=3.9"
dependencies = [
    "solara>=1.35",
    "pandasai>=2.3,<3.0",
    "plotly>=5.18",
    "pandas>=2.0",
    "openpyxl>=3.1",
    "python-dotenv>=1.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "playwright>=1.40",
    "mypy>=1.0",
    "ruff>=0.1",
]
```

**Version Rationale:**

- `solara>=1.35` — includes `use_task`, `use_cross_filter`, `InputDate`, production mode
- `pandasai>=2.3,<3.0` — Agent API, `.chat()`, LLM config; pin below 3.0 to avoid breaking changes
- `plotly>=5.18` — stable Express and graph_objects APIs
- `openpyxl>=3.1` — Excel read/write for DataService and export

---

## Project Structure

Full package-based structure:

```
solara_data_platform/
├── pyproject.toml
├── .env                    # Secrets (gitignored)
├── .gitignore
├── .opencode/              # AI coding skills and agents
│   ├── opencode.json
│   ├── skills/
│   └── agents/
├── assets/                 # Global CSS, JS, theme
│   ├── custom.css
│   ├── theme.js
│   └── favicon.png
├── public/                 # Static files served at /static/public/
│   └── images/
└── solara_data_platform/   # Package root
    ├── __init__.py
    ├── config.py           # Env var loading, app config
    ├── data.py             # Shared reactive state
    ├── models.py           # Data models / types
    ├── services/           # Business logic
    │   ├── __init__.py
    │   ├── data_service.py # Data loading, caching, connectors
    │   └── ai_service.py   # PandasAI Agent wrapper
    ├── components/         # Reusable UI
    │   ├── __init__.py
    │   ├── layout.py       # AppLayout, sidebar, navigation
    │   ├── charts.py       # Chart wrappers (FigurePlotly)
    │   ├── data_table.py   # Enhanced DataFrame with filtering
    │   ├── chat.py         # Chat UI for Explorer
    │   └── loading.py      # Skeleton, spinner components
    └── pages/              # Route components
        ├── __init__.py     # Layout + routing
        ├── dashboard.py
        ├── explorer.py
        ├── reports.py
        └── settings.py
```

**Directory Responsibilities:**

| Directory | Purpose |
|-----------|---------|
| `pages/` | Route-level components; one file per route; imports layout from `__init__.py` |
| `components/` | Reusable UI; no routing; receive props and reactive state |
| `services/` | Business logic; no Solara imports; pure Python + pandas |
| `data.py` | Module-level `solara.reactive()` for shared state |
| `config.py` | Load `.env` with `python-dotenv`; expose `OPENAI_API_KEY`, `DATABASE_URL`, etc. |
| `assets/` | CSS overrides, theme toggle JS, favicon |
| `public/` | Static images, PDFs; served at `/static/public/` |

---

## State Architecture

### State Management Strategy

| State Type | API | Scope | Use Case |
|------------|-----|-------|----------|
| **Shared app state** | `solara.reactive()` at module level | All components, all sessions* | Selected dataset, filter values, app config |
| **Per-session init** | `solara.on_kernel_start(callback)` | Per kernel/session | Load user defaults, init per-user reactive vars |
| **Component-local** | `solara.use_state(initial)` | Single component instance | Form inputs, toggles, selected tab |
| **Async data** | `solara.lab.use_task(fn, dependencies)` | Component | Data loading; `.pending`, `.error`, `.result` |
| **Derived/computed** | `solara.use_memo(fn, deps)` | Component | Expensive computations, Plotly figures |
| **Cross-filter** | `solara.use_cross_filter()`, `CrossFilterDataFrame` | Dashboard coordination | Chart click → filter all charts |

\*Module-level `reactive()` is shared across sessions in a multi-user deployment. For per-user state, create reactive vars inside `on_kernel_start()`.

### Data Flow

**User action → state update → reactive propagation → component re-render → updated UI:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  User Action (click, select, type)                                           │
│  → on_click / on_value / on_change                                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  State Update                                                                │
│  reactive_var.set(new_value)  OR  set_state(new_value)                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Reactive Propagation                                                        │
│  All components that read reactive_var.value re-render                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Component Re-render                                                          │
│  use_task re-runs if dependencies changed                                    │
│  use_memo recomputes if deps changed                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Updated UI                                                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Data loading flow: config → DataService → DataFrame → reactive store → components:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  config.py                                                                   │
│  DATABASE_URL, OPENAI_API_KEY, DATA_PATH                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  DataService / AIService                                                     │
│  load_data(), get_filtered_df(), Agent(df).chat()                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  use_task(load_data, dependencies=[filter_state])                             │
│  Returns: task.result (DataFrame), task.pending, task.error                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  reactive store (optional)                                                   │
│  selected_dataset.value = df  — for cross-page sharing                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Components                                                                  │
│  solara.DataFrame(df), solara.FigurePlotly(figure)                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Cross-Filter State (Dashboard)

For coordinated filtering across charts:

```python
from solara import use_cross_filter

@solara.component
def DashboardPage():
    df, set_cross_filter = use_cross_filter(data_key="dashboard", name="main")
    # df is filtered by chart selections; apply UI filters on top
    filtered = apply_ui_filters(df, category_filter, date_start, date_end)
    # Charts receive filtered; selections propagate via set_cross_filter
```

---

## Configuration Management

### .env File

Store secrets in `.env` (gitignored):

```bash
# .env
OPENAI_API_KEY=sk-...
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
DATA_PATH=/data
DEBUG=false
```

### config.py Module

```python
# solara_data_platform/config.py
import os
from dotenv import load_dotenv

load_dotenv()

OPENAI_API_KEY = os.getenv("OPENAI_API_KEY", "")
DATABASE_URL = os.getenv("DATABASE_URL", "")
DATA_PATH = os.getenv("DATA_PATH", "./data")
DEBUG = os.getenv("DEBUG", "false").lower() == "true"
```

### Environment-Specific Config

Use env var prefixes for multiple environments:

```bash
# .env.development
OPENAI_API_KEY=sk-dev-...
DATABASE_URL=postgresql://localhost:5432/dev_db

# .env.production (never commit)
OPENAI_API_KEY=sk-prod-...
DATABASE_URL=postgresql://prod-host:5432/prod_db
```

Load with: `load_dotenv(".env.production")` or `load_dotenv(os.getenv("ENV_FILE", ".env"))`.

### Security Rule

**Never commit secrets to git.** Ensure `.env`, `.env.*` (except `.env.example`) are in `.gitignore`.

---

## Deployment Architecture

### Docker Container

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY pyproject.toml .
RUN pip install --no-cache-dir .
COPY . .
EXPOSE 8765
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8765/ || exit 1
CMD ["solara", "run", "solara_data_platform.pages", "--host", "0.0.0.0", "--port", "8765", "--production"]
```

**Production mode:** `--production` disables hot reload and optimizes for stability.

### Nginx Reverse Proxy

Solara uses WebSockets. Configure Nginx:

```nginx
upstream solara {
    server 127.0.0.1:8765;
}

server {
    listen 80;
    server_name data.example.com;

    location / {
        proxy_pass http://solara;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Health Check Endpoint

Solara serves the app at `/`. Use HTTP 200 as health check. For a dedicated endpoint, add a simple route or use `/` with `curl -f`.

### Environment Variables at Runtime

Inject via Docker:

```bash
docker run -e OPENAI_API_KEY=sk-... -e DATABASE_URL=postgresql://... solara-platform
```

Or use `--env-file`:

```bash
docker run --env-file .env.production solara-platform
```

---

## Testing Strategy

### Unit Tests (Services, Data Transformations)

```python
# tests/services/test_data_service.py
import pytest
from solara_data_platform.services.data_service import DataService

def test_load_csv():
    service = DataService()
    df = service.load_data("fixtures/sample.csv")
    assert len(df) > 0
    assert "revenue" in df.columns
```

### Component Tests (Solara Test Utilities)

```python
# tests/components/test_charts.py
import solara
from solara_data_platform.components.charts import ChartCard
import pandas as pd

def test_chart_card_renders():
    df = pd.DataFrame({"x": [1, 2, 3], "y": [10, 20, 30]})
    box, rc = solara.render(ChartCard("Test", df, "bar", x="x", y="y"), handle_event=False)
    assert rc.find(solara.FigurePlotly)
```

### Browser Tests (Playwright)

```python
# tests/e2e/test_dashboard.py
import pytest
from solara.testing import playwright

@playwright
def test_dashboard_filters(page):
    page.goto("http://localhost:8765/")
    page.click("text=Category")
    page.click("text=Electronics")
    # Assert charts update
    assert page.locator(".v-card").count() >= 3
```

### Test Structure

Mirror source structure:

```
tests/
├── conftest.py          # Fixtures, pytest config
├── services/
│   ├── test_data_service.py
│   └── test_ai_service.py
├── components/
│   ├── test_charts.py
│   └── test_loading.py
└── e2e/
    └── test_dashboard.py
```

---

## Security Considerations

### PandasAI Code Execution

PandasAI generates and executes Python code to answer queries. **Implications:**

- Generated code runs in the same process as the app
- Malicious or malformed queries could theoretically produce harmful code
- User input is passed to the LLM and influences generated code

**Sandboxing Approach:**

1. **Input validation:** Sanitize user queries; reject obviously dangerous patterns (e.g., `import os`, `subprocess`, `eval`).
2. **Restricted execution:** PandasAI executes in a controlled context; avoid exposing filesystem or network beyond data connectors.
3. **LLM prompt hardening:** Use PandasAI's built-in prompts; avoid custom prompts that instruct the model to generate arbitrary code.
4. **Audit logging:** Log queries and generated code for debugging and security review.
5. **Rate limiting:** Limit chat requests per user/session to reduce abuse and cost.

### API Keys

- Store in environment variables only; never in code or config files committed to git
- Rotate keys periodically
- Use separate keys for dev/staging/production

### Input Validation

- Validate user queries before passing to PandasAI (length, character set)
- Validate filter values (dates, numeric ranges) before applying to DataFrames
- Use parameterized queries for any raw SQL (if extending beyond PandasAI connectors)

### Rate Limiting

- Implement per-session or per-user limits for LLM API calls
- Consider caching repeated queries
- Use `use_task` to avoid concurrent duplicate requests

---

## Performance Considerations

### Memoization

Use `solara.use_memo()` for expensive computations:

```python
figure = solara.use_memo(
    lambda: create_plotly_figure(filtered_df),
    [filtered_df]
)
```

Key memoization on the filtered DataFrame (or a hash thereof) so figures recompute only when data changes.

### Data Load Caching

- Cache loaded DataFrames in `DataService` to avoid re-fetching on every navigation
- Invalidate cache when data source config changes or on explicit refresh
- Use `use_task` with stable dependencies; avoid re-running when only UI state changes

### Skeleton Loading States

Show `v.SkeletonLoader` while data loads:

```python
if task.pending:
    SkeletonChartCard()
else:
    ChartCard(...)
```

Improves perceived performance; users see immediate feedback.

### Lazy Load Data

- Load data on page navigation, not on app startup
- Use `use_task` so each page fetches only when mounted
- Consider lazy-loading the Explorer page's PandasAI Agent until first chat

### use_task for All IO

- Never block render with synchronous IO
- Use `solara.lab.use_task()` for all data loading, API calls, file reads
- Rely on `task.pending`, `task.error`, `task.result` for UI states

---

## Solara API Quick Reference

| API | Package | Purpose |
|-----|---------|---------|
| `solara.reactive(init)` | solara | Module-level shared state; `.value`, `.set()` |
| `solara.use_state(init)` | solara | Component-local state; `(value, set_value)` |
| `solara.use_reactive(init)` | solara | Accept reactive or initial; returns reactive-like |
| `solara.use_memo(fn, deps)` | solara | Memoize; recompute when deps change |
| `solara.use_effect(fn, deps)` | solara | Side effect; run when deps change; return cleanup |
| `solara.lab.use_task(fn, dependencies)` | solara.lab | Async task; `.result`, `.pending`, `.error`, `.finished` |
| `solara.use_cross_filter(id, name)` | solara | Cross-filter coordination |
| `solara.on_kernel_start(cb)` | solara | Per-session initialization |
| `solara.FigurePlotly(figure)` | solara | Render Plotly figure |
| `solara.DataFrame(df, items_per_page)` | solara | Paginated, sortable table |
| `solara.lab.InputDate` | solara.lab | Date picker |
| `v.SkeletonLoader(type_=...)` | ipyvuetify | Skeleton placeholder |

---

## Implementation Checklist

- [ ] Create `pyproject.toml` with pinned dependency versions
- [ ] Scaffold package structure (`pages/`, `components/`, `services/`, `data.py`, `config.py`)
- [ ] Implement `config.py` with `load_dotenv()` and env var access
- [ ] Define shared reactive state in `data.py`
- [ ] Use `use_task()` for all data loading; never block render
- [ ] Use `use_memo()` for Plotly figures and expensive computations
- [ ] Add `.env.example` (no secrets) and ensure `.env` is gitignored
- [ ] Configure Dockerfile and Nginx for production deployment
- [ ] Add unit tests for services; component tests for key UI
- [ ] Document PandasAI sandboxing and input validation approach
