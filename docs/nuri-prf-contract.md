# Nuri PRF compatibility contract

## Credential request

| Property | Required value |
| --- | --- |
| WebAuthn RP ID | `nuri.com` |
| PRF input encoding | UTF-8 |
| PRF first input | `nuri-prf-salt-v1` |
| User verification | Required for wallet unlock |
| Credential type | Discoverable passkey |

The imported provider must evaluate WebAuthn `prf.eval.first` using the migrated credential's user-verified HMAC seed and return `prf.results.first`.

Bitwarden stores the portable passkey credential and its provider-internal PRF/HMAC seed state. It must not store the result for `nuri-prf-salt-v1`. PRF output is calculated fresh during each authorized WebAuthn ceremony, so the same imported passkey also works correctly with other inputs.

The provider must not apply an additional KDF, label, domain separator, or encoding conversion to the CXF HMAC seed. CXF requires the importing provider to use the exported seed as-is.

## Wallet continuity

Nuri treats the 32-byte PRF result as root key material. The current app derives:

- Bitcoin entropy with HKDF-SHA256 and the `app:nuri.com|wallet|v1` domain;
- Ethereum entropy with the same versioned domain and a chain-specific info string;
- the Nostr identity from the Bitcoin key;
- SwapKit provider roots with `HMAC-SHA256(PRF_ROOT, "nuri/swapkit/htlc/v1/" + provider)`; and
- legacy encrypted seed material through the existing versioned Nuri KDF.

Changing the passkey's PRF/HMAC seed state changes every derived PRF result and therefore changes the wallet. A migrated credential that can authenticate but returns a different or missing PRF result is incompatible and must fail closed.

## Fresh Android recovery

The decisive test starts with a fresh Android Nuri install and no Nuri private keys in local storage. The flow may use a discoverable assertion when no credential ID hint exists, but Nuri must bind the returned assertion and PRF result to the selected credential and reject mismatches.

The test must compare public identities and deterministic non-secret vectors. Raw PRF output, CXF HMAC seeds, and derived private keys must never be logged or attached to CI artifacts.

## Source references

The contract is grounded in the current Nuri implementation:

- `config/constants.ts`
- `config/PasskeyBitcoinConfig.ts`
- `services/auth/PasskeyPrfWalletBootstrapService.ts`
- `services/auth/WebAuthnPrfService.ts`
- `components/PasskeyWebAuthnPrfView.tsx`
- `lib/walletDerivation.ts`

These paths are references to the private development checkout and are intentionally not copied into this repository.
