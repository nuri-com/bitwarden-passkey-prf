# Implementation plan

## Product outcome

Deliver testable Bitwarden iOS and Android builds that preserve the same PRF-capable Apple passkey through Credential Exchange and encrypted Bitwarden sync, then evaluate that passkey's PRF extension correctly for Nuri on Android.

## Proof-first execution rule

[Orchestrator issue #68](https://github.com/nuri-com/bitwarden-passkey-prf/issues/68) is authoritative until the first physical-device golden journey in #45 works. Before that gate:

- WP1 requires CXF import, encrypted sync, reload, and PRF preservation; CXF export is postponed.
- WP2 requires evaluation of imported HMAC state; creation of new PRF passkeys inside Bitwarden is postponed.
- WP3 is Apple Passwords import only; Bitwarden export and other source providers are postponed.
- WP4 covers the exact Android/Nuri state needed for one recovery; the broad lifecycle matrix is postponed.
- WP5 means pinned installable test builds and secret-safe proof, not UI polish, public release automation, or a complete operator package.
- WP6 and all upstream packaging start only after the fork proof is green.

Only tickets labeled `priority:mvp-critical` enter the pre-proof swarm. This section narrows scheduling; the later work packages remain the longer-term upstream plan.

## Verified starting point

Bitwarden already has most of the transport surface:

- `bitwarden/credential-exchange` models CXF passkeys and `Fido2HmacCredentials`.
- `bitwarden/ios` contains iOS 26 Credential Exchange import and export flows.
- `bitwarden/android` contains a CXF module and import manager.
- `bitwarden/sdk-internal` imports and exports passkeys.
- Bitwarden clients already store and use passkeys.

The current shared implementation loses PRF portability:

- CXF import copies the private signing key but does not persist `fido2_extensions`.
- CXF export emits `fido2_extensions: None`.
- the stored-passkey authenticator forwards extension requests but keeps extension processing disabled;
- its PRF test currently asserts that no PRF output is returned.

This means the signing credential may survive migration while the Nuri wallet root does not.

## Architecture decision

Use Bitwarden's existing encrypted FIDO credential as the source of truth and extend it with optional CXF extension state. Persist the passkey's PRF/HMAC seed state, not evaluated PRF outputs. Do not store either in custom fields, notes, ordinary JSON exports, or a Nuri-specific side channel.

At minimum, the encrypted credential model needs:

- PRF/HMAC algorithm;
- user-verified HMAC seed;
- non-user-verified HMAC seed;
- optional `credBlob`;
- optional `largeBlob`; and
- enough key metadata to validate rather than hardcode the imported signing algorithm.

All new fields must remain optional so existing vaults and clients continue to decode old credentials.

Together with the existing signing key, credential ID, RP ID, and user handle, these fields are the portable passkey. Syncing only the signing key produces a credential that may authenticate but cannot reproduce the original PRF behavior.

For the first mobile proof, the complete credential is nested in Bitwarden's existing blob-encrypted
`Cipher.data` field and synced through the unchanged official Bitwarden Cloud. The iOS CXF importer
must re-encrypt imported PRF ciphers through the normal CiphersClient blob path and fail before
upload if no blob is produced. Its request and sync-response mappings must preserve `Cipher.data`
unchanged and accept the blob response's optional legacy `name`. The MVP is limited to a
blob-capable personal V2 account. Android must likewise preserve top-level `Cipher.data` through
its sync DTO, SDK mapping, cached reload, and create/update request mapping (#85). The fork-only
legacy server field is not deployed and no local Docker server is required.

## Work packages

### WP0 — Reproducible interop harness

- Add a small Nuri RP test page using RP ID `nuri.com` and the production PRF input.
- Record only credential metadata, PRF presence, and derived public test identities.
- Create synthetic CXF fixtures containing ES256 signing material and both PRF seeds.
- Add negative fixtures: missing PRF, malformed key, unsupported algorithm, non-zero counter, and credential mismatch.

Exit gate: the harness distinguishes signature continuity from PRF continuity without exposing secret material.

### WP1 — Bitwarden encrypted credential model and CXF round trip

- Extend the shared FIDO credential model with optional extension state.
- Preserve the fields through encryption, decryption, official-cloud `Cipher.data` sync, import, export, and language bindings.
- Map CXF `fido2Extensions.hmacCredentials` in both directions.
- Preserve unknown-but-valid optional extension data where the model permits it.
- Reject unsupported signing keys explicitly instead of labeling every imported key ECDSA P-256.
- Keep ordinary vault JSON export behavior unchanged unless separately reviewed.

Exit gate: `CXF -> Bitwarden Cipher.data blob -> official-cloud sync -> decrypt -> CXF` preserves the signing key, credential ID, RP ID, user handle, and both PRF seeds byte-for-byte.

### WP2 — Stored-passkey PRF authenticator

- Enable WebAuthn PRF/CTAP `hmac-secret` processing in `bitwarden-fido`.
- Evaluate WebAuthn `prf` only with the user-verified HMAC function, overriding the effective user-verification requirement when necessary.
- Preserve the non-UV HMAC seed as portable CXF/CTAP state, but never expose its output through WebAuthn `prf.results`.
- Return standards-shaped `prf.results.first` and optional `second` outputs.
- Implement creation behavior for new Bitwarden-hosted passkeys so future exports are portable too.
- Zeroize temporary seeds, salts, and outputs where the surrounding Rust and native boundaries allow it.
- Add deterministic unit and integration vectors that evaluate multiple inputs from the stored credential.

Exit gate: the same synthetic credential produces the correct PRF result for at least two independent inputs before import, after import, and after vault reload. No evaluated output is persisted.

### WP3 — iOS 26 Apple-to-Bitwarden import

- Ensure `ASImportableCredential.Passkey.fido2Extensions` reaches the shared SDK intact.
- Import into an existing or new Bitwarden login item without silently dropping passkeys.
- Display an explicit compatibility failure when the source omits PRF data.
- Verify vault lock/unlock and cross-device sync after import.
- Test Apple Passwords and 1Password as exporters.

Exit gate: an Apple-created Nuri passkey is imported into Bitwarden and the Bitwarden iOS provider returns the same Nuri public wallet identities.

### WP4 — Android Bitwarden provider and Nuri recovery

- Preserve official-cloud top-level `Cipher.data` byte-for-byte through network decoding, SDK mapping, cached reload, and create/update requests.
- Complete Android provider import/export wiring required by the supported Credential Manager version.
- Pass PRF extension inputs from Android/WebView requests to the shared authenticator.
- Return PRF outputs through the Android credential-provider response.
- Test both constrained `allowCredentials` and discoverable requests.
- Validate behavior with Bitwarden locked, recently unlocked, offline, and after process death.

Exit gate: on a fresh Android device, Nuri uses Bitwarden to recover the same public wallet identities created on iOS.

### WP5 — Mobile hardening and test release

- Run the full positive and negative matrix in `docs/test-plan.md`.
- Complete security review of secret lifetime, logs, crash reports, and export surfaces.
- Produce reproducible internal iOS and Android builds from pinned upstream commits.
- Document installation, provider selection, migration, rollback, and credential revocation.

Exit gate: a tester can complete the journey using release-like builds without developer tools or secret copying.

### WP6 — Desktop follow-up

- Browser extension PRF support first.
- Native macOS 26 Credential Exchange integration.
- Windows Credential Manager and Windows Hello interop.
- Linux behavior and explicit limitations.

Desktop is not on the critical path for the first Nuri mobile proof.

## Delivery order

```text
Interop harness
      |
Encrypted model + CXF preservation
      |
Stored-passkey PRF evaluator
      |
      +-------------------+
      |                   |
iOS import/provider   Android provider
      |                   |
      +---------+---------+
                |
       Apple -> Bitwarden -> Android
                |
         Nuri wallet continuity
```

## Definition of done for mobile v0.1

- Apple Passwords can export the PRF-capable Nuri passkey to Bitwarden on iOS 26.
- Bitwarden syncs it while retaining the exact private credential and PRF seeds.
- Bitwarden on Android can satisfy Nuri's WebAuthn request for `nuri.com`.
- `prf.results.first` is present and stable for `nuri-prf-salt-v1`.
- Bitcoin, Ethereum, Nostr, and SwapKit public test identities match the iOS baseline.
- A missing or changed PRF fails closed and cannot initialize a different wallet silently.
- No PRF seed, PRF output, signing private key, or derived wallet private key appears in logs, analytics, crash reports, normal exports, or CI artifacts.
- The changes are submitted upstream as focused Bitwarden pull requests.

## Non-goals for mobile v0.1

- extracting device-bound or hardware-security-key credentials;
- bypassing Apple or Google provider export authorization;
- migrating passkeys with non-zero signature counters;
- desktop migration;
- silently converting passkeys that never supported PRF; or
- maintaining an indefinitely divergent branded password manager.

## Key risks

| Risk | Treatment |
| --- | --- |
| Exporter omits PRF seeds | Detect and report incompatibility; never synthesize a replacement wallet root. |
| Android provider API changes | Pin the AndroidX version and isolate platform transport from CXF/core logic. |
| Bitwarden model change breaks older clients | Limit the MVP to updated forks plus a personal V2 blob account; complete the general BlobV1 downgrade boundary in post-proof issue #73. |
| PRF implementation differs by provider | Use FIDO/CXF vectors and byte-for-byte output tests. |
| WebView bypasses the chosen provider | Test on the exact Nuri WebView/Credential Manager path, not only a browser demo. |
| Upstream PR is too broad | Submit dependent PRs by repository and layer with independent tests. |
| Permanent fork becomes a security burden | Keep forks rebased, avoid branding changes, and prioritize upstream acceptance. |
