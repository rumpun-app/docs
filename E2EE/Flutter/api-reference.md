# Flutter SDK API reference

This page catalogs the callable Flutter/Dart surface exported by `package:rumpun_sdk_dart/rumpun_sdk_dart.dart`, plus the generated object-crypto calls currently imported from `src/rust/api/api_operations.dart`.

> **Status: `NOT_PRODUCTION_SAFE`.** Use synthetic data only. Protected object operations remain fail-closed in ordinary release builds until Task 6 authorization and Gate G release acceptance.

## Contents

- [Bridge and lifecycle](#bridge-and-lifecycle)
- [Enrollment, restoration, and device closure](#enrollment-restoration-and-device-closure)
- [Groups and lookup](#groups-and-lookup)
- [Membership and devices](#membership-and-devices)
- [Commit processing and inspection](#commit-processing-and-inspection)
- [Reconciliation](#reconciliation)
- [Object encryption](#object-encryption)
- [Operation control](#operation-control)
- [Version arguments](#version-arguments)
- [Imports](#imports)

For DTOs, operation classes, outcomes, and errors, see [Types, outcomes, and errors](types-outcomes-errors.md).

## Bridge and lifecycle

### `RustLib.init()`

Initializes the generated Flutter Rust Bridge runtime and loads the native library. Call once during application bootstrap when required by the selected FRB loading mode.

### `lifecycleCreate()`

```dart
Future<FfiLifecycle> lifecycleCreate()
```

Creates a new process-local lifecycle actor. Prefer one lifecycle service per application process, not one per widget.

### `lifecycleGetOrCreate()`

```dart
Future<FfiLifecycle> lifecycleGetOrCreate({required List<int> actorId})
```

Returns the process-local lifecycle associated with `actorId`, or creates it. `actorId` is an application identity selector, not an authorization token or serialized lifecycle.

### `lifecycleHandle()`

```dart
Future<NativeLifecycleHandle> lifecycleHandle({
  required FfiLifecycle lifecycle,
})
```

Returns the native lifecycle attachment handle used by supported adapter attachment flows. Treat it as opaque and process-local.

### `lifecycleShutdown()`

```dart
Future<void> lifecycleShutdown({required FfiLifecycle lifecycle})
```

Drains and shuts down the actor. All device and group handles owned by it become unusable. Call during controlled application teardown.

## Enrollment, restoration, and device closure

All calls below accept optional `versionWire`; omit it to use `CallVersion.forFfi()`.

### `lifecycleEnroll()`

```dart
Future<FfiHandleOperation> lifecycleEnroll({
  required FfiLifecycle lifecycle,
  List<int>? versionWire,
  required List<int> accountId,
  required List<int> deviceId,
})
```

Enrolls a device and returns an operation whose result is the device handle. Consume both `awaitResult()` and `awaitOutcome()` exactly once.

### `lifecycleRestore()`

```dart
Future<FfiHandleOperation> lifecycleRestore({
  required FfiLifecycle lifecycle,
  List<int>? versionWire,
})
```

Restores sealed lifecycle state and returns the restored device handle operation. Rollback or corrupted persistence fails closed.

### `lifecycleCloseDevice()`

```dart
Future<UnitOperation> lifecycleCloseDevice({
  required FfiLifecycle lifecycle,
  List<int>? versionWire,
  required FfiHandle deviceWire,
})
```

Closes the local device capability. This is not the same as removing that device from an MLS group. Consume result and publication outcome.

## Groups and lookup

### `lifecycleCreateGroup()`

```dart
Future<FfiHandleOperation> lifecycleCreateGroup({
  required FfiLifecycle lifecycle,
  List<int>? versionWire,
  required FfiHandle deviceWire,
  required List<int> groupId,
})
```

Creates a group and returns its opaque handle operation. `groupId` is authenticated group identity, not a display label.

### `lifecycleFindGroup()`

```dart
Future<FfiHandle> lifecycleFindGroup({
  required FfiLifecycle lifecycle,
  List<int>? versionWire,
  required FfiHandle deviceWire,
  required List<int> groupId,
})
```

Finds an existing group owned by the restored device and returns a fresh opaque capability.

### `lifecycleCreateKeyPackage()`

```dart
Future<BytesOperation> lifecycleCreateKeyPackage({
  required FfiLifecycle lifecycle,
  List<int>? versionWire,
  required FfiHandle deviceWire,
})
```

Creates the MLS key package sent to an existing group member. Consume bytes and terminal outcome before publishing or discarding it.

### `lifecycleJoinWelcome()`

```dart
Future<FfiHandleOperation> lifecycleJoinWelcome({
  required FfiLifecycle lifecycle,
  List<int>? versionWire,
  required FfiHandle deviceWire,
  required List<int> welcomeBytes,
})
```

Authenticates and joins an MLS Welcome, returning the joined group handle operation. Never deserialize or reinterpret Welcome bytes in Dart.

## Membership and devices

### `lifecycleAddMember()`

```dart
Future<CommitWelcomeOperation> lifecycleAddMember({
  required FfiLifecycle lifecycle,
  List<int>? versionWire,
  required FfiHandle groupWire,
  required List<int> keyPackageBytes,
})
```

Adds the key-package owner. The result contains `commit` for current members and `welcome` for the joining device. Publish neither until the operation's terminal semantics are handled.

### `lifecycleRemoveMember()`

```dart
Future<BytesOperation> lifecycleRemoveMember({
  required FfiLifecycle lifecycle,
  List<int>? versionWire,
  required FfiHandle groupWire,
  required List<int> accountId,
})
```

Removes every current device belonging to an account and returns the ordered MLS Commit bytes.

### `lifecycleRemoveDevice()`

```dart
Future<BytesOperation> lifecycleRemoveDevice({
  required FfiLifecycle lifecycle,
  List<int>? versionWire,
  required FfiHandle groupWire,
  required int leafIndex,
})
```

Removes one roster leaf and returns the ordered MLS Commit bytes. Resolve `leafIndex` from `lifecycleListMemberDevices()`, never from cached UI position.

### `lifecycleSelfUpdate()`

```dart
Future<BytesOperation> lifecycleSelfUpdate({
  required FfiLifecycle lifecycle,
  List<int>? versionWire,
  required FfiHandle groupWire,
})
```

Rotates the caller's MLS leaf material and returns the Commit for ordered delivery to peers.

## Commit processing and inspection

### `lifecycleProcessOrderedCommit()`

```dart
Future<UnitOperation> lifecycleProcessOrderedCommit({
  required FfiLifecycle lifecycle,
  List<int>? versionWire,
  required FfiHandle groupWire,
  required List<int> commitBytes,
})
```

Processes exactly the next expected Commit. Out-of-order or replayed input fails closed.

### `lifecycleCatchUpSequentially()`

```dart
Future<UnitOperation> lifecycleCatchUpSequentially({
  required FfiLifecycle lifecycle,
  List<int>? versionWire,
  required FfiHandle groupWire,
  required List<Uint8List> commitBytesList,
})
```

Processes a caller-supplied sequence in order. Do not sort, skip, or parallelize the list.

### `lifecycleInspectGroup()`

```dart
Future<FfiGroupStatus> lifecycleInspectGroup({
  required FfiLifecycle lifecycle,
  List<int>? versionWire,
  required FfiHandle groupWire,
})
```

Returns a non-secret snapshot containing the current epoch and member count.

### `lifecycleListMemberDevices()`

```dart
Future<List<FfiMemberDevice>> lifecycleListMemberDevices({
  required FfiLifecycle lifecycle,
  List<int>? versionWire,
  required FfiHandle groupWire,
})
```

Returns the current non-secret roster: account ID, leaf index, and signature public key for each device.

## Reconciliation

### `lifecycleReconcile()`

```dart
Future<FfiHandleOperation> lifecycleReconcile({
  required FfiLifecycle lifecycle,
  List<int>? versionWire,
})
```

Explicitly resolves durable ambiguity and returns a usable handle operation when reconciliation succeeds. This is the only valid path after `PublicationOutcome.ambiguous`; never automatically replay the original mutation.

## Object encryption

These generated operations are not currently re-exported by the package root. Import them explicitly as shown under [Imports](#imports). The Rust core owns keys, AAD, nonces, validation, and cryptographic work.

### `encryptObjectVersionV1()`

```dart
Future<ObjectMutationV1> encryptObjectVersionV1({
  required FfiLifecycle lifecycle,
  required List<int> versionWire,
  required FfiHandle scopeGroupHandle,
  required ObjectContentContextV1 content,
  required List<int> plaintext,
})
```

Encrypts one immutable object version with a fresh CEK and returns an encrypted bundle. Never persist plaintext beside it.

### `decryptObjectVersionV1()`

```dart
Future<Uint8List> decryptObjectVersionV1({
  required FfiLifecycle lifecycle,
  required List<int> versionWire,
  required FfiHandle scopeGroupHandle,
  required ObjectContentContextV1 expectedContentContext,
  required EncryptedObjectBundleV1 bundle,
})
```

Authenticates the independently supplied expected identity and bundle before returning plaintext. Do not derive `expectedContentContext` from the bundle.

### `rewrapObjectCekV1()`

```dart
Future<RewrapObjectCekResultV1> rewrapObjectCekV1({
  required FfiLifecycle lifecycle,
  required List<int> versionWire,
  required FfiHandle scopeGroupHandle,
  required ObjectContentContextV1 expectedContentContext,
  required ObjectWrapContextV1 sourceWrapContext,
  required EncryptedObjectBundleV1 bundle,
  required ObjectWrapContextV1 destinationWrapContext,
})
```

Adds a new CEK wrap while retaining content context, content nonce, and content ciphertext byte-for-byte. Keep the old wrap when historical access is required.

## Operation control

Mutation factories return one of `FfiHandleOperation`, `BytesOperation`, `CommitWelcomeOperation`, or `UnitOperation`. Each provides:

```dart
Future<OperationId> operationId();
Future<T> awaitResult();
Future<PublicationOutcome> awaitOutcome();
Future<CancelToken> cancel();
```

Consume result and outcome exactly once. Cancellation requests cancellation; it does not prove non-commit.

### `cancelTokenCancel()`

```dart
Future<void> cancelTokenCancel({required CancelToken cancel})
```

Signals a previously obtained cancellation token. Still await the operation's terminal outcome and reconcile if it is ambiguous.

## Version arguments

Public lifecycle wrappers default omitted `versionWire` to:

```dart
CallVersion.forFfi().toWireBytes()
```

Object operations currently require that 14-byte wire explicitly. Do not handcraft or downgrade it. Rust performs semantic admission.

## Imports

Public lifecycle surface:

```dart
import 'package:rumpun_sdk_dart/rumpun_sdk_dart.dart' as sdk;
```

Generated object DTOs and operations:

```dart
import 'package:rumpun_sdk_dart/src/rust/api.dart' as api;
import 'package:rumpun_sdk_dart/src/rust/api/api_operations.dart'
    as operations;
```

The `src/` imports are a temporary generated boundary. Pin the exact SDK revision and update imports only with an reviewed SDK change.