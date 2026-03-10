---
name: pandasai-integration
description: "PandasAI 2.3 Agent API for natural language data interaction -- placeholder, to be expanded"
version: 0.1.0
---

# PandasAI 2.3 Integration

> **Status**: Placeholder skill. Core API reference provided; detailed Solara integration patterns coming soon.

## Installation

```bash
pip install "pandasai>=2.3,<3.0"
```

## Core API

### Agent (Primary Interface)

```python
import pandas as pd
from pandasai import Agent

df = pd.DataFrame({
    "country": ["US", "UK", "France", "Germany"],
    "revenue": [5000, 3200, 2900, 4100],
    "quarter": ["Q1", "Q1", "Q1", "Q1"],
})

agent = Agent(df)
response = agent.chat("Which country has the highest revenue?")
# Returns: "US with revenue of 5000"
```

### Multiple DataFrames

```python
agent = Agent([sales_df, products_df, customers_df])
response = agent.chat("What are the top 3 products by total sales amount?")
```

### LLM Configuration

```python
from pandasai.llm import OpenAI

llm = OpenAI(api_token="sk-...", model="gpt-4")
agent = Agent(df, config={"llm": llm})
```

Supported LLMs: OpenAI, Google PaLM, Google VertexAI, Azure OpenAI, HuggingFace, LangChain-wrapped models.

### Data Connectors

```python
from pandasai.connectors import PostgreSQLConnector

connector = PostgreSQLConnector(
    config={
        "host": "localhost",
        "port": 5432,
        "database": "mydb",
        "username": "user",
        "password": "pass",
        "table": "sales",
    }
)
agent = Agent([connector])
```

Supported: CSV, XLSX, PostgreSQL, MySQL, BigQuery, Databricks, Snowflake.

### Visualization

PandasAI can generate charts when asked. The response will include the chart path:

```python
response = agent.chat("Plot a bar chart of revenue by country")
# Saves chart to exports/charts/ by default
```

## Solara Integration Pattern (Preview)

```python
import solara
from pandasai import Agent

data_agent = solara.reactive(None)

@solara.component
def ChatExplorer(df):
    messages, set_messages = solara.use_state([])
    
    def initialize():
        data_agent.value = Agent(df)
    solara.use_effect(initialize, [df])
    
    def send(message: str):
        set_messages(lambda m: m + [{"role": "user", "content": message}])
        if data_agent.value:
            response = data_agent.value.chat(message)
            set_messages(lambda m: m + [{"role": "assistant", "content": str(response)}])
    
    with solara.lab.ChatBox():
        for msg in messages:
            solara.lab.ChatMessage(children=[solara.Markdown(msg["content"])])
        solara.lab.ChatInput(send_callback=send)
```

## TODO (Future Expansion)

- Error handling and retry patterns
- Streaming response integration
- Chart rendering inline in Solara
- Memory/conversation history management
- Custom prompts and response formatting
- Security considerations for generated code execution
