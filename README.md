# Solara Resources

A collection of AI coding skills and product requirements documents (PRDs) for building data platforms with [Solara](https://solara.dev/) and [PandasAI](https://docs.pandas-ai.com/).

## What's Inside

### `.opencode/` -- AI Coding Skills

Drop-in skills for [OpenCode](https://opencode.ai/) that provide context-aware guidance when building Solara applications. Copy the `.opencode` folder into any Solara project to give your AI assistant deep knowledge of:

- **solara-components** -- Component model, hooks, built-in widgets, and composition patterns
- **solara-state-management** -- Reactive state, cross-component sharing, scoping
- **solara-multipage** -- Routing, layouts, `AppLayout`/`Sidebar`, dynamic pages
- **solara-ipyvuetify-styling** -- ipyvuetify components, skeleton loaders, spinners, custom CSS/animations, slots, theming, and common gotchas
- **solara-project-setup** -- Scaffolding, `pyproject.toml`, dependency management, dev workflow
- **pandasai-integration** -- PandasAI 2.3 Agent API and Solara integration patterns (placeholder, expanding soon)

### `docs/prd/` -- Product Requirements Documents

PRDs for a full-featured data platform combining interactive dashboards, conversational data exploration (via PandasAI), and reporting/export capabilities.

## Usage

### As OpenCode skills in a Solara project

```bash
# Copy the .opencode folder into your Solara project root
cp -r .opencode /path/to/your-solara-project/
```

OpenCode will automatically discover and use the skills when working in that project.

### As reference material

Browse the `docs/prd/` folder for detailed technical specifications covering architecture, technology choices, and implementation approaches for a Solara + PandasAI data platform.

## Technology Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| UI Framework | [Solara](https://solara.dev/) | latest |
| NL Data Queries | [PandasAI](https://docs.pandas-ai.com/) | 2.3.x |
| Charting | [Plotly](https://plotly.com/python/) | latest |
| Data Processing | [pandas](https://pandas.pydata.org/) | latest |
| Component Library | [ipyvuetify](https://ipyvuetify.readthedocs.io/) (Vuetify 2) | latest |

## License

MIT
