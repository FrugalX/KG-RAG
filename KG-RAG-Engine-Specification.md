# KG‑RAG Engine — Hybrid Retrieval Layer

### Responsibilities
- **Level‑1 Query Expansion:** grow the query via KG (neighbors, causal predecessors, label filtering)  
- **Level‑2 Retrieval Routing:** restrict vector search using KG‑derived IDs/metadata scope  
- **Level‑3 Consistency Checks:** call domain wrapper to validate generated text against symbolic rules

### Types & Config
```ts
export interface RagQuery {
  textQuery: string;                 // agent/system prompt intent
  kgSeeds?: string[];                // seed node IDs
  scope?: Record<string, unknown>;   // projectId, chapterId, datasetId, pipelineId...
  maxPromptTokens?: number;
}

export interface KgRagConfig {
  expandHops?: number;               // default 1..2
  allowedEdgeLabels?: string[];      // traversal whitelist
  maxKgNodes?: number;               // cap KG slice
  vectorK?: number;                  // top-K passages
  routeByMetadata?: boolean;         // use KG IDs in vector filter
}

export interface VectorHit {
  id: string;
  text: string;
  score: number;
  metadata: Record<string, unknown>; // e.g. {characterIds:[], taskIds:[], paperId:...}
}

export interface ContextBundle {
  kgSlice: GraphSnapshot;            // compact subgraph (<= maxKgNodes)
  passages: VectorHit[];             // vector top-K
  notes?: string[];                  // optional reasoning / constraints
}
```

### Public API
```ts
export interface DomainWrapperEngine {
  // The wrapper must expose these hooks (story/science/dag implement them):
  expandSeeds?(g: GraphEngine, seeds: string[], cfg: KgRagConfig): string[]; // return node IDs
  buildMetadataFilter?(kgSlice: GraphSnapshot): Record<string, unknown>;     // for vector routing
  checkConsistency?(draft: string, g: GraphEngine): { ok: boolean; issues: string[] };
}

export interface KgRagEngine {
  buildContext(q: RagQuery, cfg: KgRagConfig, wrapper?: DomainWrapperEngine):
    Promise<ContextBundle>;

  checkConsistency(draftText: string, wrapper: DomainWrapperEngine, g: GraphEngine):
    { ok: boolean; issues: string[] };
}
```

### Reference Flow
1) **Expand:** start from `kgSeeds` → `subgraph()` using depth + allowedEdgeLabels  
2) **Route:** compute metadata filter from KG slice → pass to vector store search  
3) **Retrieve:** call vector client with `scope + filter` → get top‑K passages  
4) **Bundle:** return `{ kgSlice, passages }` (ensure size budgets)  
5) **Post‑gen:** call `wrapper.checkConsistency(draft)` for symbolic validation

### 5.5 Vector Client (pluggable)
- Interface: `search(textQuery, filter, k)` → `VectorHit[]`
- Implementations: pgvector (SQL + cosine), Qdrant, Milvus
