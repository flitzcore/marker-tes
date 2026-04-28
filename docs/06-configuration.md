# Configuration Guide

This guide explains how to configure Marker for different use cases.

## Settings

**File**: [`marker/settings.py`](../marker/settings.py)

Marker uses Pydantic Settings for configuration, with support for environment variables.

### Core Settings

```python
# Paths
BASE_DIR: str  # Base directory (auto-detected)
OUTPUT_DIR: str  # Default: conversion_results/
FONT_DIR: str  # Default: static/fonts/
DEBUG_DATA_FOLDER: str  # Default: debug_data/

# Output
OUTPUT_ENCODING: str = "utf-8"
OUTPUT_IMAGE_FORMAT: str = "JPEG"

# LLM (from environment)
GOOGLE_API_KEY: str  # Google Gemini API key
ANTHROPIC_API_KEY: str  # Claude API key
OPENAI_API_KEY: str  # OpenAI API key

# Device (auto-detected)
TORCH_DEVICE: str  # cuda, mps, cpu (None = auto)
```

### Device Detection

```python
# Automatic device selection
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

**Override device**:
```bash
export TORCH_DEVICE=cuda  # Force CUDA
export TORCH_DEVICE=cpu   # Force CPU
```

## Converter Configuration

### PdfConverter Options

**File**: [`marker/converters/pdf.py`](../marker/converters/pdf.py)

```python
converter = PdfConverter(
    artifact_dict=model_dict,
    processor_list=["OrderProcessor", "TableProcessor", ...],
    renderer="MarkdownRenderer",
    llm_service="GoogleGeminiService",
    config={
        "use_llm": True
    }
)
```

### Common Config Options

```python
config = {
    # Enable LLM processing
    "use_llm": True,

    # PDF provider options
    "force_ocr": False,  # Force OCR on entire document
    "strip_existing_ocr": False,  # Remove PDF text layer
    "flatten_pdf": True,  # Flatten PDF forms

    # Output options
    "output_format": "markdown",  # markdown, html, json
}
```

## Processor Configuration

### Customizing Processor List

```python
from marker.converters.pdf import PdfConverter
from marker.processors import (
    OrderProcessor, TableProcessor, TextProcessor,
    LLMTableProcessor, LLMEquationProcessor
)

# Custom processor pipeline
processor_list = [
    "OrderProcessor",
    "TableProcessor",
    "LLMTableProcessor",  # Only table LLM processing
    "TextProcessor"
]

converter = PdfConverter(
    artifact_dict,
    processor_list=processor_list
)
```

### Processor-Specific Config

#### TableProcessor

```python
config = {
    "table_rec_batch_size": 12,  # Table detection batch
    "detection_batch_size": 12,  # Text detection batch
    "recognition_batch_size": 48,  # OCR batch
    "row_split_threshold": 0.5,  # Split table detection
    "disable_ocr_math": False,
}
```

#### OcrBuilder

```python
config = {
    "recognition_batch_size": 48,  # OCR batch size
    "ocr_task_name": "ocr_with_boxes",  # or "ocr_without_boxes"
    "disable_ocr_math": False,
    "drop_repeated_text": False,
    "keep_chars": False,  # Store character positions
}
```

#### LayoutBuilder

```python
config = {
    "layout_batch_size": 12,  # Layout detection batch
    "force_layout_block": None,  # Force all pages to one type
    "expand_block_types": ["Picture", "Figure", "ComplexRegion"],
    "max_expand_frac": 0.05,
}
```

## LLM Configuration

### Enable LLM Processing

```python
# Method 1: Via config
converter = PdfConverter(
    artifact_dict,
    config={"use_llm": True}
)

# Method 2: Select specific service
converter = PdfConverter(
    artifact_dict,
    llm_service="ClaudeService",
    config={"use_llm": True}
)
```

### LLM Service Configuration

#### Google Gemini

```bash
# Environment variable
export GOOGLE_API_KEY=your_api_key
```

```python
# Custom model
from marker.services.gemini import GoogleGeminiService

service = GoogleGeminiService(
    gemini_model_name="gemini-2.0-flash",
    thinking_budget=1000  # Optional thinking tokens
)

converter = PdfConverter(
    artifact_dict,
    llm_service=service,
    config={"use_llm": True}
)
```

#### Anthropic Claude

```bash
export ANTHROPIC_API_KEY=your_api_key
```

```python
from marker.services.claude import ClaudeService

service = ClaudeService(
    claude_model_name="claude-3-7-sonnet-20250219",
    max_output_tokens=8192
)
```

#### OpenAI

```bash
export OPENAI_API_KEY=your_api_key
```

```python
from marker.services.openai import OpenAIService

service = OpenAIService(
    openai_model_name="gpt-4o-mini",
    base_url=None  # For OpenAI-compatible APIs
)
```

#### Ollama (Local)

```python
from marker.services.ollama import OllamaService

service = OllamaService(
    ollama_model_name="llama3",
    base_url="http://localhost:11434"
)
```

### LLM Processor Options

#### LLMTableProcessor

```python
config = {
    "max_rows_per_batch": 60,  # Chunk size for large tables
    "max_table_rows": 175,  # Skip larger tables
    "table_image_expansion_ratio": 0,  # Expand crop
    "rotation_max_wh_ratio": 0.6,  # Rotation detection
    "max_table_iterations": 2,  # Retry attempts
}
```

## Renderer Configuration

### MarkdownRenderer

```python
from marker.renderers.markdown import MarkdownRenderer

