# Agentic RAG
**`001. Agentic Router.ipynb`**

The core of this module. This notebook introduces **agentic decision-making** as the first step in a RAG pipeline — the system thinks before it retrieves.

## How it works

```
                        User Query
                            │
                            ▼
              ┌─────────────────────────┐
              │  Router LLM (GPT-5.6)   │
              │      route_query()      │
              └────────────┬────────────┘
                           │
         ┌─────────────────┼──────────────────┐
         ▼                 ▼                  ▼
  OPENAI_QUERY     10K_DOCUMENT_QUERY    INTERNET_QUERY
         │                 │                  │
         ▼                 ▼                  ▼
  Qdrant search     Qdrant search        SerpApi
  (opnai_data)      (10k_data)          (live web)
         │                 │                  │
         └────────┬─────────┘                  │
                  ▼                            │
         RAG Response Generator                │
         rag_formatted_response()              │
                  │                            │
                  └──────────────┬─────────────┘
                                 ▼
                          Final Response
```

## Key components

| Function | Role |
|---|---|
| `route_query()` | Calls GPT-5.6-Luna with a router prompt; returns `action`, `reason`, and a short `answer` as JSON |
| `get_text_embeddings()` | Converts a query string to a 768-dim Nomic vector |
| `retrieve_and_response()` | Async function — queries Qdrant (top-3 chunks) then calls the RAG generator |
| `rag_formatted_response()` | Passes retrieved context to GPT-5.6-Luna and asks it to answer with inline citations |
| `get_internet_content()` | Calls the SerpApi Google Search API for real-time answers |
| `agentic_rag()` | Main orchestrator — ties routing, retrieval, and generation together |
| `secure_agentic_rag()` | Section 6 — the same loop with an RBAC check between routing and retrieval |

## Data sources
- **OpenAI documentation** — Agents, tools, chat completions, best practices
- **10-K SEC filings** — Lyft FY2020, FY2021, FY2022 and Uber FY2021
- **Live internet** — Any query outside the above two domains via SerpApi

## Role-Based Access Control

The last section of notebook adds an RBAC layer on top of the router. Two roles (`engineer`,
`finance_analyst`) are mapped to the route labels each may reach, and the check sits
between the router's decision and the tool call — so an unauthorized request is rejected
before anything is embedded, searched, or grounded.

| Knowledge source | Route label | `engineer` | `finance_analyst` |
|---|---|---|---|
| OpenAI documentation | `OPENAI_QUERY` | ✅ | ✅ |
| 10-K filings | `10K_DOCUMENT_QUERY` | ❌ | ✅ |
| Live internet search | `INTERNET_QUERY` | ✅ | ❌ |

File-level RBAC — gating individual documents rather than whole sources — is covered in
`003. Agentic Router_semantic_caching_rbac.ipynb`.

## Assignment

**Required — sub-query division.** Split compound questions (e.g. *"What was Uber's and Lyft's revenue in 2021?"*) into individual sub-queries, route each one independently (they may land on different sources), and synthesise a single composed answer with citations preserved.

**Bonus (optional, ungraded) — RBAC with a semantic cache.** Put a cache behind the Section 6 access gate without leaking across roles: a cache keyed only on the question will happily serve a finance answer to an engineer who was just denied. Learners build a role-partitioned (or permission-tagged) cache and pass a self-check that includes an explicit leak test. Good warm-up for **ARGUS**, where caching and cost-per-source reporting are first-class requirements.
