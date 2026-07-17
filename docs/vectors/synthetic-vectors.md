# Synthetic CXF and PRF conformance vectors

Frozen synthetic test vectors for CXF import/export and PRF evaluation. All values are deterministic, synthetic, and contain no real secrets.

## 1. Synthetic ES256 credential

| Field | Value |
| --- | --- |
| Credential ID (base64) | `AAECAwQFBgcICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8=` |
| Credential ID (hex) | `000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f` |
| RP ID | `nuri.com` |
| User handle (base64) | `ICEiIyQlJicoKSorLC0uLw==` |
| User handle (hex) | `202122232425262728292a2b2c2d2e2f` |
| Key algorithm | ES256 (ECDSA P-256, COSE alg -7) |
| Curve | P-256 (secp256r1) |

### HMAC seeds

| Field | Value |
| --- | --- |
| UV HMAC seed (base64) | `ERERERERERERERERERERERERERERERERERERERERERE=` |
| UV HMAC seed (hex) | `1111111111111111111111111111111111111111111111111111111111111111` |
| Non-UV HMAC seed (base64) | `IiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiI=` |
| Non-UV HMAC seed (hex) | `2222222222222222222222222222222222222222222222222222222222222222` |

## 2. PRF test inputs and expected outputs

PRF outputs are computed as `HMAC-SHA256(seed, UTF-8(input))` per the WebAuthn PRF specification.

### Input 1: `nuri-prf-salt-v1` (production salt)

| Field | Value |
| --- | --- |
| Input | `nuri-prf-salt-v1` |
| Input (UTF-8 bytes) | `6e7572692d7072662d73616c742d7631` |
| UV output (base64) | `dqvLIkKk7/vDfRxh0WDuqeU5l0drp+oAFc1s4lvtAxc=` |
| Non-UV output (base64) | `0FMQv1r7flEPM19o3yAim8FqbJia+2L0E9SKvYUKQY0=` |

### Input 2: `test-salt-2` (arbitrary second input)

| Field | Value |
| --- | --- |
| Input | `test-salt-2` |
| Input (UTF-8 bytes) | `746573742d73616c742d32` |
| UV output (base64) | `Eb2TCanN6CFSRN92AmXQGpFUdM6Va9YQG58ZFIGOihE=` |
| Non-UV output (base64) | `IjQDfniEIzs4QbVBRQ5e1eCY52HT2pQJYFRXeI2D0ss=` |

### Verification command

```bash
python3 -c "import hmac,hashlib,base64; seed=bytes([17]*32); out=hmac.new(seed, b'nuri-prf-salt-v1', hashlib.sha256).digest(); print(base64.b64encode(out).decode())"
# Expected: dqvLIkKk7/vDfRxh0WDuqeU5l0drp+oAFc1s4lvtAxc=
```

## 3. CXF round-trip fixture

The complete CXF JSON fixture for the synthetic credential:

```json
{
  "type": "passkey",
  "passkey": {
    "credential_id": "AAECAwQFBgcICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8=",
    "relying_party_identifier": "nuri.com",
    "user_handle": "ICEiIyQlJicoKSorLC0uLw==",
    "user_display_name": "Test User",
    "user_name": "test@nuri.com",
    "key": {
      "algorithm": -7,
      "type": "public-key",
      "key": "<synthetic ES256 private key in COSE format>"
    },
    "fido2_extensions": {
      "hmac_credentials": {
        "algorithm": "hmac-secret",
        "cred_with_uv": "ERERERERERERERERERERERERERERERERERERERERERE=",
        "cred_without_uv": "IiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiI="
      }
    }
  }
}
```

The CXF round-trip test must verify:
1. Import preserves credential_id, rp_id, user_handle, key algorithm, and both HMAC seeds byte-for-byte.
2. Encrypted sync and reload preserves all fields.
3. Export (when implemented) emits the same fido2_extensions.

## 4. Negative fixtures

### 4.1 Missing PRF (no HMAC seeds)

```json
{
  "type": "passkey",
  "passkey": {
    "credential_id": "AAECAwQFBgcICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8=",
    "relying_party_identifier": "nuri.com",
    "user_handle": "ICEiIyQlJicoKSorLC0uLw==",
    "key": { "algorithm": -7 },
    "fido2_extensions": null
  }
}
```

Expected behavior: import succeeds (credential is usable for authentication), but PRF evaluation fails closed with a clear error. Nuri wallet recovery must not proceed.

### 4.2 Malformed key (invalid COSE encoding)

```json
{
  "type": "passkey",
  "passkey": {
    "credential_id": "AAECAwQFBgcICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8=",
    "relying_party_identifier": "nuri.com",
    "user_handle": "ICEiIyQlJicoKSorLC0uLw==",
    "key": { "algorithm": -7, "key": "not-a-valid-cose-key" },
    "fido2_extensions": { "hmac_credentials": { "cred_with_uv": "ERERERERERERERERERERERERERERERERERERERERERE=" } }
  }
}
```

Expected behavior: import fails with a clear "unsupported key encoding" error. Credential is not stored.

### 4.3 Unsupported key algorithm (RS256 instead of ES256)

```json
{
  "type": "passkey",
  "passkey": {
    "credential_id": "AAECAwQFBgcICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8=",
    "relying_party_identifier": "nuri.com",
    "user_handle": "ICEiIyQlJicoKSorLC0uLw==",
    "key": { "algorithm": -257, "type": "public-key" },
    "fido2_extensions": { "hmac_credentials": { "cred_with_uv": "ERERERERERERERERERERERERERERERERERERERERERE=" } }
  }
}
```

Expected behavior: import fails with "unsupported key algorithm: RS256 (-257), expected ES256 (-7)". The implementation must not silently hardcode a different algorithm.

### 4.4 Non-zero counter

```json
{
  "type": "passkey",
  "passkey": {
    "credential_id": "AAECAwQFBgcICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8=",
    "relying_party_identifier": "nuri.com",
    "user_handle": "ICEiIyQlJicoKSorLC0uLw==",
    "key": { "algorithm": -7 },
    "counter": 42,
    "fido2_extensions": { "hmac_credentials": { "cred_with_uv": "ERERERERERERERERERERERERERERERERERERERERERE=" } }
  }
}
```

Expected behavior: import fails with "non-zero signature counter not supported for portable credentials". Per the plan, migrating passkeys with non-zero counters is a non-goal for MVP.

### 4.5 Credential mismatch

A CXF fixture with credential_id `BBBB...` but an HMAC seed that was computed with a different credential's UV seed. The PRF output will not match the expected value for the credential. This is a silent wallet-changing failure if not detected.

Expected behavior: The implementation must bind PRF evaluation to the credential ID and detect mismatches before any wallet key is persisted.

## 5. Invariants

These vectors are designed to verify:

1. **Byte-for-byte preservation**: CXF import → encrypted sync → reload → CXF export preserves all fields.
2. **Two-input PRF**: Both `nuri-prf-salt-v1` and `test-salt-2` produce stable outputs before and after reload, proving the HMAC seed was preserved (not just caching one result).
3. **UV vs non-UV separation**: UV and non-UV seeds produce different outputs for the same input.
4. **Fail-closed**: Missing PRF, malformed key, unsupported algorithm, non-zero counter, and credential mismatch all fail with clear errors before wallet commit.
5. **No persistence of PRF outputs**: The 32-byte outputs are computed on demand and never stored.