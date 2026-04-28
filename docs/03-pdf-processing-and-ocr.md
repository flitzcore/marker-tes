# PDF Processing & OCR

## Q1: How Does Marker Scan PDFs?

Marker uses a **multi-stage approach** to process PDFs, combining direct text extraction with selective OCR. Not every page gets OCR'd - only pages that need it.

### The PDF Scanning Pipeline

```
PDF File
    ↓
1. Load with pypdfium2
    ↓
2. Extract text using pdftext (direct text extraction)
    ↓
3. Analyze text quality (is OCR needed?)
    ↓
4. Render pages at two resolutions:
   - 96 DPI (low-res) for layout detection
   - 192 DPI (high-res) for OCR
    ↓
5. Perform layout detection (Surya)
    ↓
6. Perform OCR selectively (only where needed)
    ↓
7. Merge and structure results
```

### Step-by-Step Breakdown

#### 1. PDF Loading with pypdfium2

**File**: [`marker/providers/pdf.py:84-100`](../marker/providers/pdf.py#L84-L100)

```python
with self.get_doc() as doc:
    self.page_count = len(doc)
    # Initialize page storage
    self.page_lines = {i: [] for i in range(len(doc))}
    self.page_refs = {i: [] for i in range(len(doc))}
```

- Uses `pypdfium2` (PDFium library) for robust PDF handling
- Supports encrypted PDFs, forms, and annotations
- Can flatten PDF forms if needed (`flatten_pdf=True`)

#### 2. Initial Text Extraction with pdftext

**File**: [`marker/providers/pdf.py`](../marker/providers/pdf.py)

Before any OCR, Marker attempts to extract text directly from the PDF:

```python
# pdftext extraction provides:
- Text content with font information
- Character positions
- Text reading order
- Embedded links and references
```

**Why pdftext first?**
- Direct extraction is faster than OCR
- Preserves font information and metadata
- Works well for digital/native PDFs
- OCR is only used when text quality is poor

#### 3. Quality Detection - Does This Page Need OCR?

Marker analyzes each page to determine if OCR is necessary:

**Quality Thresholds** ([`marker/providers/pdf.py:55-74`](../marker/providers/pdf.py#L55-L74)):

```python
# Detect bad text quality
ocr_space_threshold: float = 0.7      # 70% spaces = likely bad OCR
ocr_newline_threshold: float = 0.6    # 60% newlines = likely bad OCR
ocr_alphanum_threshold: float = 0.3   # <30% alphanumeric = garbage
image_threshold: float = 0.65         # >65% image coverage = skip page
```

**Decision Logic**:
- If text is high quality → Skip OCR, use extracted text
- If text is low quality → Mark page for OCR (`text_extraction_method = 'surya'`)
- If page is mostly images → Skip or force OCR based on config

**Force OCR Options**:
- `force_ocr=True`: OCR entire document regardless of text quality
- `strip_existing_ocr=True`: Remove PDF's embedded text layer first

#### 4. Dual-Resolution Rendering

**File**: [`marker/builders/document.py:18-25`](../marker/builders/document.py#L18-L25)

Pages are rendered at two different resolutions:

```python
lowres_image_dpi: int = 96   # For layout detection
highres_image_dpi: int = 192 # For OCR
```

**Why two resolutions?**

| Resolution | Purpose | Why |
|------------|---------|-----|
| 96 DPI | Layout detection | Faster, sufficient for detecting regions |
| 192 DPI | OCR | Higher quality for accurate text recognition |

```python
# From DocumentBuilder
lowres_images = provider.get_images(page_range, 96)
highres_images = provider.get_images(page_range, 192)
```

#### 5. Layout Detection with Surya

**File**: [`marker/builders/layout.py`](../marker/builders/layout.py)

Uses **Surya Layout Detection** model to identify document regions:

```python
# Detected elements
- Text              # Paragraphs
- Title             # Main titles
- SectionHeader     # Section headers
- ListItem          # Bulleted/numbered lists
- Figure            # Images and figures
- Table             # Data tables
- Equation          # Math equations
- PageHeader        # Running headers
- PageFooter        # Footers
- Footnote          # Footnotes
- Caption           # Figure/table captions
- Code              # Code blocks
- Form              # Form fields
```

**Batch Processing**:
- CUDA: 12 pages per batch
- CPU/MPS: 6 pages per batch

#### 6. OCR with Surya Recognition

**File**: [`marker/builders/ocr.py`](../marker/builders/ocr.py)

## Q1: What OCR Library Does Marker Use?

**Marker uses Surya OCR, NOT Tesseract, PaddleOCR, or EasyOCR.**

### What is Surya?

Surya is a modern, transformer-based OCR and document layout analysis system:

```python
# From marker/models.py
from surya.layout import LayoutPredictor      # Layout detection
from surya.recognition import RecognitionPredictor  # Text recognition
from surya.table_rec import TableRecPredictor  # Table detection
from surya.detection import DetectionPredictor    # Text detection
from surya.ocr_error import OCRErrorPredictor     # Quality assessment
```

### Why Surya Instead of Traditional OCR?

| Feature | Surya | Tesseract |
|---------|-------|-----------|
| Architecture | Transformer-based | Traditional CNN/LSTM |
| Layout Understanding | Native support | Limited |
| Multi-language | Excellent | Variable |
| Table Detection | Built-in | Poor |
| Accuracy | Higher | Lower |
| Speed | Moderate | Fast |

### OCR Processing Details

#### Skip Blocks (Not OCR'd)

**File**: [`marker/builders/ocr.py:40-52`](../marker/builders/ocr.py#L40-L52)

```python
skip_ocr_blocks = [
    BlockTypes.Equation,        # Handled by EquationProcessor
    BlockTypes.Figure,          # Handled separately
    BlockTypes.Picture,         # Images, not text
    BlockTypes.Table,           # Handled by TableProcessor with specialized OCR
    BlockTypes.Form,            # Handled by LLMFormProcessor
    BlockTypes.TableOfContents, # Often unreliable, handled differently
]
```

These block types are skipped during OCR because:
- They're handled by specialized processors
- They contain non-text content
- They benefit from different processing approaches

#### OCR Modes

**File**: [`marker/builders/ocr.py:66-69`](../marker/builders/ocr.py#L66-L69)

```python
ocr_task_name: str = "ocr_with_boxes"
# Alternative: "ocr_without_boxes"
```

| Mode | Description | Pros | Cons |
|------|-------------|------|------|
| `ocr_with_boxes` | Preserves layout with bounding boxes | Better formatting | Slower |
| `ocr_without_boxes` | Text-only, faster | Faster performance | Loses formatting |

#### Batch Processing by Device

**File**: [`marker/builders/ocr.py:96-103`](../marker/builders/ocr.py#L96-L103)

```python
def get_recognition_batch_size(self):
    if settings.TORCH_DEVICE_MODEL == "cuda":
        return 48    # GPU can handle more
    elif settings.TORCH_DEVICE_MODEL == "mps":
        return 16    # Apple Silicon
    return 32        # CPU
```

#### Block Mode vs Line Mode

**File**: [`marker/builders/ocr.py:105-118`](../marker/builders/ocr.py#L105-L118)

OCR can operate at two levels:

```python
# Block mode: OCR entire block at once (better context)
full_ocr_block_types = [
    BlockTypes.SectionHeader,
    BlockTypes.ListItem,
    BlockTypes.Footnote,
    BlockTypes.Text,
    BlockTypes.Code,
    BlockTypes.Caption,
]

# Line mode: OCR each line individually (fallback)
# Used when:
- Block has many lines (>15)
- Block is very tall (>50% of page height)
- High intersection with other blocks (>50%)
```

### Page-by-Page Processing Flow

**Does it do OCR for each page?**

**Not necessarily.** Only pages marked for OCR get processed:

```python
# From OcrBuilder.__call__
pages_to_ocr = [page for page in document.pages 
                if page.text_extraction_method == 'surya']
```

**Per-page OCR process**:

1. **Filter blocks**: Remove skip_blocks (tables, equations, figures)
2. **Select mode**: Block mode or line mode based on complexity
3. **Prepare polygons**: Rescale and fit to image bounds
4. **Batch OCR**: Process multiple pages/blocks together
5. **Extract spans**: Convert OCR characters to structured spans
6. **Preserve links**: Copy URLs from original text to OCR result

### Table OCR

Tables are handled separately by `TableProcessor` using Surya's `TableRecPredictor`:

```python
# Specialized table detection and OCR
from surya.table_rec import TableRecPredictor

# TableProcessor:
1. Detects table structure (rows, columns, headers)
2. Performs cell-by-cell OCR
3. Preserves table formatting
4. Can be refined by LLMTableProcessor
```

### Math/Euation OCR

Equations are handled by specialized processors:

```python
# Equations detected during layout phase
# Processed by EquationProcessor and optionally LLMEquationProcessor

# Inline math in OCR
disable_ocr_math: bool = False  # Enable/disable inline math recognition
```

### OCR Quality Features

#### Character-Level Information

```python
keep_chars: bool = False  # Store individual character positions
```

When enabled, preserves exact character positions for:
- JSON output with character coordinates
- Highlighting applications
- Verification/debugging

#### Text Correction

```python
from ftfy import fix_text
# Fixes common text encoding issues
```

#### Repeated Text Detection

```python
drop_repeated_text: bool = False
# Removes duplicate text (headers, footers, watermarks)
```

### Performance Considerations

| Operation | Typical Speed |
|-----------|---------------|
| PDF text extraction | ~1-2 pages/second |
| Layout detection (96 DPI) | ~3-6 pages/second (GPU) |
| OCR (192 DPI) | ~1-3 pages/second (GPU) |

**Memory Usage**:
- Low-res images: ~100-300KB per page
- High-res images: ~500KB-2MB per page
- Model memory: ~2-4GB (GPU) or ~8GB (CPU)

### Summary

**How Marker scans PDFs:**
1. Extracts text directly using `pdftext`
2. Analyzes text quality to determine if OCR is needed
3. Renders pages at two resolutions (96 DPI for layout, 192 DPI for OCR)
4. Detects layout regions using Surya Layout model
5. Performs OCR selectively using Surya Recognition model
6. Handles special elements (tables, equations) with specialized processors

**OCR Library:**
- Uses **Surya OCR** (transformer-based), not Tesseract
- Multiple Surya models: Layout, Recognition, Table Recognition, Detection
- Batch processing with device-specific optimization
- Polygon-based OCR preserving layout information

## Next Steps

- [LLM Integration](04-llm-integration.md) - How LLMs enhance the OCR output
- [Key Components](05-key-components.md) - Deep dive into processors and builders
