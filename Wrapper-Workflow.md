# WorkflowGraphEngine (Orchestration DAG)

### Node Labels
- `Task`, `DataAsset`, `Schedule`, `Condition`

### Edge Labels
- `depends_on`, `produces`, `consumes`, `triggers`, `enables`

### Types
```ts
interface TaskProps      { name: string; description?: string; runtime?: string }
interface DataAssetProps { name: string; format?: string; sizeMB?: number }
interface ScheduleProps  { cron?: string; intervalSeconds?: number }
interface ConditionProps { expression: string }
```

### Public API
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

### KG‑RAG Behavior (DAG)
- **Expansion:** include upstream/downstream tasks up to N hops; include data assets and schedules.  
- **Routing:** restrict vector search by `{ taskIds, dataAssetIds }`.  
- **Consistency:** `validateDAG` (cycle‑free), `validateDataFlow` (inputs have producers).
