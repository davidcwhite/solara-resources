# PRD 07: PowerPoint Export of Charts and Data

## Problem Statement

Users need to share analysis results with stakeholders who work in PowerPoint. Currently, PandasAI generates charts that live only in the browser session. There is no way to export a chart, a data summary, or a collection of analysis results into a formatted PowerPoint presentation.

## Goal

Provide a "Download as PowerPoint" feature that exports PandasAI-generated charts, data tables, and text summaries into a professionally formatted `.pptx` file, downloadable directly from the Solara UI via `solara.FileDownload`.

## Technology Decision

### Core Library: python-pptx

**python-pptx** (v0.6.22+) is the standard Python library for creating PowerPoint files. It supports:

- Creating presentations from scratch or from templates
- Adding slides with various layouts (title, content, blank)
- Embedding images at arbitrary positions and sizes
- Creating native editable PowerPoint charts (bar, line, pie, scatter)
- Adding text boxes, tables, and shapes
- Controlling fonts, colors, and formatting
- Writing to file or BytesIO buffer (for in-memory download)

### Chart Export: Plotly to Static Image via Kaleido

Since charts are Plotly figures (see PRD 06), we use Plotly's built-in static export:

- `fig.to_image(format="png", width=1200, height=700, scale=2)` returns high-resolution PNG bytes
- `fig.write_image()` writes to file
- Requires **kaleido** (`pip install kaleido`) as the rendering engine
- High-resolution export at 2x scale for crisp presentation slides

### Alternative Considered: Native PowerPoint Charts

python-pptx can create native editable charts using `CategoryChartData` and `slide.shapes.add_chart()`. However:

- Requires restructuring data into python-pptx's chart data format
- Limited chart types compared to Plotly
- Cannot replicate Plotly's styling, annotations, and layout
- More complex code for each chart type

**Decision**: Embed Plotly charts as high-resolution PNG images. This preserves exact visual fidelity with what users see in the browser, works for any chart type Plotly supports, and is significantly simpler to implement. For data tables, we'll add native PowerPoint tables.

## User Stories

1. As a business analyst, I want to download a PowerPoint file of my analysis so I can present findings in a meeting.
2. As a data analyst, I want each chart to appear on its own slide with a title so my presentation is well-organized.
3. As a user, I want data tables included as formatted tables in the slides, not just images.
4. As a manager, I want the presentation to include a title slide with the analysis context.
5. As a user, I want to export the entire chat session (all charts and tables generated) into a single presentation.
6. As a user, I want a single-click download button that generates the PPTX without leaving the page.

## Functional Requirements

### FR-01: PresentationBuilder Service

A service class that assembles PowerPoint presentations from analysis results.

