# Windows support (experimental)

This guide describes the `feature/windows-support` branch of the unofficial
NativeRobot fork. Windows support is experimental, is not distributed or
supported by Scalable Capital, and has not been validated against a live broker
account. Do not treat a successful local build or test as proof that login,
token refresh, or trading works with the live service.

## Prerequisites

- Git and Rust with `rustup` and Cargo. The repository's `rust-toolchain.toml`
  selects the toolchain; the package declares Rust 1.96 as its minimum.
- A native Windows C/C++ build toolchain and the tools required to build the
  Rust `aws-lc-sys` dependency. For the MSVC Rust target, install the Visual
  Studio C++ Build Tools, including the Windows SDK.
- Network access to fetch the source and build dependencies.

## Build and run

In PowerShell, clone the Windows branch and build the production-channel binary:

```powershell
git clone --branch feature/windows-support https://github.com/NativeRobot/scalable-cli.git
Set-Location scalable-cli
cargo build --locked --release --features channel-prod --bin scalable
.\target\release\scalable.exe --version
.\target\release\scalable.exe --help
```

The binary is `target\release\scalable.exe`. `--features channel-prod` selects
the production channel; keep it on build and test commands for this checkout.
The fork also supports building `--bin sc`, but Windows has its own `sc.exe`
Service Control utility. Use `scalable.exe` or the explicit path above instead
of relying on `sc` to resolve to the intended program. Add the build directory
to your user `PATH` only if you want to invoke `scalable.exe` without its path.

## First login and read-only checks

Before authenticating, enable Scalable CLI on the Scalable web platform under
**Profile → Security → Agentic Investing**. Review the storage limitations below
and protect the configuration directory first. To try a locally enforced
read-only session, run these commands yourself from the repository directory:

```powershell
.\target\release\scalable.exe login --local-read-only
.\target\release\scalable.exe whoami
.\target\release\scalable.exe capabilities --json
.\target\release\scalable.exe broker overview --json
```

Complete the device-code login yourself in a browser. Local read-only mode
blocks CLI write commands locally; it does not limit backend token permissions.
The login and read commands contact the live service. Do not share device codes,
tokens, account data, or session files. End the local session with
`.\target\release\scalable.exe logout` when appropriate.

## Configuration and security

- The configuration directory is `%HOME%\.config\scalable-cli`, or
  `%USERPROFILE%\.config\scalable-cli` when `HOME` is unset or empty. In
  PowerShell, the following prints the directory selected by these rules:

  ```powershell
  $cliHome = if ($env:HOME) { $env:HOME } else { $env:USERPROFILE }
  Join-Path $cliHome '.config\scalable-cli'
  ```

- The default session backend is Windows Credential Manager. Session JSON is
  compressed and stored as one credential, without an automatic file fallback.
  Compression does not guarantee that the credential fits the 2,560-byte
  limit: an oversized session fails to save without overwriting the old entry.
  If a rotated refresh token cannot be saved, a new login may be required.
- An existing file-backed session is not migrated automatically. To use it,
  configure `[auth]` with `session_backend = "file"` in `config.toml`. This
  stores session tokens in plaintext on disk; prefer Credential Manager when
  possible and explicitly protect file storage if used.
- The default signing backend stores the private DPoP signing key in plaintext
  as `auth-signing-key.json` in the configuration directory, even when the
  session uses Credential Manager. The CLI does not set or verify Windows ACLs.
  Review and restrict inherited directory and file permissions before logging
  in; do not assume the profile directory is private on every machine.
- `secure_enclave` is macOS-only and `pkcs11` is Linux-only. See
  [Configuration](configuration.md#windows-experimental) for backend options.

## Local verification

From the repository directory, run the tests and build without live credentials:

```powershell
cargo test --locked --features channel-prod --tests
cargo build --locked --release --features channel-prod --bin scalable
```

An additional ignored test writes a temporary synthetic entry to the current
Windows user's Credential Manager and removes it afterward:

```powershell
cargo test --locked --features channel-prod --lib native_credential_store_round_trip -- --ignored
```

These tests do not establish that authentication, refresh, or broker operations
work with a real account. Review failed tests or build errors before proceeding.

## Troubleshooting

- **The wrong `sc.exe` runs:** invoke `scalable.exe` or its full path instead.
- **Home directory cannot be resolved:** set `HOME` or `USERPROFILE` for the
  process to the intended user profile and retry. At least one must be nonempty.
- **Credential Manager is unavailable:** check that it is accessible to the
  Windows user running the CLI. The program does not silently switch to files.
- **The compressed session exceeds 2,560 bytes:** saving fails explicitly;
  see the storage and refresh caveats above. Compression cannot guarantee that
  all session payloads fit.
- **Native dependency build fails:** check the C/C++ toolchain and SDK for the
  selected Rust target, then rerun the build command and inspect its error.
