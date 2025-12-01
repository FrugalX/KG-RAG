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

---

## 2) Architecture (Separation of Concerns)

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

---

## 3) Package Layout

```
/packages
  /graph-engine           # domain-agnostic core
  /kg-rag                 # hybrid retrieval + consistency harness
  /story-graph-engine     # wrapper: long-form story
  /scientific-kg          # wrapper: bio/med research
  /workflow-dag           # wrapper: orchestration DAG
/examples                  # small demo datasets + CLI scripts
/docs                      # MD specs & diagrams
```

---


## 4) GraphEngine — Domain‑Agnostic Core

### 4.1 Responsibilities
- CRUD: nodes/edges with labels + arbitrary properties
- Traversals: BFS/DFS, neighbors, shortest path
- Subgraph extraction by seeds/depth/edge labels
- Structural validation (label/edge constraints, dangling edges, required props)
- JSON import/export
- In‑memory + browser compatible (Graphology‑backed)

### 4.2 Types
```ts
type NodeId = string;
type EdgeId = string;

interface NodeProps { [k: string]: unknown }
interface EdgeProps { [k: string]: unknown }

interface NodeInput { id?: NodeId; label: string; props?: NodeProps }
interface EdgeInput { id?: EdgeId; label: string; from: NodeId; to: NodeId; props?: EdgeProps }

interface Node { id: NodeId; label: string; props: NodeProps }
interface Edge { id: EdgeId; label: string; from: NodeId; to: NodeId; props: EdgeProps }

interface GraphSnapshot {
  nodes: Node[];
  edges: Edge[];
}

interface StructureSchema {
  allowedNodeLabels?: string[];
  allowedEdgeLabels?: string[];
  edgeLabelConstraints?: Array<{ label: string; fromLabel: string; toLabel: string }>;
  requiredProps?: Record<string, string[]>; // node label -> required prop keys
}
```

### 4.3 Public API
```ts
export interface GraphEngine {
  // CRUD
  createNode(input: NodeInput): NodeId;
  updateNode(id: NodeId, props: Partial<NodeProps>): void;
  deleteNode(id: NodeId): void;

  createEdge(input: EdgeInput): EdgeId;
  updateEdge(id: EdgeId, props: Partial<EdgeProps>): void;
  deleteEdge(id: EdgeId): void;

  // Access & find
  getNode(id: NodeId): Node | null;
  getEdge(id: EdgeId): Edge | null;
  findNodes(filter: Partial<{label: string}> & { props?: Partial<NodeProps> }): NodeId[];
  findEdges(filter: Partial<{label: string; from: NodeId; to: NodeId}> & { props?: Partial<EdgeProps> }): EdgeId[];

  // Traversal
  neighbors(id: NodeId, opt?: { direction?: 'in'|'out'|'both'; edgeLabel?: string }): NodeId[];
  shortestPath(a: NodeId, b: NodeId, opt?: { edgeLabels?: string[] }): NodeId[] | null;
  subgraph(seeds: NodeId[], opt?: { depth?: number; edgeLabels?: string[]; maxNodes?: number }): GraphSnapshot;

  // Validation
  validateStructure(schema?: StructureSchema): { ok: boolean; issues: string[] };

  // Persistence
  toJSON(): GraphSnapshot;
  fromJSON(s: GraphSnapshot): void;
}
```

### 4.4 Structural Checks (examples)
- Edge label not allowed between `{fromLabel, toLabel}`
- Missing required node props for label
- Dangling edges (missing node endpoints)
- Disconnected components (optional warning)
- Path existence constraints (optional policies)

---

## 5) KG‑RAG Engine — Hybrid Retrieval Layer

### 5.1 Responsibilities
- **Level‑1 Query Expansion:** grow the query via KG (neighbors, causal predecessors, label filtering)  
- **Level‑2 Retrieval Routing:** restrict vector search using KG‑derived IDs/metadata scope  
- **Level‑3 Consistency Checks:** call domain wrapper to validate generated text against symbolic rules

