# Key Components Deep Dive

This document provides detailed information about the key components in Marker's architecture.

## Builders

### DocumentBuilder

**File**: [`marker/builders/document.py`](../marker/builders/document.py)

The `DocumentBuilder` orchestrates the document construction pipeline.

**Responsibilities**:
- Create page objects with dual-resolution images
- Coordinate the builder sequence
- Initialize the document structure

**Key Configuration**:

```python
lowres_image_dpi: int = 96   # For layout detection
highres_image_dpi: int = 192 # For OCR
disable_ocr: bool = False    # Skip OCR entirely
```

**Execution Flow**:

```python
def __call__(self, provider, layout_builder, line_builder, ocr_builder):
    document = self.build_document(provider)  # Create pages
    layout_builder(document, provider)         # Detect layout
    line_builder(document, provider)           # Extract lines
    ocr_builder(document, provider)            # Perform OCR
    return document
```

### LayoutBuilder

**File**: [`marker/builders/layout.py`](../marker/builders/layout.py)

Detects document structure using Surya Layout Detection.

**Detected Block Types**:
- `Text` - Paragraphs and body text
- `Title` - Main document title
- `SectionHeader` - Section and subsection headers
- `ListItem` - Bulleted and numbered lists
- `Figure` - Images and diagrams
- `Table` - Data tables
- `Equation` - Mathematical equations
- `PageHeader` - Running headers
- `PageFooter` - Footers
- `Footnote` - Footnote text
- `Caption` - Figure and table captions
- `Code` - Code blocks
- `Form` - Form fields
- `Picture` - Pictures and photos
- `ComplexRegion` - Ambiguous regions

**Key Configuration**:

```python
layout_batch_size: int = None      # Auto: 12 (CUDA), 6 (CPU)
force_layout_block: str = None     # Force all pages to one type
expand_block_types: List = [...]   # Block types to expand
max_expand_frac: float = 0.05      # Max expansion amount
```

**Why expand blocks?**

Some detected regions are slightly too small:
- Pictures often have tight crops
- Complex regions may miss edge content

### LineBuilder

Extracts and structures text lines from provider data.

**Responsibilities**:
- Parse lines from pdftext output
- Establish initial line structure
- Preserve font and position information

### OcrBuilder

**File**: [`marker/builders/ocr.py`](../marker/builders/ocr.py)

Performs OCR on pages that need it using Surya Recognition.

**Key Features**:

