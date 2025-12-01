# Core Graph Implementation Specification

## Purpose

Provide a domain-neutral, browser-compatible, TypeScript graph engine to support all wrappers and the KG-RAG engine.

### Responsibilities
- CRUD: nodes/edges with labels + arbitrary properties
- Traversals: BFS/DFS, neighbors, shortest path
- Subgraph extraction by seeds/depth/edge labels
- Structural validation (label/edge constraints, dangling edges, required props)
- JSON import/export
- In‑memory + browser compatible (Graphology‑backed)

### Types
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

### Public API
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

### Structural Checks (examples)
- Edge label not allowed between `{fromLabel, toLabel}`
- Missing required node props for label
- Dangling edges (missing node endpoints)
- Disconnected components (optional warning)
- Path existence constraints (optional policies)

### Implementation Notes

- Graphology recommended as backend
- Ensure deterministic subgraph extraction order
- Keep maxNodes caps to avoid LLM token overflow
- JSON import/export must be lossless