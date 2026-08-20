# Human test plan (approval gate handoff)

Produce this when a batch of waves is integrated and `dev` is green end-to-end. It is the gate
between `dev` and `main`. Fill it from the per-story manual-test notes plus anything that could only
run with real credentials (which never executed in worktrees).

---

## Build under test
- Branch: `dev` @ `<commit>`
- Waves included: `<list>`
- Stories included: `<IDs>`

## Automated status (for context — already green)
- Root build: ✅
- Unit tiers: ✅  · Integration tiers: ✅
- What automation does NOT cover: <the human/visual + real-credential items below>

## Manual / visual QA checklist
> Aggregated from each story's manual-test note. These are things tests genuinely can't verify —
> green ≠ correct.

- [ ] `<story S_>` — <action to perform> → <expected result>
- [ ] `<story S_>` — <action to perform> → <expected result>

## Real-credential checks (never ran in worktrees)
> The keyless agents built against fakes. These paths must be exercised once against real services.

- [ ] <flow requiring live key/service> → <expected result>

## Known assumptions & risks
- <assumptions agents made that a human should sanity-check>
- <any parked/red stories deliberately excluded from this batch>

## Approval
- [ ] **Approved** → orchestrator fast-forwards `main` to `dev`.
- [ ] **Rejected** → findings below become new stories for the next wave.

Findings:
- ...
