# Test vectors

Only synthetic credentials belong here.

Canonical positive fixture:

- [`synthetic-passkey-prf-cxf-v1.json`](synthetic-passkey-prf-cxf-v1.json) is a complete CXF v1 header containing one deterministic ES256 passkey and fixed UV/non-UV HMAC seeds.
- [`../docs/vectors/synthetic-vectors.md`](../docs/vectors/synthetic-vectors.md) freezes its digest, signing public key, two PRF input/output pairs, and negative-case recipes.

Planned additions include expected Nuri-derived public identities using synthetic root material and standalone malformed/unsupported fixture files.

Never commit a real passkey private key, real CXF export, real PRF output, or production wallet identity.
