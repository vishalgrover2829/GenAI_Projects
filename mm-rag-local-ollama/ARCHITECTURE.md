# Architecture and Data Flow

## High-level flow

```text
PDF
 |
 v
ComplexPDFParser
 |
 v
LangChain Documents with metadata
 |
 v
Text splitting + Ollama embeddings
 |
 v
Qdrant Cloud collection
 |
 v
Question -> Ollama embedding -> similarity search
                                  |
                                  v
                    retrieved text, tables, and images
                                  |
                                  v
                         Ollama ChatOllama
                                  |
                                  v
                         grounded answer + sources
```

## 1. Document selection

The Streamlit UI in `ui/app.py` supports two inputs:

- Existing PDFs under `data/`
- New PDFs uploaded through the browser and copied to `data/uploads/`

The UI calculates a file fingerprint and uses it to decide whether parsed
artifacts or an existing Qdrant index can be reused.

## 2. PDF parsing

`src/parsing.py` uses several extractors:

- PyMuPDF extracts selectable page text and renders pages.
- Tesseract OCR reads text from rendered pages and embedded images when
  available.
- pdfplumber extracts tables.
- Pillow loads and normalizes images.

The parser writes JSON and Markdown artifacts under
`data/parsed_pdf_output/`. It also creates LangChain documents for page text,
tables, and images.

Each document has metadata such as source PDF path, filename, page number,
content type, table or image index, and image path when applicable.

## 3. Chunking and embedding

`src/ingestion.py` splits long page text with
`RecursiveCharacterTextSplitter`. Tables and image records are preserved as
individual documents instead of being split like normal text.

`OllamaEmbeddings` sends each document to the local Ollama embedding endpoint.
The vector dimension is detected automatically unless
`OLLAMA_EMBEDDING_DIMENSION` is set. For `nomic-embed-text`, the expected
dimension is `768`.

## 4. Qdrant storage

The ingestion class creates the collection if necessary and creates payload
indexes used by the UI and retriever. Each vector payload contains the document
text and metadata.

The same PDF is assigned stable document and point identifiers. Re-ingesting a
PDF deletes its previous points first, preventing duplicate or stale chunks.

## 5. Retrieval

`src/retriever.py` embeds the question with the same Ollama embedding model and
performs dense cosine similarity search in Qdrant.

It supports filters for filename, document ID, content type, exact page number,
and page ranges. The retriever returns documents and similarity scores to the
generation layer.

## 6. Answer generation

`src/generation.py` builds a context block containing retrieved text, tables,
source labels, page numbers, and scores. It then sends the context and question
to `ChatOllama`.

When image records are retrieved, their local image files are converted to
inline image data and sent to the configured vision-capable Ollama model.
`llava:latest` is the default model for this reason.

The prompt instructs the model to stay grounded in retrieved evidence and cite
the relevant source pages.

## 7. Resume behavior

The UI avoids repeating expensive work when possible:

- Existing parser artifacts can be loaded from disk.
- Existing Qdrant points can be reused.
- Only missing parse, ingestion, or generation stages need to run.

Use a fresh collection name when changing embedding models or vector
dimensions.
