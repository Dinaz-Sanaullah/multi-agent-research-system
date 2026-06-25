# Multi-Agent Research Intelligence System
## Project Report

**Author:** Dinaz Sanaullah  
**GitHub:** https://github.com/Dinaz-Sanaullah/multi-agent-research-system  
**GCP Project:** multi-agent-research-system  
**Deployment URL:** https://research-agent-chfrwn7sia-uc.a.run.app/  
**Date:** June 2025

---

## 1. Executive Summary

This project implements a **Multi-Agent Research Intelligence System** using Google's **Agent Development Kit (ADK)**. The system acts as an academic research assistant that combines **Retrieval-Augmented Generation (RAG)** over a local knowledge base of scholarly PDFs with **real-time web search** via the Tavily API. Seven specialized agents collaborate through four distinct communication patterns—sequential flow, parallel execution, hierarchical delegation, and feedback loops—to produce well-cited, quality-reviewed research responses.

The solution is deployed to **Google Cloud Run** with observability, a Streamlit UI for local use, and an ADK web interface for production interaction.

---

## 2. Project Objectives

| # | Objective | Status |
|---|-----------|--------|
| 1 | Design and implement a multi-agent system using Google ADK | ✅ Complete |
| 2 | Integrate RAG for knowledge management | ✅ Complete |
| 3 | Implement web search for real-time information | ✅ Complete |
| 4 | Build custom tools demonstrating agent composition | ✅ Complete |
| 5 | Add observability/monitoring for production insights | ✅ Complete |
| 6 | Build Streamlit UI | ✅ Complete |
| 7 | Deploy to Google Cloud Platform (Cloud Run) | ✅ Complete |

**Use Case:** Research Assistant — primary knowledge base of academic papers and journals; web search for latest citations and recent developments.

---

## 3. System Architecture

### 3.1 High-Level Overview

```
User Query
    │
    ▼
┌─────────────────┐
│  Orchestrator   │  ← Hierarchical Delegation (root agent)
└────────┬────────┘
         ▼
┌─────────────────┐
│ Research Pipeline│  ← Sequential Flow
│  ┌─────────────┐│
│  │   Planner   ││
│  └──────┬──────┘│
│         ▼       │
│  ┌─────────────┐│
│  │ RAG Research││  ← Sequential or Parallel Execution
│  │ Web Research││
│  └──────┬──────┘│
│         ▼       │
│  ┌─────────────┐│
│  │ Quality Loop││  ← Feedback Loop (Synthesize→Review→Refine)
│  └─────────────┘│
└─────────────────┘
         │
         ▼
   Final Response
```

### 3.2 Technology Stack

| Layer | Technology |
|-------|------------|
| Agent Framework | Google ADK 2.x (Python) |
| LLM | Gemini 2.5 Flash (Vertex AI) |
| Embeddings | Vertex AI text-embedding-005 |
| Vector Store | FAISS (IndexFlatIP, cosine similarity) |
| PDF Processing | PyPDF |
| Web Search | Tavily API |
| Observability | structlog, OpenTelemetry (optional Cloud Trace) |
| Local UI | Streamlit |
| Production UI | ADK Dev UI (`--with_ui`) |
| Deployment | Google Cloud Run, Cloud Build, Secret Manager |
| Container | Docker (Python 3.11) |

---

## 4. Multi-Agent System (40%)

### 4.1 Specialized Agents

| Agent | Role | Tools |
|-------|------|-------|
| **Orchestrator** | Root coordinator; delegates to pipeline; presents final answer | `get_system_metrics` |
| **Planner** | Classifies query type; creates research plan | `save_research_plan` |
| **RAG Researcher** | Searches academic knowledge base | `search_knowledge_base`, `search_web` |
| **Web Researcher** | Finds latest citations and news | `search_knowledge_base`, `search_web` |
| **Synthesizer** | Merges RAG + web findings into coherent response | `format_citations` |
| **Reviewer/QA** | Quality gate; approves or requests revision | `append_to_state`, `exit_loop` |
| **Refiner** | Revises synthesis based on reviewer feedback | `format_citations` |

### 4.2 Communication Patterns (2+ Required)

#### Pattern 1: Sequential Flow
`SequentialAgent` named `research_pipeline` executes sub-agents in order:
**Planner → Research Team → Quality Review Loop**