```python
# services/pptx_service.py
from pptx import Presentation
from pptx.util import Inches, Pt, Emu
from pptx.enum.text import PP_ALIGN
from pptx.dml.color import RGBColor
from io import BytesIO
import plotly.graph_objects as go
import pandas as pd

class PresentationBuilder:
    def __init__(self, title: str = "Data Analysis Report", template_path: str = None):
        if template_path:
            self.prs = Presentation(template_path)
        else:
            self.prs = Presentation()
        self.title = title
        self._add_title_slide()

    def _add_title_slide(self):
        slide = self.prs.slides.add_slide(self.prs.slide_layouts[0])
        slide.shapes.title.text = self.title
        if slide.placeholders[1]:
            slide.placeholders[1].text = "Generated with Solara Data Platform"

    def add_chart_slide(self, fig: go.Figure, title: str, notes: str = ""):
        """Add a slide with a Plotly chart rendered as a high-res image."""
        slide = self.prs.slides.add_slide(self.prs.slide_layouts[5])  # Blank layout
        
        # Add title text box
        txBox = slide.shapes.add_textbox(Inches(0.5), Inches(0.3), Inches(9), Inches(0.6))
        tf = txBox.text_frame
        p = tf.paragraphs[0]
        p.text = title
        p.font.size = Pt(24)
        p.font.bold = True
        
        # Render Plotly figure to PNG bytes
        img_bytes = fig.to_image(
            format="png", 
            width=1200, 
            height=700, 
            scale=2,
            engine="kaleido"
        )
        
        # Add chart image centered on slide
        img_stream = BytesIO(img_bytes)
        slide.shapes.add_picture(
            img_stream, 
            Inches(0.75), Inches(1.2), 
            Inches(8.5), Inches(5.5)
        )
        
        # Add notes if provided
        if notes:
            slide.notes_slide.notes_text_frame.text = notes

    def add_table_slide(self, df: pd.DataFrame, title: str, max_rows: int = 20):
        """Add a slide with a formatted data table."""
        slide = self.prs.slides.add_slide(self.prs.slide_layouts[5])
        
        # Title
        txBox = slide.shapes.add_textbox(Inches(0.5), Inches(0.3), Inches(9), Inches(0.6))
        tf = txBox.text_frame
        p = tf.paragraphs[0]
        p.text = title
        p.font.size = Pt(24)
        p.font.bold = True
        
        # Truncate if needed
        display_df = df.head(max_rows)
        rows, cols = display_df.shape
        
        # Create table
        table_shape = slide.shapes.add_table(
            rows + 1, cols,  # +1 for header
            Inches(0.5), Inches(1.2),
            Inches(9), Inches(5)
        )
        table = table_shape.table
        
        # Header row
        for j, col_name in enumerate(display_df.columns):
            cell = table.cell(0, j)
            cell.text = str(col_name)
            p = cell.text_frame.paragraphs[0]
            p.font.size = Pt(10)
            p.font.bold = True
        
        # Data rows
        for i in range(rows):
            for j in range(cols):
                cell = table.cell(i + 1, j)
                cell.text = str(display_df.iloc[i, j])
                cell.text_frame.paragraphs[0].font.size = Pt(9)
        
        # Note if truncated
        if len(df) > max_rows:
            txBox2 = slide.shapes.add_textbox(
                Inches(0.5), Inches(6.5), Inches(9), Inches(0.4)
            )
            txBox2.text_frame.paragraphs[0].text = f"Showing {max_rows} of {len(df)} rows"
            txBox2.text_frame.paragraphs[0].font.size = Pt(8)
            txBox2.text_frame.paragraphs[0].font.italic = True

    def add_text_slide(self, title: str, body: str):
        """Add a slide with text content."""
        slide = self.prs.slides.add_slide(self.prs.slide_layouts[1])  # Title and Content
        slide.shapes.title.text = title
        slide.placeholders[1].text = body

    def add_summary_slide(self, kpis: dict):
        """Add a KPI summary slide with key metrics."""
        slide = self.prs.slides.add_slide(self.prs.slide_layouts[5])
        
        txBox = slide.shapes.add_textbox(Inches(0.5), Inches(0.3), Inches(9), Inches(0.6))
        tf = txBox.text_frame
        p = tf.paragraphs[0]
        p.text = "Key Metrics"
        p.font.size = Pt(28)
        p.font.bold = True
        
        # Arrange KPIs in a grid
        col_count = min(len(kpis), 3)
        for idx, (label, value) in enumerate(kpis.items()):
            col = idx % col_count
            row = idx // col_count
            left = Inches(0.5 + col * 3.2)
            top = Inches(1.5 + row * 2)
            
            txBox = slide.shapes.add_textbox(left, top, Inches(2.8), Inches(1.5))
            tf = txBox.text_frame
            
            p_value = tf.paragraphs[0]
            p_value.text = str(value)
            p_value.font.size = Pt(36)
            p_value.font.bold = True
            p_value.font.color.rgb = RGBColor(0x19, 0x76, 0xD2)
            p_value.alignment = PP_ALIGN.CENTER
            
            p_label = tf.add_paragraph()
            p_label.text = label
            p_label.font.size = Pt(14)
            p_label.font.color.rgb = RGBColor(0x75, 0x75, 0x75)
            p_label.alignment = PP_ALIGN.CENTER

    def to_bytes(self) -> bytes:
        """Return the presentation as bytes for download."""
        buffer = BytesIO()
        self.prs.save(buffer)
        buffer.seek(0)
        return buffer.read()
    
    def save(self, path: str):
        """Save to file."""
        self.prs.save(path)
```

### FR-02: Export from Chat Session

Build a presentation from the entire chat conversation history.

