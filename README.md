# 💬 Talk to DB PRO

> **Ask a question in plain English. Get a cited, grounded answer from technical documentation — instantly.**

[![Live Demo](https://img.shields.io/badge/🚀%20Live%20Demo-Try%20it%20now-black?style=for-the-badge)](https://udify.app/chat/1FtFMvw4odmYZ0KO)
[![GitHub Pages](https://img.shields.io/badge/📄%20Project%20Page-GitHub%20Pages-222?style=for-the-badge)](https://LifeLongLaugh.github.io/talk-to-db-pro)

![Dify](https://img.shields.io/badge/Dify%20Cloud-Chatflow-blue?style=flat-square)
![Qdrant](https://img.shields.io/badge/Qdrant-Vector%20DB-red?style=flat-square)
![Groq](https://img.shields.io/badge/Groq-Qwen3--32B%20%2B%20Llama%203.1-orange?style=flat-square)
![Jina](https://img.shields.io/badge/Jina%20AI-Embeddings%20%2B%20Reranker-purple?style=flat-square)
![Cost](https://img.shields.io/badge/Infrastructure%20cost-%240%2Fmonth-brightgreen?style=flat-square)
![Vectors](https://img.shields.io/badge/Knowledge%20base-751%20vectors-teal?style=flat-square)

---

## 🚀 Try it live

**[→ Open the chatbot](https://udify.app/chat/1FtFMvw4odmYZ0KO)**

Try asking:
- *"What does ExecuteSQLRecord do?"*
- *"How does Databridge Pro handle database connections?"*
- *"Give me 2 processors that support SQL execution"*
- *"What controller service does ExecuteSQL need?"*
- *"How do I configure DBCPConnectionPool?"*

---

## 📋 Contents

1. [What is this?](#what-is-this)
2. [What is RAG — and why does it beat regular AI?](#-what-is-rag--and-why-does-it-beat-regular-ai)
3. [How it works](#%EF%B8%8F-how-it-works)
4. [Technology stack](#-technology-stack)
5. [Key engineering decisions](#%EF%B8%8F-key-engineering-decisions)
6. [Import this project yourself](#-import-this-project-yourself)
7. [Repository structure](#-repository-structure)

---

## What is this?

**Talk to DB PRO** is a production-grade conversational RAG (Retrieval-Augmented Generation) system. It turns a large collection of dry technical documentation into an AI assistant you can have a natural conversation with.

It covers two corpora:

- **HxGN Databridge Pro** — a data integration platform used in enterprise asset management. Documentation spans multiple markdown files previously only searchable by keyword.
- **Apache NiFi processors & controller services** — 200+ data flow components, each with complex configuration properties that engineers frequently need to look up.

Instead of manually ctrl+F-ing through PDFs and markdown files, engineers can now ask questions in plain English and receive precise, cited answers in seconds — with conversation memory across follow-up questions.

**Built entirely on free tiers. Infrastructure cost: $0/month.**

---

## 🤔 What is RAG — and why does it beat regular AI?

*This section is for non-technical readers. If you already know RAG, [skip ahead](#%EF%B8%8F-how-it-works).*

### The problem with regular AI

When you ask a standard AI assistant a question, it answers from its **training data** — a fixed snapshot of the internet frozen at some point in the past. This creates two serious problems:

**Problem 1 — It doesn't know your documents.**
Your internal technical docs, your company's specific software, your product's configuration guide — none of that was in the training data. The AI will either say "I don't know", or worse, **make something up that sounds plausible but is wrong**. This is called *hallucination*, and it happens constantly with domain-specific questions.

**Problem 2 — You can't verify the answer.**
Even when a regular AI sounds confident, there's no way to know where the information came from or whether it applies to your specific version of the software.

### The solution: RAG (Retrieval-Augmented Generation)

RAG solves both problems by giving the AI a library to look things up in *before* it answers — rather than relying on memory.

The analogy: imagine the difference between asking a colleague to recall something from memory versus asking them to look it up in the actual manual before answering. RAG is the second approach.

```
Your question
      ↓
Search your documentation for the most relevant passages
      ↓
Hand those passages to the AI as its reading material
      ↓
AI generates an answer based ONLY on what it just read
      ↓
Answer includes citations — every claim is verifiable
```

### Why RAG wins

| | Regular AI | RAG (this project) |
|---|---|---|
| Knows your internal documents | ❌ No | ✅ Yes |
| Answers from your own content | ❌ No | ✅ Yes |
| Risk of hallucination | ⚠️ High | ✅ Heavily mitigated |
| Cites sources | ❌ No | ✅ Yes — with filenames |
| Works with private/internal data | ❌ No | ✅ Yes |
| Stays current as docs change | ❌ No | ✅ Re-ingest to update |
| Understands follow-up questions | ⚠️ Sometimes | ✅ Conversation memory |

Talk to DB PRO is a **production-grade RAG pipeline** — not a basic demo. It uses query expansion, semantic vector search, neural reranking, and a grounding-first synthesis prompt to produce answers that can be traced back to source documents.

---

## 🏗️ How it works

### Interactive diagrams

| Diagram | Audience |
|---|---|
| [**Pipeline Diagram**](https://LifeLongLaugh.github.io/talk-to-db-pro/Docs/Pipeline_Diagram.gif) | Technical — every node, model, and API call |

### The 8 stages

**1. You ask a question** — in plain English, typos and all.

**2. Query Rewriter** — a lightweight AI (Llama 3.1 8B) expands and clarifies your question to maximise search recall. `"ExecSQL props"` becomes `"ExecuteSQL NiFi processor properties configuration record writer"`. You see this rewrite in real time as the system thinks.

**3. Semantic Embedding** — your expanded question is converted to a 1024-dimensional vector by Jina Embeddings v3. This vector represents the *meaning* of your question mathematically — not just its keywords — enabling similarity matching even when your wording differs from the documentation.

**4. Intent Routing** — the system classifies your *original* query (not the rewritten one) to choose the best search strategy. Named processors with CamelCase → **dual path** (search filtered by category). Conceptual or how-to questions → **unified path** (all vectors without filter).

**5. Vector Search** — Qdrant Cloud finds the passages whose meaning is most similar to your question vector, using cosine similarity. Up to 10 candidates are retrieved.

**6. Neural Reranking** — Jina Reranker v2 (a cross-encoder) reads your question and each passage *together* and rescores them with much higher accuracy than the initial similarity pass. The top 5 passages survive.

**7. Grounded Synthesis** — Qwen3-32B generates an answer using *only* the 5 retrieved passages as its source material. It is explicitly instructed to cite passage numbers inline and omit anything not traceable word-for-word.

**8. Cited answer** — you receive a response with inline citations `[1][2]` and a **Sources** section listing the originating filenames.

### Full pipeline (technical view)

```
User Input
  └─► Query Rewriter LLM           [Llama 3.1 8B · Groq · temp=0]
        └─► Jina Embeddings         [POST api.jina.ai/v1/embeddings · 1024-dim]
              └─► Extract Vector    [Code: JSON → json.dumps(float[1024])]
                    └─► Intent Router  [Code: regex on sys.query]
                          └─► IF/ELSE  [search_mode == 'unified'?]
                                │
                                ├─ TRUE  ─► Qdrant Unified Search   [limit=10, no filter]
                                │              └─► Variable Aggregator
                                │
                                └─ FALSE ─► Qdrant Tech Search  [limit=5, dataset=technical_docs]
                                          ─► Qdrant Proc Search [limit=5, dataset=processors_services]
                                               └─► Merge & Sort  [combined, sorted by score]
                                                     └─► Variable Aggregator
                                                           └─► Result Mapper  [normalise + extract text]
                                                                 └─► Jina Reranker v2  [top_n=5]
                                                                       └─► Format Context  [numbered passages]
                                                                             └─► Answer Synthesis LLM  [Qwen3-32B · /no_think]
                                                                                   └─► Final Answer
```

---

## 🧰 Technology stack

| Layer | Technology | Detail |
|-------|-----------|--------|
| Orchestration | Dify Cloud (Chatflow v0.6.0) | Visual workflow builder, runtime, API gateway |
| LLM — Rewriter | Groq + Llama 3.1 8B Instant | `temp=0` · query expansion and clarification |
| LLM — Synthesis | Groq + Qwen3-32B | `temp=0.1` · `/no_think` · grounded citation synthesis |
| Embeddings | Jina Embeddings v3 | 1024-dim, cosine, `task=retrieval.query` |
| Reranker | Jina Reranker v2 | `jina-reranker-v2-base-multilingual` · `top_n=5` |
| Vector Database | Qdrant Cloud | GCP europe-west3 · named dense + sparse vectors |
| HTTP | Dify HTTP Request nodes | Full control over all external API calls |

### Knowledge base

| Dataset | Points | Content |
|---------|--------|---------|
| `technical_docs` | 539 | HxGN Databridge Pro documentation (chunked markdown) |
| `processors_services` | 212 | NiFi processor & controller service definitions |
| **Total** | **751** | Named vectors: `dense` (1024-dim, cosine) + `sparse` (BM25, ingested) |

---

## ⚙️ Key engineering decisions

**Dual-path intent routing on raw query** — the Intent Router reads `sys.query` (original user input), not the rewritten query. This prevents false dual-path triggers: if the rewriter adds `"ExecuteSQL"` as a synonym to a conceptual question, the router would incorrectly fire the processor-specific path. Raw query routing eliminates this class of error.

**Vector as JSON string (500-element workaround)** — Dify code nodes cap array outputs at 500 elements. Jina v3 produces 1024-dimensional vectors. Solution: `json.dumps(embedding)` serialises it as a string, which is then injected *unquoted* into the Qdrant HTTP request body. Qdrant's JSON parser receives a valid float array at the `query` field.

**HTTP nodes over the Qdrant plugin** — the `yaxuanm/qdrant` Dify plugin silently ignores named vector parameters and calls Qdrant's legacy endpoint, returning HTTP 400 on named-vector collections. Raw HTTP Request nodes calling `/points/query` with `"using": "dense"` give complete control.

**Bi-encoder + cross-encoder hybrid retrieval** — embedding similarity (bi-encoder) is fast but approximate; the reranker (cross-encoder) reads query and document *together* for significantly higher precision. Two stages: recall then precision — the same pattern used in production search engines.

**`/no_think` directive on Qwen3-32B** — Qwen3's extended reasoning mode produces verbose `<think>` blocks. For a RAG synthesis task this overhead is unnecessary and pollutes the user-facing output. The `/no_think` directive at the top of the system prompt disables this mode cleanly.

**Two-LLM architecture** — Llama 3.1 8B handles query rewriting (fast, cheap, the task is simple); Qwen3-32B handles synthesis (large model needed for citation-accurate grounding). Avoids paying latency cost of a 32B model where an 8B is sufficient.

**Grounding-first synthesis prompt** — the synthesis prompt instructs Qwen3-32B: *"If a claim cannot be traced to a specific passage word-for-word, omit it."* Combined with `/no_think`, this significantly reduces hallucination compared to standard prompting.

---

## 📥 Import this project yourself

Want to run your own version of this pipeline over *your* documentation?

**[→ Full import guide: IMPORT_GUIDE.md](./IMPORT_GUIDE.md)**

Quick overview:
1. Sign up for Dify Cloud, Qdrant Cloud, Groq, and Jina AI — all free
2. Create a Qdrant collection with named dense vectors (1024-dim, cosine)
3. Ingest your documents with the payload schema described in the guide
4. Import [`dify-workflow/talk-to-db-pro.yml`](./dify-workflow/talk-to-db-pro.yml) into Dify
5. Set three environment variables: `JINA_API_KEY`, `QDRANT_API_KEY`, `QDRANT_POINTS_URL`
6. Publish and chat

---

## 📁 Repository structure

```
talk-to-db-pro/
├── README.md                        ← You are here
├── IMPORT_GUIDE.md                  ← Step-by-step: run this yourself
├── docs/
│   ├── index.html                   ← GitHub Pages landing page
│   ├── pipeline-diagram.html        ← Animated node-wise technical diagram
│   └── logical-flow.html            ← Animated logical flow diagram
├── dify-workflow/
│   └── talk-to-db-pro.yml           ← Dify DSL export (API keys removed)
└── report/
    └── Talk_to_DB_PRO_Report.md     ← Full technical project report
```

---

## 📄 Full technical report

The complete project report — every node specification, all engineering decisions, 12 bugs found and resolved, performance analysis, and future improvements — is at [`report/Talk_to_DB_PRO_Report.md`](./report/Talk_to_DB_PRO_Report.md).

---

## 🙋 Author

Built by [@LifeLongLaugh](https://github.com/LifeLongLaugh).

AI Agents used: Claude, Gemini, Copilot.
