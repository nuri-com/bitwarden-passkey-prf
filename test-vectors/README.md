# Test vectors

Only synthetic credentials belong here.

Planned fixtures:

- an exportable ES256 passkey with zero signature counter;
- fixed UV and non-UV HMAC seeds;
- two public PRF input/output pairs;
- expected CXF round-trip data;
- expected Nuri-derived public identities using synthetic root material; and
- malformed/unsupported negative cases.

Never commit a real passkey private key, real CXF export, real PRF output, or production wallet identity.
