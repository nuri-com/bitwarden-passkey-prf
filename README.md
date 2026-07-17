# Bitwarden Passkey PRF Portability

Upstream-first work to make a PRF-capable passkey created with Apple Passwords usable through Bitwarden on Android without changing the derived Nuri wallet.

## Goal

A Nuri user can:

1. create a PRF-capable `nuri.com` passkey in Apple Passwords;
2. use that passkey to create a Nuri wallet on iOS;
3. transfer the passkey into Bitwarden with the platform Credential Exchange flow;
4. install Nuri and Bitwarden on Android with no copied Nuri secret material;
5. use the transferred Bitwarden passkey in Nuri; and
6. recover the same Bitcoin, Ethereum, Nostr, and SwapKit identities.

The success condition is the same passkey credential on both platforms: identical credential ID, signing key, user handle, RP ID, and PRF/HMAC secret state. Bitwarden must compute PRF results on demand; it must not store a Nuri-specific PRF output.

## Approach

This repository holds the contract, implementation plan, interoperability tests, and links to upstream work. Product code belongs in narrow forks of the relevant Bitwarden repositories and is intended to return upstream as reviewable pull requests.

- [Implementation plan](docs/plan.md)
- [Nuri compatibility contract](docs/nuri-prf-contract.md)
- [Interop and release tests](docs/test-plan.md)
- [Upstream contribution sequence](docs/upstream.md)

## Initial scope

- iOS/iPadOS 26 Credential Exchange import into Bitwarden
- encrypted Bitwarden vault sync of the full PRF-capable passkey
- Android Credential Manager use through Bitwarden
- Nuri end-to-end wallet continuity

Desktop transfer and general non-Nuri interoperability follow after the mobile proof is green.

## Status

Planning and upstream gap validation. No production-ready Bitwarden build is published from this repository.
