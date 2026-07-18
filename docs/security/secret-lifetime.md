# Passkey secret lifetime, logging, and export constraints

This document maps the lifetime of every secret that flows through the Nuri PRF portability path across the four forks (`sdk-internal`, `server`, `ios`, `android`). It is an audit artifact, not a spec: it records where each secret enters, is stored, is transmitted, is processed, and must be disposed of, and it defines the redaction and forbidden-output rules that every fork must satisfy.

It is written for the pre-proof swarm scope defined in `docs/plan.md`: CXF import, encrypted sync, reload, and on-demand PRF evaluation. CXF export-back-out, new-credential creation, and desktop paths are listed for completeness but are not required for the first mobile proof.

This document contains no real secret values. All examples are synthetic or reduced to types and lengths.

## Scope and secret inventory

The Nuri wallet root is the 32-byte WebAuthn PRF result evaluated from the passkey's `hmac-secret` seed and the production salt `nuri-prf-salt-v1`. That result is never persisted; it is derived fresh during each authorized WebAuthn ceremony. The secrets that produce it, plus the signing private key, are the lifetime subjects of this audit.

Secrets in scope, in descending order of blast radius:

| Secret | Type | Lifetime class | Why critical |
| --- | --- | --- | --- |
| PRF result / evaluated `hmac-secret` output | 32-byte symmetric secret | Ephemeral | Directly becomes the Nuri wallet root key. Leaking it leaks every derived wallet key. |
| `hmac-secret` seed, user-verified (`cred_with_uv`) | >= 32 bytes | Persisted, encrypted | Sole HMAC function exposed through WebAuthn `prf`; changing it changes the wallet. |
| `hmac-secret` seed, non-user-verified (`cred_without_uv`) | >= 32 bytes | Persisted, encrypted | Portable CXF/CTAP state for provider-internal non-UV `hmac-secret` operations; never a WebAuthn PRF output. |
| FIDO2 signing private key (`key_value`) | COSE / PKCS8 bytes | Persisted, encrypted | Authenticates to `nuri.com`. Extraction enables impersonation without the device. |
| PRF input salt (`nuri-prf-salt-v1`) | UTF-8 string | Public, fixed | Not secret per se, but its exact bytes are load-bearing. Mishandling produces a different wallet. |
| Derived Nuri wallet sub-keys (Bitcoin/Ethereum/Nostr/SwapKit roots) | 32-byte symmetric secrets | Ephemeral | Derived downstream from the PRF result inside Nuri. Out of scope for Bitwarden code, but listed because their secrecy depends on the PRF result never being logged. |
| `credBlob`, `largeBlob` (optional, future) | Opaque bytes | Persisted, encrypted | Not used by Nuri today. If preserved, they must be treated as encrypted blobs, never inspected or logged in plaintext. |

Non-secrets that are still load-bearing and must be preserved byte-for-byte: credential ID, RP ID (`nuri.com`), user handle, user name, user display name, key type / algorithm / curve metadata, discoverable flag, counter.

## Lifecycle map

The table below traces each in-scope secret through five stages: intake (where it enters a process), storage (where it is at rest), transit (where it crosses a trust boundary), processing (where it is operated on), and disposal (where it must leave memory). Fork abbreviations: SDK = `sdk-internal` (Rust), SRV = `server` (.NET), IOS = `ios` (Swift), AND = `android` (Kotlin).

### PRF result / evaluated `hmac-secret` output

