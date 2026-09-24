# Buildloom — Architecture

**Version:** 1.0  
**Status:** Draft  
**Last updated:** 2026-09-24

---

## 1. Architecture Vision

Buildloom is a pipeline, not a platform. It has three concerns and three boundaries:

1. **Ingest** — read from existing sources (git, CI, coverage, deployment) and produce normalized records.
2. **Store** — persist those records into a graph database with a clear schema.
3. **Query and report** — answer questions against the graph and emit human-readable output.

Each concern is a separable layer. Each ingestor is a separable plugin. The store is a single SQLite file. The query surface is a CLI. Nothing here requires a server, a long-running process, or a network service in v1.

The architecture is shaped by one constraint that drives every decision: **the graph must be produced as a side effect of building, not as a separate chore.** That means incremental ingestion, graceful degradation when a source is missing, and a build step that is fast enough to not be the slowest thing in the pipeline.

---

## 2. High-Level Picture

```
target repository
  |
  +-- git history, diffs, PR/issue metadata
  +-- CI logs and results (GitHub Actions)
  +-- coverage report (lcov / coverage.py JSON)
  +-- deployment signal (tag, manifest, deployment log)
  |
  v
+-------------------------------------------------------+
|  Buildloom build step                                 |
|                                                       |
|  +----------------+   +----------------+              |
|  | Git ingestor   |   | CI ingestor    |              |
|  +----------------+   +----------------+              |
|  +----------------+   +----------------+              |
|  | Coverage ingest|   | Deploy ingestor|              |
|  +----------------+   +----------------+              |
|                                                       |
|  v                                                    |
|  +--------------------------------------------------+ |
|  | Normalization + graph construction               | |
|  | (records -> nodes + edges, dedup, provenance)    | |
|  +--------------------------------------------------+ |
|  v                                                    |
|  +--------------------------------------------------+ |
|  | SQLite graph store                               | |
|  | (single file, documented schema, append-friendly)| |
|  +--------------------------------------------------+ |
|  v                                                    |
|  +--------------------------------------------------+ |
|  | Query layer (CLI) + report emission              | |
|  +--------------------------------------------------+ |
+-------------------------------------------------------+
  |
  v
graph.db  +  buildloom-report.md  (side effects alongside the build)
```

The build step runs after the normal build artifacts are produced. It reads what the build and the repository already produced, adds to the graph, and writes a report. It does not gate the build on its own success.

---

## 3. Component Diagram

### 3.1 Ingestors

Each ingestor is a self-contained module with one contract: given a source (and optionally a pointer to where it left off), produce a stream of normalized records and update its own last-seen pointer.

```
+------------------+
| Git ingestor     |
| - commits        |
| - diffs / files  |
| - PR/issue links |
+--------+---------+
         | normalized records
         v
+------------------+
| CI ingestor      |
| - build runs     |
| - build results  |
| - log pointers   |
+--------+---------+
         | normalized records
         v
+------------------+
| Coverage ingestor|
| - coverage files |
| - file/function  |
|   coverage map   |
+--------+---------+
         | normalized records
         v
+------------------+
| Deployment       |
| ingestor         |
| - tags           |
| - manifests      |
| - deploy events  |
+--------+---------+
```

### 3.2 Normalization and graph construction

A central normalizer receives records from all ingestors and:

- Maps them onto the schema's node and edge types.
- Deduplicates against what is already in the store.
- Attaches provenance (which ingestor, which source identifier, when).
- Writes nodes and edges in a defined order so foreign-key and relationship integrity is preserved.

This layer is where "what changed" becomes "what changed, in which commit, tied to which PR, and what else was touched at the same time."

### 3.3 Graph store

A single SQLite database with a documented schema. It is the system of record. The schema is designed so that:

- Every node and edge has a source, a source identifier, a timestamp, and a provenance indicator.
- The store is append-friendly: builds add to it, and corrective ingests are explicit operations, not silent rewrites.
- Basic queries work immediately after a build; no separate indexing step is required for the v1 query surface.

### 3.4 Query layer and report emission

A CLI that:

- Opens the graph store.
- Accepts a query (by name and arguments) and returns human-readable output.
- Optionally emits a build report that summarizes what this build added and what it connects to.

---

## 4. Data Flow

For a single build:

1. **Target repository and artifacts are already present** — the build has run, coverage has been generated, CI has logged, tags or manifests exist as applicable.
2. **Buildloom step starts.** It reads its own last-seen pointers (one per ingestor) from the graph store or a small state file.
3. **Each ingestor runs.** It reads only what is new since its last-seen pointer. If a source is missing or unreachable, that ingestor logs and skips; it does not fail the step.
4. **Records are normalized** into nodes and edges, deduplicated against the existing store, and written.
5. **The graph store is updated.** New nodes, new edges, updated ingestor pointers.
6. **The query layer emits** a build report and is available for on-demand queries.

For a rewrite or corrective ingest (explicit, not automatic):

1. The user requests an ingestor to re-ingest from a given point.
2. That ingestor resets its pointer, re-reads, and the normalizer reconciles the full range.
3. The store is updated. Nothing else is touched.

---

## 5. Interface Boundaries

### 5.1 Ingestor contract

Each ingestor exposes, at minimum:

- **Run** — given a scope (e.g., "since this commit" or "since this timestamp"), produce normalized records and return a new pointer.
- **Pointer** — what it uses to know where it left off (commit SHA, timestamp, log offset, file hash).
- **Availability** — a quick check that the source exists and is reachable, so the step can decide whether to run it.

