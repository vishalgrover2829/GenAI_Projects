# OpenAI Project to Local Ollama Clone

This document records the changes made in the clone compared with the
original OpenAI-based project.

Original project:

```text
Full-Stack-GenAI-Bootcamp-1.0/mm-rag-full-stack-genai-bootcamp-1.0/
```

Ollama clone:

```text
mm-rag-local-ollama/
```

The original project was preserved. Changes were made only in the clone.

## 1. `requirements.txt`

### Original

The original project used:

```text
langchain-openai
langchain-qdrant
langchain-text-splitters
```

### Ollama clone

The OpenAI integration was replaced with:

```text
langchain-ollama
langchain-qdrant
langchain-text-splitters
```

The PDF parsing, image processing, Streamlit, Qdrant, and logging dependencies
remain part of the project.

## 2. `.env.example`

### Original OpenAI configuration

The original configuration expected OpenAI settings such as:

```env
OPENAI_API_KEY=
OPENAI_CHAT_MODEL=
OPENAI_EMBEDDING_MODEL=
OPENAI_EMBEDDING_DIMENSION=
```

### Ollama configuration

The clone uses:

```env
OLLAMA_BASE_URL=http://127.0.0.1:11434
OLLAMA_CHAT_MODEL=llava:latest
OLLAMA_EMBEDDING_MODEL=nomic-embed-text
```

No `OPENAI_API_KEY` is required.

The clone also supports Qdrant Cloud:

```env
QDRANT_PATH=
QDRANT_URL=https://your-cluster-endpoint
QDRANT_API_KEY=your-qdrant-api-key
QDRANT_COLLECTION_NAME=mm-rag-ollama
```

`QDRANT_PATH` is empty when using Qdrant Cloud. It is only used for optional
local on-disk Qdrant storage.

## 3. `src/ingestion.py`

### Embedding client

Original:

```python
from langchain_openai import OpenAIEmbeddings
```

Clone:

```python
from langchain_ollama import OllamaEmbeddings
```

### Model configuration

Original ingestion used an OpenAI embedding model and a configured OpenAI
dimension.

The clone uses `OLLAMA_EMBEDDING_MODEL` and creates:

```python
OllamaEmbeddings(
    model=self.embedding_model,
    base_url=os.getenv("OLLAMA_BASE_URL"),
)
```

If `OLLAMA_EMBEDDING_DIMENSION` is empty, the clone embeds a small probe text
and detects the vector dimension automatically.

For `nomic-embed-text`, the expected dimension is `768`.

### Qdrant connection

The clone supports both:

- Qdrant Cloud through `QDRANT_URL` and `QDRANT_API_KEY`
- Local on-disk Qdrant through `QDRANT_PATH`

### Collection setup

The ingestion stage creates the collection when it does not exist and creates
payload indexes for:

- `metadata.document_id`
- `metadata.filename`
- `metadata.file_sha256`
- `metadata.content_type`
- `metadata.page_number`

It also removes previous points for the same document before re-indexing it.

## 4. `src/retriever.py`

### Embedding replacement

Original:

```python
from langchain_openai import OpenAIEmbeddings
```

Clone:

```python
from langchain_ollama import OllamaEmbeddings
```

Question embeddings are now generated locally by Ollama. The retriever must
use the same embedding model used during ingestion.

### Configuration validation

The original retriever required `OPENAI_API_KEY`.

The clone removes that requirement and validates either:

- `QDRANT_PATH`, or
- `QDRANT_URL`

## 5. `src/generation.py`

### Chat model replacement

Original:

```python
from langchain_openai import ChatOpenAI
```

Clone:

```python
from langchain_ollama import ChatOllama
```

The clone creates the local chat client with:

```python
ChatOllama(
    model=self.model_name,
    temperature=self.temperature,
    base_url=os.getenv("OLLAMA_BASE_URL"),
)
```

The default model is `llava:latest` because it supports image input.

For text-only PDFs, `llama3.2:3b` can be selected for faster responses.

### Image message format

Retrieved images are converted to data URLs and sent using the image message
format supported by `ChatOllama`.

This allows the model to reason about diagrams, screenshots, and other
retrieved PDF images when the selected Ollama model supports vision.

## 6. `ui/app.py`

### Environment status

The original UI displayed the OpenAI API key status.

The clone displays:

- Ollama endpoint
- Ollama chat model
- Qdrant storage configuration
- Qdrant API key status
- Tesseract status

### Qdrant index detection

The original UI could fail when checking a new collection that did not yet
have payload indexes.

The clone treats a missing-index check as a new collection and allows the
ingestion stage to create the indexes. This prevents errors such as:

```text
Index required but not found for metadata.filename
```

### Chat defaults

The clone defaults to `llava:latest`. The sidebar allows the model, Top-K, and
maximum image count to be changed at runtime.

## 7. `src/parsing.py`

No OpenAI-specific code was present in the parser, so its core behavior stayed
the same.

It still uses:

- PyMuPDF for PDF text and rendering
- pdfplumber for tables
- Pillow for image processing
- pytesseract for optional OCR

Tesseract remains optional for normal text-based PDFs and is useful for scanned
or image-only documents.

## 8. Prompt files

### `prompt_library/prompt.py`

The prompt library remains conceptually the same. It still instructs the model
to answer using retrieved evidence and include source references.

The prompts are now sent to `ChatOllama` instead of `ChatOpenAI`.

## 9. New documentation files

The clone also includes:

- `README.md`: complete setup and usage guide
- `README_LOCAL_OLLAMA.md`: shorter Ollama setup guide
- `ARCHITECTURE.md`: detailed pipeline and data flow
- `COMPONENT_GUIDE.md`: responsibilities of each component
- `RUN_CHECKLIST.txt`: simple end-to-end checklist
- `OPENAI_TO_OLLAMA_CHANGES.md`: this migration reference

## 10. What did not change

The following behavior remains shared with the original project:

- Streamlit is the user interface.
- PDFs are parsed into text, tables, and image records.
- LangChain `Document` objects carry content and metadata.
- Qdrant stores vectors and metadata.
- Retrieval uses dense similarity search.
- Answers are grounded in retrieved context.
- Source pages are displayed with answers.

## Result

The clone removes the OpenAI dependency for embeddings and generation while
preserving the original multimodal RAG workflow. The resulting runtime is:

```text
PDF parser -> Ollama embeddings -> Qdrant Cloud -> Ollama chat model
```
