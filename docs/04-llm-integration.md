# LLM Integration

## Q2: When Does the LLM Step In?

The LLM (Large Language Model) steps in as a **post-processing refinement layer** after the initial OCR and structure detection. It does not replace OCR - it enhances and corrects OCR output.

### LLM Processing Stage in the Pipeline

```
PDF → Provider → Builders → [OCR/Structure] → Processors → Renderers
                                     ↓            ↑
                                  Pure OCR    LLM Enhancement
                                               (Optional)
```

**Timeline**:
1. **First**: Pure OCR processors (TableProcessor, TextProcessor) extract content
2. **Second**: LLM processors refine and correct the OCR output
3. **Result**: High-quality output with structure preserved

### When is LLM Called?

LLM processors are **conditional** - they only run when enabled:

```python
# Enable LLM processing
converter = PdfConverter(artifact_dict, config={"use_llm": True})

# Each LLM processor checks this flag
if self.use_llm:
    # Process with LLM
else:
    # Skip processing
```

**Default**: `use_llm=False` - LLM processing is opt-in for performance reasons.

### LLM Integration Points

**File**: [`marker/converters/pdf.py:74-103`](../marker/converters/pdf.py#L74-L103)

The processor pipeline includes both OCR and LLM processors:

```python
default_processors = (
    # ... OCR processors
    TableProcessor,         # Pure OCR table extraction
    TextProcessor,          # Pure OCR text processing

    # LLM processors (only run if use_llm=True)
    LLMTableProcessor,      # Refine tables
    LLMTableMergeProcessor, # Merge split tables
    LLMFormProcessor,       # Extract form data
    LLMComplexRegionProcessor,  # Handle complex layouts
    LLMImageDescriptionProcessor,  # Generate captions
    LLMEquationProcessor,   # Fix math equations
    LLMHandwritingProcessor,    # Recognize handwriting
    LLMMathBlockProcessor,  # Fix inline math
    LLMSectionHeaderProcessor,   # Detect headers
    LLMPageCorrectionProcessor,  # Page-level corrections
)
```

### LLM Processor Responsibilities

| Processor | Block Types | Purpose |
|-----------|-------------|---------|
| `LLMTableProcessor` | Table, TableOfContents | Correct OCR errors in table cells, fix column alignment |
| `LLMTableMergeProcessor` | Table | Merge tables split across pages |
| `LLMFormProcessor` | Form | Extract form field labels and values |
| `LLMComplexRegionProcessor` | ComplexRegion | Disambiguate unclear layouts |
| `LLMImageDescriptionProcessor` | Figure, Picture | Generate descriptive captions |
| `LLMEquationProcessor` | Equation | Convert to proper LaTeX/MathML |
| `LLMHandwritingProcessor` | Text (handwriting) | Transcribe handwritten text |
| `LLMMathBlockProcessor` | TextInlineMath | Fix inline math formatting |
| `LLMSectionHeaderProcessor` | SectionHeader | Detect and format headers |
| `LLMPageCorrectionProcessor` | All | Final quality check and corrections |

### How LLM Processes OCR Data

#### Input to LLM

Each LLM processor receives:

1. **Image**: Cropped region from the original PDF (high resolution)
2. **Structured Context**: OCR output with metadata
   - Bounding boxes (normalized to 0-1000 range)
   - Existing HTML/text from OCR
   - Block type information
   - Reading order data

**Example: LLMTableProcessor input** ([`marker/processors/llm/llm_table.py:121-150`](../marker/processors/llm/llm_table.py#L121-L150))

```python
# 1. Extract table image from page
block_image = self.extract_image(document, block)

# 2. Get existing OCR HTML representation
block_html = block.get_html(document, page)

# 3. Send to LLM with prompt
response = llm_service(
    prompt=table_rewriting_prompt,
    image=block_image,
    block=block,
    response_schema=TableCorrectionResponse
)
```

#### Processing Flow

```python
# Standard LLM processor flow
1. Check if use_llm is enabled
2. Filter blocks by block_type
3. Extract image region (with expansion)
4. Prepare context (OCR output, block info)
5. Call LLM with structured prompt
6. Parse response with Pydantic schema
7. Update block with LLM corrections
8. Handle errors gracefully (preserve OCR output)
```

### Prompt Engineering Approach

**File**: [`marker/processors/llm/llm_table.py:43-92`](../marker/processors/llm/llm_table.py#L43-L92)

#### Example Prompt: Table Correction

```python
table_rewriting_prompt = """You are a text correction expert specializing in accurately reproducing text from images.
You will receive an image and an html representation of the table in the image.
Your task is to correct any errors in the html representation.

Guidelines:
- Reproduce the original values from the image as faithfully as possible
- Ensure column headers match the correct column values
- Use only specific allowed tags: th, td, tr, br, span, sup, sub, i, b, math, table
- Make sure the columns and rows match the image faithfully

Instructions:
1. Carefully examine the provided text block image
2. Analyze the html representation of the table
3. Write a comparison of the image and the html representation
4. Generate the corrected html representation or "No corrections needed"
5. Provide a score from 1-5 indicating how well the corrected html matches the image
"""
```

#### Key Prompt Elements

1. **Role Definition**: "You are a text correction expert..."
2. **Task Description**: Clear explanation of what to do
3. **Input Format**: Description of what inputs will be provided
4. **Guidelines**: Specific rules and constraints
5. **Output Format**: Expected response structure
6. **Examples**: Few-shot examples for better performance
7. **Context Injection**: `{block_html}` placeholder for OCR output

### Supported LLM Providers

**File**: [`marker/services/`](../marker/services/)

#### Google Gemini (Default)

**File**: [`marker/services/gemini.py`](../marker/services/gemini.py)

```python
class GoogleGeminiService(BaseService):
    gemini_model_name: str = "gemini-2.0-flash"
    thinking_budget: int = None  # Optional thinking tokens

# Alternative: Vertex AI
class GoogleVertexService(BaseService):
    gemini_model_name: str = "gemini-2.0-flash-001"
```

#### Anthropic Claude

**File**: [`marker/services/claude.py`](../marker/services/claude.py)

```python
class ClaudeService(BaseService):
    claude_model_name: str = "claude-3-7-sonnet-20250219"
    max_output_tokens: int = 8192
```

#### OpenAI

**File**: [`marker/services/openai.py`](../marker/services/openai.py)

```python
class OpenAIService(BaseService):
    openai_model_name: str = "gpt-4o-mini"
    base_url: str = None  # For OpenAI-compatible APIs
```

#### Ollama (Local)

**File**: [`marker/services/ollama.py`](../marker/services/ollama.py)

```python
class OllamaService(BaseService):
    ollama_model_name: str = "llama3"  # User-specified
    base_url: str = "http://localhost:11434"
```

### Schema Validation

LLM responses are validated using Pydantic models:

```python
# Example response schema
class TableCorrectionResponse(BaseModel):
    comparison: str  # Analysis of differences
    corrected_html: str | None  # Corrected HTML or None
    analysis: str  # Self-analysis
    score: int  # Quality score 1-5

# LLM returns JSON, validated against schema
response = llm_service(...)
validated = TableCorrectionResponse(**response)
```

### Performance Optimizations

#### 1. Conditional Execution

```python
# Skip if LLM disabled
if not self.use_llm:
    return

# Skip large tables
if row_count > self.max_table_rows:  # 175 rows
    return
```

#### 2. Batching

```python
# Process large tables in chunks
for i in range(0, row_count, self.max_rows_per_batch):
    batch_row_idxs = row_idxs[i:i + self.max_rows_per_batch]
    # Process batch
```

#### 3. Concurrency

```python
# Parallel LLM requests (max 3)
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=3) as executor:
    futures = [executor.submit(process_block, b) for b in blocks]
    results = [f.result() for f in futures]
```

#### 4. Image Optimization

```python
# Crop images to relevant regions
block_image = self.extract_image(document, block)

# Optional expansion
expansion = block.polygon.size * expansion_ratio
expanded_polygon = block.polygon.expand(expansion)
```

### Error Handling

#### Retry Logic

**File**: [`marker/services/gemini.py:63-99`](../marker/services/gemini.py#L63-L99)

```python
for tries in range(1, total_tries + 1):
    try:
        response = client.models.generate_content(...)
        return response
    except APIError as e:
        if e.code in [429, 443, 503]:  # Rate limits
            if tries == total_tries:
                logger.error(f"Max retries reached. Giving up.")
                return None
            sleep(2 ** tries)  # Exponential backoff
```

#### Graceful Degradation

```python
# If LLM fails, preserve OCR output
try:
    corrected = llm_service(...)
    if corrected and corrected.corrected_html:
        block.update_html(corrected.corrected_html)
except Exception as e:
    logger.warning(f"LLM processing failed: {e}")
    # Keep original OCR output
```

### Quality Control

#### Self-Scoring

Some processors include self-assessment:

```python
# LLM scores its own output 1-5
score: int  # 5 = perfect, 1 = poor

# Can be used for filtering
if score < 3:
    # Retry or use OCR output
```

#### Metadata Tracking

```python
# Track LLM usage in block metadata
block.update_metadata(
    llm_tokens_used=total_tokens,
    llm_request_count=1,
    llm_errors=error_count
)
```

### Configuration

#### Enable LLM Processing

```python
# Global flag
converter = PdfConverter(
    artifact_dict,
    config={"use_llm": True}
)
```

#### Select LLM Service

```python
# Use specific service
converter = PdfConverter(
    artifact_dict,
    llm_service="claude"  # or "openai", "ollama"
)
```

#### Environment Variables

```bash
# Google Gemini (default)
GOOGLE_API_KEY=your_api_key

# Anthropic Claude
ANTHROPIC_API_KEY=your_api_key

# OpenAI
OPENAI_API_KEY=your_api_key
```

### Summary: When LLM Steps In

**The LLM steps in at the post-processing stage:**

1. **After OCR completes** - LLM enhances existing OCR output, doesn't replace it
2. **Only when enabled** - `use_llm=True` flag
3. **Per block type** - Different LLM processors handle different content types
4. **With context** - Receives both image and OCR output for comparison
5. **Conditionally** - Skips large/complex blocks, handles errors gracefully

**Key Benefits:**
- Corrects OCR errors in tables and text
- Transcribes handwriting
- Generates image descriptions
- Fixes equation formatting
- Merges split tables
- Improves overall accuracy

**When to Use LLM:**
- High-accuracy requirements
- Complex table structures
- Handwritten content
- Poor quality scans
- Need for image descriptions

**When to Skip LLM:**
- High-throughput requirements
- Cost constraints
- Simple documents with good OCR
- Batch processing large volumes

## Next Steps

- [Key Components](05-key-components.md) - Deep dive into processors and builders
- [Configuration](06-configuration.md) - Full configuration guide
