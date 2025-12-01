# Development Guide

## 1) Roadmap

**T1:** GraphEngine core + tests + JSON  
**T2:** Subgraph + validation + caps + docs  
**T3:** Story wrapper + semantic checks + example dataset  
**T4:** KG‑RAG L1/L2 + mock vector client + Story demo  
**T5:** Scientific & DAG wrappers + demos + L3 checks  
**T6:** Agentic ingestion layer + LLM/rule-based extractors for document → KG conversion  
**T7:** Full agentic KG‑RAG lifecycle demo (ingestion → validation → retrieval → generation)

---

## 2) Technology Stack & Implementation Notes

### Core Languages & Frameworks
TypeScript / Node.js — primary implementation language for all packages  
Graphology — graph data structure and traversal backend for graph-engine  
Node.js Test Runner — testing framework for unit and integration tests  
rag-lite-ts — Traditional RAG framework
Persistence — JSON snapshots, or SQLite for graph storage  

### Agentic Layer (Downstream Integration)

LLM APIs: OpenAI GPT-5, Anthropic Claude, Google Gemini or local models (Ollama)  
Orchestration Frameworks: Dexto 

---

### 3) Cross‑Cutting Concerns

### 3.1 Performance & Budgets
- Default `expandHops=1`, `maxKgNodes=60`, `vectorK=8` (configurable)
- Truncate oversized KG slices by degree‑based sampling
- Apply token budget for notes/context (< 1.5k tokens bundle target)

### 3.2 Testing
- **graph-engine:** CRUD, traversal, subgraph caps, structural validation
- **wrappers:** schema coverage + semantic checks (seeded contradictions)
- **kg-rag:** query expansion unit tests + routing integration with mock vector client + L3 checks

### 3.3 CLI Demos (optional)
- `kg seed --domain story --example duel.json`
- `kg subgraph --seeds n1,n2 --depth 2`
- `rag build-context --text "..." --seeds n1 --domain story`
- `rag check --domain story --draft ./draft.txt`

---

## 4) Example End‑to‑End (Story)

1) Seeds = `{ Arion, Event: Duel@Ford }`  
2) KG expansion (1 hop) → add allies/enemies + causal predecessors  
3) Routing filter → `{ characterIds:[Arion,Mara], eventIds:[Duel], arcIds:["Reckoning"] }`  
4) Vector hits → prior scenes, tone exemplars, descriptions  
5) LLM prompt = **compact KG slice** + **top‑K passages**  
6) Draft → run `checkConsistency` → fix timeline violations if any

---

## 5) Success Criteria (PoC)

- GraphEngine: 100% domain‑agnostic, Node+Browser parity, JSON round‑trip verified
- Wrappers (3): entity/edge APIs implemented, semantic validators pass sample suites
- KG‑RAG: produces bounded **ContextBundle** and flags seeded inconsistencies
- Demos: each wrapper has a runnable example using mock vector client