| Stage | Location | Fork | Notes |
| --- | --- | --- | --- |
| Intake | Returned by the authenticator's `get_assertion` extension output, inside `passkey::types::ctap2::get_assertion::Response` unsigned extension outputs. | SDK | Produced on demand; never persisted by Bitwarden. |
| Storage | In-memory only, inside the `Fido2Authenticator` call frame. | SDK | Must not be cached on `Fido2Authenticator`, `CipherView`, or any long-lived struct. |
| Transit | Within the SDK process only. Crosses to the platform caller via the uniffi/Wasm binding as a returned `Vec<u8>` / `B64`. | SDK -> IOS / AND | The binding return is the only inter-process hop. |
| Processing | `derive_symmetric_key_from_prf` in `bitwarden-crypto/src/keys/prf.rs` stretches the first 32 bytes into an `Aes256CbcHmacKey`. Then `make_prf_user_key_set` builds a `RotateableKeySet`. | SDK | The PRF bytes are split, stretched, and dropped. The stretched key inherits `ZeroizeOnDrop`. |
| Disposal | The owning `Vec<u8>` must be dropped before the call returns. The derived `SymmetricCryptoKey` is `ZeroizeOnDrop`, so it is zeroized when its slot is cleared. | SDK | See zeroization section. |

### `hmac-secret` seed, user-verified (`cred_with_uv`)

| Stage | Location | Fork | Notes |
| --- | --- | --- | --- |
| Intake | Imported from CXF `fido2_extensions.hmacCredentials` (issue #58 / WP1), or produced by `make_credential` with `hmac_secret_mc` enabled. | SDK | The CXF importer currently drops this field (`fido2_extensions: None`). WP1 adds it. |
| Storage | Encrypted as part of the FIDO2 credential extension state inside the Bitwarden cipher. The cipher field is an `EncString` keyed by the user's symmetric key. At rest on disk inside the Bitwarden vault and on the server. | SDK, SRV | Server stores opaque encrypted blobs; it never sees plaintext seeds. |
| Transit | 1) CXF import payload (JSON, in-process on iOS/Android) -> SDK. 2) Encrypted cipher over the sync API to/from the server. 3) Encrypted cipher through uniffi to/from platform UI. | All | Only ever crosses trust boundaries encrypted, except during the initial CXF import where the plaintext seed is in memory of the importing process. |
| Processing | Decrypted inside `KeyStoreContext` when the authenticator evaluates PRF. Read into `StoredHmacSecret.cred_with_uv` and consumed by the `hmac-secret` CTAP processing in `passkey`. | SDK | The decrypted seed must live only inside the authenticator call frame. |
| Disposal | The decrypted `Vec<u8>` must be dropped at the end of the assertion. The encrypted `EncString` is the persistent form and is not zeroized (it is the stored credential). | SDK | See zeroization section. |

### `hmac-secret` seed, non-user-verified (`cred_without_uv`)

Same lifecycle as the UV seed. It must round-trip through CXF and sync unchanged as portable CTAP `hmac-secret` state, but it is only available to provider-internal/direct CTAP operations without UV. WebAuthn `prf` always exposes the user-verified function, overriding the effective user-verification requirement when necessary; if UV cannot be completed, fail closed rather than substituting this seed.

### FIDO2 signing private key (`key_value`)

| Stage | Location | Fork | Notes |
| --- | --- | --- | --- |
| Intake | CXF `PasskeyCredential.key` (base64 COSE key) on import; or produced by `make_credential` on creation. | SDK | Already wired in the existing importer (`cxf/login.rs`). |
| Storage | Encrypted as `Fido2Credential.key_value: EncString` inside the cipher. | SDK, SRV | Server column is opaque; the server never decrypts. |
| Transit | Encrypted inside sync responses. Plaintext only inside the SDK process during signing. | All | |
| Processing | Decrypted to a `Fido2CredentialFullView.key_value: String`, then parsed into a `CoseKey` / `PrivateKey` for assertion signing. | SDK | The decrypted string is the hot secret. |
| Disposal | The decrypted `String` / `CoseKey` must be dropped at the end of the assertion. `SigningKey` already implements `ZeroizeOnDrop`. | SDK | |

### PRF input salt (`nuri-prf-salt-v1`)

