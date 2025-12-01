# Agentic Ingestion & KG Construction Layer  

### Purpose  
Provide a **downstream orchestration layer** of  **autonomous agents** that leverages the established KG-RAG core to automatically convert raw documents, datasets, or structured inputs into validated **Knowledge Graph (KG) snapshots**.
This layer can be introduced after the core modules (`GraphEngine`, Wrappers, KG-RAG Engine) have been implemented and tested, enabling real-world ingestion and continuous KG maintenance through autonomous agents.  

### Position in Stack  
```
Raw Data (text, paper, workflow spec)
      │
      ▼
┌────────────────────────────────────────┐
│   Agentic Ingestion & KG Construction  │
│  (document → entities/relations → KG)  │
└────────────────────────────────────────┘
      │
      ▼
GraphEngine Core  +  Domain Wrappers
      │
      ▼
KG-RAG Engine  (query expansion, routing, checks)
```

### Responsibilities  
- **Entity & Relation Extraction** — identify domain-specific entities and symbolic relationships from text or structured inputs.  
- **Graph Population** — create corresponding nodes/edges in the `GraphEngine`.  
- **Validation & Repair** — run structural and semantic checks; auto-correct or flag issues.  
- **Enrichment & Linking** — augment nodes with metadata (e.g., vector embeddings, ontology IDs).  
- **Persistence & Audit** — emit and version `GraphSnapshot` objects, log provenance of each node/edge.

### Agent Roles  

| Agent | Responsibilities | Inputs | Outputs |
|--------|------------------|---------|----------|
| **Ingestion Agent** | Extracts entities & relations from raw content using LLM or rule pipelines. | Text / structured data | Draft node / edge JSON |
| **Validation Agent** | Runs `validateStructure()` + wrapper semantic checks; prompts auto-repair. | Draft GraphSnapshot | Clean GraphSnapshot |
| **Indexer Agent** | Embeds textual fields, updates vector store, attaches metadata links. | Valid GraphSnapshot | Indexed GraphSnapshot |
| **RAG Agent** | Builds runtime `ContextBundle` for generation/retrieval tasks. | Indexed GraphSnapshot | ContextBundle |

Agents communicate through standardized object formats (`NodeInput`, `EdgeInput`, `GraphSnapshot`) defined in the core.

### Standard Interface  
```ts
export interface KgConstructionAgent {
  run(inputDocs: string[], wrapper: DomainWrapperEngine, cfg?: KgConstructionConfig):
    Promise<GraphSnapshot[]>;

  auditLog?(snapshot: GraphSnapshot, issues: string[]): void;
}

export interface KgConstructionConfig {
  extractionMode?: 'llm' | 'hybrid' | 'rule';
  maxDocs?: number;
  validate?: boolean;
  enrichVectors?: boolean;
}
```

### Lifecycle Example (Story Domain)  
1) **Input:** Chapter text → `Ingestion Agent` extracts Characters, Events, Locations, and relations (`participates_in`, `occurs_at`, `belongs_to_arc`).  
2) **Validation:** `Validation Agent` calls `GraphEngine.validateStructure()` + `StoryGraphEngine.validateTimeline()`.  
3) **Enrichment:** `Indexer Agent` embeds descriptions and writes vector metadata.  
4) **Output:** A compact, versioned `GraphSnapshot` consumed by the `KG-RAG Engine` for query expansion and context building.

### Benefits  
- Unified, auditable pathway from **document to graph**.  
- Keeps wrappers focused on **domain semantics**, not ingestion.  
- Enables future **autonomous or continual KG updates**.  
- Compatible with both **LLM-assisted** and **classical NLP** pipelines.

### Implementation Note:
The Agentic Layer is not required for the initial proof-of-concept or unit testing phases.
It is recommended as a subsequent integration milestone once the core KG-RAG behavior and schema are validated