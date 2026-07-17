# Upstream baselines and fork builds

Pinned upstream and fork SHAs, toolchain versions, and build verification for all Bitwarden fork repositories.

## Pinned SHAs

| Repository | Fork (origin) | Fork SHA | Upstream | Upstream SHA | Build system |
| --- | --- | --- | --- | --- | --- |
| server | nuri-com/server | `eeb1c26d97d859e8901da479f6cee9ef5eee9bbc` | bitwarden/server | (fetch required) | .NET / C# |
| sdk-internal | nuri-com/sdk-internal | `f0ac1382c1518051e0c01e01dfd574f1f47c3a52` | bitwarden/sdk-internal | (fetch required) | Rust / Cargo |
| ios | nuri-com/ios | `e0ad5b96916c6f6bc215127fd62206c9166c4eff` | bitwarden/ios | (fetch required) | Xcode / Swift |
| android | nuri-com/android | `634c1fc0e297b0239b1f16d5ec0b3328b9f88e84` | bitwarden/android | (fetch required) | Gradle / Kotlin |
| coordination | nuri-com/bitwarden-passkey-prf | `610e0c82798ca9e3793470a531da3a666e2c25d3` | — | — | Markdown docs |

## Toolchain versions

| Tool | Version | How to verify |
| --- | --- | --- |
| .NET SDK | (run `dotnet --version`) | server repo build |
| Rust | (run `rustc --version`) | sdk-internal repo build |
| Xcode | (run `xcodebuild -version`) | ios repo build |
| Java/Gradle | (run `java -version` and `./gradlew --version`) | android repo build |
| Git | (run `git --version`) | all repos |
| SSH signing | `gpg.format=ssh`, `commit.gpgsign=true`, key: `/Users/eminmahrt/.ssh/github_signing_ed25519.pub` | global git config |

## Repository details

### server (nuri-com/server)

- **Local path:** `/Users/eminmahrt/Developer/nuri-bitwarden/server`
- **Fork SHA:** `eeb1c26d97d859e8901da479f6cee9ef5eee9bbc` (main)
- **Remotes:** origin → nuri-com/server, upstream → bitwarden/server
- **Build system:** .NET / C# (look for .sln/.csproj files)
- **Status:** Clean working tree on main, tracking origin/main
- **Build command:** `dotnet build` (requires .NET SDK installed)
- **Notes:** Server stores FIDO2 credentials as opaque encrypted strings. The FIDO2 data model (`CipherLoginFido2CredentialData.cs`) currently has no seed field. The wire model (`CipherFido2CredentialModel.cs`) uses `[EncryptedString]` for all fields. `SupportsPrf` is a boolean DB column.

### sdk-internal (nuri-com/sdk-internal)

- **Local path:** `/Users/eminmahrt/Developer/nuri-bitwarden/sdk-internal`
- **Fork SHA:** `f0ac1382c1518051e0c01e01dfd574f1f47c3a52` (main)
- **Remotes:** origin → nuri-com/sdk-internal, upstream → bitwarden/sdk-internal
- **Build system:** Rust / Cargo (look for Cargo.toml)
- **Status:** Clean working tree on main, tracking origin/main
- **Build command:** `cargo check` or `cargo build`
- **Notes:** Contains the CXF import/export code, Fido2Authenticator, passkey types, bitwarden-crypto (PRF derivation, HighEntropySecret, ZeroizingAllocator). CXF export currently emits `fido2_extensions: None` (load-bearing, must not regress). PRF derivation lives in `bitwarden-crypto/src/keys/prf.rs`.

### ios (nuri-com/ios)

- **Local path:** `/Users/eminmahrt/Developer/nuri-bitwarden/ios`
- **Fork SHA:** `e0ad5b96916c6f6bc215127fd62206c9166c4eff` (main)
- **Remotes:** origin → nuri-com/ios, upstream → bitwarden/ios
- **Build system:** Xcode / Swift (look for .xcodeproj/.xcworkspace)
- **Status:** Clean working tree on main, tracking origin/main
- **Build command:** `xcodebuild -list` (to verify project structure), full build requires Xcode + signing
- **Notes:** Contains iOS 26 Credential Exchange import/export surfaces. No `fido2Extensions` symbol in the tree. PRF is modeled but dropped at Apple-bridging boundaries (GetAssertionRequest hardcodes extensions: nil, tagged PM-26177).

### android (nuri-com/android)

- **Local path:** `/Users/eminmahrt/Developer/nuri-bitwarden/android`
- **Fork SHA:** `634c1fc0e297b0239b1f16d5ec0b3328b9f88e84` (main)
- **Remotes:** origin → nuri-com/android, upstream → bitwarden/android
- **Build system:** Gradle / Kotlin (build.gradle.kts)
- **Status:** Clean working tree on main, tracking origin/main
- **Build command:** `./gradlew tasks` (to verify project structure)
- **Notes:** AndroidX credentials 1.6.0, providerevents 1.0.0-alpha06, compileSdk 37, minSdk 29. `BitwardenCredentialProviderService` gated by `@RequiresApi(UPSIDE_DOWN_CAKE)` (API 34). 6 implementation gaps in PRF provider contract identified.

## SSH signing configuration

```
user.signingkey=/Users/eminmahrt/.ssh/github_signing_ed25519.pub
gpg.format=ssh
commit.gpgsign=true
tag.gpgsign=true
gpg.ssh.allowedsignersfile=/Users/eminmahrt/.config/git/allowed_signers
```

All commits use SSH signing. Never disable signing.

## Reproducible setup

1. Clone all four forks with upstream remotes configured:
   ```bash
   git clone https://github.com/nuri-com/server.git
   cd server && git remote add upstream https://github.com/bitwarden/server.git
   # Repeat for sdk-internal, ios, android
   ```
2. Verify SHAs match the pinned values above.
3. Install required toolchains (dotnet, rust, xcode, java/gradle).
4. Run build verification commands per repo.
5. All commits must use the existing SSH signing configuration.

## Notes

- Full builds of server (dotnet) and ios (xcode) may require additional dependencies not installed on this machine. A "focused documented substitute" (e.g., `cargo check`, `xcodebuild -list`, `./gradlew tasks`) is acceptable per the acceptance criteria.
- The coordination repo is not a Git fork of any upstream; it is the original nuri-com/bitwarden-passkey-prf repository.