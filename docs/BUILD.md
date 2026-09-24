# Buildloom — Build

**For:** Anyone building the project itself, including agents and new engineers.  
**Read alongside:** `ARCHITECTURE-ESSENTIALS.md` (the short version of the system), `ROADMAP.md` (what we build first).

---

## Prerequisites

- **Python 3.11 or newer.** The project targets a modern Python; no older versions are supported in v1.
- **A target repository to point it at.** Buildloom is not useful in a vacuum; it needs a repository with git history, and ideally CI logs, coverage, and a deployment signal. For development and testing, a small real repository or a prepared fixture is enough.
- **Git.** The git ingestor is the baseline ingestor and must work against any local git repository.
- **SQLite.** Bundled with Python's standard library; no separate installation is required.

No external services, no network access, and no credentials are required to run the core of Buildloom locally. Network-dependent ingestors (CI, deployment) degrade gracefully when unreachable.

---

## Layout

```
buildloom/
  buildloom/                 # package
    __init__.py
    cli.py                   # CLI entry point
    ingest/                  # ingestors
      __init__.py
      git_ingestor.py
      ci_ingestor.py
      coverage_ingestor.py
      deployment_ingestor.py
    store/                   # graph store
      __init__.py
      schema.py              # schema definition
      db.py                  # connection + CRUD
    normalize/               # normalization + graph construction
      __init__.py
      normalizer.py
    queries/                 # query layer
      __init__.py
      queries.py
    report.py                # report emission
  tests/
    conftest.py
    ...                      # tests per layer, plus fixtures
  docs/
    BUILD.md
    ROADMAP.md
    CONCEPT.md
  docs/BUILD.md              # this file (also at repo root for the project's own build)
  PRD.md
  ARCHITECTURE.md
  ARCHITECTURE-ESSENTIALS.md
  AGENTS.md
  README.md
  ...
```

This layout is the working target for v1. It is described here so a fresh engineer or agent can navigate without re-deriving it. It may be refined as the code grows; the principle is that each layer has a home and a clear boundary.

---

## Getting started

1. Clone the repository.
2. Create and activate a virtual environment.
3. Install the project in editable mode with its dev dependencies.
4. Point Buildloom at a target repository.
5. Run the CLI.

The concrete commands below assume the layout above and a `pyproject.toml` with a `buildloom` package and a CLI entry point. Adjust paths if the layout is refined.

### First run against a local repository

The git ingestor works offline from a local repository and is the first thing to try:

```
# From the project root
buildloom run --repo /path/to/target-repo --store graph.db
```

This ingests git history into `graph.db` (creating it if it does not exist), using an incremental pointer so a second run adds only what is new.

### Running the queries

Once there is a store with data:

```
buildloom query subsystem-changes \
  --store graph.db \
  --path src/auth \
  --since 2026-01-01

buildloom query impact \
  --store graph.db \
  --target src/auth/user.py
```

Both return human-readable output. No running service is required.

### Emitting a build report

```
buildloom report --store graph.db --output buildloom-report.md
```

This writes a markdown summary of what the store contains and, where applicable, what a build added.

---

## Running the tests

The project is tested per layer. Ingestor tests use fixtures; store tests use an in-memory or temporary SQLite database; query tests use a small pre-populated graph.

```
# From the project root
pytest
```

A good first run targets the git ingestor and the store, because those are the layers that must work before the rest can be meaningfully tested:

```
pytest buildloom/ingest/test_git_ingestor.py buildloom/store/test_db.py
```

---

## What "done" looks like for a first contribution

A contribution that leaves the project in a continuable state:

1. The code runs against a real or prepared target repository.
2. A git ingestor produces a populated store.
3. At least one query returns correct, readable output.
4. The tests pass.
5. The thing that was changed or added is reflected in this document if it changes how someone builds or runs the project.

---

## Incremental development

When you add an ingestor:

- It must satisfy the ingestor contract: run(scope) -> records + new pointer, an availability check, and no dependency on another ingestor for its own job.
- It must degrade gracefully when its source is missing, and leave a clear log of what it tried.
- It must not require changes to the store schema unless the schema genuinely needs to grow to represent something new.

When you add a query:

- It is a function against the schema, not against an ingestor.
- It returns human-readable output.
- It is reachable from the CLI with clear arguments.

When you change the schema:

- Update the schema documentation.
- Make sure existing queries still work, or document why they do not.

---

## Running Buildloom as a build step

For a target repository that uses Make or just, a recipe like this is the working model:

```
build:
    # ... the normal build ...

buildloom:
    buildloom run --repo . --store .buildloom/graph.db
    buildloom report --store .buildloom/graph.db --output buildloom-report.md

after-build: build buildloom
```

For CI, the model is a step that runs after the build and coverage jobs, pointing at the repository and the artifacts the build produced.

The build's success or failure is its own concern. Buildloom's failure does not silently swallow the build outcome. If Buildloom fails, that should be visible — but it should not make a successful build look like a failure.

---

## Configuration

In v1, configuration is minimal and explicit. The CLI takes what it needs as arguments: the repository path, the store path, the query, the targets, the time window. Ingestors take what they need from the environment or from the target repository's existing artifacts. There is no hidden configuration file that a reader has to hunt down to understand why a run produced a particular result.

If configuration grows, it must remain explicit and readable. A reader should be able to tell what a run did from the command and the store, without guessing.

---

## What to do if something is missing

- No coverage file? The coverage ingestor is skipped and logged. The rest still runs.
- No CI access? The CI ingestor is skipped and logged. The rest still runs.
- No deployment signal? The deployment ingestor is skipped and logged. The rest still runs.
- A git repository with no PR metadata? The git ingestor still ingests commits and diffs. The rationale link is simply absent where it would otherwise come from a PR.

The principle: a missing source is a skipped ingestor, not a failed run. The graph is whatever was obtainable; it is not an all-or-nothing artifact.

---

## Debugging

- The store is a single SQLite file. You can open it with any SQLite tool and inspect the nodes and edges directly.
- Each ingestor should log what it tried and what it produced, so a strange result is traceable to a source.
- Incremental pointers are stored explicitly, so you can tell whether an ingestor ran again or skipped.
- When a query returns something unexpected, the first check is the store contents, not the query logic. The schema is the boundary; if the data is wrong in the store, the ingestor or normalizer is the likely cause.

---

## Notes for agents

- Read `ARCHITECTURE-ESSENTIALS.md` first; it is the short version you can absorb in one sitting.
- Read `ROADMAP.md` to understand which phase a task belongs to and what "done" means for that phase.
- Read `PRD.md` for the requirements and success criteria; do not infer them from the code alone.
- Do not add a feature without deciding which phase it belongs to and whether it is in or out of v1 scope.
- Keep the layers separated. An ingestor change should not require a query change. A query change should not require a schema change unless the schema genuinely needs to grow.
- Do not introduce a web UI, a public API, a remote graph, or a mandatory per-commit opt-in. Those are out of scope for v1 and are listed as non-goals in the PRD for a reason.
- When you finish, update this document if the way to build or run the project changed. The build document must stay accurate to the code.

---

*This document is the how-to for the project itself. It is updated when the build process changes. The README is the public-facing overview; this is the working surface for the people and agents who build it.*