```python
# services/export_service.py
import plotly.graph_objects as go
import pandas as pd
from datetime import datetime

class ExportService:
    @staticmethod
    def chat_to_pptx(messages: list[dict], title: str = None) -> bytes:
        """
        Convert a chat session's messages into a PowerPoint presentation.
        
        Each message with type "plotly" gets a chart slide.
        Each message with type "dataframe" gets a table slide.
        Text responses are grouped into summary slides.
        """
        if title is None:
            title = f"Analysis Report - {datetime.now().strftime('%Y-%m-%d')}"
        
        builder = PresentationBuilder(title=title)
        
        text_buffer = []
        slide_num = 0
        
        for msg in messages:
            if msg.get("role") == "user":
                continue
            
            msg_type = msg.get("type", "text")
            value = msg.get("value")
            query = msg.get("query", "Analysis Result")
            
            if msg_type == "plotly" and isinstance(value, go.Figure):
                slide_num += 1
                builder.add_chart_slide(
                    fig=value,
                    title=query,
                    notes=f"Generated from query: {query}"
                )
            elif msg_type == "dataframe" and isinstance(value, pd.DataFrame):
                slide_num += 1
                builder.add_table_slide(df=value, title=query)
            elif msg_type == "text" and value:
                text_buffer.append(f"Q: {query}\nA: {value}")
        
        # Add collected text responses as summary slides
        if text_buffer:
            # Group into slides of ~4 Q&A pairs each
            chunk_size = 4
            for i in range(0, len(text_buffer), chunk_size):
                chunk = text_buffer[i:i + chunk_size]
                builder.add_text_slide(
                    title="Analysis Summary",
                    body="\n\n".join(chunk)
                )
        
        return builder.to_bytes()

    @staticmethod
    def single_chart_to_pptx(fig: go.Figure, title: str) -> bytes:
        """Export a single chart to a one-slide presentation."""
        builder = PresentationBuilder(title=title)
        builder.add_chart_slide(fig=fig, title=title)
        return builder.to_bytes()
```

### FR-03: Solara Download Integration

Provide download buttons in the Solara UI using `solara.FileDownload`.

```python
import solara
from datetime import datetime

@solara.component
def ExportToPptxButton(messages: list, label: str = "Download as PowerPoint"):
    """Button that generates and downloads a PPTX from chat messages."""
    
    def generate_pptx():
        return ExportService.chat_to_pptx(
            messages=messages,
            title=f"Analysis Report - {datetime.now().strftime('%Y-%m-%d %H:%M')}"
        )
    
    filename = f"report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.pptx"
    
    solara.FileDownload(
        generate_pptx,
        filename=filename,
        label=label,
    )

@solara.component
def SingleChartExportButton(fig, chart_title: str):
    """Export button for a single chart."""
    
    def generate():
        return ExportService.single_chart_to_pptx(fig=fig, title=chart_title)
    
    solara.FileDownload(
        generate,
        filename=f"{chart_title.lower().replace(' ', '_')}.pptx",
        label="Export to PPTX",
    )
```

### FR-04: Export Button Placement

Export buttons should appear in these locations:

1. **Chat interface**: A toolbar button above the chat area that exports the entire conversation
2. **Individual chart cards**: An export icon in the `CardActions` of each chart card
3. **Dashboard page**: An "Export Dashboard" button that exports all visible dashboard charts
4. **Reports page**: An export option alongside CSV/Excel export

```python
@solara.component
def ChatToolbar(messages: list):
    with solara.Row(justify="end"):
        ExportToPptxButton(messages=messages, label="Export to PowerPoint")
        solara.Button("Clear Chat", icon_name="mdi-delete", on_click=clear_chat)

@solara.component
def ChartCard(fig, title: str):
    with solara.Card(title):
        solara.FigurePlotly(fig)
        with solara.CardActions():
            SingleChartExportButton(fig=fig, chart_title=title)
```

### FR-05: Dashboard Multi-Chart Export

Export all dashboard charts into a single presentation.

```python
@solara.component
def DashboardExportButton(charts: list[tuple]):
    """
    charts: list of (title, go.Figure) tuples
    """
    def generate_dashboard_pptx():
        builder = PresentationBuilder(title="Dashboard Export")
        for title, fig in charts:
            builder.add_chart_slide(fig=fig, title=title)
        return builder.to_bytes()
    
    solara.FileDownload(
        generate_dashboard_pptx,
        filename=f"dashboard_{datetime.now().strftime('%Y%m%d')}.pptx",
        label="Export Dashboard to PPTX",
    )
```

### FR-06: Loading State During Export

PPTX generation (especially with multiple chart image renders via Kaleido) can take several seconds. Show a loading state.

