# Core Architecture

## Pipeline Overview

Marker follows a **four-stage pipeline architecture** that transforms raw input files into structured output formats:

```
Input File → Provider → Builder → Processor → Renderer → Output
```

Each stage has a specific responsibility and can be configured independently.

## Stage 1: Provider Stage

**Location**: [`marker/providers/`](../marker/providers/)

The Provider stage is responsible for loading input files and extracting raw data.

### Key Providers

| Provider | Input Format | Responsibility |
|----------|--------------|----------------|
| `PdfProvider` | PDF files | Extract page images, text, metadata, references |
| `ImageProvider` | Images | Load images for OCR processing |
| `DocxProvider` | Word documents | Parse docx structure and content |
| `XlsxProvider` | Excel spreadsheets | Extract table data and formatting |

### PdfProvider Details

The `PdfProvider` ([`marker/providers/pdf.py`](../marker/providers/pdf.py)) is the most commonly used provider:

```python
# Key operations performed by PdfProvider
1. Load PDF with pypdfium2
2. Extract pages at multiple DPI levels (96 for layout, 192 for OCR)
3. Extract text using pdftext library
4. Extract references and metadata
5. Detect pages needing OCR (based on text quality)
```

**Configuration Options**:
- `force_ocr`: Force OCR on entire document
- `strip_existing_ocr`: Remove existing OCR layer
- `flatten_pdf`: Flatten PDF forms/annotations
- `ocr_space_threshold`: Detect bad OCR (default: 0.7)
- `image_threshold`: Skip image-heavy pages (default: 0.65)

## Stage 2: Builder Stage

**Location**: [`marker/builders/`](../marker/builders/)

The Builder stage constructs the document structure from raw provider data.

### Builder Pipeline

