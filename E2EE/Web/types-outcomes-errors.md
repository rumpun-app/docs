# Web types, outcomes, errors, and contract

This page catalogs every public type, error, constant, and contract helper exported by `@rumpun/sdk-ts`.

## Contents

- [Lifecycle and opaque values](#lifecycle-and-opaque-values)
- [Mutations and publication outcomes](#mutations-and-publication-outcomes)
- [Group and membership DTOs](#group-and-membership-dtos)
- [Object encryption DTOs](#object-encryption-dtos)
- [Errors](#errors)
- [Contract constants](#contract-constants)
- [Contract helper functions](#contract-helper-functions)

## Lifecycle and opaque values

| Type | Meaning | Rule |
|---|---|---|
| `RumpunLifecycle` | One device's lifecycle, durable store, and exclusive Web Lock owner | Open through `RumpunLifecycle.open()` and always `dispose()`. |
| `DeviceHandle` | Branded opaque device capability | Never decode, persist, log, or reconstruct. |
| `GroupHandle` | Branded opaque group capability | Valid only with the lifecycle that minted or restored it. |
| `KeyPackage` | Branded MLS key-package bytes | Deliver unchanged through authenticated transport. |
| `Commit` | Branded ordered MLS Commit bytes | Deliver and process in exact order. |

Branding prevents accidental interchange at TypeScript compile time. Rust remains the semantic validator.

## Mutations and publication outcomes

### `RumpunMutation<T>`

```ts
interface RumpunMutation<T> {
  readonly operationId: string;
  cancel(): void;
  readonly result: Promise<T>;
  readonly outcome: Promise<PublicationOutcome>;
}
```

Consume both promises. `operationId` binds the mutation to its terminal outcome and exact reconciliation request; it is not authority.

### `PublicationOutcome`

```ts
type PublicationOutcome =
  | "NotCommitted"
  | "Committed"
  | "Ambiguous";
```

| Outcome | Meaning | Required response |
|---|---|---|
| `NotCommitted` | Non-commit is proven and durable state remains reusable. | A deliberate retry may be offered only when error semantics permit it. |
| `Committed` | Durable publication is proven. | Continue ordered delivery and application persistence. |
| `Ambiguous` | Neither commit nor non-commit can be proven. | Fence mutation and call `reconcile(operationId)`. Never auto-retry. |

Wire constants are `OUTCOME_NOT_COMMITTED = 0`, `OUTCOME_COMMITTED = 1`, `OUTCOME_AMBIGUOUS = 2`, and `OUTCOME_COUNT = 3`.

## Group and membership DTOs

### `GroupStatusDto`

```ts
interface GroupStatusDto {
  epoch: number;
  memberCount: number;
}
```

A non-secret snapshot returned by `inspectGroup()`.

### `MemberDeviceDto`

```ts
interface MemberDeviceDto {
  accountId: Uint8Array;
  leafIndex: number;
  signaturePublicKey: Uint8Array;
}
```

A non-secret current roster entry. `leafIndex` changes with group state and must be refreshed before removal.

### `CommitAndWelcomeDto`

```ts
interface CommitAndWelcomeDto {
  commit: Commit;
  welcome: Uint8Array;
}
```

Returned by `addMember()`. Current members receive the ordered Commit; only the joining device receives the Welcome.

## Object encryption DTOs

### `ObjectContentContextV1`

```ts
interface ObjectContentContextV1 {
  readonly familyId: Uint8Array;
  readonly objectId: Uint8Array;
  readonly scopeId: Uint8Array;
  readonly objectVersion: bigint;
  readonly schemaVersion: number;
}
```

Authenticated immutable identity. `objectVersion` must be a non-zero lossless `bigint`; the WASM boundary transports it as a decimal string. Schema version is currently `1`.

### `EncryptedObjectBundleV1`

```ts
interface EncryptedObjectBundleV1 {
  readonly contentContext: ObjectContentContextV1;
  readonly contentNonce: Uint8Array;
  readonly ciphertext: Uint8Array;
  readonly wrapContext: object;
  readonly wrappedCekNonce: Uint8Array;
  readonly wrappedCekCiphertext: Uint8Array;
}
```

Persist every field together. It contains encrypted transport and authenticated metadata, never CEK, KWK, exporter output, private keys, verifiers, or lifecycle implementation values.

`wrapContext` is intentionally opaque on the current TypeScript surface. Applications may transport the value returned by Rust but must not infer authority or fabricate destination key versions.

## Errors

```ts
class RumpunSdkError extends Error {
  readonly code: RumpunErrorCode;
}
```

Catch by class and branch on `code`. Unknown boundary values map fail-closed to `AbiMismatch`.

### Stable `RumpunErrorCode` catalog

| Code | Meaning / caller response |
|---|---|
| `MalformedInput` | Reject invalid shape, length, or encoding. |
| `UnsupportedSchema` | Upgrade or migrate through an approved path. |
| `UnsupportedProtocol` | Feature or production gate is closed; never add fallback crypto. |
| `UnsupportedCiphersuite` | Reject the unsupported suite. |
| `Downgrade` | Reject a lower contract declaration. |
| `AuthenticationFailure` | Authentication failed; do not reveal the mismatched field. |
| `InvalidSignature` | Reject unauthenticated signed material. |
| `Unauthorized` | Current authority denies the operation. |
| `StaleAuthorization` | Refresh authority through the approved Task 6 path when available. |
| `RevokedDeviceOrKey` | Stop using the revoked device or key. |
| `Replay` | Reject duplicate authenticated input. |
| `ReplayReservationConflict` | Another lifecycle or mutation owns the reservation; never silently queue. |
| `Expired` | Input or authority is outside its valid time window. |
| `TrustedTimeUnavailable` | A deliberate retry may be offered after trusted time recovers. |
| `SecureKeyUnavailable` | A deliberate retry may be offered after the secure provider recovers. |
| `SecureKeyRevokedOrDeleted` | Recovery or re-enrollment is required. |
| `SigningCanceled` | Signing was canceled; mutation outcome still governs state. |
| `PersistenceCorruption` | Stop and recover from an approved source. |
| `RollbackDetected` | Stop; never overwrite newer durable state. |
| `SequenceConflict` | Refresh and process the required order. |
| `InvalidHandle` | Resolve a fresh capability from the lifecycle. |
| `ClosedHandle` | The capability is closed; do not reuse it. |
| `AbiMismatch` | TypeScript, generated WASM, and Rust contract do not match. |
| `Canceled` | Cancellation reached the operation; outcome still governs publication. |
| `ResourceLimit` | Reduce pressure or input within documented bounds. |
| `InternalFailure` | Fail closed and retain only non-secret diagnostics. |

`RETRYABLE_ERROR_CODES` contains only `TrustedTimeUnavailable` and `SecureKeyUnavailable`. Even those never authorize replay while publication is ambiguous.

```ts
try {
  // SDK call
} catch (error) {
  if (!(error instanceof RumpunSdkError)) throw error;

  if (isRetryable(error.code)) {
    // Offer a deliberate retry only after terminal outcome is known.
  }
}
```

## Contract constants

| Export | Current value | Meaning |
|---|---:|---|
| `CONTRACT_API_VERSION` | `1` | Shared semantic API version. |
| `CONTRACT_ABI_VERSION` | `1` | Shared handle/error/outcome ABI version. |
| `WASM_ABI_VERSION` | `1` | Web adapter ABI version. |
| `MINIMUM_ADAPTER_VERSION` | `1` | Minimum supported adapter version. |
| `CONTRACT_PROTOCOL_VERSION` | `1` | Bound MLS protocol version. |
| `CONTRACT_SCHEMA_VERSION` | `1` | Bound object schema version. |
| `CONTRACT_CIPHERSUITE_ID` | `0x0001` | Bound ciphersuite. |
| `CONTRACT_HANDLE_WIRE_LEN` | `35` | Opaque handle wire length. |
| `EXPECTED_HANDLE_LAYOUT` | `[2, 1, 16, 16]` | ABI, kind, ID, and generation byte lengths. Not permission to decode handles. |
| `STABLE_ERROR_CODE_COUNT` | `26` | Stable error registry size. |
| `CALL_VERSION_WIRE_LEN` | `14` | Seven big-endian `u16` values. |
| `OUTCOME_COUNT` | `3` | Frozen terminal outcome count. |
| `HANDLE_KIND_COUNT` | `11` | Frozen handle-kind registry size. |

### `HANDLE_KINDS`

The frozen registry contains `Device`, `ScopeGroup`, `EpochKwkJournal`, `ObjectKeyReference`, `MediaSession`, `VerifiedManifest`, `ReplayStore`, `PlatformSigner`, `SecureKey`, `PersistedState`, and `LifecycleRoot`. These names are contract metadata, not constructors or authorization grants.

## Contract helper functions

### `callVersionForWasm()`

```ts
callVersionForWasm(): CallVersion
```

Returns the current seven-field WASM declaration.

### `encodeCallVersion()`

```ts
encodeCallVersion(version: CallVersion): Uint8Array
```

Encodes exactly 14 bytes as seven big-endian unsigned 16-bit fields.

### `versionWireForWasm()`

```ts
versionWireForWasm(): Uint8Array
```

Returns the cached current WASM declaration used by high-level methods.

### `validateCallVersion()`

```ts
validateCallVersion(
  version: CallVersion,
  expectedAdapterAbi: number,
): boolean
```

Checks the shared contract and an explicit adapter ABI. Primarily useful for contract verification, not runtime authority.

### `validateCallVersionForWasm()`

```ts
validateCallVersionForWasm(version: CallVersion): boolean
```

Validates against `WASM_ABI_VERSION`.

### `isRetryable()`

```ts
isRetryable(code: string): boolean
```

Returns whether the stable contract permits offering a retry for that code. It does not inspect publication outcome and never authorizes automatic retry.

### `verifyContract()`

```ts
verifyContract(): ContractVerificationResult
```

Verifies TypeScript's mirrored constants, registries, uniqueness, counts, layouts, and version declaration. Result shape:

```ts
interface ContractVerificationResult {
  readonly valid: boolean;
  readonly errors: readonly string[];
}
```

Use it for build-time or test evidence. A valid mirror does not establish production authorization.