renderer = MarkdownRenderer(
    paginate_output=False,  # Add page separators
    page_separator="---",  # Page separator text
    inline_math_delimiters=("$", "$"),  # Inline math
    block_math_delimiters=("$$", "$$"),  # Block math
    html_tables_in_markdown=False  # Use HTML tables
)
```

### Output Formats

```python
# Markdown (default)
converter = PdfConverter(artifact_dict, renderer="MarkdownRenderer")

# HTML
converter = PdfConverter(artifact_dict, renderer="HTMLRenderer")

# JSON
converter = PdfConverter(artifact_dict, renderer="JSONRenderer")
```

## CLI Configuration

### Environment Variables

Create a `.env` file in your project root:

```bash
# Device
TORCH_DEVICE=cuda

# LLM APIs
GOOGLE_API_KEY=your_google_api_key
ANTHROPIC_API_KEY=your_anthropic_api_key
OPENAI_API_KEY=your_openai_api_key

# Output
OUTPUT_DIR=my_output_folder
LOGLEVEL=INFO
```

### CLI Examples

```bash
# Basic conversion
marker convert input.pdf output.md

# With LLM
marker convert input.pdf output.md --use-llm

# Specify LLM service
marker convert input.pdf output.md --use-llm --llm-service claude

# Force OCR
marker convert input.pdf output.md --force-ocr

# Custom output format
marker convert input.pdf output.html --output-format html

# Batch processing
marker_chunk_convert input_folder/ output_folder/

# Single file (optimized)
marker_single input.pdf output.md

# GUI
marker_gui
```

## Performance Tuning

### Batch Sizes

Adjust batch sizes based on your hardware:

```python
# GPU with 16GB+ VRAM
config = {
    "layout_batch_size": 12,
    "recognition_batch_size": 48,
    "table_rec_batch_size": 12,
}

# GPU with 8GB VRAM
config = {
    "layout_batch_size": 6,
    "recognition_batch_size": 24,
    "table_rec_batch_size": 6,
}

# CPU
config = {
    "layout_batch_size": 6,
    "recognition_batch_size": 32,
    "table_rec_batch_size": 6,
}
```

### Memory Optimization

```python
# Reduce memory usage
config = {
    # Disable character storage
    "keep_chars": False,

    # Use faster but less accurate OCR
    "ocr_task_name": "ocr_without_boxes",

    # Disable LLM for faster processing
    "use_llm": False,
}
```

### Quality Optimization

```python
# Maximize quality
config = {
    # Enable all enhancements
    "use_llm": True,

    # Use high-quality OCR
    "ocr_task_name": "ocr_with_boxes",

    # Preserve character positions
    "keep_chars": True,

    # Force OCR for best results
    "force_ocr": True,
}
```

## Environment Files

### `.env` File

Marker supports `.env` files for configuration:

```bash
# Create .env file
cat > .env << EOF
TORCH_DEVICE=cuda
GOOGLE_API_KEY=your_key
OUTPUT_DIR=conversions
LOGLEVEL=INFO
EOF
```

### `local.env` for Local Overrides

```bash
# local.env (git-ignored)
TORCH_DEVICE=cpu  # Local override
GOOGLE_API_KEY=dev_key
LOGLEVEL=DEBUG
```

## Python API Configuration

### Complete Example

```python
from marker.converters.pdf import PdfConverter
from marker.models import create_model_dict
from marker.services.gemini import GoogleGeminiService
from marker.renderers.markdown import MarkdownRenderer

# Create models
model_dict = create_model_dict()

# Create LLM service
llm_service = GoogleGeminiService(
    gemini_model_name="gemini-2.0-flash"
)

# Configure converter
config = {
    "use_llm": True,
    "force_ocr": False,
    "flatten_pdf": True,
}

# Create converter
converter = PdfConverter(
    artifact_dict=model_dict,
    llm_service=llm_service,
    renderer=MarkdownRenderer(),
    config=config
)

# Convert document
result = converter("input.pdf")
with open("output.md", "w") as f:
    f.write(result)
```

## Common Configuration Patterns

### High Quality (Accuracy First)

```python
config = {
    "use_llm": True,
    "force_ocr": True,
    "keep_chars": True,
}
```

### High Speed (Performance First)

```python
config = {
    "use_llm": False,
    "force_ocr": False,
    "keep_chars": False,
    "ocr_task_name": "ocr_without_boxes",
}
```

### Balanced (Default)

```python
config = {
    "use_llm": False,  # Enable for quality
    "force_ocr": False,  # Auto-detect
    "keep_chars": False,
}
```

### Handwriting Focus

```python
config = {
    "use_llm": True,  # Required for handwriting
}

# Handwriting processor will handle it
```

## Troubleshooting

### CUDA Out of Memory

```python
# Reduce batch sizes
config = {
    "recognition_batch_size": 16,  # Reduce from 48
    "layout_batch_size": 6,  # Reduce from 12
}
```

### Slow Processing

```python
# Disable LLM
config = {"use_llm": False}

# Use faster OCR
config = {"ocr_task_name": "ocr_without_boxes"}
```

### Poor OCR Quality

```python
# Force OCR
config = {"force_ocr": True}

# Enable LLM
config = {"use_llm": True}
```

### API Rate Limits

```python
# Increase timeout and retries
service = GoogleGeminiService(
    timeout=60,  # Increase from default
    max_retries=5  # More retries
)
```

## Next Steps

- [Overview](01-overview.md) - Project overview
- [Core Architecture](02-core-architecture.md) - How Marker works
- [PDF Processing & OCR](03-pdf-processing-and-ocr.md) - OCR details
- [LLM Integration](04-llm-integration.md) - LLM usage