The builders execute in sequence ([`marker/converters/pdf.py:176-186`](../marker/converters/pdf.py#L176-L186)):

```python
document = DocumentBuilder(provider, layout_builder, line_builder, ocr_builder)
structure_builder(document)
```

### DocumentBuilder

**File**: [`marker/builders/document.py`](../marker/builders/document.py)

The `DocumentBuilder` orchestrates the construction pipeline:

```python
# 1. Build document with page images at two resolutions
lowres_images = provider.get_images(page_range, 96)   # For layout detection
highres_images = provider.get_images(page_range, 192)  # For OCR

# 2. Create Page objects with both image sets
pages = [PageGroup(lowres_image, highres_image, polygon, refs) ...]

# 3. Run builders in sequence
layout_builder(document, provider)  # Detect layout regions
line_builder(document, provider)    # Extract text lines
ocr_builder(document, provider)     # Perform OCR where needed
```

### LayoutBuilder

**File**: [`marker/builders/layout.py`](../marker/builders/layout.py)

Uses **Surya Layout Detection** to identify document regions:

```python
# Detected block types
- Text              # Regular text blocks
- Title             # Main titles
- SectionHeader     # Section headers
- ListItem          # List items
- Figure            # Figures/images
- Table             # Tables
- Equation          # Math equations
- PageHeader        # Headers/footers
- PageFooter        # Footers
- Footnote          # Footnotes
- Caption           # Captions
- Code              # Code blocks
- Form              # Form fields
- Picture           # Pictures
- ComplexRegion     # Ambiguous regions
```

**Batch Processing**: Automatically adjusts batch size based on device:
- CUDA: 12 pages per batch
- CPU/MPS: 6 pages per batch

### LineBuilder

Extracts and structures text lines from the provider data.

### OcrBuilder

**File**: [`marker/builders/ocr.py`](../marker/builders/ocr.py)

Performs OCR using **Surya Recognition** model:

```python
# Only processes pages marked for OCR
if text_extraction_method == 'surya':
    # Skip certain block types
    skip_blocks = [Equation, Figure, Picture, Table, Form]

    # Perform OCR with layout preservation
    ocr_result = recognition_model(images, polygons)
```

**Batch Processing**:
- CUDA: 48 pages per batch
- MPS: 16 pages per batch
- CPU: 32 pages per batch

## Stage 3: Processor Stage

**Location**: [`marker/processors/`](../marker/processors/)

The Processor stage refines and enhances the document content.

### Processor Pipeline

Processors run sequentially ([`marker/converters/pdf.py:74-103`](../marker/converters/pdf.py#L74-L103)):

```python
default_processors = (
    # Structure processors
    OrderProcessor,              # Establish reading order
    BlockRelabelProcessor,       # Fix block type labels
    LineMergeProcessor,          # Merge related lines

    # Format-specific processors
    BlockquoteProcessor,         # Detect blockquotes
    CodeProcessor,               # Detect code blocks
    EquationProcessor,           # Handle equations
    ListProcessor,               # Format lists
    TableProcessor,              # Extract tables (pure OCR)
    TextProcessor,               # Process text blocks

    # LLM-enhanced processors (optional, require use_llm=True)
    LLMTableProcessor,           # Refine tables with LLM
    LLMTableMergeProcessor,      # Merge split tables
    LLMFormProcessor,            # Extract form data
    LLMComplexRegionProcessor,   # Handle complex layouts
    LLMImageDescriptionProcessor,# Generate image captions
    LLMEquationProcessor,        # Fix math equations
    LLMHandwritingProcessor,     # Recognize handwriting
    LLMMathBlockProcessor,       # Fix inline math
    LLMSectionHeaderProcessor,   # Detect headers
    LLMPageCorrectionProcessor,  # Page-level corrections

    # Cleanup processors
    FootnoteProcessor,           # Handle footnotes
    ReferenceProcessor,          # Process references
    PageHeaderProcessor,         # Remove headers/footers
    DocumentTOCProcessor,        # Generate TOC
    IgnoreTextProcessor,         # Filter noise
    LineNumbersProcessor,        # Remove line numbers
    BlankPageProcessor,          # Detect blank pages
)
```

### Processor Configuration

Processors can be customized:

```python
# Disable specific processors
processor_list = [
    "OrderProcessor",
    "TableProcessor",
    "TextProcessor"
    # Skip LLM processors
]

# Enable LLM processing
converter = PdfConverter(artifact_dict, config={"use_llm": True})
```

## Stage 4: Renderer Stage

**Location**: [`marker/renderers/`](../marker/renderers/)

The Renderer stage converts the processed document into the final output format.

### Available Renderers

| Renderer | Output Format | Description |
|----------|---------------|-------------|
| `MarkdownRenderer` | Markdown (.md) | Default, with formatting preservation |
| `HTMLRenderer` | HTML | Structured HTML with semantic markup |
| `JSONRenderer` | JSON | Full document structure with blocks |

### Rendering Flow

```python
# 1. Document renders its pages
document.render() → pages

# 2. Each page renders its blocks
page.render() → block_content

# 3. Blocks render based on type
- Text → formatted text
- Table → markdown/HTML table
- Equation → LaTeX/MathML
- Image → markdown image tag or description
```

## Data Structures

### Document Schema

**Location**: [`marker/schema/`](../marker/schema/)

```python
Document
├── pages: List[PageGroup]
│   ├── page_id: int
│   ├── lowres_image: PIL.Image  # 96 DPI
│   ├── highres_image: PIL.Image # 192 DPI
│   ├── polygon: PolygonBox      # Page bounds
│   ├── structure: List[BlockId] # Reading order
│   └── blocks: Dict[BlockId, Block]
│       ├── Text, Table, Equation, Figure, etc.
│       ├── polygon: PolygonBox  # Block position
│       ├── children: List[BlockId]
│       └── content: str, html, or other data
```

### Block Types

**Location**: [`marker/schema/blocks/`](../schema/blocks/)

All document elements inherit from `Block` and have:
- Unique ID (page_id, block_id)
- Position (polygon with normalized coordinates 0-1000)
- Block type
- Content or children

## Device Detection and Model Loading

**Location**: [`marker/settings.py`](../marker/settings.py)

```python
# Automatic device detection
if torch.cuda.is_available():
    device = "cuda"
elif torch.backends.mps.is_available():
    device = "mps"  # Apple Silicon
else:
    device = "cpu"

# Dtype selection
if device == "cuda":
    dtype = torch.bfloat16  # Faster on GPU
else:
    dtype = torch.float32   # CPU compatibility
```

### Model Initialization

**Location**: [`marker/models.py`](../marker/models.py)

```python
# All Surya models loaded at startup
model_dict = {
    "layout_model": LayoutPredictor(),      # Layout detection
    "recognition_model": RecognitionPredictor(),  # OCR
    "table_rec_model": TableRecPredictor(), # Table detection
    "detection_model": DetectionPredictor(), # Text detection
    "ocr_error_model": OCRErrorPredictor()  # Quality assessment
}
```

## Performance Optimizations

1. **Batch Processing**: All models process multiple pages/images simultaneously
2. **Dual-Resolution Rendering**: Low-res for layout, high-res for OCR only where needed
3. **Selective OCR**: Pages with good text skip OCR entirely
4. **GPU Acceleration**: Automatic CUDA/MPS detection and utilization
5. **Parallel Processing**: ThreadPoolExecutor for LLM calls (max 3 concurrent)
6. **Memory Efficiency**: Page-wise processing to avoid loading entire document

## Error Handling

The pipeline includes multiple fallback mechanisms:

1. **OCR Quality Detection**: Detects and retries bad OCR results
2. **LLM Retry Logic**: Exponential backoff for rate limits
3. **Graceful Degradation**: LLM failures preserve OCR output
4. **Schema Validation**: Pydantic models ensure data consistency

## Next Steps

- [PDF Processing & OCR](03-pdf-processing-and-ocr.md) - How OCR works in detail
- [LLM Integration](04-llm-integration.md) - When and how LLMs enhance the output
- [Key Components](05-key-components.md) - Deep dive into specific components
