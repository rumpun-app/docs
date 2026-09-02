# Rumpun E2EE SDK for Flutter

A focused path for integrating the native Rumpun E2EE SDK into a Flutter application.

> **Current status: `NOT_PRODUCTION_SAFE`.** Use synthetic data only. Production authorization remains unavailable until Task 6 and release review are accepted.

## Start here

1. [Install and initialize](getting-started.md)
2. [Manage lifecycle and groups](lifecycle-and-groups.md)
3. [Encrypt, decrypt, and re-key objects](object-encryption.md)
4. [Test Flutter end-to-end](end-to-end-testing.md)

## Choose what you need

- **New integration:** start with installation, then lifecycle and groups.
- **Encrypting application data:** use the object encryption guide.
- **Writing CI or regression tests:** use the end-to-end testing guide.

## Mental model

```text
Flutter app
  -> generated Dart API
  -> flutter_rust_bridge
  -> native Rust actor
  -> authoritative Rust lifecycle
  -> sealed local persistence
```

Dart is intentionally thin. It does not generate keys, construct AAD, perform encryption, inspect opaque handles, or decide authorization.

## Golden rules

- Consume every mutation result and publication outcome exactly once.
- Never serialize lifecycle, device, or group handles.
- Never implement fallback crypto in Dart.
- Never treat a valid ciphertext as proof of authorization.
- Never retry an `ambiguous` mutation automatically.
- Never log plaintext, keys, raw handles, or provider diagnostics.

## Supported development environment

The currently verified development toolchain uses:

- Rust 1.91.1
- Flutter 3.47.0
- Dart 3.13.0
- flutter_rust_bridge 2.12.0
- Linux x86_64 as the verified native host

Android still requires NDK and device evidence. Apple targets require macOS, Xcode, and physical-device evidence.

## Source of truth

The executable Flutter example lives in the SDK repository:

- `packages/sdk-dart/test/g6_quickstart_test.dart`
- `scripts/gate-g-g6-quickstart-harness.sh`
- `.github/workflows/task-5e-flutter.yml`

Always check examples against the exact SDK revision used by your application.