### 5.2 Types & Config
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

### 5.3 Public API
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

### 5.4 Reference Flow
1) **Expand:** start from `kgSeeds` → `subgraph()` using depth + allowedEdgeLabels  
2) **Route:** compute metadata filter from KG slice → pass to vector store search  
3) **Retrieve:** call vector client with `scope + filter` → get top‑K passages  
4) **Bundle:** return `{ kgSlice, passages }` (ensure size budgets)  
5) **Post‑gen:** call `wrapper.checkConsistency(draft)` for symbolic validation

### 5.5 Vector Client (pluggable)
- Interface: `search(textQuery, filter, k)` → `VectorHit[]`
- Implementations: pgvector (SQL + cosine), Qdrant, Milvus

---

## 6) Wrapper A — StoryGraphEngine (Long‑Form Story)

### 6.1 Node Labels
- `Character`, `Event`, `Arc`, `Location`, `Artifact`

### 6.2 Edge Labels
- `relationship`, `participates_in`, `causes`, `occurs_at`, `uses`, `belongs_to_arc`, `foreshadows`

### 6.3 Types
```ts
interface CharacterProps { name: string; summary?: string; traits?: Record<string,unknown>; goals?: string[] }
interface LocationProps  { name: string; region?: string }
interface ArtifactProps  { name: string; abilities?: string[]; ownerId?: string }
interface EventProps     { name: string; summary?: string; chronology?: number }
interface ArcProps       { name: string; milestones?: string[] }
```

### 6.4 Public API
```ts
export interface StoryGraphEngine {
  // nodes
  addCharacter(p: CharacterProps): string;
  addLocation(p: LocationProps): string;
  addArtifact(p: ArtifactProps): string;
  addEvent(p: EventProps): string;
  addArc(p: ArcProps): string;

  // edges
  relateCharacters(a: string, b: string, subtype: 'allies'|'enemies'|'kin'|'mentor'|'rival'): string;
  linkParticipation(charId: string, eventId: string): string;
  setEventLocation(eventId: string, locId: string): string;
  linkUse(charId: string, artifactId: string): string;
  linkCausality(causeEventId: string, effectEventId: string): string;
  assignToArc(eventId: string, arcId: string): string;

  // queries
  getCharacterArc(id: string): { arcIds: string[]; events: string[]; order: string[] };
  getEventChain(startId: string, depth?: number): string[];
  getCharacterNetwork(id: string, hops?: number): string[];
  getArcTimeline(arcId: string): Array<{ eventId: string; chronology?: number }>;  

  // semantic checks
  validateTimeline(): { ok: boolean; issues: string[] };
  validatePossession(): { ok: boolean; issues: string[] };
  validateForeshadowing(): { ok: boolean; issues: string[] };

  // wrapper hooks for KG‑RAG
  expandSeeds(g: GraphEngine, seeds: string[], cfg: KgRagConfig): string[];
  buildMetadataFilter(kgSlice: GraphSnapshot): Record<string, unknown>;
  checkConsistency(draft: string, g: GraphEngine): { ok: boolean; issues: string[] };
}
```

### 6.5 KG‑RAG Behavior (Story)
- **Expansion:** include characters in scene, their relationships within 1–2 hops, causal predecessors of target event, arc milestones.  
- **Routing:** filter vector search by `{ characterIds, eventIds, arcIds, chapterId }`.  
- **Consistency:** detect timeline violations, artifact misuse, foreshadowing order errors.

### 6.6 Example Metadata for Vector Chunks
```json
{
  "projectId":"p1",
  "chapterId":"c12",
  "characterIds":["char_arion","char_mara"],
  "eventIds":["ev_duel_1"],
  "arcIds":["arc_reckoning"]
}
```

---

## 7) Wrapper B — ScientificKG (Bio/Medical/Research)

### 7.1 Node Labels
- `Gene`, `Protein`, `Chemical`, `Disease`, `Pathway`, `Paper`