**Skip Blocks** (doesn't OCR these):
- `Equation` - Handled separately
- `Figure` - Images, not text
- `Picture` - Photos
- `Table` - Has specialized OCR
- `Form` - Handled by LLM processor
- `TableOfContents` - Often unreliable

**Full OCR Block Types** (OCR at block level):
- `SectionHeader`
- `ListItem`
- `Footnote`
- `Text`
- `TextInlineMath`
- `Code`
- `Caption`

**Configuration**:

```python
recognition_batch_size: int = None  # Auto: 48 (CUDA), 16 (MPS), 32 (CPU)
ocr_task_name: str = "ocr_with_boxes"
disable_ocr_math: bool = False
drop_repeated_text: bool = False
keep_chars: bool = False
```

**Block Mode vs Line Mode**:

```python
# Block mode: Better context for smaller blocks
if block.block_type in full_ocr_block_types and len(lines) <= 15:
    ocr_entire_block()

# Line mode: Fallback for large or complex blocks
else:
    ocr_each_line_separately()
```

## Processors

### BaseProcessor

All processors inherit from `BaseProcessor` ([`marker/processors/base.py`](../marker/processors/base.py)).

**Common Features**:
- `use_llm` flag check
- Block type filtering
- Document and page access
- Configuration management

### TableProcessor

**File**: [`marker/processors/table.py`](../marker/processors/table.py)

Pure OCR-based table extraction using Surya Table Recognition.

**Process**:
1. Detect table structure (rows, columns, cells)
2. Extract cell polygons
3. Perform OCR on each cell
4. Reconstruct table HTML
5. Handle merged cells

**Key Configuration**:

```python
table_rec_batch_size: int = None     # Auto-based on device
detection_batch_size: int = None     # Auto-based on device
recognition_batch_size: int = None   # Auto-based on device
row_split_threshold: float = 0.5     # For split tables
disable_ocr_math: bool = False
```

**Output**: HTML table with proper cell structure.

### TextProcessor

**File**: [`marker/processors/text.py`](../marker/processors/text.py)

Merges text across pages and columns.

**Detects**:
- **Column breaks**: Text continuing in another column
- **Page breaks**: Text continuing on next page
- **Hyphenation**: Split words across lines

**Merge Logic**:

```python
# Conditions for merging text:
1. Last line is full width OR hyphenated
2. Next block is not indented
3. Column break OR page break (with position check)
```

**Key Configuration**:

```python
column_gap_ratio: float = 0.02  # Min gap for column detection
```

### EquationProcessor

**File**: [`marker/processors/equation.py`](../marker/processors/equation.py)

Handles mathematical equations.

**Features**:
- Extracts equations from layout regions
- Converts to LaTeX or MathML
- Preserves inline and block math
- Handles complex mathematical notation

### LLMTableProcessor

**File**: [`marker/processors/llm/llm_table.py`](../marker/processors/llm/llm_table.py)

Refines OCR table output with LLM.

**Features**:
- Corrects OCR errors in table cells
- Fixes column alignment
- Handles rotated tables
- Chunks large tables (60 rows per batch)
- Self-scores output quality (1-5)

**Prompt Strategy**:
- Receives image + OCR HTML
- Compares and corrects errors
- Returns corrected HTML or "No corrections needed"
- Provides analysis and quality score

**Key Configuration**:

```python
max_rows_per_batch: int = 60      # Chunk size for large tables
max_table_rows: int = 175         # Skip larger tables
table_image_expansion_ratio: float = 0
rotation_max_wh_ratio: float = 0.6  # Detect rotation
```

### Other LLM Processors

| Processor | Purpose |
|-----------|---------|
| `LLMTableMergeProcessor` | Merge tables split across pages |
| `LLMFormProcessor` | Extract form field data |
| `LLMComplexRegionProcessor` | Handle ambiguous layouts |
| `LLMImageDescriptionProcessor` | Generate image captions |
| `LLMEquationProcessor` | Fix equation formatting |
| `LLMHandwritingProcessor` | Transcribe handwriting |
| `LLMMathBlockProcessor` | Fix inline math |
| `LLMSectionHeaderProcessor` | Detect headers |
| `LLMPageCorrectionProcessor` | Page-level quality check |

### Specialized Processors

**CodeProcessor** ([`marker/processors/code.py`](../marker/processors/code.py)):
- Detects code blocks
- Preserves indentation
- Identifies programming languages

**ListProcessor** ([`marker/processors/list.py`](../marker/processors/list.py)):
- Formats bulleted and numbered lists
- Detects list hierarchy
- Handles nested lists

**BlockquoteProcessor** ([`marker/processors/blockquote.py`](../marker/processors/blockquote.py)):
- Detects quoted text
- Preserves blockquote formatting

**FootnoteProcessor** ([`marker/processors/footnote.py`](../marker/processors/footnote.py)):
- Extracts footnotes
- Links references to footnotes
- Formats footnote text

**ReferenceProcessor** ([`marker/processors/reference.py`](../marker/processors/reference.py)):
- Extracts citations and references
- Formats reference lists
- Links inline citations

**PageHeaderProcessor** ([`marker/processors/page_header.py`](../marker/processors/page_header.py)):
- Removes running headers
- Removes page numbers
- Cleans up document text

**DocumentTOCProcessor** ([`marker/processors/document_toc.py`](../marker/processors/document_toc.py)):
- Generates table of contents
- Extracts section hierarchy
- Creates navigation structure

## Renderers

### MarkdownRenderer

**File**: [`marker/renderers/markdown.py`](../marker/renderers/markdown.py)

Default renderer, produces Markdown output.

**Features**:
- Converts HTML to Markdown
- Preserves math equations (LaTeX)
- Formats tables in Markdown
- Handles code blocks with syntax highlighting
- Supports pagination markers

**Configuration**:

```python
paginate_output: bool = False
page_separator: str = "---"
inline_math_delimiters: tuple = ("$", "$")
block_math_delimiters: tuple = ("$$", "$$")
html_tables_in_markdown: bool = False
```

### HTMLRenderer

**File**: [`marker/renderers/html.py`](../marker/renderers/html.py)

Produces semantic HTML output.

**Features**:
- Preserves document structure
- Semantic HTML5 tags
- CSS classes for styling
- Accessible markup

### JSONRenderer

**File**: [`marker/renderers/json.py`](../marker/renderers/json.py)

Produces full document structure as JSON.

**Includes**:
- All blocks with IDs
- Polygon coordinates
- Block hierarchy
- Text content and formatting
- Metadata

## Schema

### Document

**File**: [`marker/schema/document.py`](../marker/schema/document.py)

Top-level document object.

**Structure**:

```python
Document:
    filepath: str
    pages: List[PageGroup]
    table_of_contents: List[TocItem]
    debug_data_path: str | None
```

### PageGroup

Represents a single page.

**Structure**:

```python
PageGroup:
    page_id: int
    lowres_image: PIL.Image  # 96 DPI
    highres_image: PIL.Image # 192 DPI
    polygon: PolygonBox
    refs: List[Reference]
    structure: List[BlockId]
    blocks: Dict[BlockId, Block]
```

### Block

Base class for all document elements.

**Common Properties**:

```python
Block:
    id: BlockId  # (page_id, block_id)
    block_type: BlockTypes
    polygon: PolygonBox  # Normalized 0-1000
    structure: List[BlockId]  # Children
    page_id: int
```

**Block Types** ([`marker/schema/__init__.py`](../marker/schema/__init__.py)):

```python
class BlockTypes(Enum):
    # Structure
    Document = "Document"
    Page = "Page"

    # Content
    Text = "Text"
    TextInlineMath = "TextInlineMath"
    Title = "Title"
    SectionHeader = "SectionHeader"
    ListItem = "ListItem"

    # Special
    Table = "Table"
    TableOfContents = "TableOfContents"
    Equation = "Equation"
    Figure = "Figure"
    Picture = "Picture"
    Code = "Code"
    Form = "Form"
    Caption = "Caption"
    Footnote = "Footnote"
    Blockquote = "Blockquote"

    # Layout
    PageHeader = "PageHeader"
    PageFooter = "PageFooter"
    ComplexRegion = "ComplexRegion"

    # Components
    Line = "Line"
    Span = "Span"
    Char = "Char"
    TableCell = "TableCell"
```

### PolygonBox

**File**: [`marker/schema/polygon.py`](../marker/schema/polygon.py)

Represents bounding boxes and polygons.

**Features**:
- Normalized coordinates (0-1000)
- Polygon operations (merge, expand, intersect)
- Rescaling between coordinate systems
- Bounding box calculation

**Operations**:

```python
polygon.merge([other])  # Merge polygons
polygon.expand(amount)   # Expand by fraction
polygon.rescale(from_size, to_size)  # Convert coordinates
polygon.intersection(other)  # Find overlap
```

## Services

### BaseService

**File**: [`marker/services/base.py`](../marker/services/base.py)

Base class for all LLM services.

**Common Interface**:

```python
def __call__(
    self,
    prompt: str,
    image: PIL.Image | List[PIL.Image] | None,
    block: Block | None,
    response_schema: type[BaseModel],
    max_retries: int | None,
    timeout: int | None
) -> dict:
    # Returns validated JSON response
```

**Features**:
- Retry logic with exponential backoff
- Image formatting for LLM
- Schema validation
- Error handling
- Token tracking

### Service Implementations

| Service | Model | Notes |
|---------|-------|-------|
| `GoogleGeminiService` | gemini-2.0-flash | Default, supports thinking |
| `GoogleVertexService` | gemini-2.0-flash-001 | Vertex AI endpoint |
| `ClaudeService` | claude-3-7-sonnet | Anthropic API |
| `OpenAIService` | gpt-4o-mini | OpenAI API |
| `OllamaService` | User-specified | Local models |

## Next Steps

- [Configuration](06-configuration.md) - Complete configuration guide
- [Overview](01-overview.md) - Back to project overview
