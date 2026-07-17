# Agent swarm operating model

GitHub is the source of truth for scheduling. Milestones express integration gates, labels express parallel waves and code ownership, and issue bodies list hard dependencies.

## Scheduler model

| Label | Meaning |
| --- | --- |
| `status:ready` | All dependencies are closed; an agent may claim it. |
| `status:claimed` | Exactly one agent owns the ticket. |
| `status:blocked` | At least one dependency remains open. |
| `status:review` | A branch and pull request are ready for independent review. |
| `parallel-safe` | The ticket has an isolated deliverable suitable for its own agent. |
| `integration-gate` | Serial coordinator task that combines several agent outputs. |
| `wave:*` | Broad concurrency wave. A later wave may still start early when its explicit dependencies are closed. |
| `track:*` | Primary repository or review specialty. |

Explicit `Blocked by` links override labels. A ticket never becomes ready merely because another ticket in the same milestone finished.

## Milestones

1. `M0 — Contract & Harness`: freeze observable behavior, fixtures, platform capability evidence, and build baselines.
2. `M1 — Portable Credential Core`: preserve the complete passkey and implement on-demand PRF evaluation.
3. `M2 — iOS + Android Integrations`: connect the shared core to both operating systems.
4. `M3 — Mobile v0.1 E2E`: prove the real Apple-to-Android Nuri journey and produce test builds.
5. `M4 — Upstream Delivery`: submit focused Bitwarden changes and close review feedback.

The `Program — Mobile v0.1` milestone contains the six parent epics only.

## Starting a swarm

The coordinator lists work that is genuinely available:

```bash
gh issue list \
  --repo nuri-com/bitwarden-passkey-prf \
  --label status:ready \
  --label parallel-safe \
  --state open \
  --limit 100
```

Launch one agent per returned issue, constrained by available compute slots. Give every agent:

- the issue URL;
- the target fork and upstream base commit;
- a dedicated clone or Git worktree;
- the branch name `swarm/<issue>-<slug>`; and
- the instruction to follow this repository's `AGENTS.md` plus the target repository's own instructions.

With four execution slots, run one coordinator and three issue agents. With a larger runner, start every `status:ready` ticket whose expected file set does not overlap another claimed ticket.

## Claim protocol

```bash
gh issue edit ISSUE \
  --repo nuri-com/bitwarden-passkey-prf \
  --add-assignee @me \
  --remove-label status:ready \
  --add-label status:claimed

gh issue comment ISSUE \
  --repo nuri-com/bitwarden-passkey-prf \
  --body "Claimed. Target: nuri-com/REPO. Branch: swarm/ISSUE-SLUG. Expected files: ..."
```

If two agents race, the second agent must stop when it sees the issue already assigned or labeled `status:claimed`.

## Coordinator loop

1. Start every safe ready ticket that fits the slot budget.
2. Review completed branches independently.
3. Merge only after required checks pass.
4. Close the implementation issue.
5. Inspect all issues that list the closed issue under `Blocked by`.
6. When every dependency is closed, replace `status:blocked` with `status:ready`.
7. Run coordinator-owned `integration-gate` issues only after all children close.
8. Refill free slots from the ready queue.

## Agent prompt template

```text
Implement GitHub issue ISSUE_URL.

Work only in TARGET_REPOSITORY on branch BRANCH_NAME, based on UPSTREAM_SHA.
Read the coordination repository AGENTS.md and the target repository instructions first.
Respect the issue scope and expected file ownership. Use synthetic secrets only.
Run the acceptance commands in the issue. Push a signed branch, open a draft PR,
and comment on the issue with evidence and remaining risks. Do not merge.
```

## Integration policy

- Core model and evaluator contracts merge before platform consumers.
- iOS and Android may develop concurrently against pinned draft SDK branches.
- Cross-platform tests consume immutable build or commit identifiers.
- Agents do not paper over a blocked upstream interface with permanent platform-specific side channels.
- Every temporary adapter must have a named removal dependency.
- The final release gate proves two arbitrary PRF inputs. This prevents an implementation from caching only the Nuri production result.

## Scaling safely

More agents help only when tickets own different outputs. Good parallel lanes include:

- synthetic vectors and harnesses;
- shared credential model versus authenticator evaluation;
- iOS transport versus Android transport;
- positive interop versus negative/security testing; and
- implementation versus upstream documentation.

Do not split two agents across the same model file, generated binding, migration, or lockfile. Make that work serial or assign one agent to own the integration gate.
