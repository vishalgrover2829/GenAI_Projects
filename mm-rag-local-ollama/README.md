# Local Ollama Multimodal PDF RAG

This project is a Streamlit application that lets you ask questions about PDF
documents. It uses local Ollama models for embeddings and answer generation and
Qdrant Cloud for vector storage and semantic search.

The original OpenAI-based project is separate. This folder is the local Ollama
clone.

## What the application does

The end-to-end pipeline is:

1. Select a PDF from `data/` or upload a new PDF.
2. Extract selectable text, page images, embedded images, OCR text, and tables.
3. Convert those results into LangChain `Document` objects with page metadata.
4. Split long text into smaller chunks.
5. Create dense embeddings locally with Ollama `nomic-embed-text`.
6. Store the vectors and metadata in the Qdrant collection.
7. Embed a user question and retrieve the most relevant chunks.
8. Send the question and retrieved evidence to Ollama `llava:latest`.
9. Display a grounded answer with source file and page references.

See [ARCHITECTURE.md](ARCHITECTURE.md) for the detailed data flow and
[COMPONENT_GUIDE.md](COMPONENT_GUIDE.md) for the responsibility of each file.

## Requirements

- macOS, Linux, or Windows
- Python 3.12
- `uv`
- Ollama
- Qdrant Cloud account, or a reachable Qdrant server
- Tesseract is optional and only needed for OCR of scanned/image-only PDFs

The required Ollama models are:

```bash
ollama pull llava:latest
ollama pull nomic-embed-text
```

`llava:latest` is used by default because it can receive retrieved images.
For text-only PDFs, `llama3.2:3b` is usually faster.

## Setup

From this directory:

```bash
uv venv env --python 3.12
source env/bin/activate
uv pip install -r requirements.txt
cp .env.example .env
```

On Windows:

```cmd
env\Scripts\activate
```

Edit `.env` and configure Qdrant Cloud:

```env
OLLAMA_BASE_URL=http://127.0.0.1:11434
OLLAMA_CHAT_MODEL=llava:latest
OLLAMA_EMBEDDING_MODEL=nomic-embed-text

QDRANT_PATH=
QDRANT_URL=https://your-cluster-endpoint
QDRANT_API_KEY=your-qdrant-api-key
QDRANT_COLLECTION_NAME=mm-rag-ollama
```

Do not commit `.env` or share the API key. `QDRANT_PATH` must be empty when
using Qdrant Cloud. It is only used for local on-disk Qdrant storage.

## Qdrant collection settings

The collection should use:

- Search: single dense vector
- Distance: cosine
- Vector dimension: `768` for `nomic-embed-text`
- Collection name: `mm-rag-ollama`

The ingestion code creates the following payload indexes automatically:

- `metadata.document_id` as Keyword
- `metadata.filename` as Keyword
- `metadata.file_sha256` as Keyword
- `metadata.content_type` as Keyword
- `metadata.page_number` as Integer

If the collection was created with another embedding model or dimension, use a
new collection name or recreate the collection before indexing documents.

## Run the application

Start Ollama if it is not already running:

```bash
ollama serve
```

In another terminal, run Streamlit:

```bash
cd mm-rag-local-ollama
source env/bin/activate
streamlit run ui/app.py
```

Then:

1. Select `data/quick_test.pdf` for a fast test, or upload another PDF.
2. Parse the PDF.
3. Ingest it into Qdrant.
4. Ask a question in the chat area.

To stop Streamlit, press `Control+C` in its terminal. Ollama can remain
running.

## Speed controls

The sidebar contains these settings:

- `Top-K retrieval`: number of chunks sent to the model. Use `3` for faster
  answers; the default is `6`.
- `Max images`: number of retrieved images sent to `llava`. Use `0` for
  text-only PDFs.
- `Model`: use `llama3.2:3b` for faster text-only responses or `llava:latest`
  when image understanding is needed.

The first question is usually slower because Ollama loads the model into
memory. Qdrant Cloud also adds a small network round trip.

## Tesseract and OCR

Tesseract is optional. Normal text PDFs can be parsed without it. Install it
for scanned PDFs or images containing text:

```bash
brew install tesseract
```

Check the installation with:

```bash
tesseract --version
which tesseract
```

If needed, set the executable path in `.env`:

```env
TESSERACT_PATH=/opt/homebrew/bin/tesseract
```

## Cached data

- `data/*.pdf`: source PDFs
- `data/uploads/`: PDFs uploaded through Streamlit
- `data/parsed_pdf_output/`: parser artifacts and extracted images
- `data/quick_test.pdf`: small one-page test document

Parsed artifacts can be reused on later runs. Delete a document's parsed
output only when you want to force a fresh parse.

## Common problems

### Ollama connection refused

Start Ollama and check that it responds:

```bash
ollama list
```

### Model not found

Pull the model named in `.env`:

```bash
ollama pull llava:latest
ollama pull nomic-embed-text
```

### Qdrant payload index error

Restart the application and run ingestion. The ingestion stage creates the
required indexes. If creating indexes manually, use the fields documented in
the Qdrant collection section above.

### Vector dimension mismatch

The collection dimension must match the embedding model. `nomic-embed-text`
uses dimension `768` in this project. Use a fresh collection if an older
collection was created with OpenAI or another embedding model.