### 7.2 Edge Labels
- `encodes`, `interacts_with`, `inhibits`, `activates`, `associated_with`, `involved_in`, `cites`, `reports_on`

### 7.3 Types
```ts
interface GeneProps     { name: string; symbol?: string; description?: string }
interface ProteinProps  { name: string; function?: string }
interface ChemicalProps { name: string; formula?: string }
interface DiseaseProps  { name: string; category?: string }
interface PathwayProps  { name: string; description?: string }
interface PaperProps    { title: string; authors: string[]; year: number; doi?: string }
```

### 7.4 Public API
```ts
export interface ScientificKG {
  addGene(p: GeneProps): string;
  addProtein(p: ProteinProps): string;
  addChemical(p: ChemicalProps): string;
  addDisease(p: DiseaseProps): string;
  addPathway(p: PathwayProps): string;
  addPaper(p: PaperProps): string;

  linkEncoding(geneId: string, proteinId: string): string;
  linkInteraction(proteinA: string, proteinB: string): string;
  linkInhibition(chemicalId: string, targetProteinId: string): string;
  linkActivation(chemicalId: string, targetProteinId: string): string;
  linkAssociation(entityId: string, diseaseId: string): string;
  linkPathwayMembership(entityId: string, pathwayId: string): string;

  linkCitation(paperA: string, paperB: string): string;
  linkReportedEntity(paperId: string, entityId: string): string;

  // queries
  getProteinInteractions(proteinId: string): string[];
  getGeneProteinPathway(geneId: string): { proteinId: string; pathways: string[] };
  findChemicalTargets(chemicalId: string): string[];
  getDiseaseRelatedEntities(diseaseId: string): string[];

  // checks
  validateNoCircularCitations(): { ok: boolean; issues: string[] };

  // wrapper hooks for KG‑RAG
  expandSeeds(g: GraphEngine, seeds: string[], cfg: KgRagConfig): string[];
  buildMetadataFilter(kgSlice: GraphSnapshot): Record<string, unknown>;
  checkConsistency(draft: string, g: GraphEngine): { ok: boolean; issues: string[] };
}
```

### 7.5 KG‑RAG Behavior (Science)
- **Expansion:** walk proteins ↔ chemicals ↔ pathways within 1 hop; include cited papers.  
- **Routing:** limit vector retrieval by `{ proteinIds, chemicalIds, diseaseIds, paperIds }`.  
- **Consistency:** basic symbol checks (no circular citations); domain teams may add stronger rules later.

---

## 8) Wrapper C — WorkflowGraphEngine (Orchestration DAG)

### 8.1 Node Labels
- `Task`, `DataAsset`, `Schedule`, `Condition`

### 8.2 Edge Labels
- `depends_on`, `produces`, `consumes`, `triggers`, `enables`

### 8.3 Types
```ts
interface TaskProps      { name: string; description?: string; runtime?: string }
interface DataAssetProps { name: string; format?: string; sizeMB?: number }
interface ScheduleProps  { cron?: string; intervalSeconds?: number }
interface ConditionProps { expression: string }
```

### 8.4 Public API
```ts
export interface WorkflowGraphEngine {
  addTask(p: TaskProps): string;
  addDataAsset(p: DataAssetProps): string;
  addSchedule(p: ScheduleProps): string;
  addCondition(p: ConditionProps): string;

  addDependency(taskId: string, dependsOnId: string): string;
  addProducedData(taskId: string, dataId: string): string;
  addConsumedData(taskId: string, dataId: string): string;
  addTrigger(scheduleId: string, taskId: string): string;
  addConditionLink(conditionId: string, taskId: string): string;

  getTaskDependencies(taskId: string): string[];
  getDownstreamTasks(taskId: string): string[];
  getUpstreamTasks(taskId: string): string[];
  getExecutionOrder(): string[]; // topological order

  getDataLineage(dataId: string): { producers: string[]; consumers: string[] };

  validateDAG(): { ok: boolean; issues: string[] };
  validateDataFlow(): { ok: boolean; issues: string[] };

  // wrapper hooks for KG‑RAG
  expandSeeds(g: GraphEngine, seeds: string[], cfg: KgRagConfig): string[];
  buildMetadataFilter(kgSlice: GraphSnapshot): Record<string, unknown>;
  checkConsistency(draft: string, g: GraphEngine): { ok: boolean; issues: string[] };
}
```

