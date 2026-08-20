# Worktree setup: use native features, don't reinvent

Read this when setting up a wave's worktrees. Prefer Claude Code's built-in worktree mechanics over
hand-rolled `git worktree` scripting — they handle branch naming, cleanup, and secret propagation.

## One worktree per agent, branched off `dev` HEAD

Each parallel agent runs in its **own** git worktree on its **own** branch, cut from the current
`dev` HEAD — which already contains the frozen contracts and Wave 0 scaffolding. Branch naming
convention: `wave<N>/<story-id>` (e.g. `wave2/S3`).

Spawn each agent with `isolation: "worktree"` so it gets an isolated copy of the repo; the worktree
is auto-cleaned if the agent makes no changes.

## Branch off in-progress work, not `main`

By default a fresh worktree is cut from `origin/HEAD` (clean from remote). For a mid-build wave you
want it cut from your **current local HEAD** on `dev` so agents see the contracts. Set:

```
worktree.baseRef: "head"
```

in project settings (or cut the branches yourself from `dev` HEAD before spawning). Otherwise Wave 2
agents will branch off a `dev` that predates your contracts and immediately drift.

## Keyless: `.worktreeinclude` for fixtures, never secrets

Worktrees start clean and don't carry gitignored files. Use a **`.worktreeinclude`** file to copy
the **non-secret** test fixtures and config each agent needs (sample data, fake service config,
`.env.test` with dummy values) into every new worktree automatically:

```
# .worktreeinclude — copied into each worktree
fixtures/**
config/test/**
.env.test
```

**Do not** put real credentials here. Live keys never enter a worktree; real-credential runs happen
only in the orchestrator's human-test handoff.

## Worktrees isolate files — NOT the environment

This is the trap. Separate worktrees still share:

- the same local **databases** and **Redis**,
- the same **ports** (dev servers collide),
- the same external **services** and **browser sessions**.

So for anything stateful:

- Prefer **in-memory / SQLite / fake** implementations for agent work (this is why Wave 0 ships the
  fake harness), **or**
- Serialize all stateful operations through the orchestrator, **or**
- Give each agent an isolated instance / unique port and say so in its prompt.

## Dependencies: install in the base, not N times in parallel

Run the dependency install once on `dev` before cutting worktrees, so agents inherit `node_modules`
(or equivalent) rather than each doing a slow independent install. Instruct agents **not to add new
packages** without orchestrator approval — parallel lockfile edits are a classic bottleneck-file
conflict, and the lockfile is orchestrator-owned.

## Cleanup

After a story's branch is merged onto `dev` and verified green, remove its worktree and delete the
merged branch. Keep the working set small so `git worktree list` stays legible.
