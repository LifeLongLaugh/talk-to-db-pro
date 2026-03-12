# 📥 Import Guide — Run Talk to DB PRO Yourself

This guide walks you through deploying your own version of Talk to DB PRO over your own documentation. Every service used has a **free tier** — total cost is $0/month.

---

## Prerequisites

You will need free accounts on:

| Service | What it does | Sign up |
|---------|-------------|---------|
| [Dify Cloud](https://cloud.dify.ai) | Hosts and runs the pipeline | Free tier |
| [Qdrant Cloud](https://cloud.qdrant.io) | Stores and searches your document vectors | Free tier (1 cluster, 1GB) |
| [Groq](https://console.groq.com) | Runs Llama 3.1 8B and Qwen3-32B | Free tier |
| [Jina AI](https://jina.ai) | Embedding and reranking models | Free tier (1M tokens/month) |

---

## Step 0 — Prepare your files

Before anything touches Qdrant or Dify, your raw documents need to be cleaned, chunked, and structured correctly. This step determines retrieval quality more than any other — a poorly chunked corpus produces poor answers regardless of how good the models are.

### 0.1 Supported file formats

The pipeline works with any format you can read into plain text. Recommended formats and how to handle each:

| Format | Recommended approach | Notes |
|--------|---------------------|-------|
| `.md` Markdown | Read directly with `open()` | Best format — structure is preserved, headings are natural chunk boundaries |
| `.txt` Plain text | Read directly with `open()` | Split by paragraph or fixed token count |
| `.pdf` PDF | Extract with `pdfplumber` or `pymupdf` | Watch for multi-column layouts and scanned pages |
| `.docx` Word | Extract with `python-docx` | Headings map well to sections |
| `.html` Web pages | Strip tags with `BeautifulSoup` | Remove nav, footer, sidebar content before chunking |
| `.json` Structured data | Flatten relevant fields to text strings | Used in this project for NiFi processor definitions |

Install extraction libraries as needed:
```bash
pip install pdfplumber python-docx beautifulsoup4 lxml
```

---

### 0.2 Chunking strategy

**What is chunking?**
A chunk is a self-contained passage of text — typically a paragraph or section — that will be stored as a single vector in Qdrant. The retriever returns whole chunks, so each chunk needs to make sense on its own when handed to the LLM as context.

**Recommended chunk sizes:**

| Corpus type | Chunk size | Overlap | Rationale |
|-------------|-----------|---------|-----------|
| Technical documentation (prose) | 400–600 tokens | 80 tokens | Preserves full explanations; overlap prevents cutting mid-sentence |
| Reference entries (processor/API definitions) | One entry per chunk | None | Each definition is already self-contained |
| FAQs / Q&A pairs | One Q+A per chunk | None | Atomic units |
| Long narrative docs (guides, manuals) | 500 tokens | 100 tokens | Enough context for the LLM, not so much that irrelevant content is included |

> **Rule of thumb:** a chunk should be long enough to answer a question on its own, but short enough that only one topic is covered. If a chunk mixes two distinct concepts, split it.

**What is overlap?**
Overlap means the last N tokens of chunk `i` are repeated as the first N tokens of chunk `i+1`. This prevents answers from being lost because a sentence was split across a chunk boundary.

---

### 0.3 Chunking by content type

**For markdown documentation** — split on headings (`##`, `###`) first, then by token count if sections are very long:

```python
import re

def split_by_heading(text: str, max_tokens: int = 500) -> list[str]:
    """Split markdown on H2/H3 headings, then by size if needed."""
    sections = re.split(r'\n(?=#{1,3} )', text.strip())
    chunks = []
    for section in sections:
        if not section.strip():
            continue
        words = section.split()
        if len(words) <= max_tokens:
            chunks.append(section.strip())
        else:
            # Further split long sections into overlapping windows
            for i in range(0, len(words), max_tokens - 80):
                chunk = ' '.join(words[i:i + max_tokens])
                if chunk.strip():
                    chunks.append(chunk.strip())
    return chunks
```

**For structured JSON entries** (like NiFi processor definitions) — one JSON object per chunk, flattened to a readable text string:

```python
def processor_to_text(processor: dict) -> str:
    """Convert a NiFi processor JSON entry to a readable text chunk."""
    lines = [
        f"Processor: {processor.get('name', '')}",
        f"Type: {processor.get('type', '')}",
        f"Description: {processor.get('description', '')}",
    ]
    props = processor.get('properties', [])
    if props:
        lines.append("Properties:")
        for p in props:
            lines.append(f"  - {p['name']}: {p.get('description', '')}")
    tags = processor.get('tags', [])
    if tags:
        lines.append(f"Tags: {', '.join(tags)}")
    return '\n'.join(lines)
```

**For PDFs:**

```python
import pdfplumber

def extract_pdf_chunks(path: str, max_tokens: int = 500) -> list[str]:
    chunks = []
    with pdfplumber.open(path) as pdf:
        for page in pdf.pages:
            text = page.extract_text()
            if not text:
                continue
            words = text.split()
            for i in range(0, len(words), max_tokens - 80):
                chunk = ' '.join(words[i:i + max_tokens])
                if len(chunk.strip()) > 80:   # skip tiny fragments
                    chunks.append(chunk.strip())
    return chunks
```

---

### 0.4 Building the payload

Every chunk must carry metadata that the pipeline uses for filtering, citations, and debugging. Build your payload at chunk time — don't try to reconstruct it later.

**Minimum required fields:**

```python
{
    "text":        str,   # the chunk text — what gets retrieved and sent to the LLM
    "source":      str,   # filename — appears in citations, e.g. "databridge_config.md"
    "dataset":     str,   # used for intent-router filtering, e.g. "technical_docs"
    "category":    str,   # optional sub-category tag
    "chunk_index": int,   # position of this chunk within its source file
}
```

**Optional but recommended fields:**

```python
{
    "file_path":   str,   # full relative path, e.g. "docs/databridge/config.md"
    "section":     str,   # heading the chunk falls under, e.g. "Database Configuration"
    "word_count":  int,   # useful for debugging chunk size
    "version":     str,   # document version if relevant
}
```

> **Important:** the `dataset` value in your payload must exactly match the filter value in the Dify search nodes. If you ingest with `"dataset": "my_docs"` but the Dify filter says `"technical_docs"`, zero results will be returned. Keep these in sync.

---

### 0.5 Complete ingestion script

This script handles the full pipeline: read files → chunk → embed in batches → upload to Qdrant.

```python
#!/usr/bin/env python3
"""
Talk to DB PRO — Document Ingestion Script
Chunks documents, embeds with Jina v3, uploads to Qdrant.

Usage:
    python ingest.py --docs-dir ./docs --dataset technical_docs
"""

import os, json, time, argparse, re
from pathlib import Path
import requests

# ── Configuration ──────────────────────────────────────────────────────────────
JINA_API_KEY   = os.environ["JINA_API_KEY"]
QDRANT_API_KEY = os.environ["QDRANT_API_KEY"]
QDRANT_URL     = os.environ["QDRANT_URL"]          # e.g. https://xxxx.cloud.qdrant.io:6333
COLLECTION     = os.environ.get("COLLECTION", "my_knowledge_base")

CHUNK_SIZE     = 500    # tokens (approx words)
CHUNK_OVERLAP  = 80
EMBED_BATCH    = 32     # chunks per Jina API call (stay under rate limits)
UPLOAD_BATCH   = 100    # points per Qdrant upsert call

# ── Chunking ───────────────────────────────────────────────────────────────────
def chunk_markdown(text: str, source: str, dataset: str) -> list[dict]:
    sections = re.split(r'\n(?=#{1,3} )', text.strip())
    chunks = []
    chunk_index = 0
    for section in sections:
        if not section.strip():
            continue
        heading_match = re.match(r'^(#{1,3}) (.+)', section)
        section_title = heading_match.group(2) if heading_match else ""
        words = section.split()
        if len(words) <= CHUNK_SIZE:
            chunks.append({
                "text": section.strip(),
                "source": source,
                "dataset": dataset,
                "category": "documentation",
                "section": section_title,
                "chunk_index": chunk_index,
                "word_count": len(words),
            })
            chunk_index += 1
        else:
            for i in range(0, len(words), CHUNK_SIZE - CHUNK_OVERLAP):
                chunk_text = ' '.join(words[i:i + CHUNK_SIZE])
                if len(chunk_text.strip()) < 60:
                    continue
                chunks.append({
                    "text": chunk_text.strip(),
                    "source": source,
                    "dataset": dataset,
                    "category": "documentation",
                    "section": section_title,
                    "chunk_index": chunk_index,
                    "word_count": len(chunk_text.split()),
                })
                chunk_index += 1
    return chunks

def chunk_text_file(text: str, source: str, dataset: str) -> list[dict]:
    """Generic plain-text chunking by paragraph then size."""
    paragraphs = [p.strip() for p in re.split(r'\n{2,}', text) if p.strip()]
    chunks, chunk_index, buffer = [], 0, []
    buffer_words = 0
    for para in paragraphs:
        words = para.split()
        if buffer_words + len(words) > CHUNK_SIZE and buffer:
            chunks.append({
                "text": ' '.join(buffer),
                "source": source, "dataset": dataset,
                "category": "documentation",
                "chunk_index": chunk_index, "word_count": buffer_words,
            })
            chunk_index += 1
            # Keep overlap
            overlap_text = ' '.join(buffer).split()[-CHUNK_OVERLAP:]
            buffer = overlap_text + words
            buffer_words = len(buffer)
        else:
            buffer.extend(words)
            buffer_words += len(words)
    if buffer:
        chunks.append({
            "text": ' '.join(buffer),
            "source": source, "dataset": dataset,
            "category": "documentation",
            "chunk_index": chunk_index, "word_count": buffer_words,
        })
    return chunks

# ── Embedding ──────────────────────────────────────────────────────────────────
def embed_batch(texts: list[str]) -> list[list[float]]:
    response = requests.post(
        "https://api.jina.ai/v1/embeddings",
        headers={"Authorization": f"Bearer {JINA_API_KEY}", "Content-Type": "application/json"},
        json={
            "model": "jina-embeddings-v3",
            "input": texts,
            "task": "retrieval.passage",
            "dimensions": 1024,
        },
        timeout=60,
    )
    response.raise_for_status()
    return [item["embedding"] for item in response.json()["data"]]

# ── Upload ─────────────────────────────────────────────────────────────────────
def upload_points(points: list[dict]):
    response = requests.put(
        f"{QDRANT_URL}/collections/{COLLECTION}/points?wait=true",
        headers={"api-key": QDRANT_API_KEY, "Content-Type": "application/json"},
        json={"points": points},
        timeout=60,
    )
    response.raise_for_status()

# ── Main ───────────────────────────────────────────────────────────────────────
def ingest(docs_dir: str, dataset: str):
    docs_path = Path(docs_dir)
    all_chunks = []

    # Collect all chunks from all files
    for filepath in sorted(docs_path.rglob("*")):
        if filepath.suffix not in (".md", ".txt"):
            continue
        print(f"  Reading {filepath.name}...")
        text = filepath.read_text(encoding="utf-8", errors="ignore")
        source = filepath.name

        if filepath.suffix == ".md":
            chunks = chunk_markdown(text, source, dataset)
        else:
            chunks = chunk_text_file(text, source, dataset)

        print(f"    → {len(chunks)} chunks")
        all_chunks.extend(chunks)

    print(f"\nTotal chunks to ingest: {len(all_chunks)}")

    # Embed in batches
    point_id = 1
    points_buffer = []

    for i in range(0, len(all_chunks), EMBED_BATCH):
        batch = all_chunks[i:i + EMBED_BATCH]
        texts = [c["text"] for c in batch]
        print(f"  Embedding batch {i // EMBED_BATCH + 1} / {(len(all_chunks) + EMBED_BATCH - 1) // EMBED_BATCH}...")

        embeddings = embed_batch(texts)
        time.sleep(0.3)  # be gentle on the free tier rate limit

        for chunk, embedding in zip(batch, embeddings):
            points_buffer.append({
                "id": point_id,
                "vector": {"dense": embedding},
                "payload": chunk,
            })
            point_id += 1

        # Upload when buffer is full
        if len(points_buffer) >= UPLOAD_BATCH:
            print(f"  Uploading {len(points_buffer)} points to Qdrant...")
            upload_points(points_buffer)
            points_buffer = []

    # Upload remainder
    if points_buffer:
        print(f"  Uploading final {len(points_buffer)} points to Qdrant...")
        upload_points(points_buffer)

    print(f"\n✓ Ingestion complete. {point_id - 1} points uploaded to '{COLLECTION}'.")

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Ingest documents into Qdrant for Talk to DB PRO")
    parser.add_argument("--docs-dir", required=True, help="Directory containing .md or .txt files")
    parser.add_argument("--dataset",  required=True, help="Dataset name tag (e.g. 'technical_docs')")
    args = parser.parse_args()

    print(f"Ingesting '{args.docs_dir}' as dataset='{args.dataset}' into collection='{COLLECTION}'")
    ingest(args.docs_dir, args.dataset)
```

**Run it:**
```bash
# Set environment variables first
export JINA_API_KEY="your_jina_key"
export QDRANT_API_KEY="your_qdrant_key"
export QDRANT_URL="https://xxxx.cloud.qdrant.io:6333"
export COLLECTION="my_knowledge_base"

# Ingest your first dataset
python ingest.py --docs-dir ./docs/technical --dataset technical_docs

# Ingest a second dataset into the same collection
python ingest.py --docs-dir ./docs/processors --dataset processors_services
```

---

### 0.6 Verify ingestion

Check your collection point count via the Qdrant dashboard, or with `curl`:

```bash
curl "YOUR_CLUSTER_URL/collections/YOUR_COLLECTION_NAME" \
  -H "api-key: YOUR_QDRANT_API_KEY" | python3 -m json.tool
```

Look for `"points_count"` in the response. It should match your expected total.

Do a quick sanity search — embed a test question and check that results come back:

```bash
curl -X POST "YOUR_CLUSTER_URL/collections/YOUR_COLLECTION_NAME/points/query" \
  -H "api-key: YOUR_QDRANT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": [0.01, 0.02, ...],
    "using": "dense",
    "limit": 3,
    "with_payload": true,
    "with_vector": false
  }'
```

If you get 3 results with `text` and `source` in the payload, ingestion is correct. Proceed to Step 1.

---

### 0.7 Common file preparation mistakes

| Mistake | Effect | Fix |
|---------|--------|-----|
| Chunks too large (1000+ tokens) | LLM context filled by one chunk; low precision | Keep chunks under 600 tokens |
| Chunks too small (<50 tokens) | Fragments without enough context to answer questions | Merge short paragraphs before chunking |
| Missing `text` field in payload | Result Mapper returns empty documents; answers say "insufficient information" | Always verify payload schema before uploading |
| `dataset` value mismatched with Dify filter | Filtered searches return zero results | Use exact same string in payload and Dify filter node |
| No chunk overlap | Answers lost at chunk boundaries | Add 60–100 token overlap between consecutive chunks |
| Keeping boilerplate content (headers, footers, nav) | Noise in retrieval; irrelevant passages ranked highly | Strip structural elements before chunking |
| Ingesting duplicate content | Redundant passages waste vector space and confuse reranker | Deduplicate source files before ingestion |

---

## Step 1 — Set up Qdrant

### 1.1 Create a cluster

1. Log in to [Qdrant Cloud](https://cloud.qdrant.io)
2. Create a new cluster — select the **Free** tier
3. Note your **Cluster URL** (format: `https://xxxx-xxxx.cloud.qdrant.io:6333`)
4. Create an **API key** under Access → API Keys. Copy it — you'll need it later.

### 1.2 Create the collection

You need a collection with a named `dense` vector (1024-dim, cosine). Run this via the Qdrant Cloud console or `curl`:

```bash
curl -X PUT "YOUR_CLUSTER_URL/collections/YOUR_COLLECTION_NAME" \
  -H "api-key: YOUR_QDRANT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "vectors": {
      "dense": {
        "size": 1024,
        "distance": "Cosine"
      }
    }
  }'
```

Replace `YOUR_COLLECTION_NAME` with whatever you'd like (e.g. `my_knowledge_base`).

### 1.3 Index payload fields for filter performance

If you're using multiple datasets (like this project does), index the filter fields:

```bash
curl -X PUT "YOUR_CLUSTER_URL/collections/YOUR_COLLECTION_NAME/index" \
  -H "api-key: YOUR_QDRANT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"field_name": "dataset", "field_schema": "keyword"}'
```

---

## Step 2 — Ingest your documents

You need to chunk your documents, embed them with Jina Embeddings v3, and upload the resulting vectors to Qdrant.

### 2.1 Required payload schema

Each point you upload to Qdrant must have at minimum:

```json
{
  "id": 1,
  "vector": {
    "dense": [0.12, -0.34, ...]
  },
  "payload": {
    "text": "The actual passage text...",
    "source": "your_document_filename.md",
    "dataset": "your_dataset_name",
    "category": "optional_category_tag",
    "chunk_index": 0
  }
}
```

The `text` field is what gets retrieved and sent to the LLM. The `source` field appears in citations. The `dataset` field is what the intent router uses to filter searches.

### 2.2 Embed your chunks

Call Jina Embeddings v3 for each chunk:

```python
import requests, json

JINA_API_KEY = "your_jina_api_key"

def embed(texts: list[str]) -> list[list[float]]:
    response = requests.post(
        "https://api.jina.ai/v1/embeddings",
        headers={"Authorization": f"Bearer {JINA_API_KEY}"},
        json={
            "model": "jina-embeddings-v3",
            "input": texts,
            "task": "retrieval.passage",   # use 'retrieval.passage' for ingestion
            "dimensions": 1024
        }
    )
    return [item["embedding"] for item in response.json()["data"]]
```

> **Note:** Use `task=retrieval.passage` when embedding documents for ingestion, and `task=retrieval.query` when embedding the user's question at query time (this is already set in the Dify workflow).

### 2.3 Upload to Qdrant

```python
import requests

QDRANT_URL   = "YOUR_CLUSTER_URL"
QDRANT_KEY   = "YOUR_QDRANT_API_KEY"
COLLECTION   = "YOUR_COLLECTION_NAME"

def upload_points(points: list[dict]):
    requests.put(
        f"{QDRANT_URL}/collections/{COLLECTION}/points",
        headers={"api-key": QDRANT_KEY, "Content-Type": "application/json"},
        json={"points": points}
    )

# Example point
points = [
    {
        "id": 1,
        "vector": {"dense": embed(["Your passage text here"])[0]},
        "payload": {
            "text": "Your passage text here",
            "source": "my_doc.md",
            "dataset": "my_dataset",
            "category": "general",
            "chunk_index": 0
        }
    }
]
upload_points(points)
```

---

## Step 3 — Get your API keys

| Key | Where to find it |
|-----|----------------|
| `JINA_API_KEY` | [jina.ai](https://jina.ai) → Sign in → API Keys |
| `QDRANT_API_KEY` | Qdrant Cloud → Your cluster → Access → API Keys |
| `GROQ_API_KEY` | [console.groq.com](https://console.groq.com) → API Keys (used inside Dify model settings, not as an env var) |

Build your `QDRANT_POINTS_URL` — this is the full endpoint for vector search:

```
https://YOUR_CLUSTER_URL/collections/YOUR_COLLECTION_NAME/points/query
```

Example:
```
https://abc123.europe-west3-0.gcp.cloud.qdrant.io:6333/collections/my_knowledge_base/points/query
```

---

## Step 4 — Import the Dify workflow

### 4.1 Create a Dify app

1. Log in to [Dify Cloud](https://cloud.dify.ai)
2. Click **Create App** → choose **Chatflow** (Advanced Chat)

### 4.2 Import the DSL

1. In the Chatflow editor, click the **⋯ menu** (top right) → **Import DSL**
2. Upload [`dify-workflow/talk-to-db-pro.yml`](./dify-workflow/talk-to-db-pro.yml)
3. The full 17-node pipeline will be imported

### 4.3 Add model providers

1. Go to **Settings** → **Model Providers**
2. Add **Groq** — paste your Groq API key
3. Enable `llama-3.1-8b-instant` and `qwen3-32b` models

### 4.4 Set environment variables

In Dify, go to **Settings** → **Environment Variables** and add:

| Variable | Value |
|----------|-------|
| `JINA_API_KEY` | Your Jina AI API key |
| `QDRANT_API_KEY` | Your Qdrant API key |
| `QDRANT_POINTS_URL` | Full `/points/query` URL (from Step 3) |

These are referenced in the workflow as `{{#env.JINA_API_KEY#}}` etc. — they are never hardcoded.

---

## Step 5 — Adapt for your collection

If your Qdrant collection name or dataset names differ from the original, update the Qdrant search node bodies in the Dify editor:

**Unified Search node** — update the URL to your `QDRANT_POINTS_URL` (already done via env var).

**Tech Search node** — change the filter value to match your dataset name:
```json
"filter": {"must": [{"key": "dataset", "match": {"value": "YOUR_DATASET_NAME"}}]}
```

**Intent Router code node** — update the `dual_signals` regex patterns to match your own processor/component naming conventions if you're not using NiFi.

---

## Step 6 — Publish and chat

1. Click **Publish** in the Dify editor
2. Click **Run** to open the chatbot
3. Ask a question about your documents

If you see an error on first run, check the **Logs** panel in Dify — it shows the output of every node. Common issues are listed below.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Qdrant returns HTTP 400 "Not existing vector name error" | Collection does not have a named `dense` vector | Re-create collection with `"vectors": {"dense": {...}}` schema |
| Empty results from search | `QDRANT_POINTS_URL` wrong | Ensure URL ends in `/points/query`, not `/search` |
| `AttributeError: 'str' object has no attribute 'get'` in Format node | Jina Reranker returns document as string, not dict | Already handled in the workflow — check you imported the latest DSL |
| All answers say "insufficient information" | Documents not ingested correctly | Check payload has a `text` field; verify `dataset` filter value matches your ingestion |
| Vector extraction fails | Dify array cap hit | Already handled via `json.dumps()` — do not change output type to `array[number]` |
| Wrong search path fired | Intent router regex too broad/narrow | Edit the `dual_signals` patterns in the Intent Router code node |

---

## Adapting to your own use case

This pipeline is **domain-agnostic**. The only things that are HxGN/NiFi-specific are:

1. The documents you ingest into Qdrant
2. The `dual_signals` regex patterns in the Intent Router (tune for your naming conventions)
3. The dataset names (`technical_docs`, `processors_services`) in the filter bodies

Everything else — the embedding model, reranker, LLMs, routing logic, synthesis prompt — works identically for any document corpus.

---

## Security notes

- API keys are stored as **Dify environment variables** (masked as `*****`) — never visible in the YAML export
- The DSL file in this repo has placeholder comments where keys were — it is safe to share publicly
- Rotate keys via each provider's dashboard; update only the Dify environment variable

---

*Questions? Open an issue on the [GitHub repo](https://github.com/YOUR_GITHUB_USERNAME/talk-to-db-pro).*
