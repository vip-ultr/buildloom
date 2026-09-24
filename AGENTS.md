# Buildloom — AGENTS.md

**For:** Agents and automated contributors working on this repository.  
**Read first:** `ARCHITECTURE-ESSENTIALS.md` (the short version), then `ROADMAP.md` (what to build next and in what order).

---

## What this repository is

Buildloom is a build-time pipeline that produces a cumulative, queryable knowledge graph as a side effect of building. It ingests git history, CI logs, coverage, and deployment signals into a single SQLite graph store, answers questions with a CLI, and emits a human-readable report.

It is not a documentation tool, not a static dependency visualizer, not an architecture diagram generator, and not a replacement for the tools you already use. Those non-goals are stated in the PRD and the concept document for a reason — respect them.

---

## The documents and what each is for

| Document | Use it for |
|----------|------------|
| `README.md` | The public-facing overview. The concept, in brief. |
| `docs/CONCEPT.md` | The idea and the problem it attacks. Read this to understand why the project exists. |
| `PRD.md` | The requirements and success criteria. Read this before writing code that claims to satisfy a requirement. |
| `ARCHITECTURE.md` | The full shape of the system, the data flow, and the boundaries. Read this when you need to know how the pieces fit. |
| `ARCHITECTURE-ESSENTIALS.md` | The short version. Read this first if you are new to the project and only have a little time. |
| `docs/ROADMAP.md` | The plan in phases. Read this to understand which phase a task belongs to and what "done" means for that phase. |
| `docs/BUILD.md` | How to build and run the project itself. Read this when you are setting up or when the build process changes. |
| This file (`AGENTS.md`) | The rules and expectations for agents working on the project. Read this first, every time. |

---

## Rules for agents

### 1. Plan before you build

The project follows a plan-first workflow. Before writing code for a new capability, confirm:

- Which phase of the roadmap it belongs to.
- Whether it is in scope for v1.
- Which document captures the decision, and update that document if the decision is not already recorded.

If a task touches a decision that should be locked — ecosystem, CI system, coverage format, deployment signal, the two v1 queries — stop and confirm the decision before coding. These are listed as open questions in the PRD and the roadmap; they are not things to guess.

### 2. Stay inside the v1 boundaries

v1 scope:

- One ecosystem.
- One CI system.
- One coverage format.
- One deployment signal.
- A SQLite graph store with a documented schema.
- A CLI that answers at least two useful queries.
- A build report as markdown or text.
- Incremental ingestion with per-ingestor pointers.
- Graceful degradation when a source is missing.

v1 non-goals (do not add these in v1):

- A web UI.
- A public API.
- A persisted remote graph.
- Multi-ecosystem support.
- A mandatory per-commit opt-in.
- A separate indexing or compaction step required for basic queries.

If you are tempted to add one of these, stop and ask whether it belongs in a later phase. The PRD non-goals are the guardrail.

### 3. Respect the layer boundaries

The system has three layers and a contract for each:

- **Ingestors** — self-contained, independently skippable, satisfy the ingestor contract. An ingestor must not depend on another ingestor for its own job.
- **Normalization and store** — the schema is the boundary. Once something is a node or edge, it is queried through the schema, not through the ingestor.
- **Query layer and report** — queries are functions against the schema, not against ingestors. The CLI is the entry point.

When you touch one layer, do not reach into another unless the boundary itself is what you are changing and you have a reason for it.

### 4. Make missing sources a skip, not a failure

If a source is missing or unreachable, the responsible ingestor is skipped and logged. It does not fail the run. It does not block the other ingestors. The graph is whatever was obtainable; it is not an all-or-nothing artifact.

When you write or change an ingestor, make sure this is true for it.

### 5. Keep the build document accurate

If your work changes how the project is built or run, update `docs/BUILD.md`. The build document must stay accurate to the code. A reader should be able to build and run the project from this document alone.

### 6. Update the roadmap when a phase changes

If your work changes what a phase contains, what "done" means for it, or the order of phases, update `docs/ROADMAP.md`. The roadmap is the plan; it should reflect the current state of the plan, not a stale one.

### 7. Do not infer the requirements from the code

The requirements are in the PRD. If the code and the PRD disagree, the PRD is the source of truth until it is updated. Do not silently reinterpret a requirement to match existing code.

### 8. Keep the schema honest and minimal

The v1 schema should be honest and minimal. It can be revised as queries reveal what is actually needed. Do not pre-engineer the schema for queries that do not exist yet. Do not pre-engineer it for ecosystems that are not in v1.

When you change the schema, update the schema documentation and make sure existing queries still work, or document why they do not.

### 9. Two queries is the floor, not the ceiling — but validate before you lock

The v1 query surface is at minimum two queries. The current candidates are:

- Subsystem change with rationale.
- Impact / dependency.

Validate these against the target repository before locking them as the v1 surface. If they are not the most valuable two, the v1 surface is wrong.

### 10. Leave the project continuable

A contribution should leave the project in a state where a fresh engineer or agent can continue without re-deriving context. Concretely:

- The code runs against a real or prepared target repository.
- A git ingestor produces a populated store.
- At least one query returns correct, readable output.
- The tests pass.
- Any decision that matters is recorded in a document, not only in the code or in the agent's memory.

---

## What to do on arrival

1. Read `ARCHITECTURE-ESSENTIALS.md`.
2. Read `docs/ROADMAP.md` to see which phase is current.
3. Read `PRD.md` if the task is about satisfying a requirement.
4. Read `docs/BUILD.md` if the task is about building or running the project.
5. If the task is ambiguous, confirm the decision in the relevant document before proceeding. Do not guess at a decision that should be locked.

---

## What not to do

- Do not add a feature without deciding which phase it belongs to.
- Do not add a v1 non-goal without an explicit decision to extend v1 scope (and a record of that decision).
- Do not introduce a UI, an API, a remote graph, or a per-commit opt-in as a "small addition." Those are scope changes, not tweaks.
- Do not silently reinterpret a requirement.
- Do not leave a changed build process undocumented.
- Do not leave a changed roadmap unrecorded.

---

## Success looks like

- The code satisfies a requirement in the PRD, and the requirement is reflected in the code or the document.
- The contribution fits a phase in the roadmap, and the roadmap is accurate to it.
- The layers stay separated.
- The build document is accurate.
- A fresh engineer or agent can continue from where you left off without re-deriving context.

---

*This file is the agent contract for the repository. It is updated when the expectations for agents change. It is not a substitute for the other documents; it points to them.*
