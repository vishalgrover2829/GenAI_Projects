# Component Guide

## User interface

### `ui/app.py`

The Streamlit entry point. It is responsible for page layout, sidebar controls,
PDF discovery and upload, session state, parser artifact reuse, Qdrant checks,
calling the pipeline, and displaying answers and sources.

The `Top-K retrieval` slider controls how many chunks are retrieved. The
`Max images` slider controls how many image records are passed to the vision
model.

## Parsing

### `src/parsing.py`

Defines `ComplexPDFParser`.

Responsibilities:

- validate the PDF and create output folders
- extract selectable text
- render pages for OCR
- extract embedded images
- run optional Tesseract OCR
- extract tables with pdfplumber
- create LangChain documents
- save JSON, Markdown, and image artifacts

## Ingestion

### `src/ingestion.py`

Defines `MultimodalDocumentIngestion`.

Responsibilities:

- load Ollama embedding configuration
- detect the embedding vector dimension
- split page text into chunks
- preserve table and image documents
- add stable document and point IDs
- create or validate the Qdrant collection
- create Qdrant payload indexes
- remove stale points for a re-ingested PDF
- upload documents and vectors to Qdrant

This is the indexing stage. It must use the same embedding model and dimension
as the collection.

## Retrieval

### `src/retriever.py`

Defines `MultimodalQdrantRetriever`.

Responsibilities:

- connect to the configured Qdrant collection
- create Ollama query embeddings
- build optional metadata filters
- perform dense similarity search
- return documents with scores
- expose collection status for diagnostics

## Generation

### `src/generation.py`

Defines `MultimodalRAGGenerator`.

Responsibilities:

- call the retriever
- enforce a context character limit
- format source labels and citations
- attach retrieved images when available
- call `ChatOllama`
- normalize the model response
- return the answer, sources, used images, and usage metadata

## Prompts

### `prompt_library/prompt.py`

Contains the system prompt, user prompt template, image evidence instructions,
and suggested questions shown in the UI. Prompt changes affect grounding,
citation style, and answer behavior without changing retrieval code.

## Exceptions and logging

### `exception/custom_exception.py`

Provides the project-specific exception wrapper used to preserve the original
cause while presenting a consistent application error.

### `logger/custom_logger.py`

Provides the structured logger used by parsing, ingestion, retrieval, and
generation.

## Configuration and data

### `.env`

Local secrets and runtime configuration. It should not be committed.

### `.env.example`

Safe configuration template for Ollama, Qdrant, models, and optional Tesseract.

### `requirements.txt`

Python dependencies for PDF processing, LangChain, Ollama, Qdrant, Streamlit,
and logging.

### `data/`

Source PDFs and test documents.

### `data/uploads/`

PDFs uploaded through the UI.

### `data/parsed_pdf_output/`

Cached parser output, including records, extracted tables, page images, and
embedded images.

## Typical request lifecycle

```text
User selects PDF
  -> UI checks cached parser output
  -> Parser creates LangChain documents
  -> Ingestion embeds and indexes documents
  -> User asks a question
  -> Retriever searches Qdrant
  -> Generator builds grounded context
  -> Ollama returns an answer
  -> UI renders answer and citations
```