Not a secret, but load-bearing. It is a UTF-8 string sent in `prf.eval.first`. Its exact bytes must be preserved by every fork; any normalization, truncation, or re-encoding produces a different wallet. It is safe to log, attach to CI artifacts, and include in tests. Do not log it together with the resulting PRF output.

### Derived Nuri wallet sub-keys

Out of scope for Bitwarden code. They are produced inside Nuri from the PRF result. They are listed only to record that their secrecy is entirely dependent on the PRF result never being logged, attached to crash reports, or written to CI artifacts by any fork in this audit.

## Zeroization points

Zeroization is applied where the host language and existing architecture permit it. The rule is: every temporary buffer that has held a plaintext PRF result, plaintext HMAC seed, or plaintext signing private key must be cleared before the buffer goes out of scope. Persistent encrypted forms are not zeroized; they are the stored credential.

Required zeroization points in `sdk-internal` (Rust):

1. **PRF result buffer** in `bitwarden-fido/src/device_auth_key.rs` after `derive_symmetric_key_from_prf` consumes it. Wrap in `Zeroizing<Vec<u8>>` or ensure the owning `Vec<u8>` is dropped at scope exit.
2. **`StoredHmacSecret` decrypted seed** used by the `passkey` authenticator during `get_assertion`. The `cred_with_uv` / `cred_without_uv` `Vec<u8>` fields should be `Zeroizing<Vec<u8>>` or copied into a `Zeroizing` buffer and the copy dropped after evaluation.
3. **`Fido2CredentialFullView.key_value`** after it is parsed into a signing key. The `SigningKey` types already implement `ZeroizeOnDrop` (see `bitwarden-crypto/src/signing/signing_key.rs`); the intermediate `String` does not, so prefer parsing into the signing key and dropping the `String` immediately, or move the secret into a `Zeroizing<String>`.
4. **`HighEntropySecret`** (`bitwarden-crypto/src/safe/high_entropy_secret.rs`) already wraps its bytes in `Zeroizing<Vec<u8>>`. Any PRF output held as a `HighEntropySecret` inherits this. Use this type for the PRF result when feasible.
5. **`SymmetricCryptoKey`** derived from the PRF is `ZeroizeOnDrop` (`bitwarden-crypto/src/keys/symmetric_crypto_key.rs`). The `RotateableKeySet` built from it inherits this.
6. **`KeyStoreContext`** zeroizes its slots on drop via `ZeroizeOnDrop` on the backend (`bitwarden-crypto/src/store/backend/implementation/basic.rs`).
7. **Global allocator**: `bitwarden-crypto` sets a `ZeroizingAllocator<std::alloc::System>` as the global allocator (`bitwarden-crypto/src/lib.rs`), so heap-allocated secret bytes are zeroized on free. This is a defense-in-depth backstop, not a substitute for the explicit points above.

Required zeroization points in `ios` (Swift):

1. **CXF import payload bytes** (`Data` containing the JSON with the plaintext HMAC seed) must be deallocated and not retained beyond the import call. Swift `Data` backed by a region of memory is freed by ARC; there is no `memset_s` equivalent that is universally safe on iOS, so the mitigation is to keep the plaintext `Data` scoped to the import function and never copy it into a long-lived type.
2. **`ASPasskeyCredentialRequest` / `ASPasskeyAssertionCredential`** responses containing the assertion and PRF output must not be retained beyond the credential provider callback. Hand them to `extensionContext.completeAssertionRequest` and let ARC reclaim them.
3. **Any `Data` derived from `passkey.key` or `passkey.fido2Extensions`** during import must be consumed by the SDK binding immediately and not stored on a long-lived view model.

Required zeroization points in `android` (Kotlin):

