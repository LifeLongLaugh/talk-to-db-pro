# Talk to DB PRO — Full Technical Project Report

**Conversational RAG Pipeline over Technical Documentation**

| Platform | Vector DB | Embedding | Reranker | LLM |
|----------|-----------|-----------|----------|-----|
| Dify Cloud (Advanced-Chat) | Qdrant Cloud | Jina Embeddings v3 | Jina Reranker v2 | Qwen3-32B + Llama 3.1 8B via Groq |

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Technology Stack](#2-technology-stack)
3. [Pipeline Architecture](#3-pipeline-architecture)
4. [Key Engineering Decisions](#4-key-engineering-decisions)
5. [Bugs Found and Resolved](#5-bugs-found-and-resolved)
6. [Environment Variables & Security](#6-environment-variables--security)
7. [Performance & Quality](#7-performance--quality)
8. [Skills Demonstrated](#8-skills-demonstrated)
9. [Potential Future Improvements](#9-potential-future-improvements)
10. [Quick Reference](#10-quick-reference)

---

## 1. Project Overview

**Talk to DB PRO** is a production-grade conversational Retrieval-Augmented Generation (RAG) system built on Dify Cloud. It enables natural language querying over HxGN Databridge Pro technical documentation and NiFi processor/controller service definitions — a corpus that is otherwise navigated manually through PDFs and text files.

The system takes a user's question, expands and rewrites it using an LLM, embeds it with Jina Embeddings v3, retrieves semantically similar passages from Qdrant Cloud, reranks them with Jina Reranker v2, and synthesises a grounded, cited answer using Qwen3-32B hosted on Groq.

### 1.1 The Problem Being Solved

- HxGN Databridge Pro documentation spans multiple markdown files and is not searchable by meaning — only by keyword
- The NiFi processor catalogue contains 200+ processors with complex property descriptions that engineers frequently need to look up
- Finding which processors support a specific capability (e.g. SQL execution, SFTP transfer) required manual scanning
- No system existed to answer follow-up questions in context across multiple document types simultaneously

### 1.2 Solution Summary

- A fully automated pipeline that accepts a natural language question and returns a cited, grounded answer
- Dual-dataset intelligent routing: conceptual queries search across all documents; processor-specific queries apply targeted filters
- Conversation memory: the system maintains context across multi-turn dialogues
- Zero infrastructure cost: all components operate on free tiers (Dify Cloud, Qdrant Cloud free, Groq free, Jina free)

---

## 2. Technology Stack

| Layer | Technology | Specifics | Role |
|-------|-----------|-----------|------|
| Orchestration | Dify Cloud | Advanced-chat / Chatflow v0.6.0 | Visual workflow builder, runtime, API gateway |
| LLM — Rewriter | Groq + Llama 3.1 8B Instant | llama-3.1-8b-instant, temp=0 | Query expansion and rewriting |
| LLM — Synthesis | Groq + Qwen3-32B | qwen3-32b, temp=0.1, /no_think | Grounded answer generation with citations |
| Embeddings | Jina Embeddings v3 | jina-embeddings-v3, 1024-dim, cosine | Dense semantic vector generation |
| Reranker | Jina Reranker v2 | jina-reranker-v2-base-multilingual, top_n=5 | Cross-encoder relevance rescoring |
| Vector Database | Qdrant Cloud | GCP europe-west3, hxgn_knowledge_base | Vector storage and retrieval |
| HTTP Client | Dify HTTP Request nodes | Native Dify nodes, POST to Qdrant /points/query | All external API calls |

### 2.1 Knowledge Base (Qdrant Collection)

| Property | Value |
|----------|-------|
| Collection name | hxgn_knowledge_base |
| Total points | 751 |
| Vector: dense | jina-embeddings-v3, 1024 dimensions, cosine similarity |
| Vector: sparse | BM25 (built-in Qdrant sparse) — ingested, not active in current search |
| Dataset 1: technical_docs | 539 points — chunked markdown from HxGN Databridge Pro documentation |
| Dataset 2: processors_services | 212 points — NiFi processor and controller service definitions |
| Payload fields (technical_docs) | text, source, dataset, category, chunk_index, file_path |
| Payload fields (processors_services) | text, source, dataset, category, processor_name, full_type, bundle, version, tags, restricted |
| Indexed fields | All keyword/category fields indexed for filter performance |

---

## 3. Pipeline Architecture

The flow is built as a Dify Advanced-Chat Chatflow consisting of 17 nodes connected in a linear chain with a single conditional branch. Every component is wired by explicit variable references using Dify's `{{#node_id.output_field#}}` syntax.

### 3.1 Full Node Sequence

```
User Input (Start)
  └─► Query Rewriter LLM           [Groq / llama-3.1-8b-instant]
        └─► Jina Embeddings         [HTTP POST api.jina.ai/v1/embeddings]
              └─► Extract Vector    [Code: parse embedding from JSON response]
                    └─► Expanded Query Display  [Answer node — streams rewritten query to user]
                          └─► Intent Router     [Code: regex pattern matching on sys.query]
                                └─► IF/ELSE     [search_mode == 'unified'?]
                                      │
                                      ├─ TRUE  ─► Qdrant Unified Search     [HTTP POST /points/query]
                                      │              └─► Variable Aggregator
                                      │
                                      └─ FALSE ─► Qdrant Tech Search        [HTTP POST /points/query, filter: technical_docs]
                                                ─► Qdrant Proc Search        [HTTP POST /points/query, filter: processors_services]
                                                     └─► Merge Results       [Code: combine + sort by score]
                                                           └─► Variable Aggregator
                                                                 └─► Qdrant Result Mapper  [Code: normalise + extract text]
                                                                       └─► Jina Reranker   [HTTP POST api.jina.ai/v1/rerank]
                                                                             └─► Format for LLM Context  [Code: build numbered passages]
                                                                                   └─► Answer Synthesis LLM  [Groq / Qwen3-32B]
                                                                                         └─► Final Answer  [Answer node]
```

### 3.2 Node-by-Node Specifications

---

#### Node 1 — User Input (Start)
Standard Dify Start node. Exposes `sys.query` (current user message) and `sys.files` to downstream nodes.

---

#### Node 2 — Query Rewriter (LLM)

| Property | Value |
|----------|-------|
| Node ID | 1772973692102 |
| Model | llama-3.1-8b-instant via Groq |
| Temperature | 0 (deterministic) |
| Memory | Enabled, window size 5 turns |
| Input | sys.query + conversation history via memory |
| Output | `.text` — rewritten and expanded query string |
| Purpose | Fix typos, add synonyms, make implicit references explicit to maximise retrieval recall |

**System prompt:**
```
You are a query expansion assistant for a retrieval-augmented system.
Given the conversation history and the user's latest message, rewrite
and expand the query to maximize retrieval recall.
- Fix typos and ambiguities
- Add synonyms or related terms if helpful
- Make implicit context explicit (e.g. referencing a previously mentioned topic)
- Output ONLY the rewritten query. No explanations, no preamble, no quotes.
```

**User prompt:**
```
Current user message: {{#sys.query#}}

Rewritten query:
```

---

#### Node 3 — Jina Embeddings (HTTP Request)

| Property | Value |
|----------|-------|
| Node ID | 1773157495040 |
| URL | https://api.jina.ai/v1/embeddings |
| Method | POST |
| Auth header | `Authorization: Bearer {{#env.JINA_API_KEY#}}` |
| Content-Type | application/json |
| Input | `{{#1772973692102.text#}}` — the rewritten query |
| Output | `.body` — raw JSON response containing embedding array |

**Request body:**
```json
{
  "model": "jina-embeddings-v3",
  "input": ["{{#1772973692102.text#}}"],
  "task": "retrieval.query",
  "dimensions": 1024
}
```

---

#### Node 4 — Extract Vector (Code)

| Property | Value |
|----------|-------|
| Node ID | 1773158737461 |
| Input | `jina_response` ← Jina Embeddings `.body` (string) |
| Output | `query_vector` — type: **string** (JSON-serialised float array) |

**Code:**
```python
import json

def main(jina_response: str) -> dict:
    data = json.loads(jina_response)
    embedding = data["data"][0]["embedding"]
    return {"query_vector": json.dumps(embedding)}  # serialised as string
```

> **Key design decision:** Dify code node array outputs are capped at 500 elements. Jina v3 produces 1024-dimensional vectors. The solution is to serialise the embedding as a JSON string and inject it unquoted into the Qdrant HTTP body template. Qdrant's JSON parser receives a valid float array.

---

#### Node 5 — Expanded Query Display (Answer)

| Property | Value |
|----------|-------|
| Node ID | 1773065598617 |
| Type | Dify Answer node (streaming) |
| Content | `> 🔍 **Thinking:** {{#1772973692102.text#}}` |
| Purpose | Shows the user the expanded query in real time — transparency UX pattern while retrieval proceeds |

---

#### Node 6 — Intent Router (Code)

Classifies the **original** user query (`sys.query`, not the rewritten version) into one of two search strategies. Using the raw query prevents false dual-search triggers from synonyms added by the rewriter.

| Property | Value |
|----------|-------|
| Node ID | 1773077552800 |
| Input | `query` ← `sys.query` (string) |
| Output | `search_mode` — either `"unified"` or `"dual"` |

**Code:**
```python
import re

def main(query: str) -> dict:
    dual_signals = [
        r'\b[A-Z][a-zA-Z]+(?:Processor|Service|Pool|Reader|Writer|Controller)\b',  # CamelCase component
        r'\bconfigure\b',
        r'\bproperties\b',
        r'\bwhat does .+ do\b',
        r'\bcontroller service\b',
        r'\bListSFTP|PutS3|GetFile|DBCPConnection|JsonPath\b',  # known processor names
    ]
    for pattern in dual_signals:
        if re.search(pattern, query, re.IGNORECASE):
            return {"search_mode": "dual"}
    return {"search_mode": "unified"}
```

| search_mode | Triggers |
|-------------|----------|
| unified | Conceptual questions, broad how-to, cross-dataset |
| dual | Named processor/service, "properties", "configure", "what does X do", "controller service" |

---

#### Node 7 — IF/ELSE Branch

| Property | Value |
|----------|-------|
| Node ID | 1773077675139 |
| Condition | `search_mode == "unified"` |
| True branch | Routes to Unified Qdrant Search |
| False branch | Routes to both Tech and Proc Qdrant Search nodes in parallel |

---

#### Nodes 8 / 9 / 10 — Qdrant Dense Search (HTTP Request × 3)

All three nodes call the same Qdrant endpoint. Authentication via `api-key` header. All use POST.

| Parameter | Unified | Tech | Proc |
|-----------|---------|------|------|
| Node ID | 1773185085153 | 17731854106500 | 17731854155860 |
| Title | UNIFIED DENSE SEARCH | TECH DENSE SEARCH | PROC DENSE SEARCH |
| URL | `{{#env.QDRANT_POINTS_URL#}}` | same | same |
| limit | 10 | 5 | 5 |
| using | "dense" | "dense" | "dense" |
| filter | none | `dataset == technical_docs` | `dataset == processors_services` |
| Active when | search_mode == unified | search_mode == dual | search_mode == dual |

**Unified body:**
```json
{
  "query": {{#1773158737461.query_vector#}},
  "using": "dense",
  "limit": 10,
  "with_payload": true,
  "with_vector": false
}
```

**Tech body:**
```json
{
  "query": {{#1773158737461.query_vector#}},
  "using": "dense",
  "filter": {"must": [{"key": "dataset", "match": {"value": "technical_docs"}}]},
  "limit": 5,
  "with_payload": true,
  "with_vector": false
}
```

**Proc body:**
```json
{
  "query": {{#1773158737461.query_vector#}},
  "using": "dense",
  "filter": {"must": [{"key": "dataset", "match": {"value": "processors_services"}}]},
  "limit": 5,
  "with_payload": true,
  "with_vector": false
}
```

> Note: `query_vector` is injected **unquoted** — it is a JSON string that evaluates to a valid float array when parsed by Qdrant.

---

#### Node 11 — Merge Results (Code — dual path only)

Combines Tech and Proc results, unwrapping Qdrant's `result.points[]` envelope from each response, then sorts by cosine similarity descending.

| Property | Value |
|----------|-------|
| Node ID | 1773078688584 |
| Input: tech_results | Tech HTTP node `.body` (string) |
| Input: proc_results | Proc HTTP node `.body` (string) |
| Output: merged_results | string — JSON array of combined, sorted point objects |

**Code:**
```python
import json

def main(tech_results: str, proc_results: str) -> dict:
    tech = json.loads(tech_results).get("result", {}).get("points", [])
    proc = json.loads(proc_results).get("result", {}).get("points", [])
    combined = tech + proc
    combined.sort(key=lambda x: x.get("score", 0), reverse=True)
    return {"merged_results": json.dumps(combined)}
```

---

#### Node 12 — Variable Aggregator

Merges the unified and dual execution paths back into a single variable for downstream processing.

| Property | Value |
|----------|-------|
| Node ID | 1773078889010 |
| Input A | Unified path: UNIFIED DENSE SEARCH `.body` |
| Input B | Dual path: Merge Results `.merged_results` |
| Output type | string |
| Output variable | `output` |

---

#### Node 13 — Qdrant Result Mapper (Code)

Normalises aggregated results into a clean list of document dicts. Handles two different input shapes: the Qdrant response envelope `{result: {points: [...]}}` from the unified path, and the pre-merged bare list from the dual path.

| Property | Value |
|----------|-------|
| Node ID | 1773067376966 |
| Input: qdrant_results | Variable Aggregator `.output` (string) |
| Input: rewritten_query | Query Rewriter `.text` (string) |
| Output: reranker_documents | JSON array of passage text strings |
| Output: candidate_docs | JSON array of full document dicts with metadata |
| Output: query_for_reranker | The rewritten query string |

**Code:**
```python
import json

def main(qdrant_results: str, rewritten_query: str) -> dict:
    if not qdrant_results or not qdrant_results.strip():
        return {
            "reranker_documents": "[]",
            "candidate_docs": "[]",
            "query_for_reranker": rewritten_query
        }

    data = json.loads(qdrant_results)

    if isinstance(data, list):
        results = data                                        # dual path: bare list
    elif isinstance(data, dict):
        results = data.get("result", {}).get("points", [])   # unified path: wrapped
    else:
        results = []

    documents = []
    for r in results:
        payload = r.get("payload", {})
        text = payload.get("text", "")
        if not text:
            continue
        documents.append({
            "id": str(r.get("id", "")),
            "text": text,
            "source": payload.get("source", "unknown"),
            "category": payload.get("category", ""),
            "qdrant_score": r.get("score", 0)
        })

    return {
        "reranker_documents": json.dumps([d["text"] for d in documents]),
        "candidate_docs": json.dumps(documents),
        "query_for_reranker": rewritten_query
    }
```

---

#### Node 14 — Jina Reranker (HTTP Request)

| Property | Value |
|----------|-------|
| Node ID | 1773067852821 |
| URL | https://api.jina.ai/v1/rerank |
| Method | POST |
| Auth | `Authorization: Bearer {{#env.JINA_API_KEY#}}` |
| Model | jina-reranker-v2-base-multilingual |
| top_n | 5 |
| return_documents | true |
| Input: query | `{{#1773067376966.query_for_reranker#}}` |
| Input: documents | `{{#1773067376966.reranker_documents#}}` |
| Output | `.body` — reranked results with `relevance_score` per passage |

**Request body:**
```json
{
  "model": "jina-reranker-v2-base-multilingual",
  "query": "{{#1773067376966.query_for_reranker#}}",
  "documents": {{#1773067376966.reranker_documents#}},
  "top_n": 5,
  "return_documents": true
}
```

> **Purpose:** A cross-encoder reads query and document together, yielding significantly more accurate relevance judgements than the bi-encoder embedding similarity alone. Reduces up to 10 candidates to 5 highest-quality passages.

---

#### Node 15 — Format for LLM Context (Code)

Builds the numbered passage list and citation metadata. Handles Jina's `document` field being a plain string rather than a nested dict.

| Property | Value |
|----------|-------|
| Node ID | 1773068077928 |
| Input: reranker_response | Jina Reranker `.body` (string) |
| Input: candidate_docs | Result Mapper `.candidate_docs` (string) |
| Output: context_passages | Numbered, source-labelled passage block |
| Output: citations_metadata | JSON array of citation dicts with doc_id, score, snippet |

**Code:**
```python
import json

def main(reranker_response: str, candidate_docs: str) -> dict:
    reranker_data = json.loads(reranker_response)
    candidates = json.loads(candidate_docs)

    id_lookup = {d["text"]: d for d in candidates}
    results = reranker_data.get("results", [])

    passages = []
    citation_map = []

    for i, r in enumerate(results, 1):
        doc = r.get("document", "")
        text = doc if isinstance(doc, str) else doc.get("text", "")  # handle both string and dict
        relevance_score = r.get("relevance_score", 0)
        original = id_lookup.get(text, {})
        source = original.get("source", "unknown")

        passages.append(f"[{i}] (source: {source})\n{text}")
        citation_map.append({
            "citation_num": i,
            "doc_id": original.get("id", "unknown"),
            "relevance_score": round(relevance_score, 4),
            "snippet": text[:120] + "..." if len(text) > 120 else text
        })

    return {
        "context_passages": "\n\n".join(passages),
        "citations_metadata": json.dumps(citation_map, indent=2)
    }
```

---

#### Node 16 — Answer Synthesis (LLM)

| Property | Value |
|----------|-------|
| Node ID | 1773068310795 |
| Model | Qwen3-32B via Groq |
| Temperature | 0.1 |
| Context inputs | `context_passages` + `citations_metadata` from Format node |
| Query inputs | `sys.query` (original) + `{{#1772973692102.text#}}` (rewritten) |

**System prompt:**
```
/no_think
You are a precise, helpful research assistant. Answer the user's question
using ONLY the provided context passages.

Rules:
- Use inline citations like [1], [2], etc. matching the passage numbers in the context.
- If multiple passages support a claim, cite all of them: [1][3].
- Do NOT fabricate information not present in the passages.
- If the context is insufficient, say so explicitly.
- Quote passages as closely as possible rather than paraphrasing.
- If a claim cannot be traced to a specific passage word-for-word, omit it.
- At the end, include a "**Sources**" section listing each cited passage number and its source filename.
- Do NOT show doc_ids.
- Be concise but complete.
```

**User prompt:**
```
Context passages:
{{#1773068077928.context_passages#}}

Citation metadata (use these doc_ids and sources in your Sources section):
{{#1773068077928.citations_metadata#}}

User question: {{#sys.query#}}

Rewritten/expanded query used for retrieval: {{#1772973692102.text#}}

Answer with inline citations:
```

---

#### Node 17 — Final Answer (Answer node)

```
{{#1773068310795.text#}}
```

Streams the Answer Synthesis LLM output directly to the chat interface.

---

## 4. Key Engineering Decisions

### 4.1 Dense-Only Search (Why Hybrid Was Abandoned)

The system was originally designed with full hybrid search — dense + sparse BM25 via Qdrant's Query API fusion using the `yaxuanm/qdrant` Dify plugin v0.0.1. This was abandoned after discovering that the plugin silently ignores the `using_dense` and `using_sparse` named-vector parameters and sends requests to Qdrant's legacy search endpoint with an **unnamed** vector field — which fails on collections that only have named vectors (HTTP 400: "Not existing vector name error").

Switching to direct HTTP Request nodes calling `/points/query` with an explicit `"using": "dense"` field resolved this completely and gave full control over the request. Dense semantic search is sufficient for the corpus size (751 points).

### 4.2 Vector as JSON String (500-Element Array Limit Workaround)

Dify's code node output system enforces a maximum of 500 elements on `array[number]` type outputs. Jina Embeddings v3 produces 1024-dimensional vectors. The solution: serialise the embedding as a JSON string using `json.dumps()` in the Extract Vector node, and inject it **unquoted** into the Qdrant HTTP body template. Qdrant's JSON parser receives a valid float array at the `query` field.

### 4.3 Intent Router on Raw Query (Not Rewritten Query)

The Intent Router reads `sys.query` (original user input) rather than the Query Rewriter output. This prevents false dual-search triggers: if a user asks "how does Databridge handle databases?", the rewriter might add "ExecuteSQL QueryDatabaseTable" as related terms, causing CamelCase pattern detection to fire incorrectly and route to dual search when a unified search is more appropriate.

### 4.4 Qwen3-32B with /no_think

Qwen3-32B has a built-in extended reasoning mode that produces verbose `<think>...</think>` blocks before generating its answer. For a RAG synthesis task, this reasoning overhead is unnecessary and would pollute the Final Answer output. The `/no_think` directive at the start of the system prompt disables this mode, making responses concise and direct while retaining the model's instruction-following quality.

### 4.5 Reranker as Quality Gate

Rather than passing all retrieved documents directly to the LLM, the pipeline uses Jina Reranker v2 as a cross-encoder quality gate. A bi-encoder (the embedding model) scores documents independently of the query; a cross-encoder reads query and document together, yielding significantly more accurate relevance judgements. The reranker reduces up to 10 candidates to 5, keeping the LLM context concise and prioritising the most relevant passages.

### 4.6 Conversation Memory via LLM Memory Toggle

Early versions attempted to inject conversation history using a `sys.dialogue_turns` variable — which does not exist in Dify. The correct approach is to enable the **Memory toggle** directly on the LLM node with window size 5. Dify automatically prepends the last 5 conversation turns as user/assistant message pairs before the current prompt, requiring no variable injection or prompt engineering.

### 4.7 Two-LLM Architecture

Using two different models for two different tasks:
- **llama-3.1-8b-instant** for Query Rewriting — fast, cheap, and more than capable for simple query expansion
- **Qwen3-32B** for Answer Synthesis — large, accurate, strong instruction-following needed for grounded citation

This avoids paying the latency cost of a large model where it isn't needed.

---

## 5. Bugs Found and Resolved

All 12 bugs encountered across 5 YAML iterations and multiple runtime debugging cycles:

| # | Bug | Fix |
|---|-----|-----|
| 1 | `sys.dialogue_turns` variable does not exist in Dify | Enable Memory toggle on LLM node; remove variable from prompt |
| 2 | `yaxuanm/qdrant` plugin ignores `using_dense`/`using_sparse`; sends unnamed vector; Qdrant returns HTTP 400 "Not existing vector name error" | Replace plugin tool nodes with HTTP Request nodes calling `/points/query` directly with `"using": "dense"` |
| 3 | Jina Embeddings URL left as placeholder text "Add Jina Embeddings" | Set to `https://api.jina.ai/v1/embeddings` |
| 4 | All 3 Qdrant nodes used `method: get`; GET cannot carry a request body | Change all to `method: post` |
| 5 | `QDRANT_POINTS_URL` env var missing `/collections/` path segment | Add `/collections/`: `.../6333/collections/hxgn_knowledge_base/points/query` |
| 6 | Result Mapper input `value_type` was `array[object]` leftover from plugin era; Variable Aggregator outputs `string` | Change `value_type` to `string` |
| 7 | Extract Vector outputs 1024-element array; Dify caps code node arrays at 500 elements — runtime error | Return `json.dumps(embedding)` as string; inject unquoted in Qdrant body |
| 8 | Qdrant response envelope is `{result: {points: [...]}}` but code did `data.get("result", [])` returning a dict not a list | Fix to `data.get("result", {}).get("points", [])` |
| 9 | Jina Reranker returns `document` as a plain string; Format for LLM Context called `.get("text")` on it — `AttributeError: 'str' object has no attribute 'get'` | Add isinstance check: `text = doc if isinstance(doc, str) else doc.get("text", "")` |
| 10 | Score threshold of 0.15 in Result Mapper filtered all results — RRF scores are ~0.01–0.05 and never reach 0.15; Jina received empty documents array | Remove threshold entirely; quality controlled by `limit` on Qdrant nodes and `top_n` on Reranker |
| 11 | Merge Results returned `json.dumps(combined)` but output declared as `array[object]` — type mismatch | Return bare list or use consistent `string` output type with `json.dumps` |
| 12 | Jina Reranker body had unquoted query: `"query": {{var}}` — produces invalid JSON when query contains special characters | Wrap in quotes: `"query": "{{#var#}}"` |

---

## 6. Environment Variables & Security

| Variable | Used In | Purpose |
|----------|---------|---------|
| `JINA_API_KEY` | Jina Embeddings + Jina Reranker HTTP nodes | `Authorization: Bearer` header |
| `QDRANT_API_KEY` | All 3 Qdrant search HTTP nodes | `api-key` header |
| `QDRANT_POINTS_URL` | All 3 Qdrant search HTTP nodes | Full `/points/query` URL to avoid repetition |

All keys stored as Dify environment variables (masked as `*****`) and referenced via `{{#env.KEY_NAME#}}`. Never hardcoded in node bodies or YAML exports. Rotate via provider dashboards; update only the Dify environment variable.

---

## 7. Performance & Quality

### 7.1 Retrieval Quality

| Metric | Value |
|--------|-------|
| Embedding model | Jina Embeddings v3 — multilingual, 1024-dim, task=retrieval.query |
| Cosine similarity range observed | 0.41–0.52 on relevant queries |
| Reranker effect | Cross-encoder rescores and reorders — top cosine result not always top reranked result |
| Context window to LLM | 5 passages maximum after reranking |

### 7.2 Routing Accuracy

| Route | When triggered |
|-------|---------------|
| Unified | Conceptual, how-to, cross-dataset questions — searches all 751 points |
| Dual | Named processor/service with CamelCase, "properties", "configure" keywords — filters to relevant dataset |
| Known behaviour | "Give me 2 processors that support SQL" routes to unified — correct, since the user wants results across all data |

### 7.3 Answer Quality

| Mechanism | Detail |
|-----------|--------|
| Hallucination mitigation | `/no_think` on Qwen3-32B + "omit if not traceable word-for-word" instruction |
| Citation faithfulness | Source filenames in passage headers; LLM instructed to use filenames not doc_ids |
| Query rewriter safety | Small model (8B) for rewriting — simple task, no hallucination risk |
| Synthesis quality | Qwen3-32B — strong instruction following for grounded RAG, better citation discipline than smaller models |

---

## 8. Skills Demonstrated

### AI / ML Engineering
- Designed and implemented a multi-stage RAG pipeline from scratch on a no-code/low-code platform
- Applied bi-encoder + cross-encoder hybrid retrieval pattern (embedding for recall, reranker for precision)
- Built intent-based query routing using regex classification with deliberate fallback logic
- Implemented query expansion using LLM rewriting for improved retrieval recall
- Debugged and resolved vector dimension mismatch, API response schema issues, and score calibration problems
- Selected and configured models for specific sub-tasks: fast small model for rewriting, large accurate model for synthesis

### API Integration & Systems
- Integrated four external APIs (Groq, Jina Embeddings, Jina Reranker, Qdrant Cloud) all via HTTP with full control over request/response handling
- Diagnosed and worked around a Dify platform-level constraint (500-element array cap) with a serialisation workaround
- Built a vector database search pipeline using Qdrant's REST API with named vectors, payload filters, and response envelope unwrapping
- Managed authentication, error handling, and retry configuration across all HTTP nodes

### Data Engineering
- Designed a dual-dataset Qdrant collection schema with named dense and sparse vectors
- Defined payload field indexing strategy for efficient filter-based retrieval
- Chunked and ingested 751 documents across two distinct corpus types with different metadata schemas

### Prompt Engineering
- Crafted query expansion prompts that prevent over-engineering the rewritten query
- Designed grounding-first system prompts with explicit hallucination prevention rules
- Used `/no_think` directive to control model reasoning mode output
- Built citation instructions that produce source-attributed, verifiable answers with filename references

### Debugging & Problem Solving
- Traced 12 distinct bugs across 5 YAML iterations through systematic log and error analysis
- Identified a Dify plugin implementation bug by reading raw Qdrant HTTP error responses
- Resolved data type mismatches between nodes through careful input/output schema tracing
- Diagnosed empty results caused by RRF score range miscalibration (scores ~0.01–0.05, not 0–1 as assumed)

---

## 9. Potential Future Improvements

| Improvement | Description | Complexity |
|-------------|-------------|------------|
| Hybrid Search | Re-enable BM25 sparse search by adding sparse vector computation alongside dense embedding, then fusing with Qdrant RRF via `/points/query` prefetch API | Medium |
| Streaming Citations | Surface passage sources as inline UI elements using Dify's `retriever_resource` feature — requires migrating to Dify Knowledge Base nodes | High |
| Dynamic Routing | Extend Intent Router with an LLM-based classifier to handle ambiguous queries beyond current regex patterns | Medium |
| Metadata Filtering UI | Allow users to specify dataset scope via conversation variable toggles ("search only processors") | Low |
| Answer Confidence | If top reranker score is below threshold, respond with explicit "insufficient information" rather than low-quality answer | Low |
| Ingestion Pipeline | Automated document chunking and ingestion pipeline to keep knowledge base current | Medium |
| Multi-collection | Extend to multiple Qdrant collections for different product versions or customer environments | Medium |

---

## 10. Quick Reference

### 10.1 Example Queries by Route

| Route | Example Query |
|-------|--------------|
| Unified (conceptual) | "How does Databridge Pro handle database connections?" |
| Unified (cross-dataset) | "Give me 2 processors that support SQL execution" |
| Unified (how-to) | "How do I configure EAM for Databridge Pro?" |
| Dual (processor name) | "What are the properties of QueryDatabaseTable?" |
| Dual (configure) | "How do I configure DBCPConnectionPool?" |
| Dual (what does X do) | "What does ExecuteSQLRecord do?" |
| Dual (controller service) | "What controller service does ExecuteSQL need?" |

### 10.2 All Node IDs

| Node ID | Title | Type |
|---------|-------|------|
| 1772970532640 | User Input | Start |
| 1772973692102 | Query Rewriter | LLM (llama-3.1-8b-instant) |
| 1773157495040 | Jina Embeddings | HTTP Request |
| 1773158737461 | Extract Vector | Code |
| 1773065598617 | Expanded Query Display | Answer |
| 1773077552800 | Intent Router | Code |
| 1773077675139 | IF/ELSE | Condition |
| 1773185085153 | UNIFIED DENSE SEARCH | HTTP Request |
| 17731854106500 | TECH DENSE SEARCH | HTTP Request |
| 17731854155860 | PROC DENSE SEARCH | HTTP Request |
| 1773078688584 | Merge Results | Code |
| 1773078889010 | Variable Aggregator | Aggregator |
| 1773067376966 | Qdrant Result Mapper | Code |
| 1773067852821 | Jina Reranker | HTTP Request |
| 1773068077928 | Format for LLM Context | Code |
| 1773068310795 | Answer Synthesis | LLM (Qwen3-32B) |
| 1773068440860 | Final Answer | Answer |

### 10.3 Qdrant Collection Details

| Property | Value |
|----------|-------|
| Cluster URL | `https://d79d8acd-5f80-4b94-8e04-b8855744ff41.europe-west3-0.gcp.cloud.qdrant.io:6333` |
| Collection | `hxgn_knowledge_base` |
| Points/query endpoint | `.../collections/hxgn_knowledge_base/points/query` |
| Named dense vector | `dense` — 1024-dim, cosine, Jina Embeddings v3 |
| Named sparse vector | `sparse` — BM25 (ingested, not active in current search) |
| Dataset 1 filter | `{"must":[{"key":"dataset","match":{"value":"technical_docs"}}]}` |
| Dataset 2 filter | `{"must":[{"key":"dataset","match":{"value":"processors_services"}}]}` |

### 10.4 API Endpoints Summary

| Service | Endpoint | Auth Header |
|---------|----------|-------------|
| Jina Embeddings | `POST https://api.jina.ai/v1/embeddings` | `Authorization: Bearer {{JINA_API_KEY}}` |
| Jina Reranker | `POST https://api.jina.ai/v1/rerank` | `Authorization: Bearer {{JINA_API_KEY}}` |
| Qdrant Search | `POST .../collections/hxgn_knowledge_base/points/query` | `api-key: {{QDRANT_API_KEY}}` |
| Groq (via Dify) | Managed by Dify Groq plugin | Configured in Dify Model Providers |

### 10.5 Dify Platform Constraints Encountered

| Constraint | Impact | Workaround |
|------------|--------|------------|
| Code node array output capped at 500 elements | Cannot output 1024-dim float array directly | Serialise as JSON string with `json.dumps()` |
| `sys.dialogue_turns` does not exist | Cannot inject conversation history via variable | Enable Memory toggle on LLM node |
| Plugin tool nodes may not expose full API parameters | `yaxuanm/qdrant` plugin silently ignored named vector fields | Replace with HTTP Request nodes for full control |
| Citations & Attributions widget requires native Knowledge Base nodes | Cannot use built-in citation UI with external Qdrant | Implement manual citation formatting in code node |
