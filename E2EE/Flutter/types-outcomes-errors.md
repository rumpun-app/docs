# Flutter types, outcomes, and errors

This page covers every type intentionally exposed by the Flutter package root and the generated DTOs required by object encryption.

## Contents

- [Opaque capabilities](#opaque-capabilities)
- [Operation types](#operation-types)
- [Publication outcomes](#publication-outcomes)
- [Group and roster DTOs](#group-and-roster-dtos)
- [Object crypto DTOs](#object-crypto-dtos)
- [Errors](#errors)
- [Versions and wire helpers](#versions-and-wire-helpers)

## Opaque capabilities

| Type | Meaning | Rule |
|---|---|---|
| `FfiLifecycle` | Native actor capability | Keep process-local; shut down explicitly. |
| `NativeLifecycleHandle` | Supported native lifecycle attachment handle | Opaque; do not serialize or inspect. |
| `FfiHandle` | Device or group capability validated by Rust | Never persist, log, compare as identity, or move through unsupported isolate transport. |
| `OpaqueHandleWire` | Provisional 35-byte wire wrapper | Length-check only; Rust remains the sole semantic validator. |
| `CancelToken` | Cancellation request capability | Cancellation does not prove non-commit. |

`handleWireLen` is `35`. Its existence does not authorize applications to decode or construct handles.

## Operation types

| Type | `awaitResult()` value | Typical producers |
|---|---|---|
| `FfiHandleOperation` | `FfiHandle` | enroll, restore, create group, join Welcome, reconcile |
| `BytesOperation` | `Uint8List` | key package, remove member/device, self update |
| `CommitWelcomeOperation` | `FfiCommitAndWelcome` | add member |
| `UnitOperation` | `void` | close device, process Commit, catch up |

Every operation also exposes `operationId()`, `awaitOutcome()`, and `cancel()`. Result and outcome are separate terminal values and must both be consumed exactly once.

### `OperationId`

```dart
class OperationId {
  final U8Array16 incarnation;
  final int sequence;
}
```

Correlates one admitted operation with its terminal publication outcome. It is diagnostic identity, not authorization.

## Publication outcomes

| Outcome | Meaning | Caller action |
|---|---|---|
| `PublicationOutcome.notCommitted` | Non-commit is proven and durable state remains reusable. | A user-directed retry may be offered if the original error semantics allow it. |
| `PublicationOutcome.committed` | Durable publication is proven. | Continue with ordered delivery and application persistence. |
| `PublicationOutcome.ambiguous` | Commit and non-commit cannot be proven. | Fence ordinary mutation and call `lifecycleReconcile()`. Never auto-retry. |

The package also exposes a provisional discriminant helper with `fromDiscriminant(int)`: `0`, `1`, and `2` map to the outcomes above; unknown values fail closed as ABI mismatch.

## Group and roster DTOs

### `FfiGroupStatus`

```dart
class FfiGroupStatus {
  final int epoch;
  final int memberCount;
}
```

A non-secret current group snapshot.

### `FfiMemberDevice`

```dart
class FfiMemberDevice {
  final Uint8List accountId;
  final int leafIndex;
  final Uint8List signaturePublicKey;
}
```

A non-secret roster entry. `leafIndex` is epoch-sensitive; refresh before device removal.

### `FfiCommitAndWelcome`

```dart
class FfiCommitAndWelcome {
  final Uint8List commit;
  final Uint8List welcome;
}
```

Returned by member addition. Send `commit` to current members in order and `welcome` only to the joining device through the authenticated delivery path.

The package also exports provisional equivalents `GroupStatus`, `MemberDevice`, and `CommitAndWelcome`. Prefer the generated `Ffi*` forms returned by callable APIs; do not convert them into authorization decisions.

## Object crypto DTOs

### `ObjectContentContextV1`

```dart
class ObjectContentContextV1 {
  final Uint8List familyId;
  final Uint8List objectId;
  final Uint8List scopeId;
  final Uint8List objectVersion;
  final int schemaVersion;
}
```

Authenticated immutable object identity. `objectVersion` is exactly eight big-endian bytes representing a non-zero `u64`; schema version is currently `1`.

### `ObjectWrapContextV1`

Extends the same identity with:

```dart
final Uint8List keyVersion; // eight-byte big-endian non-zero u64
final int ciphersuite;      // currently 0x0001
```

The authorized lifecycle supplies the key-version mapping. UI code must not invent it.

### `EncryptedObjectBundleV1`

```dart
class EncryptedObjectBundleV1 {
  final ObjectContentContextV1 contentContext;
  final Uint8List contentNonce;
  final Uint8List ciphertext;
  final ObjectWrapContextV1 wrapContext;
  final Uint8List wrappedCekNonce;
  final Uint8List wrappedCekCiphertext;
}
```

Persist every field together. The bundle contains ciphertext and authenticated metadata, never CEK, KWK, exporter output, or private keys.

### `ObjectMutationV1`

```dart
class ObjectMutationV1 {
  final EncryptedObjectBundleV1 bundle;
}
```

Return value of `encryptObjectVersionV1()`.

### `RewrapObjectCekResultV1`

Contains retained content context, nonce, and ciphertext plus the added destination wrap context, wrapped-CEK nonce, and wrapped-CEK ciphertext. Re-keying is additive and must not alter retained content fields.

## Errors

Callable generated APIs throw:

```dart
class FfiError implements FrbException {
  final FfiErrorCode code;
}
```

### Stable `FfiErrorCode` catalog

| Code | Meaning / caller response |
|---|---|
| `malformedInput` | Reject invalid shape, length, or encoding. |
| `unsupportedSchema` | Upgrade or migrate using an approved path. |
| `unsupportedProtocol` | Feature or production gate is closed; never add fallback crypto. |
| `unsupportedCiphersuite` | Reject the unsupported suite. |
| `downgrade` | Reject a lower contract declaration. |
| `authenticationFailure` | Authentication failed; do not reveal which field mismatched. |
| `invalidSignature` | Reject unauthenticated signed material. |
| `unauthorized` | Current authority denies the operation. |
| `staleAuthorization` | Refresh authority through Task 6's approved path when available. |
| `revokedDeviceOrKey` | Stop using the revoked device or key. |
| `replay` | Reject duplicate authenticated input. |
| `replayReservationConflict` | Another operation owns the replay reservation. |
| `expired` | Input or authority is outside its valid time window. |
| `trustedTimeUnavailable` | A user-directed retry may be offered after trusted time recovers. |
| `secureKeyUnavailable` | A user-directed retry may be offered after the secure provider recovers. |
| `secureKeyRevokedOrDeleted` | Recovery or re-enrollment is required. |
| `signingCanceled` | Signing was canceled; still obey operation outcome semantics. |
| `persistenceCorruption` | Stop and recover from an approved source. |
| `rollbackDetected` | Stop; never silently overwrite newer durable state. |
| `sequenceConflict` | Refresh and process the required order. |
| `invalidHandle` | Resolve a fresh capability from the lifecycle. |
| `closedHandle` | The capability is closed; do not reuse it. |
| `abiMismatch` | SDK and generated adapter do not match. |
| `canceled` | Cancellation reached the operation; terminal outcome still governs mutation state. |
| `resourceLimit` | Reduce pressure or input within documented bounds. |
| `internalFailure` | Fail closed and capture non-secret diagnostics. |

The provisional `RumpunErrorCode.retryable` property is `true` only for `trustedTimeUnavailable` and `secureKeyUnavailable`. Retryability never authorizes automatic replay of an ambiguous mutation.

```dart
try {
  // SDK call
} on FfiError catch (error) {
  switch (error.code) {
    case FfiErrorCode.trustedTimeUnavailable:
    case FfiErrorCode.secureKeyUnavailable:
      // Offer a deliberate retry only when no ambiguous mutation exists.
      break;
    case FfiErrorCode.unsupportedProtocol:
      // Keep the production gate closed.
      break;
    default:
      rethrow;
  }
}
```

## Versions and wire helpers

### `CallVersion`

Declares seven unsigned 16-bit contract values: shared API, shared ABI, protocol, schema, ciphersuite, adapter ABI, and minimum adapter version.

```dart
final wire = CallVersion.forFfi().toWireBytes(); // exactly 14 bytes
```

`toWireBytes()` uses big-endian encoding. Do not handcraft, truncate, or downgrade it.

### Version constants

```dart
sharedCoreApiVersion == 1
sharedCoreAbiVersion == 1
```

These constants describe the shared core contract. Pin the SDK revision and regenerate FRB bindings with the repository-pinned codegen when the contract changes.