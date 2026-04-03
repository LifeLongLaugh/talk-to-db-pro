# 📥 Import Guide — Run Talk to DB PRO Yourself

This guide walks you through deploying your own instance of Talk to DB PRO over your own documentation. The pipeline is domain-agnostic — the only things specific to HxGN Databridge Pro and NiFi are the documents ingested and a handful of regex patterns in the Intent Router. Everything else transfers directly to any document corpus.

Every service used has a free tier. Total infrastructure cost: **$0/month**, with every layer independently substitutable if you later want to migrate to paid tiers.

---

## Prerequisites

Free accounts on:

| Service | What it does | Sign up |
| --- | --- | --- |
| [Dify Cloud](https://cloud.dify.ai) | Hosts and runs the pipeline | Free tier |
| [Qdrant Cloud](https://cloud.qdrant.io) | Stores and searches your document vectors | Free tier (1 cluster, 1 GB) |
| [Groq](https://console.groq.com) | Runs Llama 3.1 8B and Qwen3-32B | Free tier |
| [Jina AI](https://jina.ai) | Embedding and reranking models | Free tier (1 M tokens/month) |

---

## Step 0 — Prepare your files

Before anything touches Qdrant or Dify, your raw documents need to be cleaned, chunked, and structured correctly. This step determines retrieval quality more than any other — a poorly chunked corpus produces poor answers regardless of how good the models are.

### 0.1 Supported file formats

The pipeline works with any format you can read into plain text:

| Format | Recommended approach | Notes |
| --- | --- | --- |
| `.md` Markdown | Read directly with `open()` | Best format — structure is preserved, headings are natural chunk boundaries |
| `.txt` Plain text | Read directly with `open()` | Split by paragraph or fixed token count |
| `.pdf` PDF | Extract with `pdfplumber` or `pymupdf` | Watch for multi-column layouts and scanned pages |
| `.docx` Word | Extract with `python-docx` | Headings map well to sections |
| `.html` Web pages | Strip tags with `BeautifulSoup` | Remove nav, footer, and sidebar content before chunking |
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
| --- | --- | --- | --- |
| Technical documentation (prose) | 400–600 tokens | 80 tokens | Preserves full explanations; overlap prevents cutting mid-sentence |
| Reference entries (processor/API definitions) | One entry per chunk | None | Each definition is already self-contained |
| FAQs / Q&A pairs | One Q+A per chunk | None | Atomic units |
| Long narrative docs (guides, manuals) | 500 tokens | 100 tokens | Enough context for the LLM, not so much that irrelevant content is included |

> **Rule of thumb:** a chunk should be long enough to answer a question on its own, but short enough that only one topic is covered. If a chunk mixes two distinct concepts, split it.

**What is overlap?**
Overlap means the last N tokens of chunk `i` are repeated as the first N tokens of chunk `i+1`. This prevents answers from being lost because a sentence was split across a chunk boundary.

---

### 0.3 Chunking by content type

**For markdown documentation** — split on headings (`##`, `###`) first, then by token count if sections are very long.

**For structured JSON entries** (like NiFi processor definitions) — one JSON object per chunk, flattened to a readable text string.

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

### 0.5 Verify ingestion

After uploading points, confirm the count via the Qdrant REST API:

```bash
curl "YOUR_CLUSTER_URL/collections/YOUR_COLLECTION_NAME" \
  -H "api-key: YOUR_QDRANT_API_KEY"
```

Look for `"points_count"` in the response. It should match your expected total. If it's lower, check for upload errors or payload validation failures in your ingestion script.

---

### 0.6 Common file preparation mistakes

| Mistake | Effect | Fix |
| --- | --- | --- |
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
3. Note your **Cluster URL** (format: `https://YOUR_CLUSTER_ID.YOUR_REGION.gcp.cloud.qdrant.io:6333`)
4. Create an **API key** under Access → API Keys

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

Chunk your documents, embed them with Jina Embeddings v3, and upload the resulting vectors to Qdrant.

### 2.1 Required payload schema

Each point you upload to Qdrant must have at minimum:

```python
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

> **Note:** Use `task=retrieval.passage` when embedding documents for ingestion, and `task=retrieval.query` when embedding the user's question at query time. This asymmetric task designation is how Jina v3 achieves better retrieval accuracy than symmetric embedding.

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
| --- | --- |
| `JINA_API_KEY` | [jina.ai](https://jina.ai) → Sign in → API Keys |
| `QDRANT_API_KEY` | Qdrant Cloud → Your cluster → Access → API Keys |
| `GROQ_API_KEY` | [console.groq.com](https://console.groq.com) → API Keys (configured inside Dify model settings, not as an env var) |

Build your `QDRANT_POINTS_URL` — this is the full endpoint for vector search:

```
https://YOUR_CLUSTER_URL/collections/YOUR_COLLECTION_NAME/points/query
```

Example format:

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
2. Upload [`dify-workflow/talk-to-db-pro.yml`](https://github.com/LifeLongLaugh/talk-to-db-pro/blob/main/dify-workflow/talk-to-db-pro.yml)
3. The full 17-node pipeline will be imported

### 4.3 Add model providers

1. Go to **Settings** → **Model Providers**
2. Add **Groq** — paste your Groq API key
3. Enable `llama-3.1-8b-instant` and `qwen3-32b` models

### 4.4 Set environment variables

In Dify, go to **Settings** → **Environment Variables** and add:

| Variable | Value |
| --- | --- |
| `JINA_API_KEY` | Your Jina AI API key |
| `QDRANT_API_KEY` | Your Qdrant API key |
| `QDRANT_POINTS_URL` | Full `/points/query` URL (from Step 3) |

These are referenced in the workflow as `{{#env.JINA_API_KEY#}}` etc. — they are never hardcoded in node bodies or visible in the YAML export.

---

## Step 5 — Adapt for your collection

If your Qdrant collection name or dataset names differ from the original, update the search node bodies in the Dify editor:

**Unified Search node** — the URL is already parameterised via `{{#env.QDRANT_POINTS_URL#}}`. No change needed here.

**Tech Search node** — change the filter value to match your dataset name:

```json
"filter": {"must": [{"key": "dataset", "match": {"value": "YOUR_DATASET_NAME"}}]}
```

**Intent Router code node** — update the `dual_signals` regex patterns to match your own component naming conventions if you're not using NiFi:

```python
dual_signals = [
    r'\b[A-Z][a-zA-Z]+(?:YourComponentSuffix)\b',  # adapt to your naming scheme
    r'\bconfigure\b',
    r'\bproperties\b',
    # add domain-specific terms here
]
```

---

## Step 6 — Publish and chat

1. Click **Publish** in the Dify editor
2. Click **Run** to open the chatbot
3. Ask a question about your documents

If you see an error on first run, open the **Logs** panel in Dify — it shows the output of every node in the pipeline, making it straightforward to trace exactly where a failure occurred.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Qdrant returns HTTP 400 "Not existing vector name error" | Collection does not have a named `dense` vector | Re-create collection with `"vectors": {"dense": {...}}` schema |
| Empty results from search | `QDRANT_POINTS_URL` incorrect | Ensure URL ends in `/points/query`, not `/search` |
| `AttributeError: 'str' object has no attribute 'get'` in Format node | Jina Reranker returns document as string, not dict | Already handled in the workflow — check you imported the latest DSL |
| All answers say "insufficient information" | Documents not ingested correctly, or `dataset` filter mismatch | Check payload has a `text` field; verify `dataset` value matches the filter string in the Dify search nodes exactly |
| Vector extraction fails | Dify array cap hit | Already handled via `json.dumps()` — do not change output type to `array[number]` |
| Wrong search path fired | Intent router regex too broad or too narrow | Edit the `dual_signals` patterns in the Intent Router code node |
| `points_count` lower than expected after ingestion | Upload batching errors or payload validation failures | Add error handling to your upload loop; print response status per batch |

---

## Adapting to your own use case

This pipeline is domain-agnostic. The only things that are HxGN/NiFi-specific are:

1. The documents you ingest into Qdrant
2. The `dual_signals` regex patterns in the Intent Router (tune for your naming conventions)
3. The dataset names (`technical_docs`, `processors_services`) in the filter bodies

Everything else — the embedding model, reranker, LLMs, routing logic, synthesis prompt — works identically for any document corpus. To port to a new domain, change only those three things.

---

## Security notes

- API keys are stored as **Dify environment variables** (displayed as `*****`) — never visible in the YAML export or accessible to end users of the chatbot
- The DSL file in this repo has placeholder comments where keys were removed — it is safe to share publicly
- Rotate keys via each provider's dashboard; update only the corresponding Dify environment variable — no code changes needed

---

*Questions? Open an issue on the [GitHub repo](https://github.com/LifeLongLaugh/talk-to-db-pro).*
