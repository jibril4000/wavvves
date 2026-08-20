---
name: wavvves
description: >
  Orchestrates a large build across parallel sub-agents in isolated git worktrees using a
  contracts-first, wave-based methodology. Use when decomposing a substantial multi-file build
  into agent-sized "stories," running independent stories in parallel worktrees, and integrating
  them onto a dev branch behind a human approval gate. Triggers on "parallel build," "wave build,"
  "orchestrate agents," "worktree build," "fan out the build," or planning a big multi-package feature.
allowed-tools: >
  Bash(git worktree*) Bash(git branch*) Bash(git checkout*) Bash(git switch*)
  Bash(git merge*) Bash(git rebase*) Bash(git status*) Bash(git log*) Bash(git diff*)
license: MIT
metadata:
  version: "1.0.0"
---

# wavvves — parallel wave build

An orchestrator plays lead: it plans a build, decomposes it into agent-sized **stories**, groups
them into **waves** by dependency, runs independent stories as **parallel sub-agents in isolated
git worktrees**, then integrates their branches onto `dev` in dependency order and hands the human
a test plan. You are that orchestrator.

The whole method rests on one idea: **parallel agents can only be safe if they code against frozen
contracts and never touch the same files.** Everything below serves that.

## 0. STOP — confirm before any fan-out

This workflow spawns multiple paid sub-agents. Before launching *any* wave of agents you MUST:

1. Show the user the **story table** and the **wave plan**.
2. State the **expected agent count**, the **model per agent**, and a **rough token cost**.
3. Get explicit confirmation.

Never fan out silently. After a wave finishes, report **actual** spend by counting the transcripts
on disk, not the agents visible in your context — recursive helpers never report back, so
context-based counts undercount. (Agents are told not to recurse; see the agent template.)

**Sub-agents cost tokens — a lot.** Each sub-agent loads its own copy of context (contracts,
fixtures, file tree, its prompt) and generates its own work, so you pay for shared context *once per
agent*, plus your own orchestration + integration overhead. As a rule of thumb from Anthropic's
measurements: a multi-agent run burns **~15× the tokens of a plain chat**, and **~4× a single agent
doing the same work**. Parallelism buys wall-clock speed and isolation, paid for in tokens — which is
exactly why fan-out is gated. Estimate a wave as **Σ(per-agent context + per-agent output) + your
orchestration/integration**, then compare it to what building the same stories inline would cost. If
the wave isn't ~3–4× a single agent's cost *and* genuinely parallelizable, do it inline. See
`templates/story-table.md` for the worked estimate format.

## 1. Should you even use this? (pre-flight gate)

Use parallel waves ONLY if a majority of these are true. If not, **build inline yourself** — a
single-agent plan is the correct output when the answer is "no."

- [ ] The build is large enough that setup + merge overhead pays off (each worktree costs an install and a merge).
- [ ] Work splits into **file-disjoint** chunks — stories that own non-overlapping file sets.
- [ ] The shared **interfaces/contracts can be frozen first** (Wave 0), so parallel work codes against stable types.
- [ ] Each chunk is independently specifiable (clear inputs, outputs, acceptance criteria).
- [ ] You (or the human) have **review bandwidth** to verify output at parallel speed — verification, not generation, is the bottleneck.

Coding with heavy cross-file interdependencies is a *poor* fit for multi-agent today. Be honest.

**Hybrid, not dogmatic:** small non-conflicting edits and *all integration* are done inline by the
orchestrator. Reserve worktree agents for substantial, independent chunks.

## 2. The method at a glance

```
Plan → Stories → Waves → [Wave 0 inline] → fan out wave → integrate → next wave → human approval → main
```

1. **Plan** the build. Decompose into stories. → `references/decomposition.md`
2. **Wave 0 (Foundation)** — you do this INLINE, never parallelized (§3).
3. For each later wave: **confirm → fan out agents in worktrees → collect reports → integrate onto `dev`.**
4. Keep `dev` **always green**. Never merge red.
5. Hand the human a **test plan**; fast-forward `main` only after they approve. → `references/branch-integration.md`

## 3. Wave 0 (Foundation) — inline, a hard prerequisite

Do this yourself before any agent runs. Wave 0 lands, on `dev`:

