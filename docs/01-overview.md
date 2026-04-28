# Marker Project Overview

## What is Marker?

Marker is a high-performance document conversion tool that transforms PDFs and other documents into structured formats like Markdown, JSON, HTML, and chunks. It combines advanced OCR (Optical Character Recognition) with optional LLM (Large Language Model) enhancement to produce highly accurate conversions that preserve document structure, tables, equations, and formatting.

## Key Capabilities

### Supported Input Formats
- **PDF** (primary format)
- **Images** (PNG, JPG, etc.)
- **Word documents** (.docx)
- **Excel spreadsheets** (.xlsx)
- **HTML files**
- **EPUB files**

### Supported Output Formats
- **Markdown** (.md) - Default output
- **JSON** - Structured data with block-level information
- **HTML** - Web-ready format
- **Chunks** - For RAG (Retrieval Augmented Generation) applications

### Advanced Features
- **Layout preservation**: Maintains document structure with headers, columns, and sections
- **Table extraction**: High-accuracy table detection and formatting
- **Equation rendering**: Math equations preserved in LaTeX or MathML
- **Code block detection**: Identifies and formats code blocks
- **Handwriting recognition**: Optional LLM-based handwriting transcription
- **Image descriptions**: Optional LLM-generated image captions
- **Batch processing**: Convert multiple documents efficiently
- **GPU acceleration**: Automatic CUDA/MPS/CPU device detection

## Entry Points

Marker provides multiple ways to interact with the system:

### Command Line Interface (CLI)

```bash
# Convert a single file
marker convert input.pdf output.md

# Single file converter (optimized for single files)
marker_single input.pdf output.md

# Batch conversion
marker_chunk_convert input_folder output_folder

# GUI interface
marker_gui

# API server
marker_server
```

### Python API

```python
from marker.converters.pdf import PdfConverter
from marker.models import create_model_dict

# Initialize models and converter
artifact_dict = create_model_dict()
converter = PdfConverter(artifact_dict)

# Convert a document
result = converter("input.pdf")
```

### REST API

Start the server with `marker_server` and make HTTP requests to convert documents.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Marker Pipeline                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │ Provider │───▶│ Builders │───▶│Processors│───▶│ Renderers│  │
│  │          │    │          │    │          │    │          │  │
│  │ - PDF    │    │ - Doc    │    │ - Table  │    │ - Markdown│  │
│  │ - Image  │    │ - Layout │    │ - Text   │    │ - JSON    │  │
│  │ - Docx   │    │ - Line   │    │ - LLM*   │    │ - HTML    │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│                                                 │                │
│                                          *Optional                │
└─────────────────────────────────────────────────────────────────┘
```

### Pipeline Stages

1. **Provider Stage**: Loads the input file and extracts page images and metadata
2. **Builder Stage**: Constructs document structure with layout, lines, and OCR data
3. **Processor Stage**: Refines content, handles special elements, applies LLM enhancement
4. **Renderer Stage**: Converts the processed document to the desired output format

## Technology Stack

### Core Technologies
- **Surya OCR**: Layout detection and text recognition (replaces traditional OCR like Tesseract)
- **PyTorch**: Deep learning framework for model inference
- **pypdfium2**: PDF rendering and processing
- **pdftext**: Initial text extraction from PDFs
- **FastAPI**: REST API server
- **Click**: CLI framework
- **Pydantic**: Data validation and settings

### Supported LLM Providers
- **Google Gemini** (default)
- **Anthropic Claude**
- **OpenAI** (GPT-4, etc.)
- **Google Vertex AI**
- **Ollama** (local models)

## Key Design Principles

1. **Modularity**: Each stage can be configured or extended independently
2. **Hybrid Processing**: Traditional OCR combined with optional LLM refinement
3. **Performance**: GPU acceleration, batch processing, efficient memory usage
4. **Accuracy**: Multiple quality checks and fallback mechanisms
5. **Flexibility**: Easy to add new processors, renderers, or providers

## Project Structure

```
marker/
├── builders/       # Document construction pipeline
├── config/         # Configuration management
├── converters/     # Format converters
├── extractors/     # Content extractors
├── processors/     # Content refinement (including LLM)
│   └── llm/       # LLM-based processors
├── providers/      # Input file handlers
├── renderers/      # Output formatters
├── schema/         # Data structures and validation
├── services/       # LLM service providers
├── scripts/        # CLI entry points
└── utils/          # Utility functions
```

## Next Steps

- [Core Architecture](02-core-architecture.md) - Detailed pipeline explanation
- [PDF Processing & OCR](03-pdf-processing-and-ocr.md) - How OCR works
- [LLM Integration](04-llm-integration.md) - When and how LLMs are used
