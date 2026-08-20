# Story table template

Fill this in during planning and show it to the user for confirmation before any fan-out
(SKILL.md §0). One row per story. **The "Owns (files)" column must be non-overlapping across rows.**

## Wave plan (one paragraph)

> _Example:_ Wave 0 (inline): CI + vitest harness, fake API + fixtures, frozen `types/contracts.ts`,
> root manifest & barrel scaffolding. Wave 1 (3 agents, parallel): data layer, API client, UI
> primitives — all file-disjoint, all against contracts. Wave 2 (2 agents): feature screens + server
> routes, depend on Wave 1. Orchestrator integrates onto `dev` after each wave; human approves before
> `main`.

## Cost declaration (state before fan-out)

Sub-agents cost tokens because each one re-loads its own context and generates its own work — you pay
for shared context once *per agent*, plus your orchestration + integration. Rule of thumb: multi-agent
≈ **15× a plain chat**, ≈ **4× a single agent** doing the same work. So estimate, then justify the
multiple against building inline.

**Method:** for each agent, `context tokens` (contracts + fixtures + its file set + prompt) + `output
tokens` (the work it generates). Sum across agents, add your own orchestration/integration overhead.

| Wave | Agents | Models | Est. context/agent | Est. output/agent | Wave est. | Inline est. | Multiple |
|------|--------|--------|--------------------|-------------------|-----------|-------------|----------|
| __   | __     | __×sonnet __×opus | ~__k | ~__k | **~__k** | ~__k | ~__× |

- Only fan out if the work is genuinely parallelizable AND the multiple is worth the wall-clock win.
- Report actuals afterward by counting transcripts on disk (recursive helpers don't report up).

## Stories

| ID | Title | Wave | Depends on | Model | Test tiers | Owns (files) |
|----|-------|------|-----------|-------|-----------|--------------|
| S0 | Foundation (inline) | 0 | — | opus (you) | — | CI, test harness, fakes/fixtures, `contracts.*`, shared scaffolding |
| S1 |  | 1 | S0 | sonnet | unit, integration | `path/a/**` |
| S2 |  | 1 | S0 | sonnet | unit | `path/b/**` |
| S3 |  | 1 | S0 | opus | unit, integration | `path/c/**` |
| S4 |  | 2 | S1, S3 | sonnet | integration, human-QA | `path/d/**` |

**Orchestrator-owned (no agent touches these):** workspace/root manifest, lockfile, shared schema,
registration barrels / route registry / DI container, CI config.

## Checklist before you fan out

- [ ] Every "Owns (files)" set is disjoint from every other.
- [ ] Contracts are frozen (Wave 0 merged on `dev`).
- [ ] Each story's dependencies are satisfied by earlier waves.
- [ ] Models assigned explicitly per story.
- [ ] Agent count + rough cost stated and user confirmed.
