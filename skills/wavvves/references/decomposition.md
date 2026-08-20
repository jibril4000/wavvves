# Decomposition: stories, file ownership, waves

Read this during planning, before showing the user the story table.

## Story schema

A **story** is one agent-sized unit of work. Each story specifies:

| Field | Meaning |
|---|---|
| **ID** | Short stable handle, e.g. `S3`. |
| **Title** | One line. |
| **Wave** | Which wave it runs in (0 = foundation, inline). |
| **Depends on** | Story IDs whose output this needs before it can start. |
| **Model** | `opus` (design/architecture) or `sonnet` (mechanical). Set explicitly. |
| **Test tiers** | Which of unit / integration / human-QA this story delivers. |
| **Owns (files)** | The **exact, non-overlapping** file set this agent may write. |

The single most important field is **Owns (files)**. Two stories must never list the same file.
If two stories want the same file, either merge them, or extract the shared part into a contract
(Wave 0) or an orchestrator-owned file.

## Step 1 — inventory the build

List every file/module the build creates or changes. For each, note whether it is:

- **Leaf work** — implementation an agent can own end-to-end.
- **A contract** — an interface/type/schema other work depends on. → freeze in Wave 0.
- **A bottleneck/shared file** — touched by many stories. → orchestrator-owned, never handed to an agent.

Typical bottleneck files: workspace/monorepo config, root manifest (`package.json`, `Cargo.toml`,
`pyproject.toml`), lockfiles, shared schema, DI/registration barrels, route registries, `index`
re-export files, CI config. The orchestrator wires these at merge time.

## Step 2 — partition by non-overlapping file sets (NOT by "story idea")

Group the leaf work into stories such that each story owns a disjoint file set. Partition by
**file ownership, not by feature description** — "add auth" and "add billing" may both want to edit
the same router file, which makes them *not* parallelizable as written. Split along the seams where
files don't overlap; push the shared seam into a contract or an orchestrator-owned file.

## Step 3 — build the dependency graph and group into waves

1. Draw edges: story B depends on story A if B needs A's output (a type, a module, a built artifact).
2. **Wave 0** = the foundation you build inline: test tooling + CI, fakes/fixtures, frozen contracts,
   and the initial orchestrator-owned scaffolding.
3. **Wave N** = all stories whose dependencies are fully satisfied by waves `< N`. This is a
   topological layering: everything in a wave can run in parallel because nothing in it depends on
   anything else in it.

### Intra-wave dependencies

If two stories land in the same wave but one actually depends on the other, you have two options:

- **Demote** the dependent story to the next wave (simplest, default), or
- Use the native **task dependency graph** (Agent Teams) so the dependent task auto-unblocks when
  its prerequisite completes.

Never let a real dependency hide inside a single parallel wave — that is how contract drift and
half-built interfaces happen.

## Step 4 — right-size and assign models

- Cap each story's scope (N files) so an agent can finish it in one focused session.
- Assign **`sonnet`** to mechanical stories (wrappers, boilerplate, templated CRUD, format
  conversion) and **`opus`** to stories requiring design judgment or that author/extend contracts.
- If a story is too big to state crisply, split it. If it's trivial and non-conflicting, don't make
  it a story at all — the orchestrator does it inline.

## Wave-aggressiveness heuristic

Prefer **fewer, larger, cleanly-separated** stories over many small ones. 3–5 parallel agents per
wave is the empirical sweet spot; coordination overhead grows fast beyond that. More agents is not
more speed once integration and review saturate.

Output of this phase: a filled `templates/story-table.md` plus a one-paragraph wave plan, shown to
the user for confirmation (see SKILL.md §0).
