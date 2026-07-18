# ADR: Encrypted extension-state contract for portable PRF passkeys

Status: Proposed
Date: 2026-07-18
Coordination issue: [nuri-com/bitwarden-passkey-prf#56](https://github.com/nuri-com/bitwarden-passkey-prf/issues/56)
Depends on: `docs/plan.md`, `docs/nuri-prf-contract.md`, `docs/orchestrator.md`, `docs/upstream.md`

## Context

Bitwarden already transports passkeys through CXF import/export, encrypted server wire models, SDK cipher encryption, and cross-platform sync. The current shared path preserves the signing key but drops FIDO2 extension state, including the PRF/HMAC seed state that Nuri uses as wallet root material. A credential that authenticates but returns a missing or different PRF result silently produces a different wallet, which is unacceptable for funds management.

This ADR fixes the contract for the optional encrypted extension-state fields that must travel with a passkey so that:

- the existing signing credential remains the source of truth;
- the PRF/HMAC seed state required to evaluate WebAuthn PRF is preserved byte-for-byte through encrypted sync, lock/unlock, CXF import/export, and language bindings;
- updated Bitwarden clients continue to decode existing legacy credentials without breakage;
- ordinary vault JSON export does not disclose PRF seeds;
- evaluated PRF outputs are never persisted anywhere; and
- the model stays generic for any PRF-capable relying party, with no Nuri-specific runtime behavior in Bitwarden.

The contract deliberately mirrors what the CXF `fido2Extensions.hmacCredentials` structure already models, so the encrypted Bitwarden credential and the CXF representation can be mapped bidirectionally without inventing a parallel side channel.

## Decision

Extend Bitwarden's encrypted FIDO2 credential model with a single optional encrypted container that carries PRF/HMAC extension state. The container is part of the cipher's encrypted payload, not a custom field, note, attachment, or Nuri-specific side channel. The new nested fields are optional so updated clients continue to decode legacy credentials. When a valid CXF `hmacCredentials` object is imported, however, both of its functions are present and must be preserved in the encrypted container. The MVP does not claim that an older BlobV1 client can decrypt, modify, and reseal the new payload without losing unknown nested state; that downgrade boundary is isolated in #73.

The container MUST carry, at minimum:

1. **PRF/HMAC algorithm** — the algorithm identifier needed to validate that the stored signing key and HMAC seed match the actual imported credential, rather than assuming ES256. This enables explicit rejection of unsupported keys instead of silent mislabeling.
2. **User-verified HMAC seed** — the sole HMAC function exposed through WebAuthn `prf.eval.first`/`prf.eval.second`; the effective user-verification requirement is overridden when necessary.
3. **Non-user-verified HMAC seed** — portable CTAP `hmac-secret` state used only by provider-internal/direct CTAP operations without user verification. It is required and preserved when sourced from CXF `hmacCredentials`; it may be absent only on legacy or non-CXF internal credentials and must never be exposed through WebAuthn `prf.results`.
4. **Optional `credBlob`** — preserved only when the imported credential actually carries it.
5. **Optional `largeBlob`** — preserved only when the imported credential actually carries it.
6. **Key metadata for algorithm validation** — enough metadata to validate the imported signing-key algorithm rather than hardcoding ECDSA P-256. This exists to make algorithm mismatches explicit instead of silently relabeling keys.

The container MUST NOT carry evaluated PRF outputs, derived wallet material, or Nuri-specific values. It stores only the portable passkey state required to evaluate PRF on demand during an authorized WebAuthn ceremony.

The passkey identity invariants that bind the container to a single credential are:

1. **Credential ID** — preserved byte-for-byte across import, sync, and export.
2. **Signing key + validated algorithm** — the private signing key plus the validated algorithm from the container's key metadata. The algorithm is validated against the imported key material, not assumed.
3. **RP ID** — preserved; for Nuri this is `nuri.com`.
4. **User handle** — preserved byte-for-byte.

These four invariants plus the optional extension container together form the portable passkey. Syncing only the signing key is insufficient: the result can authenticate but cannot reproduce the original PRF behavior.

## Detailed rules

### 1. Optional encrypted fields

| Field | Required | Purpose |
| --- | --- | --- |
| `prf_hmac_algorithm` | yes, when extension state is present | Algorithm identifier used to validate the signing key and select PRF/HMAC evaluation |
| `uv_hmac_seed` | yes, when extension state is present | User-verified HMAC seed; the sole HMAC function exposed through WebAuthn `prf` |
| `non_uv_hmac_seed` | required for CXF `hmacCredentials`; optional for legacy/non-CXF internal state | Portable non-UV CTAP `hmac-secret` state; preserved for CXF/internal CTAP use, never exposed through WebAuthn `prf` |
| `cred_blob` | optional | FIDO2 `credBlob` if present on the imported credential |
| `large_blob` | optional | FIDO2 `largeBlob` if present on the imported credential |
| `key_algorithm_metadata` | yes, when extension state is present | Metadata sufficient to validate the signing-key algorithm instead of hardcoding ECDSA P-256 |

The container is absent on existing passkey ciphers and on any passkey that does not carry PRF/HMAC extension state. Absence is distinct from an empty container and MUST be preserved through round trips.

Per [WebAuthn Level 3 section 10.1.4](https://www.w3.org/TR/webauthn-3/#prf-extension), the WebAuthn client extension exposes one PRF per credential. An implementation backed by CTAP `hmac-secret` MUST use the user-verified function and override the effective `userVerification` preference if necessary; it MUST NOT choose between the two stored HMAC functions based on the assertion's initial UV preference.

Per the [CXF FIDO2 HMAC Credentials section](https://fidoalliance.org/specs/cx/cxf-v1.0-ps-errata-20260309.html#fido2hmaccredentials), a valid `Fido2HmacCredentials` exchange object contains both `credWithUV` and `credWithoutUV`. Import must therefore retain both byte-for-byte. A future CXF exporter starting from a legacy/internal credential that stores only one function must create the missing stable function before exchange as required by CXF; it must not emit a one-function `hmacCredentials` object.

### 2. Passkey identity invariants

The following four fields are the identity invariants of the passkey and MUST be preserved unchanged across every transformation:

1. **Credential ID** — the WebAuthn credential identifier.
2. **Signing key + validated algorithm** — the private signing key plus the algorithm obtained from `key_algorithm_metadata` and validated against the actual key material. The validated algorithm is authoritative; the authenticator MUST NOT substitute a default.
3. **RP ID** — the WebAuthn relying-party identifier (for Nuri, `nuri.com`).
4. **User handle** — the WebAuthn user handle.

A credential that survives a round trip with any of these changed is a different credential and MUST fail closed before any wallet state is committed.

### 3. Old-client downgrade behavior

All new wire fields are optional for mixed-version decoding. The following downgrade rules apply; they do not relax the two-function requirement for a CXF `hmacCredentials` object:

- An older client that does not know the extension container may be unable to use a PRF-capable blob cipher; it MUST fail rather than replace or silently downgrade the cipher.
- An older BlobV1-capable client that can decode and reseal the containing blob MUST not silently drop the unknown nested state. The general mixed-version prevention mechanism is post-proof issue #73.
- An older client MUST NOT synthesize a replacement for missing PRF state. If it cannot evaluate a requested PRF extension, it MUST return no PRF result so that the relying party fails closed.
- A mixed-version fixture set MUST prove all three paths: old-cipher-decoded-by-new-client, new-cipher-decoded-by-old-client, and new-cipher-decoded-by-new-client.

The contract is intentionally additive. No existing field's encoding, ordering, or semantics changes.

### 4. Server wire model

The mobile MVP uses Bitwarden's existing blob-encrypted `Cipher.data` wire field. Specifically:

- The complete FIDO2 credential, including extension state, is serialized inside the versioned cipher blob before transmission.
- `Cipher.data` is an opaque authenticated-encryption envelope with a top-level `format_version`; the server stores and returns it without parsing the nested credential or PRF/HMAC state.
- No PRF-specific server field, column, DTO, or API shape is required. The unchanged official Bitwarden Cloud already accepts and returns the existing `Cipher.data` field.
- iOS CXF import MUST pass the imported cipher through the normal SDK CiphersClient blob encryptor before `/ciphers/import`. A PRF-capable import that does not produce `Cipher.data` MUST fail before upload.
- The iOS request and sync-response models MUST map `Cipher.data` unchanged and accept the blob response's optional legacy `name`; a forced post-import sync must not replace the blob with `nil`.
- The blob request MUST NOT also send extension state through legacy structured `login.fido2Credentials` fields.
- Only updated client paths proven to forward `Cipher.data` unchanged are in the MVP: iOS import,
  request, and response mapping in #83, plus Android request, response, and cached-sync mapping in
  #85. Backup, key rotation, and mixed-version resealing are not assumed safe merely because the
  outer field is opaque; those broader guarantees remain in #73.

The controlled MVP therefore requires updated iOS and Android forks and a blob-capable personal V2 account, but no local Docker server or Nuri backend deployment. The fork-only legacy server DTO remains compatibility research for field-level/V1 clients and is not part of the first device proof. This keeps the server out of the PRF trust boundary and avoids a Nuri-specific production server surface.

### 5. Re-encryption behavior

On the pinned updated mobile forks, the container MUST survive the exact encrypt/decrypt/sync cycle
used by the golden journey unchanged:

- Encrypting a decrypted credential that contains extension state produces the same extension state after re-encryption. Byte-for-byte preservation is required for the seeds, credential ID, user handle, and RP ID.
- Decryption on a new client or after vault reload yields the same container that was encrypted.
- Lock/unlock and ordinary vault sync on the pinned updated clients preserve the container.
- A round trip of `CXF -> pinned iOS import -> official-cloud Cipher.data sync -> pinned Android decrypt` preserves the signing key, credential ID, RP ID, user handle, and both PRF seeds byte-for-byte.
- A focused `decrypt -> normal personal-V2 encrypt -> decrypt` round trip on the pinned SDK yields the same container.

Migration across arbitrary server/SDK versions, account-key rotation, organization-key rotation, and
old-client decrypt/reseal paths are not part of this MVP claim. They remain fail-closed follow-up
work in #73 and must not be inferred from the opaque server transport alone.

If any field would be mutated by re-encryption, the round trip MUST fail closed rather than silently producing a different wallet.

### 6. JSON-export behavior

Ordinary vault JSON export MUST NOT disclose PRF seeds. Concretely:

- A normal Bitwarden vault JSON export of a cipher containing extension state excludes the extension container, the UV seed, the non-UV seed, `credBlob`, and `largeBlob`.
- The signing credential, credential ID, RP ID, and user handle follow the existing Bitwarden export policy for FIDO2 credentials.
- A dedicated, separately reviewed export path (for example an encrypted CXF export or an explicitly user-authorized encrypted export) may carry extension state, but only behind an explicit user action and an explicit review gate. That path is out of scope for this ADR and for the mobile v0.1 proof.
- The default export surface is the ordinary vault JSON export. Its behavior remains unchanged: no PRF seed bytes, no HMAC seed bytes, no `credBlob`/`largeBlob` contents.

This rule exists because an unencrypted JSON export of PRF seed material is equivalent to exporting the wallet root for any relying party that uses PRF.

### 7. Prohibition: evaluated PRF outputs must never be persisted

This is an explicit, non-negotiable prohibition:

- Evaluated PRF outputs (`prf.results.first`, `prf.results.second`, any derived value) MUST NOT be persisted in the cipher, the extension container, the server, sync state, logs, crash reports, analytics, CI artifacts, or any local cache.
- PRF outputs are computed fresh during each authorized WebAuthn ceremony using the stored HMAC seed and the requested input.
- The extension container stores only the seed state required to compute outputs, never the outputs themselves.
- Synthetic tests MUST assert that no evaluated output is recoverable after a ceremony, across encrypted reload, and across sync.
- Zeroization of temporary seeds, salts, and outputs is required where the surrounding Rust and native boundaries allow it.

This prohibition is the load-bearing security invariant of the entire contract. Storing a PRF result would turn the encrypted vault into a wallet-key oracle.

## Consequences

- Bitwarden's encrypted FIDO2 credential becomes the single source of truth for the portable passkey, including its PRF/HMAC extension state.
- The official server remains outside the PRF trust boundary; it carries the opaque encrypted `Cipher.data` envelope.
- The first proof is intentionally limited to updated fork clients and a blob-capable personal V2 account. Broader old-client compatibility is a separate post-proof gate.
- CXF and Bitwarden-internal representations can be mapped bidirectionally without a parallel side channel.
- Ordinary vault JSON export stays safe for users; PRF seed material never appears in a default export.
- The same encrypted credential supports any PRF-capable relying party, not only Nuri. Nuri-specific runtime behavior is prohibited in Bitwarden.
- The mobile v0.1 golden journey becomes possible: the imported passkey's PRF behavior is preserved through Apple Passwords -> Bitwarden iOS -> encrypted sync -> Bitwarden Android -> Nuri WebView assertion, with public wallet identities matching the iOS baseline.

## Open questions

1. The MVP wire encoding is the existing versioned BlobV1 `Cipher.data` envelope, with optional extension state nested in the encrypted FIDO2 credential. A future new blob version may be required for general old-client downgrade protection in #73.
2. Whether `credBlob` and `largeBlob` are needed for the first Apple Passwords -> Android journey is to be confirmed by the captured iOS 26 Credential Exchange contract. They remain optional in the model so the contract is forward-compatible.
3. The explicit review gate for any future encrypted export path that would carry PRF seed material is deferred to the post-proof upstream delivery milestone and is out of scope here.
