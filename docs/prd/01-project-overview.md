# PRD 01: Project Overview -- Solara Data Platform

## Vision

Build a multi-page data platform that combines interactive dashboards, conversational data exploration powered by PandasAI, and structured reporting/export capabilities. The platform enables both technical and non-technical users to explore, analyze, and report on data through visual interfaces and natural language queries.

## Goals

1. **Interactive Dashboards** -- Filterable charts and data tables with cross-filter coordination
2. **Conversational Data Explorer** -- Natural language queries via PandasAI with inline chart rendering
3. **Reporting & Export** -- Saved query templates, parameterized reports, CSV/PDF export
4. **Modern UX** -- Skeleton loading states, responsive layout, dark/light theming

## Technology Stack

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| UI Framework | Solara | latest | Reactive Python web components |
| Component Library | ipyvuetify | latest | Vuetify 2 material design widgets |
| NL Data Queries | PandasAI | 2.3.x | Natural language to pandas/SQL |
| Charting | Plotly | latest | Interactive visualizations |
| Data Processing | pandas | latest | DataFrames, transformations |
| Spreadsheet Support | openpyxl | latest | Excel file read/write |
| Configuration | python-dotenv | latest | Environment variable management |

## Target Architecture

```
Browser
  └── Solara App (WebSocket connection)
       ├── AppLayout (shell: AppBar + Sidebar + Content)
       ├── Pages
       │   ├── Dashboard      ── Plotly charts + cross-filters + data tables
       │   ├── Explorer        ── Chat UI + PandasAI Agent + inline results
       │   ├── Reports         ── Saved queries + parameterized export
       │   └── Settings        ── Data source config + LLM config + theme
       ├── Shared State
       │   ├── solara.reactive()  ── App-wide data stores
       │   └── on_kernel_start()  ── Per-session initialization
       └── Services
            ├── DataService     ── Data loading, caching, connectors
            └── AIService       ── PandasAI Agent wrapper
```

## Application Pages

### 1. Dashboard (`/`)
Interactive analytics dashboard with multiple visualization cards. Users can filter data using sliders, selects, and date pickers. Charts update reactively. See [PRD 02](02-data-analytics-dashboard.md).

### 2. Explorer (`/explorer`)
Chat-based interface where users type natural language questions about their data. PandasAI translates queries into pandas operations. Results render as text, tables, or charts inline. See [PRD 03](03-conversational-data-explorer.md).

### 3. Reports (`/reports`)
Saved query templates with parameterizable inputs. Users can generate, preview, and export reports as CSV or PDF. Includes pivot table views. See [PRD 04](04-reporting-and-export.md).

### 4. Settings (`/settings`)
Configuration page for data source connections, LLM API keys, and theme preferences.

## Project Structure

```
solara_data_platform/
├── pyproject.toml
├── .env
├── .opencode/
├── assets/
│   ├── custom.css
│   └── theme.js
└── solara_data_platform/
    ├── __init__.py
    ├── config.py
    ├── data.py                 # Shared reactive state
    ├── models.py               # Data models / types
    ├── services/
    │   ├── __init__.py
    │   ├── data_service.py     # Data loading and caching
    │   └── ai_service.py       # PandasAI Agent wrapper
    ├── components/
    │   ├── __init__.py
    │   ├── layout.py           # App Layout with sidebar
    │   ├── charts.py           # Reusable chart components
    │   ├── data_table.py       # Enhanced data table with filtering
    │   ├── chat.py             # Chat UI components
    │   └── loading.py          # Skeleton and spinner components
    └── pages/
        ├── __init__.py         # Layout component
        ├── dashboard.py
        ├── explorer.py
        ├── reports.py
        └── settings.py
```

## Non-Functional Requirements

- **Performance**: Skeleton loading for all data-dependent views. Memoize expensive computations.
- **Responsiveness**: 2-column layout on desktop, single column on mobile via `ColumnsResponsive` or `v.Row`/`v.Col`.
- **Theming**: Support dark and light modes via `theme.js` and `ThemeToggle`.
- **Error Handling**: Graceful degradation when PandasAI or data sources fail. Inline error messages, not crashes.
- **Security**: API keys in `.env` (gitignored). PandasAI code execution sandboxed.
- **Deployment**: Dockerized, runs behind reverse proxy with WebSocket support.

## Success Criteria

1. Users can view and filter dashboard charts without page reloads
2. Users can ask natural language questions and receive data-backed answers with charts
3. Users can export filtered data and reports as CSV
4. App loads with skeleton placeholders and transitions smoothly to content
5. App works in both light and dark mode
