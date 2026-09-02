# Install and initialize

## 1. Prerequisites

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

From the E2EE SDK repository:

```bash
cd packages/sdk-dart
flutter pub get
dart analyze lib
```

Use the repository's FRB bootstrap and native build scripts. Do not install an arbitrary codegen version and do not edit generated files manually.

## 3. Import the public SDK

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

```dart
final lifecycle = await lifecycleCreate();
```

One lifecycle owns one native actor and its process-local capabilities. Keep it in a dedicated application service rather than recreating it in every widget.

## 5. Enroll a synthetic device

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

`awaitResult()` and `awaitOutcome()` are separate, single-consumption values. Always consume both for mutations.

## 6. Shut down cleanly

```dart
await lifecycleShutdown(lifecycle: lifecycle);
```

Shutdown invalidates handles and drains the actor. Do not reuse `device` or group handles after shutdown.

## Common setup mistakes

### Editing generated Dart files

Wrong. Regenerate them from the pinned Rust bridge and reject unexpected diffs.

### Passing opaque handles between isolates

Wrong. Opaque lifecycle and handle capabilities are not transferable through `SendPort`. Use the supported attach-by-identity flow when multiple isolates need one actor.

### Ignoring `UnsupportedProtocol`

Wrong. Ordinary release builds intentionally fail protected object operations until production authorization exists. Never replace this with Dart-side crypto.

## Next

Continue with [Manage lifecycle and groups](lifecycle-and-groups.md).