1. **CXF import payload `String`** containing the plaintext HMAC seed. Kotlin/JVM does not guarantee zeroization of `String` (interned, immutable, GC-managed). Mitigation: parse the payload to `ByteArray`, hand the bytes to the SDK, and `Arrays.fill(bytes, 0)` before the import function returns. Avoid keeping the seed in `String` form.
2. **`BeginGetCredentialResponse`** payloads containing the assertion and PRF output. Hand them to the Credential Manager callback and do not retain.
3. **Any `ByteArray` used to ferry the PRF result across the JNI boundary** to the SDK. `Arrays.fill(array, 0)` after the SDK call returns.

## Redaction points

Logs, analytics, crash reports, and CI artifacts may carry metadata about credentials but may never carry secret bytes. The redaction rule is: log lengths, status codes, types, and identifiers; never raw bytes, hex, base64, or decoded COSE key material.

Allowed in logs (all forks):

- credential ID (it is a public identifier, not a secret)
- RP ID (`nuri.com`)
- user handle (it is a public user identifier; treat as PII but not as a secret)
- key type / algorithm / curve strings (`public-key`, `ECDSA`, `P-256`)
- discoverable flag
- counter value
- PRF presence / status enum (`WebAuthnPrfStatus.Enabled`, `PrfStatus`)
- `supportsPrf` boolean
- error enum variants and short error strings (`PrfFailure`, `MissingHmacSecret`, `InvalidPrfInput`)
- lengths, e.g. `prf_result.len() = 32`, `seed.len() = 64`
- operation outcome (success / fail-closed)

Forbidden in logs (all forks):

- raw PRF result bytes, in any encoding
- raw `hmac-secret` seed bytes (`cred_with_uv` or `cred_without_uv`), in any encoding
- raw signing private key bytes (`key_value`), in any encoding, including decoded COSE key JSON
- derived Nuri wallet sub-key bytes
- full CXF JSON payloads from real accounts (the test-plan already forbids this)
- encrypted cipher blobs are not secret-by-construction; they may appear in server logs at debug level, but they must not be attached to CI artifacts or crash reports

Redaction implementation points:

- `sdk-internal`: `HighEntropySecret::fmt` already overrides `Debug` to print only `HighEntropySecret` (`bitwarden-crypto/src/safe/high_entropy_secret.rs`). Apply the same manual `Debug` override to any new struct that holds a plaintext PRF result or HMAC seed. Audit `tracing::error!` / `warn!` / `info!` call sites in `bitwarden-fido` (22 sites, listed below) to confirm none interpolates a secret-bearing variable with `?` or `%`. The existing sites interpolate error enums and status codes only.
- `server`: `ILogger` usage near WebAuthn/FIDO2 handling must log only `PrfStatus`, `SupportsPrf`, credential IDs, and exception types. The server never decrypts cipher fields, so plaintext seeds do not reach it; the residual risk is logging encrypted blobs at info level, which should be debug-only.
- `ios`: `Logger.flightRecorder` and `Logger.application` are the two sinks. Audit any log call near `ASImportableCredential.Passkey`, `fido2Extensions`, and `AutofillCredentialService` to ensure it logs only the allowed metadata list. The flight recorder writes to a file that may be attached to a support ticket, so this is a high-leak-risk sink.
- `android`: `Timber` is the sink. The `FlightRecorderTree` (`data/src/main/kotlin/com/bitwarden/data/manager/flightrecorder/FlightRecorderManagerImpl.kt`) writes logs to a file that may be shared. Audit any `Timber.*` call near the CXF importer, Credential Provider Service, and SDK binding to ensure it logs only allowed metadata.

## Forbidden outputs

No fork may emit any in-scope secret byte to any of the following sinks:

1. **Logs** — application logs, flight recorder files, `tracing` spans, `Timber` trees, `os_log`. See redaction points above.
2. **Analytics** — telemetry events. Bitwarden telemetry must not include cipher field values; the PRF portability work must not add any new telemetry that reads `fido2_extensions`, `key_value`, `hmac_secret`, or PRF result bytes.
3. **Crash artifacts** — crash reports, exception stack traces, and diagnostic dumps. A crash during PRF evaluation must not dump the authenticator response, the decrypted seed, or the PRF result. Ensure crash reporters (iOS `OSLogErrorReporter`, Android `FirebaseCrashlytics`/equivalent) are configured to strip cipher fields and extension outputs. Do not attach the CXF import payload to a crash report.
4. **Normal JSON exports** — Bitwarden's normal vault JSON export (`bitwarden-exporters`) must continue to emit `fido2_extensions: None`. The current code already does this (`cxf/export.rs:219`, `cxf/import.rs:376/456/510`, asserted by `export.rs:323`). This is a load-bearing forbidden output: it must not regress. A future change that adds PRF seeds to the normal export must go through a separate security review and must not be part of the MVP.
5. **CI artifacts** — test output, golden journey recordings, screenshots, and evidence bundles. The test-plan already forbids raw CXF payloads from real accounts. Extend this to: no synthetic CXF fixture that contains a real PRF seed may be committed; fixtures must use the synthetic seeds defined in the interop harness (issue #2). No test that evaluates PRF may attach the result to CI output; tests assert derived public identities and stable non-secret vectors only.

## Per-fork audit checklist

### `sdk-internal` (Rust)

Key files and what they must / must not do:

- `crates/bitwarden-fido/src/authenticator.rs` — owns the `Fido2Authenticator` call frame. Must not store PRF results or decrypted seeds on `self`. The `selected_cipher: Mutex<Option<CipherView>>` stores the encrypted cipher only, which is correct.
- `crates/bitwarden-fido/src/device_auth_key.rs` — `DeviceAuthKeyRecord.hmac_secret: Vec<u8>` (line 539) is the persisted UV seed. This is the device-auth-key path, not the imported-passkey path, but it shows the pattern: the seed is a `Vec<u8>`. For the imported-passkey path (WP1), the equivalent field must be `Zeroizing<Vec<u8>>` while in memory and `EncString` while at rest. `TryFrom<Passkey> for DeviceAuthKeyRecord` (line 554) clones the seed from the passkey extension; ensure the source passkey's extension buffer is dropped after the clone.
- `crates/bitwarden-fido/src/crypto.rs` — extracts the private key from the COSE key. The `tracing::error!` sites here (lines 20, 27, 42) interpolate error enums only; confirm no future change interpolates the COSE key bytes.
- `crates/bitwarden-crypto/src/keys/prf.rs` — `derive_symmetric_key_from_prf` takes `&[u8]`. The caller owns the slice; ensure the caller drops it. The function rejects all-zero input (defends against uninitialized PRF) and truncates to 32 bytes (the remainder is dropped — confirm the caller drops the remainder too, or pass exactly 32 bytes).
- `crates/bitwarden-crypto/src/safe/high_entropy_secret.rs` — the approved-source comment (lines 29-30) explicitly lists "A PRF output from a passkey with the hmac-secret extension" as a valid source. Use this type for the PRF result where feasible; it gives `Zeroizing<Vec<u8>>` and a redacted `Debug` for free.
- `crates/bitwarden-vault/src/cipher/login.rs` — `Fido2Credential` (line 92) is the encrypted at-rest form; `Fido2CredentialFullView` (line 150) is the decrypted form with `key_value: String`. WP1 will add extension fields here. The full view must not be serialized to logs or to the normal export.
- `crates/bitwarden-exporters/src/cxf/login.rs` — `impl TryFrom<Fido2Credential> for PasskeyCredential` (line 197) sets `fido2_extensions: None` (line 219). This is the load-bearing forbidden output. WP1 must add a *separate*, opt-in path that emits the seeds only through an explicitly encrypted, user-authorized export flow, never through the normal export.
- `crates/bitwarden-exporters/src/cxf/import.rs` — `parse_item` consumes CXF passkey credentials. WP1 must read `fido2_extensions` from the CXF payload and route it into the encrypted `Fido2Credential` extension state, without logging the plaintext.
- `crates/bitwarden-core/src/key_management/crypto.rs` — `make_prf_user_key_set` (line 683) takes `prf: B64` and derives the user key set. The `B64` is the PRF result in base64; ensure the `B64` is dropped after use.

Logging audit: the 22 `tracing` sites in `bitwarden-fido` (enumerated above) all interpolate error enums, status codes, or `?options` (attestation options, which are public). None interpolates a secret. Any new `tracing` call added by WP1/WP2 must follow the same rule.

### `server` (.NET)

- `src/Core/Vault/Models/Data/CipherLoginFido2CredentialData.cs` — the server-side FIDO2 data model has no PRF/HMAC seed field. This is correct for the MVP: the server stores opaque encrypted blobs only. If WP1 adds extension state, it will live inside the encrypted `Login` data blob, not as a new server column. Confirm no new server column is added for HMAC seeds.
- `src/Api/Vault/Models/CipherFido2CredentialModel.cs` — the wire model. All fields are `[EncryptedString]`. Keep it that way. Do not add a plaintext `HmacSeed` field.
- `src/Api/Vault/Models/Response/SyncResponseModel.cs` — emits `WebAuthnPrfOptions` (line 69), which is metadata (which credentials support PRF), not seed material. Correct.
- `src/Api/Auth/Controllers/WebAuthnController.cs` — handles passkey login. The `EncryptedUserKey`, `EncryptedPublicKey`, `EncryptedPrivateKey` fields are encrypted; the server does not decrypt them. Confirm.
- `src/Sql/dbo/Tables/WebAuthnCredential.sql` — the `SupportsPrf` column (line 13) is a boolean, not a seed. Correct.
- `src/Api/KeyManagement/Validators/WebAuthnLoginKeyRotationValidator.cs` — validates PRF-enabled credentials during key rotation. The error messages (lines 41, 46, 51) are safe strings.

Server-side residual risk: the server never sees plaintext seeds, so the only leak vector is logging encrypted blobs at info level. Mitigation: keep cipher-field logging at debug level, and never attach sync responses to crash reports.

### `ios` (Swift)

- `BitwardenAutoFillExtension/CredentialProviderViewController.swift` — handles `ASPasskeyCredentialRequest` and returns `ASPasskeyAssertionCredential`. The assertion credential contains the authenticator data, signature, and (when PRF is requested) the PRF result. Do not retain the response; hand it to `extensionContext.completeAssertionRequest` and return.
- `BitwardenShared/Core/Autofill/Services/AutofillCredentialService.swift` — bridges to the SDK. The `ASPasskeyAssertionCredential` constructor (line 447, 707) takes `clientDataHash`, `relyingParty`, etc. Ensure the PRF result, if carried in the response, is not logged.
- `BitwardenShared/Core/Vault/Services/TestHelpers/ASImportableAccount+Extensions.swift` — test helper that prints `passkey.key` (line 59) and other passkey fields for debugging. This is a test helper, but it shows the pattern to avoid: never log `passkey.key` or `passkey.fido2Extensions` in production code.
- iOS 26 Credential Exchange import: `ASImportableCredential.Passkey.fido2Extensions` must reach the SDK binding intact. The binding hands the bytes to the SDK importer. Do not copy the plaintext into a view model or a log line.
- `BitwardenKit/Core/Platform/Services/FlightRecorder.swift` — the flight recorder writes logs to a file that may be attached to support tickets. This is the highest-leak-risk sink on iOS. Audit any flight-recorder log call near passkey/PRF code to confirm it logs only allowed metadata.

### `android` (Kotlin)

- `cxf/` module — parses CXF payloads. The importer hands parsed bytes to the SDK. Ensure the parsed `ByteArray` for the HMAC seed is `Arrays.fill`-cleared after the SDK call.
- `network/src/main/kotlin/com/bitwarden/network/model/SyncResponseJson.kt` — `Cipher.Fido2Credential` (line 1236) is the sync wire model. All fields are encrypted strings on the wire. Keep it that way; do not add a plaintext seed field.
- Credential Provider Service — `BeginGetCredentialResponse` and the assertion response carry the PRF output. Hand the response to the Credential Manager callback; do not retain.
- `data/src/main/kotlin/com/bitwarden/data/manager/flightrecorder/FlightRecorderManagerImpl.kt` — `FlightRecorderTree` writes to a file. Same high-leak-risk concern as iOS. Audit `Timber.*` calls near passkey/PRF code.

## Cross-cutting invariants

1. The PRF result is ephemeral. No fork may persist it, cache it, or store it on a long-lived object.
2. The HMAC seeds are persisted only in encrypted form (`EncString` / encrypted cipher blob). The server never sees plaintext.
3. The normal JSON export always emits `fido2_extensions: None`. Any change to this requires a separate security review and is not part of the MVP.
4. Fail closed: missing PRF state, missing UV seed for any WebAuthn `prf` evaluation, unsupported key, or credential mismatch must abort before any wallet state is committed. No fork may synthesize a replacement seed or expose the non-UV HMAC function through WebAuthn `prf`; the request must result in effective UV or fail.
5. Zeroization is defense-in-depth. The global `ZeroizingAllocator` in `bitwarden-crypto` is a backstop; the explicit zeroization points above are the primary controls.
6. This document contains no real secret values. Every example refers to types, lengths, or synthetic test vectors.

## Open questions for review

1. Should the imported-passkey path store the HMAC seed as `Zeroizing<Vec<u8>>` while in memory, matching `HighEntropySecret`, or is the `EncString`-at-rest plus immediate-drop-in-memory pattern sufficient? (Recommend: use `Zeroizing<Vec<u8>>` for the in-memory decrypted seed, `EncString` for at-rest.)
2. Is the Android JNI boundary a zeroization gap? Kotlin `ByteArray` passed to Rust via uniffi is copied; the Kotlin-side copy should be `Arrays.fill`-cleared. Confirm the uniffi binding does not retain the copy.
3. Should the flight recorder (iOS and Android) be disabled by default in the test builds, or should it be redacted to exclude any line that mentions `passkey`, `fido2`, `prf`, or `hmac`? (Recommend: redact, do not disable — the recorder is useful for debugging transport issues.)
4. For the CXF export-back-out path (post-MVP), what authorization model is required before emitting the HMAC seed? (Out of scope here; tracked in the post-proof backlog.)

## References

- `coordination/docs/nuri-prf-contract.md` — the PRF contract.
- `coordination/docs/plan.md` — work packages WP1 and WP2.
- `coordination/docs/test-plan.md` — release evidence rules, including "no raw CXF payloads from real accounts".
- `sdk-internal/crates/bitwarden-fido/src/authenticator.rs` — authenticator call frame.
- `sdk-internal/crates/bitwarden-fido/src/device_auth_key.rs` — `DeviceAuthKeyRecord`, `StoredHmacSecret`, PRF derivation path.
- `sdk-internal/crates/bitwarden-crypto/src/keys/prf.rs` — `derive_symmetric_key_from_prf`.
- `sdk-internal/crates/bitwarden-crypto/src/safe/high_entropy_secret.rs` — `HighEntropySecret` with redacted `Debug`.
- `sdk-internal/crates/bitwarden-crypto/src/lib.rs` — global `ZeroizingAllocator`.
- `sdk-internal/crates/bitwarden-vault/src/cipher/login.rs` — `Fido2Credential` and view types.
- `sdk-internal/crates/bitwarden-exporters/src/cxf/login.rs` — `fido2_extensions: None` export path.
- `server/src/Core/Vault/Models/Data/CipherLoginFido2CredentialData.cs` — server data model (no seed field).
- `server/src/Api/Vault/Models/CipherFido2CredentialModel.cs` — wire model (all `EncryptedString`).