#### Pattern 2: Parallel / Sequential Execution
`ParallelAgent` or `SequentialAgent` for the research team:
- **Parallel mode** (`SEQUENTIAL_RESEARCH=false`): RAG and web researchers run concurrently (fan-out/gather)
- **Sequential mode** (`SEQUENTIAL_RESEARCH=true`): Reduces Vertex AI 429 quota pressure

#### Pattern 3: Hierarchical Delegation
The **Orchestrator** acts as parent agent with `research_pipeline` as a sub-agent. ADK's delegation mechanism transfers execution based on agent descriptions.

#### Pattern 4: Feedback Loop
`LoopAgent` named `quality_review_loop` runs:
**Synthesizer → Reviewer → Refiner** until the reviewer calls `exit_loop` or `max_iterations` is reached.

### 4.3 State Management

- **Session state** shared across agents via ADK `output_key` and `tool_context.state`
- Key state variables: `user_query`, `research_plan`, `rag_findings`, `web_findings`, `research_synthesis`, `review_feedback`
- State templating in instructions: `{user_query?}`, `{rag_findings?}`, etc.

---

## 5. RAG Support (25%)

### 5.1 Pipeline

```
PDF Files (data/papers/)
    │
    ▼
PDF Text Extraction (PyPDF)
    │
    ▼
Semantic Chunking (paragraph-aware, 600 chars, 80 overlap)
    │
    ▼
Vertex AI text-embedding-005 (batched, 8 chunks/request)
    │
    ▼
FAISS Vector Store (IndexFlatIP, L2-normalized)
    │
    ▼
Similarity Search (top-k retrieval)
```

### 5.2 Sample Knowledge Base

Three open-access arXiv papers were ingested:

| Paper | Chunks |
|-------|--------|
| Attention Is All You Need (Vaswani et al., 2017) | 15 |
| BERT (Devlin et al., 2018) | 16 |
| Language Models are Few-Shot Learners (Brown et al., 2020) | 73 |
| **Total** | **104 chunks** |

### 5.3 Retrieval Performance

RAG smoke tests returned relevant chunks with similarity scores of 0.71–0.75 for transformer, BERT, and few-shot learning queries.

---

## 6. Web Search Capabilities (15%)

### 6.1 Tavily API Integration

The `search_web` tool uses Tavily's advanced search with:
- Up to 5 results per query
- Included AI-generated answer summary
- Source URLs for citation

### 6.2 Intelligent Routing

`classify_query_source()` analyzes query signals:

| Signal Type | Keywords | Route |
|-------------|----------|-------|
| RAG | paper, journal, cite, methodology, abstract | Knowledge Base |
| Web | latest, recent, 2025, news, breaking | Tavily |
| Hybrid | Both signal types present | Both sources |

### 6.3 Agent Tool Strategy

Both researcher agents register both search tools to prevent `ValueError` when the LLM calls the wrong tool name. Instructions enforce role-specific tool usage.

---

## 7. Custom Tools & Agent Composition

| Tool | Purpose |
|------|---------|
| `search_knowledge_base` | FAISS retrieval over academic PDFs |
| `search_web` | Tavily real-time search |
| `save_research_plan` | Persists plan + classification to session state |
| `append_to_state` | Cross-agent feedback communication |
| `format_citations` | Merges paper and web citation sources |
| `get_system_metrics` | Returns observability metrics dashboard data |
| `exit_loop` | ADK built-in; terminates quality review loop |

---

## 8. Observability & Monitoring

### 8.1 Structured Logging
JSON logs via `structlog` capture:
- Agent events (`agent_event`)
- Tool calls with parameters (`tool_call`)
- Span completions with latency (`span_completed`)

### 8.2 In-Memory Metrics
`AgentMetrics` tracks:
- Total queries, RAG/web/hybrid counts
- Average latency (ms)
- Review loop iterations

### 8.3 Optional Cloud Trace
Enable via `ENABLE_CLOUD_TRACE=true` for distributed tracing on GCP.

---

## 9. User Interfaces

### 9.1 Streamlit UI (`streamlit_app/app.py`)
- Chat interface for research queries
- PDF upload and ingestion sidebar
- Live metrics dashboard
- Session state debugging

### 9.2 ADK Web UI (Production)
Deployed with `--with_ui` flag on Cloud Run:
- Interactive agent graph visualization
- Event inspection per agent
- State tab for session debugging

