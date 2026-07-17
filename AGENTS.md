# Agent swarm rules

Read `README.md`, `docs/nuri-prf-contract.md`, `docs/plan.md`, and `docs/swarm.md` before claiming work.

## Claiming work

- Claim only an issue carrying `status:ready` and `parallel-safe`.
- Issue #68 is the sole exception: exactly one program coordinator claims it even though it is not `parallel-safe`.
- Until the real-device golden journey in #45 is green, claim only `priority:mvp-critical` work. `priority:post-proof` is parked unless a proven blocker makes it unavoidable.
- One agent owns one issue, one branch, and one pull request at a time.
- Atomically replace `status:ready` with `status:claimed`, assign yourself, and comment with the target repository, branch, and files you expect to touch.
- Do not start an issue with an open item in its `Blocked by` section.
- `integration-gate` issues are coordinator-owned and must not be claimed by an implementation agent.
- Functional proof outranks cleanup. Do not add cosmetic work, speculative abstractions, general provider support, desktop work, or upstream packaging to an MVP-critical branch.

## Branch and repository boundaries

- Shared Rust SDK work belongs in `nuri-com/sdk-internal`.
- Apple platform work belongs in `nuri-com/ios`.
- Android platform work belongs in `nuri-com/android`.
- Contracts, synthetic vectors, swarm documentation, and cross-repository evidence belong here.
- Use branches named `swarm/<issue-number>-<short-slug>`.
- Never mix unrelated cleanup, formatting, dependency updates, or generated-file churn into an issue branch.
- If another claimed issue overlaps your intended files, stop and coordinate before editing.

## Security

- Use synthetic credentials only.
- Never commit or log a real passkey key, CXF payload, PRF/HMAC seed, evaluated PRF output, or derived wallet private key.
- PRF results are evaluated on demand and are not persisted.
- Preserve fail-closed behavior for missing PRF state, credential mismatches, unsupported keys, and missing required user verification.
- Zeroize temporary secret buffers where the host language and existing architecture permit it.

## Handoff

- Run the focused tests named in the issue.
- Comment with exact commands, results, changed files, remaining risks, and the pull-request URL.
- Replace `status:claimed` with `status:review` only when the branch is pushed and independently reviewable.
- Do not merge your own pull request unless the coordinator explicitly delegates integration.
- Upstream Bitwarden pull requests must be generic, tested, and contain no Nuri-specific runtime behavior.
