# Flutter and native guide

> **Status: `NOT_PRODUCTION_SAFE`.** Use synthetic data only until production authority, platform providers, and release review are accepted.

## Architecture

```text
Flutter public API
-> generated Dart/FRB bindings
-> Rust bridge facade
-> native actor
-> worker-owned Rust lifecycle
-> sealed platform persistence
```

Dart must not perform cryptography, construct AAD, inspect handles, infer publication outcomes, or provide authority booleans.

## Prerequisites

The verified development toolchain currently uses Rust 1.91.1, Flutter 3.47.0, Dart 3.13.0, and flutter_rust_bridge 2.12.0.

```bash
cd packages/sdk-dart
flutter pub get
dart analyze lib
flutter test
```

Use the SDK repository's FRB bootstrap and build scripts for the selected native target. Do not hand-edit generated bindings.

## Create a lifecycle and enroll

```dart
import 'dart:typed_data';
import 'package:rumpun_sdk_dart/rumpun_sdk_dart.dart';

final lifecycle = await lifecycleCreate();
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

Consume `awaitResult()` and `awaitOutcome()` exactly once. A successful result alone is not publication proof.

## Create and inspect a group

```dart
final create = await lifecycleCreateGroup(
  lifecycle: lifecycle,
  deviceWire: device,
  groupId: Uint8List.fromList('synthetic-family-group'.codeUnits),
);

final group = await create.awaitResult();
if (await create.awaitOutcome() != PublicationOutcome.committed) {
  throw StateError('Group creation did not commit');
}

final status = await lifecycleInspectGroup(
  lifecycle: lifecycle,
  groupWire: group,
);
print('epoch=${status.epoch}, members=${status.memberCount}');
```

Production object authority is not created by a group ID. Task 6 must verify current authority before protected production operations become reachable.

## Cancel an operation

```dart
final operation = await lifecycleCreateKeyPackage(
  lifecycle: lifecycle,
  deviceWire: device,
);

final token = await operation.cancel();
await cancelTokenCancel(cancel: token);

try {
  await operation.awaitResult();
} on FfiError catch (error) {
  if (error.code != FfiErrorCode.canceled) rethrow;
}

final terminal = await operation.awaitOutcome();
if (terminal != PublicationOutcome.notCommitted) {
  throw StateError('Unexpected cancellation outcome: $terminal');
}
```

Cancellation is cooperative at defined boundaries. Never assume cancellation means no commit without checking the terminal outcome.

## Restore

Shut down the current lifecycle, create a fresh one over the same platform-owned state location, then restore:

```dart
await lifecycleShutdown(lifecycle: lifecycle);

final restoredLifecycle = await lifecycleCreate();
final restore = await lifecycleRestore(lifecycle: restoredLifecycle);
final restoredDevice = await restore.awaitResult();

if (await restore.awaitOutcome() != PublicationOutcome.committed) {
  throw StateError('Restore did not commit');
}
```

Every pre-restart handle is stale. Obtain fresh handles through the restored lifecycle; never serialize or rebuild handle bytes.

## Error handling

Catch `FfiError` and branch on `FfiErrorCode`. Resolve fresh handles for `invalidHandle` or `closedHandle`; treat `replayReservationConflict` as an ownership conflict. Offer explicit retry for `secureKeyUnavailable` or `trustedTimeUnavailable` only after the prior outcome is known.

Do not surface provider text, stack traces, secret material, plaintext, or raw handles in logs and analytics.

## Isolates and ownership

Opaque lifecycle and handle capabilities are not transferable between Dart isolates. Use the supported attach-by-identity API when multiple isolates must reach one actor; do not pass opaque values through `SendPort`.

## Current limitations

- Linux is the currently verified native development host.
- Android requires NDK and device evidence.
- Apple targets require macOS, Xcode, and physical-device evidence.
- Production authorization remains blocked by Task 6.
- There is no offline mutation replay or reconnect auto-replay.
- Raw key export and plaintext backend processing are forbidden.

The authoritative executable quickstart is `packages/sdk-dart/test/g6_quickstart_test.dart`, run by `scripts/gate-g-g6-quickstart-harness.sh` in the SDK repository. Check it against the exact SDK revision before integrating.
