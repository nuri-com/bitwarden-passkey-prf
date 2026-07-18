# Start here: fastest testable Apple-to-Android passkey MVP

## URL-only execution contract

This issue is intentionally self-contained. **If this issue URL is the entire user prompt, that URL is the complete instruction to execute this program now.** Do not merely summarize the issue, propose another plan, ask what to do next, or wait for the user to repeat its contents.

Receiving only this URL authorizes the orchestrator to perform all normal, reversible implementation actions required by the `priority:mvp-critical` scope in the repositories listed below, including:

- inspect the repositories and current upstream state;
- claim and update GitHub issues and scheduling labels;
- create isolated branches and worktrees;
- start and coordinate the maximum safe number of parallel workers;
- edit code and tests;
- make SSH-signed commits and push branches to the Nuri forks;
- open draft pull requests against the Nuri forks;
- independently review, request fixes, integrate reviewed Nuri-fork changes, and re-test them;
- create narrowly scoped bug/fix issues discovered by testing; and
- prepare installable test builds and all automatable real-device prerequisites.

This authority does not permit production deployment, disclosure of secrets, bypassing platform authorization, destructive cleanup outside the named worktrees, or merging Bitwarden upstream pull requests. The existing security and physical-action boundaries in this issue still apply.

### Mandatory first actions

1. Read this entire issue, `AGENTS.md`, and every file in **Required reading** before changing code.
2. Work from `/Users/eminmahrt/Developer/nuri-bitwarden`; verify every listed checkout, `origin`, `upstream`, Git status, and effective SSH-signing configuration.
3. Claim this issue exactly once: assign the coordinator, replace `status:ready` with `status:claimed`, and post the coordinator session, model/harness, available worker capacity, and pinned starting SHAs.
4. Query the live `priority:mvp-critical` + `status:ready` queue and immediately start the maximum safe non-overlapping workers.
5. Continue the review -> fix -> rebuild -> test loop without requiring another user prompt.

The only valid stopping conditions are:

- **success:** issue #45 is proven on physical iOS and Android devices with the required secret-safe evidence; or
- **genuine external blocker:** all automatable work is complete and the orchestrator reports one precise user-only action, the exact failed step and evidence, what has already been tried, and how it will resume afterward.

Planning completion, exhausted initial workers, open pull requests, passing unit tests, simulator success, a manual test remaining, or an upstream review pending are not stopping conditions.

## Mission

Claim this issue as the single program orchestrator and execute the scoped implementation end to end. Start the maximum safe number of parallel workers, keep refilling free slots, independently review every result, send defects back for correction, integrate the Nuri forks, and continue until the real-device golden journey works or one precise physical/user action is genuinely required.

This is not a request for another plan. The plan and tickets already exist. The required outcome is an installable Bitwarden iOS build and an installable Bitwarden Android build that let a tester move a real PRF-capable Nuri passkey from Apple Passwords into Bitwarden on iOS, sync that exact credential, and use it from Bitwarden on Android to recover the same Nuri wallet identities.

Functionality is the only priority before that proof. Do not spend time on cosmetic work, generalization, broad compatibility matrices, desktop support, or upstream presentation while the golden journey is still red.

## Product goal

A Nuri user must be able to:

1. create a discoverable PRF-capable passkey for `nuri.com` in Apple Passwords on iOS;
2. use it in Nuri on iOS and record the resulting public Bitcoin, Ethereum, Nostr, and SwapKit identities;
3. transfer the credential from Apple Passwords into the Nuri Bitwarden iOS fork through iOS 26 Credential Exchange;
4. let Bitwarden encrypt and sync the complete credential;
5. receive the same credential in the Nuri Bitwarden Android fork;
6. start from a fresh or wiped Nuri Android install with no copied Nuri secret material;
7. select Bitwarden as the Android third-party passkey provider and complete Nuri's PRF assertion; and
8. recover exactly the same public identities and complete one non-destructive signing proof.

## Non-negotiable architecture decisions

### Store and sync the passkey, never a Nuri PRF result

The portable object is the same passkey. At minimum, continuity includes:

- credential ID;
- private signing key and its validated public-key algorithm;
- RP ID, which is `nuri.com`;
- user handle;
- user-verified PRF/HMAC seed state;
- non-user-verified PRF/HMAC seed state when present or required by the imported CXF credential; and
- only the additional FIDO2 extension metadata required to preserve the actual Apple credential.

Bitwarden must evaluate WebAuthn PRF on demand during an authorized assertion. It must not persist the result for Nuri's input, precompute a Nuri wallet root, add a Nuri-specific side channel, or substitute newly generated HMAC state during import. An implementation that can sign with the imported credential but returns a missing or different PRF result has failed.

