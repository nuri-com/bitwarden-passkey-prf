# Upstream contribution strategy

## Repository layout

This coordination repository remains independent. Create GitHub forks only when implementation begins:

| Upstream | Planned fork | Purpose |
| --- | --- | --- |
| `bitwarden/server` | `nuri-com/server` | optional encrypted FIDO2 request, data, sync, and compatibility models |
| `bitwarden/sdk-internal` | `nuri-com/sdk-internal` | encrypted model, CXF mappings, PRF authenticator |
| `bitwarden/ios` | `nuri-com/ios` | Apple Credential Exchange and provider integration |
| `bitwarden/android` | `nuri-com/android` | Android provider transport and integration |
| `bitwarden/clients` | `nuri-com/clients` | later browser/desktop work |

The existing `bitwarden/credential-exchange` model already contains PRF/HMAC credential fields. Fork it only if conformance fixes are actually required.

## Pull-request sequence

### PR 1 — SDK CXF extension preservation

Target: `bitwarden/sdk-internal`

- optional encrypted extension fields;
- bidirectional CXF mapping;
- correct signing-key validation;
- old/new vault compatibility tests; and
- byte-preserving synthetic CXF round trips.

This PR must not enable runtime PRF evaluation yet.

### PR 2 — Stored-passkey PRF evaluation

Target: `bitwarden/sdk-internal`

- enable PRF/`hmac-secret` in the authenticator configuration;
- UV and non-UV seed selection;
- WebAuthn-shaped outputs;
- secret-lifetime hardening; and
- deterministic extension tests replacing the current "PRF is not evaluated" expectation.

Depends on PR 1.

### PR 3 — iOS Credential Exchange preservation

Target: `bitwarden/ios`

- prove `ASImportableFIDO2Extensions` survives the Swift-to-SDK boundary;
- user-visible unsupported-credential reporting; and
- Apple Passwords/1Password fixtures and integration tests.

Depends on PR 1; provider PRF use depends on PR 2.

### PR 4 — Android credential-provider PRF integration

Target: `bitwarden/android`

- current AndroidX provider event transport;
- PRF request/output plumbing;
- locked/offline/process-death coverage; and
- end-to-end provider tests.

Depends on PRs 1 and 2.

### PR 5 — Documentation and interoperability report

Targets: the Bitwarden repositories chosen by maintainers.

- supported import/export sources;
- platform and OS requirements;
- non-exportable credential limitations; and
- a redacted Apple-to-Android interoperability report.

## Engagement rule

Before opening PR 1, open or locate the canonical Bitwarden feature issue and share:

- the precise data-loss finding (`fido2_extensions` dropped during CXF conversion);
- the FIDO CXF requirement for stable HMAC seeds;
- the proposed optional model change;
- synthetic interoperability tests; and
- the intended dependent PR sequence.

Do not open an upstream planning-only PR. The first upstream PR should compile, include tests, and be independently reviewable.

## Branch conventions

- Base every branch on a recorded upstream commit.
- Use one concern per branch and avoid Nuri branding in generic Bitwarden code.
- Keep the Nuri production salt out of Bitwarden unit tests; use public synthetic vectors upstream.
- Link dependent PRs explicitly without merging temporary compatibility hacks.
- Rebase rather than merge upstream main into contribution branches unless maintainers request otherwise.

## Upstream acceptance criteria

The feature is upstream-ready when it is useful to any PRF-capable relying party, follows CXF and WebAuthn semantics, preserves Bitwarden's encrypted-vault boundary, and includes no Nuri-specific runtime behavior.
