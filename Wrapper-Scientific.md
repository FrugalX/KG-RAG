# ScientificKG (Bio/Medical/Research)

### Node Labels
- `Gene`, `Protein`, `Chemical`, `Disease`, `Pathway`, `Paper`

### Edge Labels
- `encodes`, `interacts_with`, `inhibits`, `activates`, `associated_with`, `involved_in`, `cites`, `reports_on`

### Types
```ts
interface GeneProps     { name: string; symbol?: string; description?: string }
interface ProteinProps  { name: string; function?: string }
interface ChemicalProps { name: string; formula?: string }
interface DiseaseProps  { name: string; category?: string }
interface PathwayProps  { name: string; description?: string }
interface PaperProps    { title: string; authors: string[]; year: number; doi?: string }
```

### Public API
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

### KG‑RAG Behavior (Science)
- **Expansion:** walk proteins ↔ chemicals ↔ pathways within 1 hop; include cited papers.  
- **Routing:** limit vector retrieval by `{ proteinIds, chemicalIds, diseaseIds, paperIds }`.  
- **Consistency:** basic symbol checks (no circular citations); domain teams may add stronger rules later.
