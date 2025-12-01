# KG‑RAG ENGINE SPECIFICATION
### A domain-agnostic modular KG-RAG engine

---

## 0) Purpose & Outcomes

Build a **domain‑agnostic KG‑RAG engine** that cleanly separates:
1) **GraphEngine (core)** — generic nodes/edges/traversal/validation (no domain semantics)  
2) **KG‑RAG Engine** — hybrid retrieval: KG query expansion + retrieval routing + consistency checks  
3) **Wrappers** — thin domain layers: **Story**, **Scientific**, **Workflow DAG**

**Deliverables**
- TypeScript packages: `graph-engine`, `kg-rag`, `story-graph-engine`, `scientific-kg`, `workflow-dag`
- Tests, example datasets, CLI demos
- Ready to integrate with vector stores (pgvector) and LLMs

---

## 1) What is KG‑RAG (One‑Paragraph Definition)

**KG‑RAG (Knowledge‑Graph–Augmented Retrieval‑Augmented Generation)** fuses **symbolic knowledge graphs** (explicit nodes/edges, constraints, timelines) with **semantic vector retrieval** (embeddings over text and metadata). The KG contributes **query expansion**, **scoped routing** (restrict search to relevant subgraphs), and **post‑generation consistency checks**; the vector store contributes **high‑recall, style‑aware context**. The result: grounded, coherent outputs at long horizons.

## 2) KG-RAG Architecture Overview

```
Application Layer (agents, UI, API)
      ▲                            ▲
      │                            │  Domain rules
      │                ┌────────────────────────────┐
      │                │   Domain Wrappers (3)      │
      │                │ Story | Scientific |  DAG  │
      │                └────────────────────────────┘
      │                            ▲
      │                            │  Generic graph ops
      │                ┌────────────────────────────┐
      │                │     GraphEngine (Core)     │
      │                │  nodes | edges | traversal │
      ▼                └────────────────────────────┘
   ┌──────────────────────────────────────────────────────┐
   │                    KG‑RAG Engine                     │
   │  (KG query expansion + retrieval routing + checks)   │
   └──────────────────────────────────────────────────────┘
      ▲                            ▲
      │ KG slice                   │ Vector hits
Knowledge Graph               Vector Store (pgvector / Qdrant)
```

**Key principles**  
- Core is domain‑neutral.  
- Wrappers translate domain → generic graph calls.  
- KG‑RAG is an orchestration/service layer consuming both KG and vector store.  
- Consistency checks are **structural** in core; **semantic** in wrappers.

## 3) Components

### 3.1) GraphEngine

- Domain-neutral graph core handling:
- CRUD
- traversal
- validation
- subgraph extraction

### 3.2) KG-RAG Engine

Responsible for:

- KG-based query expansion
- vector routing filters
- context bundle assembly
- symbolic consistency checks

### 3.3) Domain Wrappers

Specialized layers:
- StoryGraphEngine
- ScientificKG
- WorkflowGraphEngine

They define:
- domain entities (node labels)
- relationships (edge labels)
- semantic validation rules

## 4) Development Strategy

Build components in this order:

- GraphEngine
- Story wrapper
- KG-RAG engine
- Scientific & DAG wrappers
- Ingestion agents