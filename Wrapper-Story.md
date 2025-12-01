# StoryGraphEngine (Long‑Form Story)

### Node Labels
- `Character`, `Event`, `Arc`, `Location`, `Artifact`

### Edge Labels
- `relationship`, `participates_in`, `causes`, `occurs_at`, `uses`, `belongs_to_arc`, `foreshadows`

### Types
```ts
interface CharacterProps { name: string; summary?: string; traits?: Record<string,unknown>; goals?: string[] }
interface LocationProps  { name: string; region?: string }
interface ArtifactProps  { name: string; abilities?: string[]; ownerId?: string }
interface EventProps     { name: string; summary?: string; chronology?: number }
interface ArcProps       { name: string; milestones?: string[] }
```

### Public API
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

### KG‑RAG Behavior (Story)
- **Expansion:** include characters in scene, their relationships within 1–2 hops, causal predecessors of target event, arc milestones.  
- **Routing:** filter vector search by `{ characterIds, eventIds, arcIds, chapterId }`.  
- **Consistency:** detect timeline violations, artifact misuse, foreshadowing order errors.

### Example Metadata for Vector Chunks
```json
{
  "projectId":"p1",
  "chapterId":"c12",
  "characterIds":["char_arion","char_mara"],
  "eventIds":["ev_duel_1"],
  "arcIds":["arc_reckoning"]
}
```
