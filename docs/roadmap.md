# Mobile v0.1 swarm roadmap

## Outcome

Preserve the same PRF-capable passkey from Apple Passwords through Bitwarden iOS import, encrypted Bitwarden sync, and Bitwarden Android use, then prove that a wiped Nuri Android install recovers the same wallet identities.

The stored object is the passkey, including its provider-internal PRF/HMAC seed state. Evaluated PRF outputs are computed during authorized ceremonies and are never persisted.

The immediate target is a working physical-device proof, not a polished or generally complete password-manager release. Start at [issue #68](https://github.com/nuri-com/bitwarden-passkey-prf/issues/68); it is the claimable orchestrator task and the authoritative proof-first scope.

## GitHub structure

| Milestone | Tickets | Gate |
| --- | ---: | --- |
| [Program — Mobile v0.1](https://github.com/nuri-com/bitwarden-passkey-prf/milestone/1) | 6 epics + [claimable orchestrator](https://github.com/nuri-com/bitwarden-passkey-prf/issues/68) | Program-level outcome and proof-first execution |
| [M0 — Contract & Harness](https://github.com/nuri-com/bitwarden-passkey-prf/milestone/2) | 11 | Frozen vectors, platform contracts, baseline builds, and a secret-safe continuity harness |
| [M1 — Portable Credential Core](https://github.com/nuri-com/bitwarden-passkey-prf/milestone/3) | 14 | MVP gate: CXF import through encrypted sync, reload, and arbitrary-input PRF assertion; export follows after proof |
| [M2 — iOS + Android Integrations](https://github.com/nuri-com/bitwarden-passkey-prf/milestone/4) | 14 | Both platform transports preserve and use the shared credential correctly |
| [M3 — Mobile v0.1 E2E](https://github.com/nuri-com/bitwarden-passkey-prf/milestone/5) | 7 | Real Apple-to-fresh-Android Nuri recovery with release-like builds |
| [M4 — Upstream Delivery](https://github.com/nuri-com/bitwarden-passkey-prf/milestone/6) | 7 | Focused server, SDK, iOS, and Android contributions submitted upstream |

Milestones are integration gates, not rigid time boxes. A later-milestone ticket can start as soon as every explicit dependency closes.

## Initial proof-first parallel wave

These nine `priority:mvp-critical` tickets have no dependencies and can be assigned immediately to independent agents:

- [#7 — Synthetic CXF and PRF vectors](https://github.com/nuri-com/bitwarden-passkey-prf/issues/7)
- [#56 — Encrypted extension-state and mixed-version contract](https://github.com/nuri-com/bitwarden-passkey-prf/issues/56)
- [#57 — Upstream baselines and fork builds](https://github.com/nuri-com/bitwarden-passkey-prf/issues/57)
- [#9 — iOS 26 Credential Exchange contract](https://github.com/nuri-com/bitwarden-passkey-prf/issues/9)
- [#11 — Android Credential Manager PRF contract](https://github.com/nuri-com/bitwarden-passkey-prf/issues/11)
- [#58 — Fresh-Android Nuri recovery audit](https://github.com/nuri-com/bitwarden-passkey-prf/issues/58)
- [#16 — Nuri WebView third-party provider proof](https://github.com/nuri-com/bitwarden-passkey-prf/issues/16)
- [#13 — Android Digital Asset Links verification](https://github.com/nuri-com/bitwarden-passkey-prf/issues/13)
- [#59 — Secret lifetime, logging, and export constraints](https://github.com/nuri-com/bitwarden-passkey-prf/issues/59)
The live ready queue is always authoritative:

```bash
gh issue list \
  --repo nuri-com/bitwarden-passkey-prf \
  --label priority:mvp-critical \
  --label status:ready \
  --label parallel-safe \
  --state open \
  --limit 100
```

## Parallel lanes after discovery

```text
contracts/vectors +-> official-cloud Cipher.data proof (#83 + #85) ---+
                  +-> SDK encrypted credential model -----+-> shared portability gate
                  +-> key validation ----------------------+             |
                  +-> passkey-rs PRF evaluator ------------+             |
                                                                          +-> iOS import/export ----+
                                                                          +-> Android provider ------+-> real-device gate
Nuri recovery/WebView/DAL ------------------------------------------------+                         |
                                                                                                     +-> upstream PR lanes
```

Expected safe concurrency before the first device proof:

- M0: up to 9 implementation agents immediately, plus one coordinator.
- M1: four to six agents across the official-cloud blob boundary, SDK model, CXF mappings, evaluator, validation, and security tests.
- M2: six to eight agents across independent iOS, Android, Nuri, and QA branches.
- M3: iOS build/import and Android build/provider work can overlap; exact cross-device continuity and golden recovery remain serial gates.
- M4 and broad hardening remain parked until the golden recovery gate is green.

## Critical path

1. Freeze the synthetic vectors and encrypted credential contract.
2. Add optional encrypted extension state through SDK BlobV1 and bindings, using the existing official-cloud `Cipher.data` field without a custom server deployment.
3. Preserve CXF import and enable arbitrary-input PRF evaluation.
4. Pass the shared-core portability gate.
5. Complete iOS blob import (#83) and Android blob transport (#85), then the remaining provider integrations in parallel.
6. Import from Apple Passwords and sync the exact credential to Android.
7. Complete the wiped-Android Nuri golden journey.
8. Only after the working proof, run broad hardening, add export paths, and prepare focused upstream contributions.

## Deliberately parked work

Tickets carrying `priority:post-proof` must not delay the first installable iOS/Android proof. They cover Bitwarden CXF export, creation of new PRF passkeys inside Bitwarden, iOS export, broad Android lifecycle testing, the full negative/rollback matrix, operator documentation, and upstream issue/PR packaging. Cosmetic UI work, desktop support, provider matrices, speculative refactors, and broad algorithm support are outside the proof-first queue entirely.

## Definition of a swarm-ready ticket

Every claimable ticket has:

- one observable outcome;
- a target repository and ownership track;
- direct `Blocked by` and reverse `Unblocks` issue links;
- checkable acceptance criteria;
- required verification evidence;
- `parallel-safe` plus exactly one scheduling status; and
- no overlap with an already claimed file set.

See [swarm.md](swarm.md) for claim, branch, handoff, and coordinator commands.
