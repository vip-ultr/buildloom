# Buildloom — Product Requirements Document

**Version:** 1.0  
**Status:** Draft  
**Last updated:** 2026-09-24

---

## 1. Overview

Buildloom is a build-time pipeline that, as a side effect of a normal build, produces a structured, cumulative, and queryable knowledge graph of a software system. It compiles what already exists across git history, CI logs, coverage reports, deployment manifests, and issue trackers into one place — without asking anyone to write documentation.

The core insight: every input Buildloom needs already exists. It is scattered. Buildloom joins it.

The graph knows:

- **What changed** — files, functions, modules, with the diff tied to the commit and the rationale from the PR or issue.
- **Why** — the decision trail behind each change, linked to discussions, designs, and the issues that justified it.
- **What depends on what** — not just static imports, but build dependencies, runtime call relationships, and deployment order.
- **What is covered** — which tests touch which code, which paths are exercised, where coverage is thin.
- **What deployed where** — artifact, environment, version, with the release trail attached.

---

## 2. Problem Statement

Teams accumulate knowledge about their system across git history, CI logs, coverage reports, deployment manifests, issue trackers, PR descriptions, and people's heads. None of these is a queryable map of the system's evolution. The result:

- New engineers spend weeks reconstructing what the code does and why it is the way it is.
- "Who decided this?" has no answer beyond folklore.
- Impact analysis before a change is guesswork — "if I touch this, what else should I worry about?"
- Coverage is a number, not a map showing where the gaps are relative to what changed.
- When a deployment goes wrong, tracing from symptom back to the change that introduced it is manual archaeology.

Documentation tools ask people to write. Buildloom does not. It observes the work already happening and records it.

---

## 3. Goals

1. Make the system's evolution queryable without creating a documentation chore.
2. Produce the graph as a side effect of building — no separate step, no opt-in per commit.
3. Start small and believable: one ecosystem, a few sources, a clear schema, a CLI that answers real questions.
4. Accumulate across time — every build adds nodes and edges; nothing is thrown away.
5. Emit something human-readable so the value is visible without a UI.

---

## 4. User Personas and Use Cases

### 4.1 Onboarding engineer

**Need:** "Show me every place that touches the auth flow and was changed in the last 6 months, with the rationale behind each one."

**Value:** Cuts the time to understand a subsystem from weeks to hours. The rationale trail is attached to the code, not locked in someone's head.

### 4.2 Engineer about to make a change

**Need:** "If I touch this module, what else should I be worried about?"

**Value:** Impact analysis grounded in actual dependency relationships and historical change patterns, not tribal knowledge.

### 4.3 Tech lead / architect

**Need:** "Where are the decisions documented for this subsystem, and who made them?"

**Value:** Decision trail tied to code and commits, not lost in a Slack thread or a PR that got merged and forgotten.

### 4.4 Post-incident investigator

**Need:** Trace from a symptom in production back to the change that introduced it, and see what else that change touched.

**Value:** The deployment trail and the change graph together make this a query, not a hunt.

### 4.5 Coverage-conscious team

**Need:** "Where is coverage thin relative to the code we just changed?"

**Value:** Coverage mapped onto the change graph — not a global percentage, but a localized picture of what changed and what tested it.

---

## 5. Functional Requirements

### 5.1 Ingestors

| ID | Requirement |
|----|-------------|
| F-1 | Git ingestor: read commit history, diffs, file paths, and (where available) PR/issue metadata from the local repository. |
| F-2 | CI ingestor: read build logs and results from at least one CI system (GitHub Actions as the first target). |
| F-3 | Coverage ingestor: read at least one coverage format (lcov / coverage.py JSON as the first target) and map coverage to files and functions. |
| F-4 | Deployment ingestor: read at least one deployment signal (a release tag, a manifest, or a CI deployment log) and attach artifact, environment, and version. |
| F-5 | Each ingestor must be independently runnable and independently failable without breaking the others. |

### 5.2 Graph store

| ID | Requirement |
|----|-------------|
| F-6 | Persist the graph in a single SQLite database with a clear, documented schema. |
| F-7 | The schema must support nodes for: commits, files, functions/modules, PRs/issues, decisions, tests, coverage entries, artifacts, environments, deployments. |
| F-8 | The schema must support edges for: changed_in, references, depends_on, covered_by, deployed_to, and a small number of additional relationship types as needed. |
| F-9 | Every node and edge must carry at least: source, source identifier, timestamp, and a confidence or provenance indicator. |
| F-10 | The database must be append-friendly: builds add to it; they do not rewrite history except where a corrective ingest is explicitly requested. |

### 5.3 Query layer

