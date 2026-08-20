# Branch model, integration, and DoD enforcement

Read this when collecting agent reports and merging a wave. The orchestrator owns everything here;
agents never merge.

## Branch model

- **`main`** — approved & green. Only ever advanced by a fast-forward after the human approves.
- **`dev`** — accumulates completed, green work awaiting human test-approval. Always green; never
  merge red into it.
- **`wave<N>/<story-id>`** — one per agent, cut from `dev` HEAD, deleted after merge.

```
main ──────────────────────────●(ff after human approval)
                               ╱
dev  ──●──●──●──●──●──●────────●  (each ● = a story merged green)
        ╲   ╲       ╲
   wave1/S1 wave1/S2 wave2/S4   (agent branches, merged in dependency order, then deleted)
```

## Enforce Definition of Done before accepting a story

For each returned story, confirm the agent's report contains and the branch actually satisfies:

- [ ] Builds clean.
- [ ] Lints clean.
- [ ] Declared test tiers green — **with pasted output**, not a claim.
- [ ] No cross-package / cross-module type breaks.
- [ ] Files-touched list matches the story's owned file set (no stray edits to shared files).
- [ ] A written manual-test note (what a human should click/verify).
- [ ] Assumptions & blockers surfaced.

If any item fails, the story is not done. Send it back with specific feedback, or salvage inline.
Optionally wire a **`TaskCompleted` hook** (exit code 2 blocks completion) to enforce "tests green"
mechanically rather than trusting the self-report.

## Merge order and procedure

1. **Order by dependency.** Merge prerequisite stories before their dependents. Within a wave, order
   is usually free, but merge contract-adjacent stories first.
2. For each branch: merge onto `dev`, then immediately run the **full root build + full test suite**.
   Never batch several merges and test once — you lose the ability to attribute a break.
3. **Resolve semantic conflicts yourself.** Textual merges can succeed while the *meaning* breaks
   (two agents implemented the same contract differently, duplicated a helper, diverged on naming).
   This is the orchestrator's core job and the reason agents don't self-merge.
4. **Wire the orchestrator-owned shared files** now: add the new modules to the registration barrel /
   route registry / DI container, update the root manifest, reconcile the lockfile. Agents never
   touched these; you connect their work here.
5. Only when `dev` is green again do you move to the next merge.

## Never merge red

If a branch can't be made green quickly, do not merge it. Park it (keep the branch), fix forward or
re-scope, and continue with the rest of the wave. A red `dev` poisons every subsequent story's
integration test and destroys the "always shippable" guarantee.

## Human approval gate → `main`

When a batch of waves is integrated and `dev` is green end-to-end:

1. Produce the **human test plan** (`templates/human-test-plan.md`): the human/visual-QA items each
   story flagged, plus any real-credential checks (which never ran in worktrees).
2. The human executes it against `dev`.
3. On approval, **fast-forward `main` to `dev`**. On rejection, capture the findings as new stories
   and run another wave.

`main` therefore only ever contains work a human has actually approved — green tests alone never
promote code to `main`.