```python
@solara.component
def ExportToPptxButton(messages: list, label: str = "Download as PowerPoint"):
    generating, set_generating = solara.use_state(False)
    
    if generating:
        with solara.Row(align="center"):
            solara.SpinnerSolara(size="small")
            solara.Text("Generating presentation...")
    else:
        def generate_pptx():
            set_generating(True)
            try:
                return ExportService.chat_to_pptx(messages=messages)
            finally:
                set_generating(False)
        
        solara.FileDownload(
            generate_pptx,
            filename=f"report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.pptx",
            label=label,
        )
```

### FR-07: Custom Slide Templates (Optional)

Support custom PowerPoint templates for branded presentations.

```python
# Users can provide a .pptx template file with their company branding
# The template defines slide layouts, fonts, colors, and logo placement

builder = PresentationBuilder(
    title="Q1 Revenue Analysis",
    template_path="assets/company_template.pptx"
)
```

- Template placed in `assets/` or `public/` directory
- Template defines custom slide masters with company logos, colors, fonts
- `PresentationBuilder` uses the template's layouts instead of defaults
- Configurable via settings page or environment variable

## Dependencies

```toml
[project]
dependencies = [
    "python-pptx>=0.6.22",
    "plotly>=5.18",
    "kaleido>=1.0",         # Plotly static image export
    "Pillow>=10.0",         # Image processing (python-pptx dependency)
]
```

**Note**: Kaleido v1 requires Chrome/Chromium. Install via:
```bash
pip install kaleido
plotly_get_chrome  # or: python -c "import plotly.io; plotly.io.get_chrome()"
```

## Implementation Plan

### Phase 1: Core Export
1. Add `python-pptx` and `kaleido` to dependencies
2. Implement `PresentationBuilder` service with chart, table, and text slides
3. Implement `ExportService.chat_to_pptx()` for chat session export
4. Add `ExportToPptxButton` Solara component using `FileDownload`
5. Wire export button into the chat interface toolbar

### Phase 2: Multi-Location Export
6. Add single-chart export buttons to chart cards
7. Add dashboard export button
8. Add PPTX option to the reports page export actions

### Phase 3: Polish
9. Add KPI summary slide generation
10. Support custom PPTX templates
11. Add loading states during generation
12. Optimize multi-chart export performance (batch Kaleido renders)

## Technical Architecture

```
User clicks "Download as PowerPoint"
    │
    ▼
┌──────────────────────────────────┐
│  solara.FileDownload             │
│  (calls generate_pptx callback)  │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  ExportService.chat_to_pptx()   │
│                                  │
│  For each message in chat:       │
│    type == "plotly"  ──► chart slide     │
│    type == "dataframe" ──► table slide   │
│    type == "text"    ──► text slide      │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  PresentationBuilder             │
│                                  │
│  add_chart_slide(fig, title):    │
│    fig.to_image(png, 2x) ──►    │
│    slide.shapes.add_picture()    │
│                                  │
│  add_table_slide(df, title):     │
│    slide.shapes.add_table()      │
│    ──► native PPTX table         │
│                                  │
│  to_bytes() ──► BytesIO buffer   │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  Browser downloads .pptx file    │
└──────────────────────────────────┘
```

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Kaleido/Chrome not available in deployment | Chart images fail to render | Include Chrome in Docker image; document requirement; fall back to text-only slides |
| Large presentations (many charts) are slow | Poor UX during export | Show progress indicator; batch image generation; limit max slides |
| python-pptx table formatting is basic | Tables look plain | Apply consistent cell padding, header background color, alternating row shading |
| BytesIO buffer grows large with many high-res images | Memory pressure | Use scale=1.5 instead of 2 for presentations with >10 charts; stream slides if possible |
| Custom templates have incompatible layouts | Crash during generation | Validate template layouts on load; fall back to default if layout index doesn't exist |

## Acceptance Criteria

1. Clicking "Download as PowerPoint" in the chat produces a valid `.pptx` file
2. Each Plotly chart appears as a high-resolution image on its own slide with a title
3. DataFrames appear as native PowerPoint tables with headers and formatted data
4. The presentation opens correctly in Microsoft PowerPoint, Google Slides, and LibreOffice Impress
5. A title slide is always included with the report name and generation date
6. The download works via `solara.FileDownload` without page navigation
7. Export completes within 10 seconds for a typical session (5-10 charts)
8. Tables exceeding 20 rows are truncated with a note indicating total row count
