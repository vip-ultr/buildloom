# Buildloom — Architecture Essentials

**For:** New engineers and agents working on the codebase.  
**Read alongside:** `ARCHITECTURE.md` (full), `PRD.md` (what it must do), `CONCEPT.md` (why it exists).

---

## What it is

Buildloom is a build-time pipeline that produces a cumulative, queryable knowledge graph as a side effect of building. It reads what already exists — git history, CI logs, coverage, deployment signals — and writes a single SQLite graph database plus a human-readable report. Nothing it needs is created by Buildloom; everything is already produced by the work.

## What the graph knows

- **What changed** — files, functions, modules, with diffs tied to commits and rationale from PRs/issues.
- **Why** — the decision trail behind changes.
- **What depends on what** — build dependencies, runtime relationships, deployment order.
- **What is covered** — which tests touch which code, where coverage is thin.
- **What deployed where** — artifact, environment, version, release trail.

## The three layers

1. **Ingest** — a small set of ingestors (git, CI, coverage, deployment), each self-contained, each independently skippable.
2. **Store** — a single SQLite file with a documented schema. Append-friendly; builds add to it.
3. **Query and report** — a CLI that answers questions and emits a build report.

## How a build step runs

1. The normal build has already run; artifacts, coverage, CI logs, tags are present.
2. Buildloom reads its per-ingestor pointers (where each ingestor left off).
3. Each ingestor reads only what is new since its pointer. Missing or unreachable sources are logged and skipped; they do not fail the step.
4. Records are normalized into nodes and edges, deduplicated, and written to the store.
5. Pointers are updated. A report is emitted.

## The contracts that matter

- **Ingestor contract:** run(scope) -> records + new pointer; availability check; no dependency on other ingestors for its own job.
- **Normalized record:** a small, source-agnostic structure — kind, identifiers, provenance, immediate relationships. Not the graph; input to graph construction.
- **Store schema:** the boundary between normalization and query. Once something is a node or edge, it is queried through the schema, not through the ingestor.
- **CLI:** open the store, accept a query + arguments, return readable output. No running service required.

## v1 scope

- One ecosystem (Python assumed; confirm against a real target).
- One CI system (GitHub Actions assumed; confirm).
- One coverage format (lcov or coverage.py JSON).
- One deployment signal (tag + manifest entry minimum).
- SQLite graph store, documented schema.
- CLI answering at least two queries.
- Build report as markdown or text.
- Incremental ingestion; graceful degradation.

## v1 non-goals

No web UI, no public API, no remote graph, no multi-ecosystem support, no mandatory per-commit opt-in, no separate indexing step required for basic queries. Not a documentation tool. Not a static dependency visualizer. Not an architecture diagram generator. Not a replacement for the tools you already use.

## Where to look first

- `PRD.md` — the requirements and success criteria.
- `ARCHITECTURE.md` — the full shape, data flow, and boundaries.
- `CONCEPT.md` — the idea and the problem it attacks.
- `docs/ROADMAP.md` — what we build first and why.
- `docs/BUILD.md` — how to build and run the project itself.

## Good first questions to validate

- Which real repository do we point this at first?
- Which two queries are most valuable against that repository?
- Which ecosystem and CI system does that repository actually use?

Answer those before locking the v1 implementation choices.