| ID | Requirement |
|----|-------------|
| F-11 | Provide a CLI that answers at least two useful questions well in v1. |
| F-12 | Query 1 — "What changed in this subsystem over a time window, and why?" Returns files, functions, commits, and linked PR/issue rationale. |
| F-13 | Query 2 — "What depends on this module, and what would I be worried about if I changed it?" Returns direct and transitive dependency relationships with provenance. |
| F-14 | Queries must be expressible from the command line with clear arguments; no UI required. |
| F-15 | Query output must be human-readable (text or structured markdown) without requiring a separate rendering step. |

### 5.4 Build integration

| ID | Requirement |
|----|-------------|
| F-16 | Buildloom must run as a step alongside a normal build, not as a replacement for it. |
| F-17 | The build must succeed or fail on its own terms; Buildloom's failure must not silently swallow the build's outcome. |
| F-18 | Provide a clear integration point for at least one build system (a CI step, a pre/post-build hook, or a Make/just recipe). |

### 5.5 Report emission

| ID | Requirement |
|----|-------------|
| F-19 | Emit a human-readable report (markdown or text) as part of the build output, so the value is visible without a UI. |
| F-20 | The report must show, at minimum: what changed in this build, what it connects to, and any notable gaps (e.g., changed code with no coverage). |

### 5.6 Scope and ecosystem

| ID | Requirement |
|----|-------------|
| F-21 | v1 targets one language or ecosystem, chosen for ease of instrumentation. |
| F-22 | v1 reads from a small, fixed set of sources: git, one CI system, one coverage format, one deployment signal. |
| F-23 | Extending to another ecosystem or source is a documented, separable effort, not a fork of the v1 code. |

---

## 6. Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NF-1 | **Incremental.** A build must not re-ingest the entire history every time. It must add only what is new since the last build. |
| NF-2 | **Fast enough to not be a chore.** The Buildloom step must complete in bounded time for a typical mid-size repository; it must not be the slowest step in the build. |
| NF-3 | **Queryable.** The graph must be queryable immediately after the build completes; no separate indexing or compaction step required for basic queries. |
| NF-4 | **Portable.** The graph store is a single file (SQLite); it can be moved, backed up, or inspected with standard tools. |
| NF-5 | **Debuggable.** When an ingestor fails or produces odd results, there must be a clear log or artifact explaining what it tried and why it failed. |
| NF-6 | **No mandatory network dependency for core operation.** Git ingestion works offline from the local repository. Network-dependent ingestors (CI, deployment) must degrade gracefully when unreachable. |
| NF-7 | **No mandatory per-commit opt-in.** The build produces the graph as a side effect; engineers are not asked to annotate commits or files for Buildloom to work. |

---

## 7. Out of Scope / Non-Goals

- Not a documentation tool. It does not ask anyone to write docs.
- Not a static dependency visualizer. It is cumulative, queryable, and tied to reasoning and deployment, not just import graphs.
- Not an architecture diagram generator.
- Not a replacement for the tools you already use. It sits beside them and pulls their output into one queryable fabric.
- Not a product spec. This repo is for an idea and the code that tries to make it real. Scope, APIs, storage format, and the query surface are all to be discovered by building.
- v1 does not include a web UI, a public API, a persisted remote graph, or multi-ecosystem support.

---

## 8. Success Criteria

| Criterion | How we know |
|-----------|-------------|
| S-1 | A build of a small target repository produces a populated SQLite graph database. |
| S-2 | The CLI answers Query 1 and Query 2 against that database with correct, readable output. |
| S-3 | Running Buildloom again after a new commit adds only the new information; the database grows, it is not rewritten. |
| S-4 | The emitted report shows what changed and what it connects to, in plain text or markdown. |
| S-5 | A failing ingestor (e.g., no coverage file present) does not prevent the other ingestors from running or the build from completing. |
| S-6 | The codebase is structured so that adding a second ecosystem or a second CI system is a separable effort, not a rewrite. |

---

## 9. Open Questions

- **Ecosystem for v1:** Which language/ecosystem is the easiest to instrument without fighting the toolchain? The working assumption is Python (coverage.py, clear diff tooling, easy CI integration), but this should be confirmed against a real target repository.
- **CI system for v1:** GitHub Actions is the likely first target; confirm against the target repo's actual CI.
- **Coverage format for v1:** lcov vs. coverage.py JSON — pick the one that maps most cleanly to functions/modules for the target ecosystem.
- **Deployment signal for v1:** What is the smallest deployment signal that is still meaningful? A git tag plus a manifest entry may be enough; a full deployment log is phase 2.
- **Query surface for v1:** Two queries is the floor; which two are most valuable against a real repository? The current candidates are listed in F-12 and F-13 and should be validated against real use.
- **Name scope:** "buildloom" may collide with other projects. The local repo is `buildloom-local` and the remote is `vip-ultr/buildloom`. Confirm whether the name is usable as a public project name before any broader publication.

---

*This document is a living requirements artifact. It grows as the code does. The README captures the concept; this document captures what the code must do.*
