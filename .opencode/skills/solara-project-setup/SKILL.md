---
name: solara-project-setup
description: "Scaffolding and setup guide for Solara projects: structure, dependencies, configuration, and dev workflow"
version: 1.0.0
---

# Solara Project Setup

## Installation

```bash
pip install solara
```

For a full data platform with PandasAI and Plotly:

```bash
pip install solara pandasai plotly pandas openpyxl python-dotenv
```

## Scaffolding a New Project

### Quick Start (Directory-Based)

```bash
mkdir my_app
solara create button my_app/01-home.py
solara create markdown my_app/02-about.py
solara run ./my_app
```

### Production App (Package-Based)

```bash
solara create portal my_app
cd my_app
pip install -e .
solara run my_app.pages
```

This generates the recommended structure:

```
my_app/
├── LICENSE
├── Procfile
├── mypy.ini
├── pyproject.toml
└── my_app/
    ├── __init__.py
    ├── components/
    │   ├── __init__.py
    │   ├── header.py
    │   └── layout.py
    ├── content/
    │   └── articles/
    ├── data.py
    └── pages/
        ├── __init__.py
        ├── dashboard.py
        └── settings.py
```

## Recommended Project Structure (Data Platform)

```
my_platform/
├── pyproject.toml
├── .env                        # API keys (gitignored)
├── .gitignore
├── .opencode/                  # AI coding skills (copy from solara-resources)
│   ├── opencode.json
│   ├── skills/
│   └── agents/
├── assets/                     # Global CSS, JS, theme, favicon
│   ├── custom.css
│   ├── custom.js
│   ├── theme.js
│   └── favicon.png
├── public/                     # Static files served at /static/public/
│   └── images/
└── my_platform/                # Python package
    ├── __init__.py
    ├── config.py               # App configuration, env var loading
    ├── data.py                 # Shared reactive state, data stores
    ├── models.py               # Data models / types
    ├── services/               # Business logic, API clients
    │   ├── __init__.py
    │   ├── data_service.py
    │   └── ai_service.py       # PandasAI wrapper
    ├── components/             # Reusable UI components
    │   ├── __init__.py
    │   ├── layout.py           # App shell, navigation
    │   ├── charts.py           # Chart wrappers
    │   ├── data_table.py       # Enhanced data table
    │   └── loading.py          # Skeleton/spinner components
    └── pages/                  # Route pages
        ├── __init__.py         # Layout component
        ├── dashboard.py
        ├── explorer.py
        ├── reports.py
        └── settings.py
```

## pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-platform"
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
    "pytest",
    "playwright",
    "mypy",
    "ruff",
]
```

## Configuration

### Environment Variables (.env)

```bash
# .env (gitignored)
OPENAI_API_KEY=sk-...
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
SOLARA_APP=my_platform.pages
```

### Config Module

```python
# my_platform/config.py
import os
from dotenv import load_dotenv

load_dotenv()

OPENAI_API_KEY = os.getenv("OPENAI_API_KEY", "")
DATABASE_URL = os.getenv("DATABASE_URL", "")
DEBUG = os.getenv("DEBUG", "false").lower() == "true"
```

## Running the App

```bash
# Development (with hot reload)
solara run my_platform.pages --host 0.0.0.0 --port 8765

# The app auto-reloads when you save Python files

# Production
solara run my_platform.pages --production
```

## Dev Workflow

1. **Edit** any `.py` file in the project
2. **Save** -- Solara automatically detects changes and reloads
3. **Browser** refreshes with updated components
4. No need to restart the server for most changes

## Deployment

### Self-Hosted (Docker)

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY pyproject.toml .
RUN pip install .
COPY . .
EXPOSE 8765
CMD ["solara", "run", "my_platform.pages", "--host", "0.0.0.0", "--port", "8765", "--production"]
```

### With Procfile (Heroku / Railway)

```
web: solara run my_platform.pages --host 0.0.0.0 --port $PORT --production
```

### Behind a Reverse Proxy (Nginx)

Solara uses WebSockets. Ensure your proxy forwards them:

```nginx
location / {
    proxy_pass http://127.0.0.1:8765;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

## Testing

```python
# test_components.py
import solara

def test_my_component():
    # Unit test with Solara's test utilities
    box, rc = solara.render(MyComponent(), handle_event=False)
    assert "Expected text" in rc.find(solara.Text).widget.value
```

```bash
# Run tests
pytest

# Browser tests with Playwright
pytest --headed
```

## Key Points

- Always use the package-based structure for anything beyond a prototype
- Put `assets/` one level above `pages/` for CSS/JS/theme files
- Use `.env` for secrets, load with `python-dotenv`
- Solara hot-reloads on save -- no manual restart needed
- WebSocket support is required for deployment proxies
