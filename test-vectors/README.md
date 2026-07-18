# Test vectors

Only synthetic credentials belong here.

Canonical positive fixture:

- [`synthetic-passkey-prf-cxf-v1.json`](synthetic-passkey-prf-cxf-v1.json) is a complete CXF v1 header containing one deterministic ES256 passkey and fixed `credWithUV`/`credWithoutUV` HMAC seeds.
- [`../docs/vectors/synthetic-vectors.md`](../docs/vectors/synthetic-vectors.md) freezes its digest, signing public key, two WebAuthn PRF input/output pairs, internal CTAP non-UV oracles, and negative-case recipes.

WebAuthn Level 3 exposes only the user-verified HMAC function through the `prf` client extension. The non-UV seed and literals in these vectors exist solely to prove CXF preservation and internal CTAP `hmac-secret` behavior; they must never be returned as WebAuthn `prf.results`.

Planned additions include expected Nuri-derived public identities using synthetic root material and standalone malformed/unsupported fixture files.

Never commit a real passkey private key, real CXF export, real PRF output, or production wallet identity.