An ingestor must not depend on another ingestor's output to do its own job. Cross-ingestor connections happen in normalization, not in ingestion.

### 5.2 Normalized record contract

A normalized record is a small, source-agnostic structure that says:

- What kind of thing this is (commit, file, function, PR, test, coverage entry, deployment, etc.).
- The identifying information for that thing (repo-relative path, function name, SHA, number, etc.).
- The provenance (which ingestor, which source identifier, when observed).
- Any immediate relationships that are cheap and unambiguous to attach at ingest time (e.g., a commit record carries the files it touched; a coverage record carries the file it covers).

Records are not the graph. They are the input to graph construction. The graph construction step is where records become nodes and edges in the schema.

### 5.3 Store schema boundary

The store schema is the contract between the normalizer and the query layer. Once a record is in the store as a node or edge, it is queried through the schema, not through the ingestor that produced it.

### 5.4 CLI contract

The CLI is the query surface. It must accept a query and arguments and return human-readable output. It must not require a running service. It must be able to point at a graph store and answer immediately.

---

## 6. Storage Design

### 6.1 Why SQLite

- Single file, portable, backable, inspectable with standard tools.
- Good enough for the graph sizes we expect in v1.
- No server, no migration infrastructure, no long-running process.
- Append-friendly with the right schema design.

### 6.2 Schema principles

- One table per node type, one table per edge type, or a small number of well-documented tables — the exact shape is to be worked out in code, but the principle is: **a reader should be able to look at the schema and understand what a node and an edge mean without reading the ingestors.**
- Every node and edge carries source, source identifier, timestamp, and provenance.
- The ingestor pointer state lives either in a small metadata table in the same database or in a parallel state file; the principle is: **the state that makes ingestion incremental must itself be queryable and recoverable.**

### 6.3 Append-friendly design

- Normal ingestion inserts new rows; it does not update or delete existing ones except where a corrective ingest explicitly reconciles a range.
- If two ingestors produce overlapping information (e.g., a commit's files come from git, and the same files appear in coverage), the schema should make that overlap visible rather than silently collapsing it.

---

## 7. Query Design

### 7.1 v1 query surface

At minimum, two queries:

- **Subsystem change with rationale** — given a path or set of paths and a time window, return the files, functions, commits, and linked PR/issue rationale.
- **Impact / dependency** — given a module or file, return what depends on it, with provenance and the nature of each relationship.

These two are the floor. The working assumption is that they cover the most valuable onboarding and impact-analysis questions for a real repository. They should be validated against a real target before we commit to them as the final v1 surface.

### 7.2 Output

- Human-readable text or structured markdown.
- No UI required in v1.
- The build report is a special case of query output: a summary of what this build added and what it connects to, written alongside the build artifacts.

### 7.3 Extensibility

- New queries are added as functions against the schema, not as new ingestors or a new storage layer.
- The CLI is the entry point; the schema is the surface. Adding a query should not require touching an ingestor.

---

## 8. Build Integration

### 8.1 Where it sits

Buildloom runs as a step after the normal build. The build produces its artifacts; Buildloom reads them and adds to the graph. The build's success or failure is its own concern; Buildloom's failure does not silently swallow the build outcome.

### 8.2 Integration points

- A CI step (e.g., a GitHub Actions step that runs after the build and coverage jobs).
- A pre/post-build hook in a Make/just recipe.
- A standalone invocation for local development, where the engineer runs it by hand after a build.

### 8.3 Local development

- An engineer working locally should be able to run Buildloom against their local repository and get a graph and a report without any CI involvement.
- The git ingestor is the one ingestor that must work offline from the local repository; it is the baseline that makes the tool useful even before any CI integration.

---

## 9. v1 Scope Boundaries

In scope for v1:

- One ecosystem (Python or the ecosystem of a real target repository).
- One CI system (GitHub Actions, or the CI of the real target).
- One coverage format (lcov or coverage.py JSON).
- One deployment signal (tag plus manifest entry, or the deployment signal of the real target).
- A SQLite graph store with a documented schema.
- A CLI that answers at least two useful queries.
- A build report emitted as markdown or text.
- Incremental ingestion with per-ingestor pointers.
- Graceful degradation when a source is missing.

Out of scope for v1:

- A web UI.
- A public API.
- A persisted remote graph.
- Multi-ecosystem support.
- A separate indexing or compaction step required for basic queries.
- A mandatory per-commit opt-in.

---

## 10. Assumptions and Risks

### 10.1 Assumptions

- The target repository already produces most of the inputs (git history, CI logs, coverage, deployment signal). Buildloom joins them; it does not create them.
- A single SQLite file is sufficient for v1 graph sizes and query latency.
- Two well-chosen queries are enough to demonstrate value in v1.
- Incremental ingestion with per-ingestor pointers is sufficient to keep the step fast.

### 10.2 Risks

- **Scope creep into documentation tooling.** The project must stay a build-time graph, not a docs platform. The PRD non-goals are the guardrail.
- **Over-engineering the schema.** v1 schema should be honest and minimal; it can be revised as queries reveal what is actually needed.
- **Dependency on a specific CI or coverage format too early.** The ingestor contract is the guardrail; swapping a source should be a pinch replacement, not a rewrite.
- **The build step becoming the slowest thing in the pipeline.** Incremental ingestion and a bounded first run are the mitigation; measure on a real repository.

---

*This document describes the shape of the system and the boundaries between its parts. The essentials version distills this for a new engineer or agent on day one. The concept and PRD capture why and what; this document captures how the pieces fit.*
