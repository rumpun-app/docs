# Rumpun E2EE SDK for Flutter

A focused path for integrating the native Rumpun E2EE SDK into a Flutter application.

**In plain terms:** your Flutter app talks to a thin layer of generated Dart code, that generated code calls across `flutter_rust_bridge` into a native build of the Rust core, and that Rust core does all the real cryptography. The Dart layer stays deliberately thin so that no security decision is ever made twice: every key, encryption step, and authorization check lives in the one audited Rust core, and Dart only carries typed values back and forth.

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

This is the chain a request travels through, from your app down to storage. Each arrow is one thin hop, and the authoritative decisions happen only in the Rust lifecycle near the bottom.

```text
Flutter app
  -> generated Dart API
  -> flutter_rust_bridge
  -> native Rust actor
  -> authoritative Rust lifecycle
  -> sealed local persistence
```

Dart is intentionally thin. It does not generate keys, construct AAD, perform encryption, inspect opaque handles, or decide authorization. Everything sensitive stays behind the bridge in the Rust core, so Dart cannot leak a key or misjudge an authorization even by accident. (MLS is the group-messaging security protocol that decides who is currently in a family group and derives the group's shared keys.)

## Golden rules

These are the non-negotiable habits that keep your integration safe. Follow every one exactly; each maps to a rule enforced by the Rust core.

- Consume every mutation result and publication outcome exactly once.
- Never serialize lifecycle, device, or group handles.
- Never implement fallback crypto in Dart.
- Never treat a valid ciphertext as proof of authorization.
- Never retry an `ambiguous` mutation automatically.
- Never log plaintext, keys, raw handles, or provider diagnostics.

## Supported development environment

These are the exact tool versions the SDK has been verified against. Pin them for development so your build matches the one the SDK team tests, and treat anything newer as unverified until it is validated.

The currently verified development toolchain uses:

- Rust 1.91.1
- Flutter 3.47.0
- Dart 3.13.0
- flutter_rust_bridge 2.12.0
- Linux x86_64 as the verified native host

Android still requires NDK and device evidence. Apple targets require macOS, Xcode, and physical-device evidence.

## Source of truth

When the docs and the code disagree, the code wins. The executable Flutter example and harness live in the SDK repository; check these against the exact SDK revision your application pins.

- `packages/sdk-dart/test/g6_quickstart_test.dart`
- `scripts/gate-g-g6-quickstart-harness.sh`
- `.github/workflows/task-5e-flutter.yml`

Always check examples and signatures against the exact SDK revision used by your application.