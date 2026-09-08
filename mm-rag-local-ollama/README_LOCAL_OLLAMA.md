# Local Ollama MM-RAG

This is a separate Ollama-based clone of the original multimodal PDF RAG
project. The original project remains unchanged and uses OpenAI. This clone
uses local Ollama models for embeddings and answer generation.

For the complete explanation of the pipeline and components, see
[`README.md`](README.md), [`ARCHITECTURE.md`](ARCHITECTURE.md), and
[`COMPONENT_GUIDE.md`](COMPONENT_GUIDE.md).

## 1. Install and start Ollama

Make sure the Ollama application/service is running, then pull the default
models:

```bash
ollama pull llava:latest
ollama pull nomic-embed-text
```

If different models are already installed, put their names in `.env`.

## 2. Qdrant storage

The application uses Qdrant for vector search. This clone defaults to
Qdrant's local on-disk mode, stored in `data/qdrant`, so Docker is not needed.
To use a remote Qdrant server instead, clear `QDRANT_PATH` and set
`QDRANT_URL` and optionally `QDRANT_API_KEY` in `.env`.

## 3. Install Python dependencies

From this directory:

```bash
uv venv env --python 3.12
source env/bin/activate
uv pip install -r requirements.txt
cp .env.example .env
```

On Windows, activate with `env\\Scripts\\activate`.

The default `.env` values point to Ollama at `http://127.0.0.1:11434`. For the
current Qdrant Cloud setup, leave `QDRANT_PATH` empty and set `QDRANT_URL` and
`QDRANT_API_KEY`.

The default chat model is `llava:latest` so images extracted from PDFs can be
sent to the model. For text-only PDFs, `llama3.2:3b` is also supported.

## 4. Run the UI

```bash
streamlit run ui/app.py
```

Upload a PDF, wait for indexing, and ask questions about it.

For a fast first test, select `data/quick_test.pdf`.

## Tesseract

Tesseract is optional for text-based PDFs. Without it, scanned pages and text
inside images will not be OCR'd. The rest of the pipeline can still run.

## Troubleshooting

- `connection refused` on port `11434`: start Ollama.
- `model not found`: run `ollama pull <model-name>` or update `.env`.
- Qdrant errors: check that `data/qdrant` is writable, or configure a remote
  Qdrant URL.
- `vector dimension mismatch`: use a new `QDRANT_COLLECTION_NAME`, because a
  collection created with another embedding model has a different dimension.