### Nuri's current contract is fixed for this MVP

- RP ID: `nuri.com`
- PRF input encoding: UTF-8
- first PRF input: `nuri-prf-salt-v1`
- user verification: required for wallet unlock
- credential: discoverable passkey
- response: standards-shaped `prf.results.first`

Nuri treats the 32-byte result as root key material. It derives Bitcoin, Ethereum, Nostr, and SwapKit identities from it. Therefore, changing the imported credential's HMAC state changes the wallet.

### Preserve general PRF behavior

Synthetic tests must evaluate at least two independent PRF inputs before and after encrypted reload. This proves that Bitwarden preserved the passkey's HMAC state instead of caching only `nuri-prf-salt-v1`. Real-device evidence must compare public identities only; never print real seeds, raw PRF outputs, or private keys.

## Verified starting point and actual gap

Bitwarden already has most of the transport:

- `bitwarden/credential-exchange` models CXF passkeys and FIDO2 HMAC credentials;
- `bitwarden/ios` has iOS 26 Credential Exchange import/export surfaces;
- `bitwarden/android` has Credential Manager/provider and CXF surfaces;
- `bitwarden/sdk-internal` imports, exports, stores, and uses passkeys; and
- Bitwarden clients already encrypt and sync passkeys.

The current shared path is not PRF-portable:

- CXF import preserves signing material but drops `fido2_extensions`;
- CXF export emits no FIDO2 extension state;
- the stored-passkey authenticator forwards extension requests while PRF processing remains disabled; and
- the existing PRF test expects no PRF output.

That can preserve authentication while silently losing the Nuri wallet root. The fork must preserve the imported HMAC state through the encrypted server/SDK model and use it in Android assertions.

For the physical MVP, the sync boundary is the existing opaque encrypted `Cipher.data` field on the
official Bitwarden Cloud. The iOS import must convert the SDK's legacy CXF-import result through the
normal CiphersClient blob encryptor and fail before upload unless a PRF-capable cipher has `data`.
Use a personal V2/security-state-2 Bitwarden test account. Do not start local Docker or deploy a
custom Bitwarden backend for this path. The fork server PR is retained only for legacy compatibility
research and is not an MVP runtime dependency.

## Definition of done for the first proof

Do not close this issue until every applicable item is evidenced:

- [ ] A pinned Bitwarden iOS fork build installs on a physical iOS 26 device.
- [ ] A real test passkey created by Apple Passwords for `nuri.com` imports into that build through the supported OS flow.
- [ ] Import preserves the credential ID, signing credential, RP ID, user handle, algorithm, and PRF/HMAC state without logging their secret bytes.
- [ ] Bitwarden encrypts, syncs, locks/unlocks, and reloads that credential without changing it.
- [ ] A pinned Bitwarden Android fork build installs on the target physical Android device and can be enabled as the passkey provider.
- [ ] The Android build receives the exact synced credential and returns PRF results for Nuri's actual WebView/Credential Manager request.
- [ ] A fresh/wiped Nuri Android install contains no copied Nuri private key or cached PRF output before recovery.
- [ ] Bitcoin, Ethereum, Nostr, and SwapKit public identities match the iOS baseline exactly.
- [ ] One non-destructive signature verifies against the original public identity.
- [ ] Synthetic tests prove on-demand evaluation for `nuri-prf-salt-v1` and one unrelated input across encrypted reload.
- [ ] Missing, malformed, wrong-credential, or unsupported PRF state fails clearly before Nuri commits a different wallet.
- [ ] Focused automated tests, build commands, fork commit SHAs, device/OS versions, and secret-safe real-device evidence are attached to the relevant issues.
- [ ] Every code result has an independently reviewed PR in the appropriate Nuri fork; discovered defects have been fixed and re-tested.

Passing unit tests alone, opening draft PRs, producing a simulator build, or proving only a signature is not done.

## Minimum implementation scope

Do only what the golden journey requires:

1. Freeze synthetic CXF and PRF vectors and pin compiling upstream baselines.
2. Confirm the exact Apple iOS 26 and Android Credential Manager wire contracts used by the target devices.
3. Add optional encrypted FIDO2 HMAC extension state through SDK models, BlobV1 cipher encryption/reload, and Swift/Kotlin bindings; route it through the existing official-cloud `Cipher.data` wire field.
4. Preserve the Apple credential's extension state during CXF import into Bitwarden iOS.
5. Validate the imported signing algorithm instead of silently hardcoding a different one.
6. Bridge stored HMAC state into the authenticator and evaluate arbitrary requested PRF inputs with the correct user-verification seed.
7. Carry Android PRF request inputs into the SDK and return standards-shaped results to Nuri.
8. Preserve the credential through real Bitwarden encrypted sync to Android.
9. Make only the smallest Nuri app changes required for the actual Android provider flow, credential binding, and fail-closed wallet commit boundary.
10. Produce installable device builds and run the real journey.

