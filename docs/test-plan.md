# Mobile interoperability and release tests

## Proof-first rule

Until golden recovery issue #45 is green, run the golden journey plus only the focused synthetic/fail-closed checks needed to avoid creating a different wallet. The broad matrix below is post-proof hardening and must not delay the first installable iOS and Android builds. UI polish, desktop coverage, additional exporters, and general provider compatibility are not part of the first run.

## Golden journey

1. Start with a clean iOS 26 device and a Nuri test account.
2. Select Apple Passwords as the passkey provider.
3. Create the Nuri passkey and wallet.
4. Record the credential ID fingerprint and public Bitcoin, Ethereum, Nostr, and SwapKit test identities.
5. Export the credential from Apple Passwords to the test Bitwarden build using Credential Exchange.
6. Lock and unlock Bitwarden, force a vault sync, and verify the item remains usable on iOS.
7. Start with a clean Android device or work profile containing no Nuri local keys.
8. Install the matching Bitwarden test build, sync the vault, and enable Bitwarden as the passkey provider.
9. Install Nuri and choose the existing-wallet/passkey path.
10. Complete a discoverable assertion through Bitwarden.
11. Verify PRF presence and compare all public identities with the iOS baseline.
12. Exercise one non-destructive signing proof and verify it against the original public credential.

## Post-proof hardening matrix

| Area | Cases |
| --- | --- |
| Exporter | Apple Passwords, Bitwarden, 1Password |
| iOS state | Bitwarden unlocked, locked, backgrounded, after reboot |
| Android state | unlocked, locked, offline after sync, process killed, after reboot |
| Request | constrained credential ID, discoverable selection |
| User verification | WebAuthn PRF evaluation performs effective UV; if verification cannot complete, no PRF result is returned |
| Vault | fresh import, duplicate item, multiple `nuri.com` passkeys |
| Failure | no PRF extension, missing UV seed, malformed seed, changed credential ID, unsupported key, sync conflict |

## Assertions

- The imported signing public key equals the registration public key.
- The credential ID and user handle survive unchanged.
- PRF output exists only when requested.
- The production salt produces the same derived public identities on both platforms.
- A second arbitrary test salt also produces stable output, proving preservation of the passkey's PRF seed rather than caching one Nuri result.
- Vault inspection and round-trip tests prove that evaluated PRF outputs are not persisted.
- Selecting the wrong `nuri.com` passkey is detected before local wallet state is committed.
- Normal Bitwarden JSON export does not unexpectedly disclose PRF seeds.
- Logs and crash artifacts contain lengths/status only, never secret bytes.

## Release evidence bundle

- Exact upstream commits and dependency versions
- Test build identifiers
- Device and OS versions
- Credential-provider selections
- Redacted screenshots of the migration journey
- Public identity comparison
- Automated test output
- Known limitations and rollback instructions

The evidence bundle must not include raw CXF payloads from real accounts.
