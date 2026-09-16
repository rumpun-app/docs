# Install and initialize

This guide takes you from an empty Flutter project to an enrolled synthetic device in six numbered steps. Each step opens with a one-sentence explanation of what it does and why, followed by the exact commands or code to run. Work through them in order.

## 1. Prerequisites

First make sure your machine runs the exact toolchain the SDK is verified against, so your build matches the one the SDK team tests.

Install the versions pinned by the SDK repository:

```bash
rustup show
flutter --version
dart --version
```

Expected development baseline:

```text
Rust 1.91.1
Flutter 3.47.0
Dart 3.13.0
flutter_rust_bridge_codegen 2.12.0
```

## 2. Prepare the Dart package

Next pull the package's dependencies and confirm it analyzes cleanly, so you start from a known-good state before writing any code.

From the E2EE SDK repository:

```bash
cd packages/sdk-dart
flutter pub get
dart analyze lib
```

Use the repository's FRB bootstrap and native build scripts. Do not install an arbitrary codegen version and do not edit generated files manually.

## 3. Import the public SDK

Now bring the SDK into your Dart code. The first import is the public entry point you will use everywhere; keep the extra operation and DTO imports on only the pages that actually encrypt objects.

```dart
import 'dart:typed_data';
import 'package:rumpun_sdk_dart/rumpun_sdk_dart.dart';
```

Object encryption uses generated operation and DTO types. Import those only on pages that need them:

```dart
import 'package:rumpun_sdk_dart/src/rust/api.dart' as api;
import 'package:rumpun_sdk_dart/src/rust/api/api_operations.dart'
    as operations;
```

## 4. Create a lifecycle

A lifecycle is your handle to one device's Rust core. Creating it starts the native actor and claims that device's process-local capabilities, so open exactly one and keep it around.

```dart
final lifecycle = await lifecycleCreate();
```

One lifecycle owns one native actor and its process-local capabilities. Keep it in a dedicated application service rather than recreating it in every widget.

## 5. Enroll a synthetic device

Enrollment registers this device under a family account so it can join groups. It is a protected mutation, so it returns two things you must both read: a result (the value produced) and a publication outcome (the separate, terminal answer to "was this change actually saved?").

```dart
final enroll = await lifecycleEnroll(
  lifecycle: lifecycle,
  accountId: Uint8List.fromList('synthetic-account'.codeUnits),
  deviceId: Uint8List.fromList('synthetic-device-01'.codeUnits),
);

final device = await enroll.awaitResult();
final outcome = await enroll.awaitOutcome();

if (outcome != PublicationOutcome.committed) {
  throw StateError('Enrollment did not commit: $outcome');
}
```

`awaitResult()` and `awaitOutcome()` are separate, single-consumption values: await each exactly once. The result gives you the produced value, and the outcome tells you whether it durably committed. A successful result on its own is not proof that the change was published, so always consume both for mutations.

## 6. Shut down cleanly

Finally, shut the lifecycle down when you are done so the native actor drains and every handle it minted is invalidated. Do this even on error paths.

```dart
await lifecycleShutdown(lifecycle: lifecycle);
```

Shutdown invalidates handles and drains the actor. Do not reuse `device` or group handles after shutdown.

## Common setup mistakes

These are the errors first-time integrators hit most often. Each one lists what not to do and the short reason it matters.

### Editing generated Dart files

Wrong. Regenerate them from the pinned Rust bridge and reject unexpected diffs. Why: the generated files are the exact contract with the audited Rust core, so a hand edit silently breaks the guarantee that Dart only carries typed values across the boundary.

### Passing opaque handles between isolates

Wrong. Opaque lifecycle and handle capabilities are not transferable through `SendPort`. Use the supported attach-by-identity flow when multiple isolates need one actor. Why: a handle is only valid inside the process and lifecycle that minted it, so shipping its bytes elsewhere yields a stale, unusable capability.

### Ignoring `UnsupportedProtocol`

Wrong. Ordinary release builds intentionally fail protected object operations until production authorization exists. Never replace this with Dart-side crypto. Why: this error is the SDK enforcing `NOT_PRODUCTION_SAFE`, and working around it in Dart would move cryptography out of the one audited core.

## Next

Continue with [Manage lifecycle and groups](lifecycle-and-groups.md).