- **Test tooling + CI** — the harness every story's tests will run in.
- **Mock/fake harness + fixtures** — so agents build & test with **no live credentials** (see §6).
- **Frozen shared contracts** — the interfaces/types/schemas that decouple stages so they can be
  built in parallel. This is the #1 defense against merge hell. **Add optional fields later; never
  rename or remove.** Agents code against these and must not change them.

Also decide **file ownership** now: list the **bottleneck/shared files** (workspace config, root
manifest, lockfile, shared schema, registration barrels). These are **orchestrator-owned** — agents
never touch them; you wire them at merge time. → algorithm in `references/decomposition.md`.

## 4. The per-wave execution loop

For each wave after 0:

1. **Confirm** (see §0).
2. **Set up worktrees.** Each agent gets its own worktree branched off the current `dev` HEAD (which
   already holds the contracts). Use native flags, don't reinvent them. → `references/worktree-setup.md`
3. **Launch agents** with the standard prompt: keyless, no-recursion, scope-capped, owns only its
   file set, must deliver its declared test tiers, reports in the fixed format. → `templates/agent-prompt.md`
4. **Collect reports** and enforce the **Definition of Done** (§7).
5. **Integrate** the branches onto `dev` in dependency order, run the **full root build + test green**,
   resolve semantic conflicts yourself, wire the orchestrator-owned shared files. → `references/branch-integration.md`
6. **Circuit breaker:** if an agent loops on the same error (~3 failed iterations) or burns ~85% of
   its budget, stop it, salvage what's usable, and finish that story inline or re-scope it.

## 5. Roles: who owns what

- **Orchestrator (you):** Wave 0, all shared/bottleneck files, all integration & merges, conflict
  resolution, the full-repo green build, the human handoff. Model: **top tier (Opus)**.
- **Agents:** build only within their owned file set, against frozen contracts and fakes. They
  **report**; they never merge. Do not let them touch shared files or spawn sub-agents.

## 6. Keyless-by-default agents

Agents build and test against **fakes + fixtures — no live credentials in worktrees.** Copy the
non-secret test fixtures/config into each worktree automatically with **`.worktreeinclude`**
(never real secrets). Real-credential runs happen **only** in the orchestrator's human-test handoff.
Worktrees isolate files but **not** databases, ports, or running services — so stateful services
must be faked or serialized through the orchestrator. → `references/worktree-setup.md`

## 7. Three honest test tiers + Definition of Done

Each story declares which tiers it delivers:

- **Unit** — fast, keyless, against fakes.
- **Integration** — real tooling, synthetic inputs, CI-friendly (catches "passes-locally-breaks-merged").
- **Human / visual QA** — what tests genuinely can't cover. Don't pretend green = correct.

**Definition of Done** (orchestrator enforces before accepting a story): builds clean · lints ·
declared test tiers green **with pasted output** · no cross-package type breaks · files-touched list ·
a written **manual-test note** · assumptions/blockers surfaced. → format in `templates/agent-prompt.md`.

## 8. Model tiering + fan-out discipline

- **Cheap tier (Sonnet)** for mechanical work: wrappers, boilerplate, templates, format conversion.
- **Top tier (Opus)** for design/architecture/contract work and the orchestrator itself.
- Set each agent's model **explicitly** per story in the table — inheriting the parent's tier makes
  every mechanical agent expensive.
- Cap the work per agent (N files, N stories) so cost scales with a number you chose.

## 9. Human approval gate

Branch model: **`main` = approved & green; `dev` = completed green work awaiting human test-approval.**
When a batch of waves is integrated and `dev` is green, hand the human a **test plan** covering the
human/visual-QA tier and any real-credential checks. Fast-forward `main` only after they approve.
→ `templates/human-test-plan.md`, `references/branch-integration.md`.

## Reference & template map

Load these when you reach the matching phase:

- `references/decomposition.md` — story schema, wave-grouping & file-ownership algorithm.
- `references/worktree-setup.md` — native worktree flags (`isolation: worktree`, `.worktreeinclude`, `baseRef: head`), env isolation caveats.
- `references/branch-integration.md` — `main`/`dev` model, merge order, conflict rules, DoD enforcement.
- `templates/story-table.md` — the story table to fill in and show the user.
- `templates/agent-prompt.md` — the standard keyless / no-recursion / scoped / DoD / report prompt.
- `templates/human-test-plan.md` — the approval-gate handoff.