Use `credBlob`, `largeBlob`, additional algorithms, or compatibility adapters only if the captured contract or real Apple test credential proves they are necessary for this exact journey. Record the evidence before widening scope.

## Explicitly parked until the golden journey is green

Do not let any of these block the first live-device build:

- Bitwarden-to-Apple or general CXF export;
- creating new PRF-capable passkeys inside Bitwarden;
- desktop, browser-extension, macOS, Windows, or Linux support;
- 1Password, Google Password Manager, or a general provider matrix;
- UI redesign, copy polish, branding work, animations, screenshots for marketing, or accessibility cleanup unrelated to completing the flow;
- broad lock/offline/reboot/process-death testing beyond the single state needed for the first proof;
- a full rollback, negative, or release-hardening matrix beyond fail-closed checks that prevent a wrong wallet;
- broad key-algorithm support beyond the real Apple credential;
- speculative abstractions, refactors, dependency upgrades, generated-file churn, or cleanup not forced by the path;
- operator handbooks, release automation, public distribution, or production deployment;
- upstream issue/PR packaging and review-polish work.

The following tickets are intentionally labeled `priority:post-proof`: #12, #23, #29, #32, #33, #37, #47, #48, #49, #50, #51, #52, #53, #54, and #55. Do not claim them before #45 is green unless a concrete blocker proves one contains unavoidable work; document that evidence before changing its priority.

## Repositories and local checkouts

| Responsibility | Fork | Expected local checkout |
| --- | --- | --- |
| Coordination, vectors, evidence | `nuri-com/bitwarden-passkey-prf` | `/Users/eminmahrt/Developer/nuri-bitwarden/coordination` |
| Optional legacy-format compatibility (not MVP runtime) | `nuri-com/server` | `/Users/eminmahrt/Developer/nuri-bitwarden/server` |
| Shared Rust SDK, CXF, authenticator, bindings | `nuri-com/sdk-internal` | `/Users/eminmahrt/Developer/nuri-bitwarden/sdk-internal` |
| Apple Credential Exchange integration | `nuri-com/ios` | `/Users/eminmahrt/Developer/nuri-bitwarden/ios` |
| Android Credential Manager/provider | `nuri-com/android` | `/Users/eminmahrt/Developer/nuri-bitwarden/android` |
| Real relying-party app | `nuri-com/nuri-expo` | `/Users/eminmahrt/Developer/nuri-expo` |

The umbrella directory `/Users/eminmahrt/Developer/nuri-bitwarden` is intentionally not a Git repository. Put isolated agent worktrees under `/Users/eminmahrt/Developer/nuri-bitwarden/worktrees/<issue>-<slug>`. For each Bitwarden clone, `origin` must remain the Nuri fork and `upstream` the corresponding `bitwarden/*` repository. Pin and record base commits before editing. Read this repository's `AGENTS.md` and every target repository's instructions first.

## Orchestrator authority and boundaries

Within this project's scope, the orchestrator is expected to do the implementation work, not merely delegate it:

- claim and schedule the MVP-critical tickets;
- create isolated branches/worktrees and draft PRs in Nuri forks;
- review diffs and test evidence independently;
- request or implement corrections when results are incomplete;
- integrate compatible Nuri-fork changes after review and required checks;
- create narrowly scoped bug tickets when real testing uncovers missing work;
- rebuild and re-test until the golden journey works; and
- prepare the devices/build artifacts and exact test steps as far as automation permits.

Do not merge Bitwarden upstream PRs, deploy to production, disclose real credential material, bypass OS/provider authorization, or weaken signing/security checks. Upstream contributions happen after the fork proof and remain maintainer-controlled.

All commits and tags must use the machine's existing SSH signing configuration. Never disable signing. If signing fails, stop that commit/tag and repair or report the signing problem.

If a physical-device tap, Apple export authorization, Bitwarden login, code-signing identity, or another user-only action is required, finish every automatable prerequisite first, then ask for one precise action and resume immediately afterward. Do not report completion without direct real-device evidence.

## Swarm execution loop

### 1. Claim this coordinator issue

Issue #68 is the only exception to the normal `parallel-safe` claim rule. Exactly one coordinator claims it, changes `status:ready` to `status:claimed`, and remains responsible until the full proof is green.

