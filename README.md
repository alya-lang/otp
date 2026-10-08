# otp

[![CI](https://github.com/alya-lang/otp/actions/workflows/ci.yml/badge.svg)](https://github.com/alya-lang/otp/actions/workflows/ci.yml)
[![License](https://img.shields.io/github/license/alya-lang/otp?color=blue&label=License)](LICENSE)
[![Alya](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Fotp%2Fmain%2Falya.toml&query=%24.package.alya-version&label=Alya&color=orange&prefix=%3E%3D)](https://github.com/alya-lang/alya)
[![Package Version](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Fotp%2Fmain%2Falya.toml&query=%24.package.version&label=Version&color=brightgreen)](alya.toml)

RFC 4226 HOTP and RFC 6238 TOTP one-time password toolkit for Alya backed by alya-lang/crypto

---

## 🌟 Features

- 🔑 **RFC 4226 HOTP**: Counter-based one-time passwords with 8-byte big-endian encoding and dynamic truncation
- ⏱️ **RFC 6238 TOTP**: Time-based codes over HOTP with configurable step, epoch offset, and live-clock generation via `time()`
- 🧮 **Multi-Algorithm HMAC**: SHA-1 (interoperable default) plus SHA-256 and SHA-512 TOTP extensions, dispatched through `OtpAlgorithm`
- ✅ **Constant-Time Verification**: HOTP resync windows returning the matched counter, TOTP look-around windows, all comparisons via `crypto::constant_time_eq`
- 🎲 **Secret Provisioning**: Fresh random secrets from `crypto::random_bytes`, emitted as unpadded Base32 for authenticator enrollment
- 🔗 **otpauth Enrollment URIs**: Authenticator QR-code flows with `build_otpauth_uri` / `parse_otpauth_uri` (default-on `uri` feature slice)
- 🧪 **RFC Test Vectors**: Full Appendix D (HOTP) and Appendix B (TOTP SHA-1/256/512) conformance coverage in `tests/test_basic.alya`
- 📦 **Zero Native Code**: Pure Alya on top of `alya-lang/crypto` — no C toolchain beyond the standard compiler pipeline

---

## 📁 Project Architecture

```
otp/
├── .alyalint               # Linter configuration (rules, exclusions, severity overrides)
├── .editorconfig           # Uniform formatting rules across IDEs and editors
├── .gitignore              # Ecosystem standard ignore filters
├── .vscode/                # VS Code workspace settings, DAP launch configurations & tasks
├── alya.toml               # Package manifest with crypto dependency and [features]
├── src/
│   ├── lib.alya            # Public API facade (pub exports, config-bound wrappers, uri stubs)
│   ├── types.alya          # Data models: OtpAlgorithm enum, OtpConfig struct, factories, methods
│   └── core/               # Subdirectory module hierarchy
│       ├── hotp.alya       # HOTP generation: counter encoding, digest dispatch, truncation
│       ├── totp.alya       # TOTP generation, verify_hotp/verify_totp, generate_secret
│       └── uri.alya        # Feature-gated (`uri`) otpauth enrollment URI builder & parser
├── examples/
│   └── demo.alya           # Comprehensive runnable walkthrough of all package capabilities
├── tests/
│   └── test_basic.alya     # Automated test suite with RFC vector conformance coverage
└── benches/
    └── bench_basic.alya    # Micro-benchmarks measuring generation and verification throughput
```

> [!NOTE]
> **Visibility & Modularity:** Symbols annotated with `pub` (`pub function`, `pub struct`, `pub enum`) are exported to external consumers and re-exporting modules. Symbols without `pub` remain strictly internal to their declaring module, preventing symbol collisions and implementation leakage.

---

## 📦 Installation

Add `otp` to the `[dependencies]` section in your `alya.toml`:

```toml
[dependencies]
otp = { git = "https://github.com/alya-lang/otp", branch = "main" }
```

Or install it directly using the Alya package CLI:

```bash
alya add otp --git https://github.com/alya-lang/otp --branch main
alya install
```

---

## 🚀 Quick Start

```alya
import "otp" as otp

function main()
    let secret = "GEZDGNBVGY3TQOJQGEZDGNBVGY3TQOJQ"

    # 1. Counter-based code (RFC 4226)
    let hotp_code = otp::hotp(secret, 0)
    say f"HOTP: {hotp_code}" # "755224"

    # 2. Time-based code for the current moment
    let totp_code = otp::totp_now(secret)
    say f"TOTP: {totp_code}"

    # 3. Verify a user-supplied code with a look-around window
    if otp::verify_totp_now(secret, totp_code)
        say "Accepted"
    end

    # 4. Config-bound facade with enrollment URI
    let cfg = otp::new_totp_config("alice@example.com", "Example")
    say f"Config: {cfg.summary()}"
    say otp::build_otpauth_uri("totp", secret, cfg)
end

main()
```

---

## 📖 API Reference

| Symbol | Visibility | Description |
|---|---|---|
| `hotp(secret, counter, digits, algorithm)` | `pub function` | Generates an HOTP code from a Base32 secret. Defaults: `digits = 6`, `OtpAlgorithm.Sha1`. |
| `hotp_bytes(secret_bytes, counter, digits, algorithm)` | `pub function` | Generates an HOTP code from raw secret bytes. |
| `totp_at(secret, timestamp, step, t0, digits, algorithm)` | `pub function` | Generates a TOTP code at an explicit Unix timestamp. Defaults: `step = 30`, `t0 = 0`. |
| `totp_at_bytes(secret_bytes, timestamp, step, t0, digits, algorithm)` | `pub function` | Generates a TOTP code from raw secret bytes at an explicit timestamp. |
| `totp_now(secret, step, t0, digits, algorithm)` | `pub function` | Generates a TOTP code for the current system time. |
| `verify_hotp(secret, code, counter, window, digits, algorithm)` | `pub function` | Verifies an HOTP code over a resync window; returns the matched counter or `-1`. |
| `verify_totp(secret, code, timestamp, window, step, t0, digits, algorithm)` | `pub function` (`-> bool`) | Verifies a TOTP code with a look-around window at an explicit timestamp. |
| `verify_totp_now(secret, code, window, step, t0, digits, algorithm)` | `pub function` (`-> bool`) | Verifies a TOTP code against the current time. |
| `generate_secret(length)` | `pub function` | Generates a fresh random secret as unpadded Base32. Defaults to `20` bytes. |
| `counter_to_bytes(counter)` | `pub function` | Encodes a counter as 8 big-endian bytes (RFC 4226 section 5.1). |
| `new_totp_config(label, issuer, digits, period, algorithm)` | `pub function` | Factory constructing a time-based `OtpConfig` (defaults: 6 digits, 30s step, SHA-1). |
| `new_hotp_config(label, issuer, digits, algorithm)` | `pub function` | Factory constructing a counter-based `OtpConfig` (`period` is `0`). |
| `make_totp_config(label, issuer, digits, period, algorithm)` | `pub function` | Full validating constructor for time-based `OtpConfig`. |
| `make_hotp_config(label, issuer, digits, algorithm)` | `pub function` | Full validating constructor for counter-based `OtpConfig`. |
| `totp_for(cfg, secret, timestamp)` | `pub function` | Config-bound TOTP generation at an explicit timestamp. |
| `totp_now_for(cfg, secret)` | `pub function` | Config-bound TOTP generation for the current time. |
| `verify_totp_for(cfg, secret, code, timestamp, window)` | `pub function` (`-> bool`) | Config-bound TOTP verification at an explicit timestamp. |
| `algorithm_name(algorithm)` | `pub function` | Returns `"SHA1"`, `"SHA256"`, or `"SHA512"` for an `OtpAlgorithm` value. |
| `build_otpauth_uri(kind, secret, cfg, counter)` | `pub function` (`uri` feature, default-on) | Builds an `otpauth://` enrollment URI; stub throws a descriptive error when the feature is off. |
| `parse_otpauth_uri(uri)` | `pub function` (`uri` feature, default-on) | Parses an `otpauth://` URI into a parameter map; stub throws a descriptive error when the feature is off. |
| `OtpAlgorithm` | `pub enum` | Hash selectors (`Sha1 = 0`, `Sha256 = 1`, `Sha512 = 2`). |
| `OtpConfig` | `pub struct` | Account model (`issuer`, `label`, `digits`, `period`, `algorithm`). |
| `OtpConfig.summary()` | `pub method` | Single-line summary (e.g. `"Example:alice (6 digits, SHA1)"`). |
| `OtpConfig.describe()` | `pub method` | Detailed multi-field description. |
| `OtpConfig.is_valid()` | `pub method` | Returns 1 if the config satisfies OTP constraints, 0 otherwise. |
| `OtpConfig.is_hotp()` | `pub method` | Returns 1 for counter-based configs (`period == 0`), 0 otherwise. |
| `OtpConfig.with_digits(new_digits)` | `pub method` | Immutable copy with updated code length. |
| `OtpConfig.with_algorithm(new_algorithm)` | `pub method` | Immutable copy with updated hash selector. |

> [!TIP]
> **Internal Helpers & Documentation:** Public symbols are documented with `##` Markdown docstrings, enabling automatic API documentation generation via `alya doc`. Private functions such as `truncate_internal`, `digest_internal`, and `find_first_internal` are not annotated with `pub` and remain encapsulated within their respective modules.

---

## 🧪 Running Tests & Benchmarks

Run the automated test suite using `alya test`:

```bash
alya test
```

Exercise feature selection (the `uri` slice is default-on):

```bash
alya test --features uri
alya test --no-default-features
```

Generate static API documentation:

```bash
alya doc . -o docs --markdown
```

Run the benchmark suite:

```bash
alya run benches/bench_basic.alya
```

Run the example demo:

```bash
alya run examples/demo.alya
```

Check code formatting:

```bash
alya fmt . --check
```

Run static code linter:

```bash
alya lint . --check
```

---

### 💻 Developer Tooling & VS Code Integration

This package comes preconfigured with recommended workspace settings and tasks for **Visual Studio Code**:
- **LSP & Formatting**: Auto-formatting on save and real-time Language Server diagnostics via `alya-lang.vscode-alya`.
- **DAP Debugging**: Launch configurations in `.vscode/launch.json` ready for interactive step-debugging via `F5`.
- **Predefined Tasks**: Press `Ctrl+Shift+B` or run tasks (`Test`, `Lint`, `Format`, `Build Docs`) directly from the Command Palette.

---

## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository and clone it locally
2. Install dependencies:
   ```bash
   alya install
   ```
3. Create your feature branch (`git checkout -b feature/my-feature`)
4. Verify tests and formatting before opening a PR:
   ```bash
   alya test
   ```
5. Commit your changes (`git commit -m "feat: add feature"`) and open a Pull Request

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
