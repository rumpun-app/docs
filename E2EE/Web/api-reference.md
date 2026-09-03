# Web and TypeScript API reference

This page catalogs every callable export from `@rumpun/sdk-ts` on the current `develop` surface.

> **Status: `NOT_PRODUCTION_SAFE`.** Protected object operations remain fail-closed until Task 6 authorization and Gate G release acceptance. Never bypass this with application-side crypto.

## Contents

- [Open and close a lifecycle](#open-and-close-a-lifecycle)
- [Enroll and restore](#enroll-and-restore)
- [Create and find groups](#create-and-find-groups)
- [Manage members and devices](#manage-members-and-devices)
- [Process Commits and inspect state](#process-commits-and-inspect-state)
- [Reconcile ambiguity](#reconcile-ambiguity)
- [Encrypt, decrypt, and re-key objects](#encrypt-decrypt-and-re-key-objects)
- [Mutation control](#mutation-control)
- [Low-level verification APIs](#low-level-verification-apis)

For all types, outcomes, errors, constants, and contract helpers, see [Types, outcomes, errors, and contract](types-outcomes-errors.md).

## Open and close a lifecycle

### `RumpunLifecycle.open()`

```ts
static open(deviceId: Uint8Array): Promise<RumpunLifecycle>
```

Initializes WASM once, acquires the exclusive Web Lock for `deviceId`, opens its durable store, and creates the native lifecycle. A competing owner fails with `ReplayReservationConflict`; the call never silently queues.

```ts
const lifecycle = await RumpunLifecycle.open(deviceId);
```

Use a stable, application-issued device identity. It is not a display name or authorization token.

### `lifecycle.dispose()`

```ts
dispose(): Promise<void>
```

Closes and zeroizes owned in-memory state where supported, drains the lifecycle, and releases the Web Lock. Always call it in `finally`.

```ts
try {
  // use lifecycle
} finally {
  await lifecycle.dispose();
}
```

Every handle minted by the disposed lifecycle becomes invalid.

## Enroll and restore

### `lifecycle.enroll()`

```ts
enroll(accountId: Uint8Array): RumpunMutation<DeviceHandle>
```

Enrolls the lifecycle-bound device under `accountId`. The device identity comes from `open()` and cannot diverge from the Web Lock/store binding.

```ts
const operation = lifecycle.enroll(accountId);
const device = await operation.result;
const outcome = await operation.outcome;
```

Consume both promises exactly once and handle `Ambiguous` before any further mutation.

### `lifecycle.restore()`

```ts
restore(): RumpunMutation<DeviceHandle>
```

Restores sealed state into a fresh lifecycle and returns a fresh device capability. Dispose the previous lifecycle before opening and restoring the same device identity.

## Create and find groups

### `lifecycle.createGroup()`

```ts
createGroup(
  device: DeviceHandle,
  groupId: Uint8Array,
): RumpunMutation<GroupHandle>
```

Creates an MLS group and returns its opaque handle. `groupId` is authenticated identity, not human-readable authorization.

### `lifecycle.findGroup()`

```ts
findGroup(
  device: DeviceHandle,
  groupId: Uint8Array,
): Promise<GroupHandle>
```

Resolves a fresh group capability from restored state. Never cache a handle across lifecycle disposal, reload, or restore.

### `lifecycle.createKeyPackage()`

```ts
createKeyPackage(
  device: DeviceHandle,
): RumpunMutation<KeyPackage>
```

Creates the MLS key package delivered to a current group member. Treat the branded bytes as opaque.

### `lifecycle.joinWelcome()`

```ts
joinWelcome(
  device: DeviceHandle,
  welcome: Uint8Array,
): RumpunMutation<GroupHandle>
```

Authenticates and joins an MLS Welcome, returning the joined group handle. Do not decode or reinterpret the Welcome in TypeScript.

## Manage members and devices

### `lifecycle.addMember()`

```ts
addMember(
  group: GroupHandle,
  keyPackage: KeyPackage,
): RumpunMutation<CommitAndWelcomeDto>
```

Adds the key-package owner. Deliver `commit` to current members in order and `welcome` only to the joining device through the authenticated transport.

### `lifecycle.removeMember()`

```ts
removeMember(
  group: GroupHandle,
  accountId: Uint8Array,
): RumpunMutation<Commit>
```

Removes every current device belonging to an account and returns the ordered MLS Commit.

### `lifecycle.removeDevice()`

```ts
removeDevice(
  group: GroupHandle,
  leafIndex: number,
): RumpunMutation<Commit>
```

Removes one current roster leaf. Resolve `leafIndex` immediately beforehand with `listMemberDevices()`, never from UI position or a stale cache.

### `lifecycle.selfUpdate()`

```ts
selfUpdate(group: GroupHandle): RumpunMutation<Commit>
```

Rotates the caller's MLS leaf material and returns the Commit for ordered delivery.

## Process Commits and inspect state

### `lifecycle.processOrderedCommit()`

```ts
processOrderedCommit(
  group: GroupHandle,
  commit: Commit,
): RumpunMutation<void>
```

Processes exactly the next expected Commit. Replay, gaps, and out-of-order delivery fail closed.

### `lifecycle.catchUpSequentially()`

```ts
catchUpSequentially(
  group: GroupHandle,
  commits: Commit[],
): RumpunMutation<void>
```

Processes the caller-supplied sequence in order. Do not sort, skip, deduplicate, or parallelize it.

### `lifecycle.inspectGroup()`

```ts
inspectGroup(group: GroupHandle): Promise<GroupStatusDto>
```

Returns the current non-secret epoch and member count.

### `lifecycle.listMemberDevices()`

```ts
listMemberDevices(
  group: GroupHandle,
): Promise<MemberDeviceDto[]>
```

Returns the current non-secret roster: account ID, epoch-sensitive leaf index, and signature public key.

## Reconcile ambiguity

### `lifecycle.reconcile()`

```ts
reconcile(operationId: string): Promise<PublicationOutcome>
```

Reconciles one exact ambiguous operation. Call only when that operation's outcome is `Ambiguous`. Do not substitute restore, invent an ID, clear local state, or replay the original mutation.

```ts
const outcome = await operation.outcome;
if (outcome === "Ambiguous") {
  const reconciled = await lifecycle.reconcile(operation.operationId);
  // Continue only after applying the returned terminal semantics.
}
```

## Encrypt, decrypt, and re-key objects

These methods are public release-shaped APIs, but ordinary builds currently return `UnsupportedProtocol` before admission until production authority exists. TypeScript performs no crypto or AAD construction.

### `lifecycle.encryptObjectVersionV1()`

```ts
encryptObjectVersionV1(
  group: GroupHandle,
  context: ObjectContentContextV1,
  plaintext: Uint8Array,
): Promise<EncryptedObjectBundleV1>
```

Encrypts one immutable object version with a fresh CEK and returns only encrypted transport data. Never store plaintext beside the bundle.

### `lifecycle.decryptObjectVersionV1()`

```ts
decryptObjectVersionV1(
  group: GroupHandle,
  context: ObjectContentContextV1,
  bundle: EncryptedObjectBundleV1,
): Promise<Uint8Array>
```

Authenticates the independently supplied expected context and complete bundle before returning plaintext. Never copy the expected context from untrusted bundle fields.

### `lifecycle.rewrapObjectCekV1()`

```ts
rewrapObjectCekV1(
  group: GroupHandle,
  context: ObjectContentContextV1,
  sourceWrapContext: object,
  bundle: EncryptedObjectBundleV1,
  destinationWrapContext: object,
): Promise<EncryptedObjectBundleV1>
```

Adds a destination CEK wrap while preserving content context, content nonce, and content ciphertext. Destination key-version authority must come from the approved lifecycle flow, never UI code.

## Mutation control

Every protected mutation returns:

```ts
interface RumpunMutation<T> {
  readonly operationId: string;
  cancel(): void;
  readonly result: Promise<T>;
  readonly outcome: Promise<PublicationOutcome>;
}
```

`result` carries the produced value. `outcome` carries terminal publication semantics. Cancellation is a request, not proof of non-commit; always drain both promises and reconcile `Ambiguous`.

## Low-level verification APIs

These exports exist for contract vectors and boundary tests. **Application code should not use them.** They expose raw lifecycle bytes without granting permission to inspect, persist, log, or reconstruct capabilities.

### `rawWasmNewLifecycle()`

```ts
rawWasmNewLifecycle(
  deviceId: Uint8Array,
  versionWire?: Uint8Array,
): Promise<Uint8Array>
```

Calls the same lifecycle admission boundary with an optional explicit contract declaration.

### `rawWasmEnroll()`

```ts
rawWasmEnroll(
  handleWire: Uint8Array,
  accountId: Uint8Array,
  versionWire?: Uint8Array,
): Promise<Uint8Array>
```

Enrolls through the raw WASM boundary and returns only the result bytes. It is unsuitable for application mutations because it does not expose the full high-level outcome workflow.

### `rawWasmRestore()`

```ts
rawWasmRestore(
  handleWire: Uint8Array,
  versionWire?: Uint8Array,
): Promise<Uint8Array>
```

Restores through the explicit contract-vector boundary.

### `rawWasmDrop()`

```ts
rawWasmDrop(
  handleWire: Uint8Array,
  versionWire?: Uint8Array,
): Promise<void>
```

Drops the raw lifecycle and releases its resources.

### `lifecycle.rawHandleBytes()`

```ts
rawHandleBytes(): Uint8Array
```

Returns a copy of raw lifecycle wire bytes for verification evidence. This is not an application persistence or transport API. Prefer `RumpunLifecycle` methods and treat use outside controlled tests as a security review finding.