### 8.5 KG‑RAG Behavior (DAG)
- **Expansion:** include upstream/downstream tasks up to N hops; include data assets and schedules.  
- **Routing:** restrict vector search by `{ taskIds, dataAssetIds }`.  
- **Consistency:** `validateDAG` (cycle‑free), `validateDataFlow` (inputs have producers).

---


## 9) Agentic Ingestion & KG Construction Layer  

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
It is recommended as a subsequent integration milestone once the development team has validated the core KG-RAG behavior and schema.

---


## 10) Cross‑Cutting Concerns

### 10.1 Performance & Budgets
- Default `expandHops=1`, `maxKgNodes=60`, `vectorK=8` (configurable)
- Truncate oversized KG slices by degree‑based sampling
- Apply token budget for notes/context (< 1.5k tokens bundle target)

### 10.2 Testing
- **graph-engine:** CRUD, traversal, subgraph caps, structural validation
- **wrappers:** schema coverage + semantic checks (seeded contradictions)
- **kg-rag:** query expansion unit tests + routing integration with mock vector client + L3 checks

### 10.3 Security & Privacy
- Keep KG + vector metadata project‑scoped
- Redact sensitive properties in `toJSON()` via allowlist (optional)

### 10.4 CLI Demos (optional)
- `kg seed --domain story --example duel.json`
- `kg subgraph --seeds n1,n2 --depth 2`
- `rag build-context --text "..." --seeds n1 --domain story`
- `rag check --domain story --draft ./draft.txt`

---

## 11) Example End‑to‑End (Story)

1) Seeds = `{ Arion, Event: Duel@Ford }`  
2) KG expansion (1 hop) → add allies/enemies + causal predecessors  
3) Routing filter → `{ characterIds:[Arion,Mara], eventIds:[Duel], arcIds:["Reckoning"] }`  
4) Vector hits → prior scenes, tone exemplars, descriptions  
5) LLM prompt = **compact KG slice** + **top‑K passages**  
6) Draft → run `checkConsistency` → fix timeline violations if any

---

## 12) Success Criteria (PoC)

- GraphEngine: 100% domain‑agnostic, Node+Browser parity, JSON round‑trip verified
- Wrappers (3): entity/edge APIs implemented, semantic validators pass sample suites
- KG‑RAG: produces bounded **ContextBundle** and flags seeded inconsistencies
- Demos: each wrapper has a runnable example using mock vector client

---

## 13) Roadmap

**T1:** GraphEngine core + tests + JSON  
**T2:** Subgraph + validation + caps + docs  
**T3:** Story wrapper + semantic checks + example dataset  
**T4:** KG‑RAG L1/L2 + mock vector client + Story demo  
**T5:** Scientific & DAG wrappers + demos + L3 checks  
**T6:** Agentic ingestion layer + LLM/rule-based extractors for document → KG conversion  
**T7:** Full agentic KG‑RAG lifecycle demo (ingestion → validation → retrieval → generation)

---

## 14) Technology Stack & Implementation Notes

### Core Languages & Frameworks
TypeScript / Node.js — primary implementation language for all packages  
Graphology — graph data structure and traversal backend for graph-engine  
Node.js Test Runner — testing framework for unit and integration tests  
rag-lite-ts — Traditional RAG framework
Persistence — JSON snapshots, or SQLite for graph storage  

### Agentic Layer (Downstream Integration)

LLM APIs: OpenAI GPT-5, Anthropic Claude, Google Gemini or local models (Ollama)  
Orchestration Frameworks: Dexto  

**End of Spec**
