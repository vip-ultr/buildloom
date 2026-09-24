# Buildloom — Roadmap

**Version:** 1.0  
**Status:** Draft  
**Last updated:** 2026-09-24

---

## How to read this

This is a working roadmap, not a commitment schedule. It is ordered by dependency and value, not by date. Phases are separable; each phase should leave the project in a state where a fresh engineer or agent can continue without re-deriving context.

The goal of the roadmap is to get to a believable v1 proof as directly as possible: one ecosystem, a few sources, a real schema, two real queries, and a report that shows the value.

---

## Phase 0 — Lock the target and the choices

**Goal:** Before writing code, confirm the decisions that the code depends on. This phase is about de-risking, not drafting.

- Pick the v1 target repository. Prefer a real one the team already works with, so the value of the queries can be felt early.
- Confirm the v1 ecosystem. Working assumption: Python (coverage.py, clear diff tooling, easy CI integration). Confirm against the target.
- Confirm the v1 CI system. Working assumption: GitHub Actions. Confirm against the target.
- Confirm the v1 coverage format. Working assumption: coverage.py JSON or lcov. Pick the one that maps most cleanly to functions/modules for the target.
- Confirm the v1 deployment signal. Working assumption: a git tag plus a manifest entry. Validate that this is meaningful enough for the target.
- Validate the two v1 queries against the target repository by hand first — can you answer "what changed in this subsystem and why" and "what depends on this module" using existing tools? If not, the queries are the wrong ones.

**Deliverable:** A short note in `docs/BUILD.md` or a separate decision log capturing each confirmed choice and the reason. No code yet.

---

## Phase 1 — Skeleton and schema

**Goal:** A repository that runs, a schema you can read, and a graph file you can open.

- Set up the project skeleton: a Python package, a clear entry point, a way to run Buildloom as a step.
- Define the v1 schema in code and in writing. At minimum: nodes for commits, files, functions/modules, PRs/issues, tests, coverage entries, artifacts, environments, deployments; edges for changed_in, references, depends_on, covered_by, deployed_to.
- Implement the git ingestor first. It is the one ingestor that must work offline from a local repository; it is the baseline that makes the tool useful before any CI integration.
- Implement a store that accepts nodes and edges and persists them to a single SQLite file.
- Implement a first naive normalization path: git records in, nodes and edges out.

**Deliverable:** Run Buildloom against a local repository and get a populated `graph.db` you can open with a standard SQLite tool. The schema is documented. The git ingestor works incrementally (a second run adds only what is new).

---

## Phase 2 — Two real queries and a report

**Goal:** The tool answers real questions and shows its value in plain text.

- Implement Query 1 — subsystem change with rationale. Given a path or set of paths and a time window, return files, functions, commits, and linked PR/issue rationale in readable output.
- Implement Query 2 — impact / dependency. Given a module or file, return what depends on it, with provenance and the nature of each relationship.
- Implement the build report: a markdown or text summary of what this build added and what it connects to, written alongside the build artifacts.
- Hook up the CLI so these are reachable from the command line with clear arguments.

**Deliverable:** A human can run the CLI against the graph from Phase 1 and get correct, readable answers to both queries, plus a build report. The value is visible without a UI.

---

## Phase 3 — Remaining ingestors and build integration

**Goal:** The graph is no longer just git. It is joined from the other sources, and it runs as part of a build.

- Implement the CI ingestor for the confirmed v1 CI system. It reads build runs, results, and log pointers.
- Implement the coverage ingestor for the confirmed v1 coverage format. It maps coverage to files and functions.
- Implement the deployment ingestor for the confirmed v1 deployment signal.
- Wire the ingestors together so one build step runs them all, each independently skippable, each updating its own pointer.
- Integrate with the target build: a CI step, a Make/just recipe, or a pre/post-build hook. The build's success or failure is its own concern; Buildloom's failure does not silently swallow the build outcome.

**Deliverable:** A build of the target repository produces a graph joined from git, CI, coverage, and deployment. Running Buildloom again after a new commit adds only the new information.

---

## Phase 4 — Hardening and local developer experience

**Goal:** The tool is something a developer can use by hand, and it survives real-world misses gracefully.

- Make sure every ingestor degrades gracefully when its source is missing or unreachable, with a clear log of what it tried and why it failed.
- Ensure the graph store is queryable immediately after a build; no separate indexing or compaction step required for basic queries.
- Ensure the incremental ingestion is fast enough that the Buildloom step is not the slowest thing in the build. Measure on the target repository.
- Provide a clear local development path: an engineer can run Buildloom by hand against their local repository and get a graph and a report without any CI involvement.
- Document the schema well enough that a reader understands what a node and an edge mean without reading the ingestors.

**Deliverable:** A developer can use the tool locally, the step is fast and bounded, and failures are debuggable.

---

## Phase 5 — Ecosystem extensibility and query growth

**Goal:** Adding a second ecosystem or a second CI system is a separable pinch, not a rewrite. The query surface grows beyond two.

- Restructure so that adding a new ingestor for a new ecosystem or source is a separable effort. The ingestor contract is the guardrail.
- Add queries beyond the two v1 queries, driven by real questions from the target repository.
- Decide whether a small human-readable visual artifact (a graph, a timeline, a dependency map) adds value beyond the report. Keep it optional and text-first.

**Deliverable:** A second ingestor can be added without touching the existing ones. The query surface is wider. The project is no longer a one-ecosystem proof.

---

## What is not on the roadmap

These are explicitly deferred and, in v1, out of scope:

- A web UI.
- A public API.
- A persisted remote graph.
- Multi-ecosystem support in v1.
- A mandatory per-commit opt-in.
- A separate indexing or compaction step required for basic queries.
- Any departure from the PRD non-goals.

---

## Success gates between phases

Use these to tell whether a phase is actually done, not just "written."

- **After Phase 1:** A local run produces a populated, openable `graph.db`. The schema is documented. The git ingestor is incremental.
- **After Phase 2:** Two queries return correct, readable answers. A build report is emitted.
- **After Phase 3:** A full build produces a graph joined from all four sources. A second run is incremental.
- **After Phase 4:** Local use works. Failures are debuggable. The step is fast enough on the target.
- **After Phase 5:** A new ingestor is separable. The query surface has grown.

---

## Open questions that shape the roadmap

- What is the real target repository for v1? The roadmap assumes one exists and is known before Phase 1 code is written.
- Which two queries are most valuable against that repository? The current candidates are in the PRD and must be validated before they are locked as the v1 surface.
- Which ecosystem, CI system, coverage format, and deployment signal does that repository actually use? The working assumptions are Python, GitHub Actions, coverage.py JSON or lcov, and tag plus manifest — confirm before coding Phase 3.

Answer these before Phase 1 is finalized. They are not things to discover during coding; they are things to lock so the code does not have to be redone.

---

*This roadmap is live. It is rewritten as the code and the target repository teach us something. The phase boundaries are the commitment: each phase should leave the project continuable by a fresh engineer or agent.*