### 2. Start the maximum safe MVP queue

List only functional MVP work:

```bash
gh issue list \
  --repo nuri-com/bitwarden-passkey-prf \
  --state open \
  --label priority:mvp-critical \
  --label status:ready \
  --label parallel-safe \
  --limit 100
```

With four execution slots, keep one coordinator and three workers active. With more slots, start every ready issue whose expected files do not overlap. Refill slots as soon as a PR enters review or a task closes.

### 3. Isolate every worker

One issue means one owner, one target repository, one branch or worktree, and one PR. Use `swarm/<issue-number>-<short-slug>`. A worker must claim the ticket atomically, state expected files, use synthetic secrets, run focused tests, make signed commits, push the branch, open a draft PR against the Nuri fork, and attach exact evidence. Workers do not self-merge.

### 4. Review and fix, not just collect PRs

The coordinator independently inspects the diff, target behavior, tests, generated bindings, security boundary, and dependency impact. If a defect or gap exists, return it to the same worker or create a narrowly scoped ready fix ticket. Re-run the review and tests after every correction. A PR that compiles but does not preserve the real credential path is not acceptable.

### 5. Promote dependencies continuously

After a reviewed change is integrated into the Nuri fork, close its implementation issue. Inspect dependent issues and change `status:blocked` to `status:ready` only when every `Blocked by` item is closed. Run integration gates serially. Keep issue bodies and reverse links accurate when real evidence changes the path.

### 6. Integrate and prove on devices

Pin cross-repository SDK/client commits, prove the unchanged official-cloud `Cipher.data` boundary, produce installable iOS and Android builds, and run the exact golden journey. Use a real test passkey but secret-safe evidence. When a device run fails, diagnose the actual boundary, fix it in the smallest responsible repository, review, rebuild, and repeat.

### 7. Stop only at the correct terminal condition

Success is #45 green with installable builds and evidence. A genuine blocker must identify the exact failed step, command/log evidence, responsible repository, attempted fixes, and the one external action required. "The PRs are open" or "the remaining test is manual" is not a terminal condition.

## Initial parallel queue

These nine MVP-critical tickets are intentionally dependency-free and should start first:

- #7 — synthetic CXF and PRF vectors
- #56 — encrypted extension-state and mixed-version contract
- #57 — upstream baselines and fork builds
- #9 — iOS 26 Credential Exchange contract
- #11 — Android Credential Manager PRF contract
- #58 — fresh-Android Nuri recovery/commit audit
- #16 — Nuri WebView third-party provider proof
- #13 — Android Digital Asset Links verification
- #59 — passkey secret lifetime, logging, and export constraints

## Functional critical path

GitHub issue dependencies are authoritative. The MVP-critical set is grouped here so no essential lane is forgotten:

- Contract/harness: #7, #56, #57, #9, #11, #58, #16, #13, #59, #19
- Portable credential core: #21, #60, #17, #22, #26, #61, #62, #63, #31 (#20 is retained legacy compatibility, not an MVP runtime dependency)
- iOS import and build: #28, #27, #83, #64, #67, #46
- Android provider and build: #35, #65, #66, #40, #41, #39, #42
- Cross-device proof: #44, then #45

The proof-first sequence is:

```text
contracts + vectors + build baselines
                 |
encrypted SDK BlobV1 credential + CXF import + on-demand PRF
                 |
       shared import/sync/reload/PRF gate (#31)
                 |
       +---------+---------+
       |                   |
 iOS import/build     Android provider/build
       |                   |
 Apple real import -> official-cloud Cipher.data sync -> fresh Android Nuri recovery (#45)
```

## Worker prompt

Use this template for each spawned worker:

```text
Implement ISSUE_URL as one scoped worker in the target repository.

Read nuri-com/bitwarden-passkey-prf issue #68, its AGENTS.md, and the target
repository instructions first. Claim only this issue. Use a dedicated
swarm/<issue>-<slug> branch/worktree based on the pinned upstream commit.
Do only the functional acceptance scope; no cleanup or polish. Use synthetic
secrets. Run focused tests, create signed commits, push to the Nuri fork, open
a draft PR, and comment with exact commands, results, changed files, commit SHA,
risks, and PR URL. Do not merge. If blocked, prove the exact blocker and notify
the coordinator; do not silently widen scope.
```

## Required reading

- `AGENTS.md`
- `docs/nuri-prf-contract.md`
- `docs/plan.md`
- `docs/roadmap.md`
- `docs/swarm.md`
- `docs/test-plan.md`
