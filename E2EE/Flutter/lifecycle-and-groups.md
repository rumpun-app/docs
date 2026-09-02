# Manage lifecycle and groups

This guide assumes you already created `lifecycle` and enrolled `device` in [Install and initialize](getting-started.md).

## Create a group

```dart
final createGroup = await lifecycleCreateGroup(
  lifecycle: lifecycle,
  deviceWire: device,
  groupId: Uint8List.fromList('synthetic-family-group'.codeUnits),
);

final group = await createGroup.awaitResult();
final groupOutcome = await createGroup.awaitOutcome();

if (groupOutcome != PublicationOutcome.committed) {
  throw StateError('Group creation did not commit: $groupOutcome');
}
```

A group ID is not authorization. Production object-capable groups require Task 6 verified authority.

## Inspect a group

```dart
final status = await lifecycleInspectGroup(
  lifecycle: lifecycle,
  groupWire: group,
);

print('epoch=${status.epoch}');
print('members=${status.memberCount}');
```

Inspection is read-only and has no publication outcome.

## Invite another device

On the joining device:

```dart
final keyPackageOperation = await lifecycleCreateKeyPackage(
  lifecycle: joiningLifecycle,
  deviceWire: joiningDevice,
);

final keyPackage = await keyPackageOperation.awaitResult();
if (await keyPackageOperation.awaitOutcome() !=
    PublicationOutcome.committed) {
  throw StateError('Key package did not commit');
}
```

On the existing group member:

```dart
final addMember = await lifecycleAddMember(
  lifecycle: lifecycle,
  groupWire: group,
  keyPackageBytes: keyPackage,
);

final invitation = await addMember.awaitResult();
if (await addMember.awaitOutcome() != PublicationOutcome.committed) {
  throw StateError('Add member did not commit');
}
```

Deliver `invitation.welcome` and ordered commits through the approved transport. Do not log or reinterpret their bytes.

On the joining device:

```dart
final join = await lifecycleJoinWelcome(
  lifecycle: joiningLifecycle,
  deviceWire: joiningDevice,
  welcomeBytes: invitation.welcome,
);

final joinedGroup = await join.awaitResult();
if (await join.awaitOutcome() != PublicationOutcome.committed) {
  throw StateError('Welcome join did not commit');
}
```

Production authorization and authenticated mapping transport remain Task 6 responsibilities.

## Process an ordered commit

```dart
final process = await lifecycleProcessOrderedCommit(
  lifecycle: joiningLifecycle,
  groupWire: joinedGroup,
  commitBytes: invitation.commit,
);

await process.awaitResult();
if (await process.awaitOutcome() != PublicationOutcome.committed) {
  throw StateError('Commit did not commit');
}
```

Commit order matters. Never process N+1 before N and never silently retry an ambiguous mutation.

## Restore after restart

Create a fresh lifecycle over the same platform-owned state location:

```dart
final restoredLifecycle = await lifecycleCreate();
final restore = await lifecycleRestore(lifecycle: restoredLifecycle);
final restoredDevice = await restore.awaitResult();

if (await restore.awaitOutcome() != PublicationOutcome.committed) {
  throw StateError('Restore did not commit');
}
```

Every old handle is stale after restart. Resolve a fresh group handle:

```dart
final restoredGroup = await lifecycleFindGroup(
  lifecycle: restoredLifecycle,
  deviceWire: restoredDevice,
  groupId: Uint8List.fromList('synthetic-family-group'.codeUnits),
);
```

Restore is not reconciliation. If a mutation ends `ambiguous`, reconcile the exact operation through the supported authority flow.

## Cancel safely

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
```

Check `terminal`. Cancellation alone does not prove that nothing committed.

## Next

Continue with [Encrypt, decrypt, and re-key objects](object-encryption.md).
