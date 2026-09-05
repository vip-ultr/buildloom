# buildloom - a build-time knowledge graph

Every build emits an artifact. Usually that artifact is a binary. Sometimes it is a test report. Almost never is it a **queryable map of the system's evolution**: what changed, why each decision was made, what depends on what, what is covered by what, and what deployed where. That information exists. It is scattered across git history, CI logs, issue trackers, PR descriptions, and people's heads.

buildloom compiles it into one place as a side effect of building.

---

## What it is

A build-time pipeline that, alongside your normal build output, produces a structured knowledge graph. The graph is persisted, queryable, and cumulative across time. Each build adds nodes and edges; nothing is thrown away.

The graph knows:

- **what changed**: files, functions, modules, with the diff tied to the commit and the rationale from the PR or issue
- **why**: the decision trail behind each change, linked to discussions, designs, and the issues that justified it
- **what depends on what**: not just static imports, but the actual dependency relationships that matter: build dependencies, runtime call relationships, deployment order
- **what's covered**: which tests touch which code, which paths are exercised, where coverage is thin
- **what deployed where**: artifact, environment, version, with the release trail attached

Query it across time:

- "Show me every place that touches the auth flow and was changed in the last 6 months, with the rationale behind each one."
- "If I touch this module, what else should I be worried about?"
- "Where are the decisions documented for this subsystem, and who made them?"

---

## What it is not

- Not a documentation tool. It does not ask anyone to write docs now.
- Not a static dependency visualizer. It is cumulative, queryable, and tied to reasoning and deployment, not just import graphs.
- Not an architecture diagram generator. It is an artifact that accumulates as a side effect of doing real work.
- Not a replacement for the tools you already use. It sits beside them and pulls their output into one queryable fabric.

---

## Why now

Three things make this more tractable than it used to be:

1. **Builds already produce most of the inputs.** Git history, CI logs, coverage reports, deployment manifests, issue trackers. They all exist; they are just not joined.
2. **Graph query is mainstream now.** People are comfortable with graph-shaped answers. The question is getting the graph populated without a separate chore.
3. **The "docs rot" problem is real and expensive.** The teams that still know why their system is the way it are the ones where the knowledge survived someone's memory. A build-time artifact that accumulates the reasoning as a side effect is a direct attack on that fragility.

---

## Shape of the first version

A minimal buildloom is honest about its scope:

- one language or ecosystem to start (pick the one you can instrument without fighting the toolchain)
- read from a small, fixed set of sources: git history, one CI system, one coverage format, one deployment signal
- produce a graph store (even a single SQLite database with a clear schema is enough to start)
- expose a query layer (even if it begins as a CLI that answers one or two useful questions well)
- emit something human-readable (a report, a graph, a timeline) so the value is visible without writing a UI

The point of v1 is not completeness. It is a believable proof that build events can be turned into a cumulative, queryable artifact without becoming a documentation chore.

---

## Non-goal

This is not a product spec. It is a repo for an idea and the code that tries to make it real. Scope, APIs, storage format, and the query surface are all to be discovered by building. The README will grow as the code does.

---

## Status

Placeholder. First real commit incoming.