---

## 10. Cloud Deployment

### 10.1 Infrastructure

| Component | Configuration |
|-----------|---------------|
| Service | `research-agent` |
| Region | us-central1 |
| Memory | 2 GiB |
| CPU | 2 |
| Timeout | 300s |
| Secrets | `tavily-api-key` → `TAVILY_API_KEY` |

### 10.2 CI/CD
`cloudbuild.yaml` automates:
1. Docker image build
2. Push to GCR
3. Cloud Run deploy with env vars and secrets

### 10.3 Environment Variables
```
GOOGLE_CLOUD_PROJECT=multi-agent-research-system
GOOGLE_GENAI_USE_VERTEXAI=True
GEMINI_MODEL=gemini-2.5-flash
EMBEDDING_MODEL=text-embedding-005
SEQUENTIAL_RESEARCH=true
MAX_REVIEW_ITERATIONS=2
```

---

## 11. Testing & Results

### 11.1 RAG Smoke Test
```
Q: What is the self-attention mechanism in transformers?
→ [0.736] attention_is_all_you_need.pdf (chunk 1)

Q: How does BERT pre-training work?
→ [0.751] bert_pretraining.pdf (chunk 3)
```

### 11.2 Live Multi-Agent Test
```
Q: What is the transformer architecture and self-attention mechanism?
Routing: rag
Pipeline: planner → rag_researcher → synthesizer
Result: Comprehensive cited response from Attention paper (57s)
```

### 11.3 Web Search Test
```
Q: latest transformer research papers 2025
→ 3 Tavily results in ~6 seconds
```

---

## 12. Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| Vertex AI 429 RESOURCE_EXHAUSTED | Retry config (5 attempts, 2s delay); sequential research mode; reduced loop iterations |
| `gemini-2.0-flash` not available | Switched to `gemini-2.5-flash` |
| Embedding batch token limit | Batched embeddings (8 chunks per API call); smaller chunk size (600 chars) |
| Tool `search_knowledge_base` not found | Both tools registered on both researcher agents |
| Cloud Run 403 Forbidden | Granted `allUsers` run.invoker IAM role |
| Tavily key not in local `.env` | Documented Secret Manager for Cloud; `.env` for local dev |

---

## 13. Project Structure

```
multi-agent-research-system/
├── research_agent/           # ADK agent package
│   ├── agent.py              # Workflow composition
│   ├── agents/definitions.py # Agent definitions
│   ├── tools/custom_tools.py # RAG, web, state tools
│   ├── rag/                  # PDF → FAISS pipeline
│   ├── observability/        # Logging & metrics
│   └── model_config.py       # Gemini retry settings
├── streamlit_app/app.py      # Local UI
├── scripts/                  # ingest, test, deploy
├── data/papers/              # Academic PDFs
├── vector_store/             # FAISS index
├── Dockerfile                # Cloud Run container
└── cloudbuild.yaml           # CI/CD pipeline
```

---

## 14. Conclusion

The Multi-Agent Research Intelligence System successfully demonstrates a production-oriented research assistant using Google ADK. It fulfills all core requirements:

- **3+ specialized agents** with four communication patterns
- **RAG pipeline** with PDF extraction, semantic chunking, Vertex AI embeddings, and FAISS
- **Tavily web search** with intelligent query routing
- **Custom tools** for search, state management, and citation formatting
- **Observability** via structured logging and metrics
- **Streamlit UI** and **Cloud Run deployment**

Future enhancements could include: persistent session storage (Firestore), larger document corpora, agent evaluation benchmarks, and quota-aware dynamic parallel/sequential switching.

---

## 15. References

1. Google Agent Development Kit Documentation — https://google.github.io/adk-docs/
2. Vaswani et al., "Attention Is All You Need" (2017) — arXiv:1706.03762
3. Devlin et al., "BERT" (2018) — arXiv:1810.04805
4. Brown et al., "Language Models are Few-Shot Learners" (2020) — arXiv:2005.14165
5. Tavily API Documentation — https://docs.tavily.com/
6. Vertex AI Gemini Models — https://cloud.google.com/vertex-ai/generative-ai/docs/models
7. FAISS: Facebook AI Similarity Search — https://github.com/facebookresearch/faiss

---

*End of Report*
