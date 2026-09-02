# Rumpun E2EE SDK for Flutter

A focused path for integrating the native Rumpun E2EE SDK into a Flutter application.

> **Current status: `NOT_PRODUCTION_SAFE`.** Use synthetic data only. Production authorization remains unavailable until Task 6 and release review are accepted.

## Complete table of contents

### Guides

1. [Install and initialize](getting-started.md)
2. [Manage lifecycle and groups](lifecycle-and-groups.md)
3. [Manage family members and devices](members-and-devices.md)
4. [Encrypt, decrypt, and re-key objects](object-encryption.md)
5. [Test Flutter end-to-end](end-to-end-testing.md)

### API reference

6. [All callable Flutter SDK APIs](api-reference.md)
   - bridge initialization and lifecycle
   - enrollment, restore, and device closure
   - group creation, lookup, and joining
   - member and device mutations
   - ordered Commit processing and inspection
   - reconciliation
   - object encryption, decryption, and CEK rewrap
   - cancellation and operation control
7. [All public types, outcomes, and errors](types-outcomes-errors.md)
   - opaque capabilities and handles
   - operation classes and operation IDs
   - publication outcomes
   - group, roster, and object DTOs
   - complete stable error-code catalog
   - version declarations and wire helpers

## Choose what you need

- **New integration:** start with installation, then lifecycle and groups.
- **Inviting or removing people/devices:** use the member and device management guide.
- **Encrypting application data:** use the object encryption guide.
- **Looking up a signature or return type:** use the callable API reference.
- **Handling outcomes or errors:** use the types, outcomes, and errors reference.
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

Always check examples and signatures against the exact SDK revision used by your application.