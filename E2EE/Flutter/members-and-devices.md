# Manage family members and devices

This guide covers listing devices, removing one device, removing an entire family member, and distributing the resulting MLS Commit.

Adding and removing people or devices changes who can read the group's content, so each change is a protected mutation that produces an MLS Commit the rest of the group must apply in order. Read the roster before you act, and always handle the terminal outcome.

> **Current status: `NOT_PRODUCTION_SAFE`.** Removal changes who can participate in future group state, so production calls require Task 6 current authorization. A valid group handle or cryptographic signature alone is not enough.

## Member removal vs device removal

The key distinction: a family member (a person) can own several devices, while a device is one phone or computer. Removing a member evicts every device that person owns; removing a device evicts exactly one. Choose the action that matches the product decision, because removing a single device leaves that person's other devices in the group.

| Action | Use when | SDK operation | Effect |
|---|---|---|---|
| Remove member | A person must leave the family group completely | `lifecycleRemoveMember` | Removes every current MLS device leaf whose account ID matches that member |
| Remove device | One phone or computer is lost, replaced, or revoked | `lifecycleRemoveDevice` | Removes exactly one current MLS device leaf selected by its ephemeral leaf index |

Do not remove only one device when the product decision is to remove the person. A member may have multiple active devices.

## List current member devices

Start by reading the current members and devices so you act on live state rather than a stale UI cache. Each entry's `leafIndex` is that device's slot in the current MLS tree, and the numbering shifts whenever the group changes (a new epoch), so a value read earlier can point at a different device now. That is why you always resolve it fresh right before a removal.

Always obtain the leaf index immediately before removing a device:

```dart
final devices = await lifecycleListMemberDevices(
  lifecycle: lifecycle,
  groupWire: group,
);

for (final item in devices) {
  print('account=${String.fromCharCodes(item.accountId)}');
  print('leafIndex=${item.leafIndex}');
}
```

`leafIndex` is non-secret but ephemeral. Do not persist it as a permanent device identifier. Re-resolve it from the current group before each removal.

The returned signature public key is informational cryptographic identity. It does not authorize removal by itself.

## Remove one device

Use this when a single phone or computer is lost, replaced, or revoked. It evicts exactly one device leaf and leaves the account's other devices in the group.

Choose the current device entry using authoritative application state, then submit its current leaf index:

```dart
final devices = await lifecycleListMemberDevices(
  lifecycle: lifecycle,
  groupWire: group,
);

final target = devices.singleWhere(
  (item) =>
      String.fromCharCodes(item.accountId) == 'family:hadi' &&
      item.leafIndex == selectedCurrentLeafIndex,
);

final removal = await lifecycleRemoveDevice(
  lifecycle: lifecycle,
  groupWire: group,
  leafIndex: target.leafIndex,
);

final commit = await removal.awaitResult();
final outcome = await removal.awaitOutcome();

if (outcome != PublicationOutcome.committed) {
  throw StateError('Device removal did not commit: $outcome');
}
```

This removes exactly one MLS device leaf. Other devices belonging to the same account remain members.

## Remove a family member completely

Use this when a person must leave the family group entirely. One call evicts every device that person owns in a single Commit, so you do not have to remove their devices one by one.

Pass the exact account ID bytes used in MLS credentials:

```dart
final removal = await lifecycleRemoveMember(
  lifecycle: lifecycle,
  groupWire: group,
  accountId: Uint8List.fromList('family:hadi'.codeUnits),
);

final commit = await removal.awaitResult();
final outcome = await removal.awaitOutcome();

if (outcome != PublicationOutcome.committed) {
  throw StateError('Member removal did not commit: $outcome');
}
```

`lifecycleRemoveMember` removes all current device leaves whose credential account ID matches. If no current leaf matches, the operation rejects instead of pretending success.

## Deliver the removal Commit

Removing someone only takes full effect once every remaining member applies the resulting Commit. Each member must apply Commits in the exact order they were produced; applying them out of order, skipping one, or applying one twice fails closed.

The returned `commit` must be delivered to every remaining group member through the approved ordered transport. On each remaining device:

```dart
final process = await lifecycleProcessOrderedCommit(
  lifecycle: recipientLifecycle,
  groupWire: recipientGroup,
  commitBytes: commit,
);

await process.awaitResult();
final processOutcome = await process.awaitOutcome();

if (processOutcome != PublicationOutcome.committed) {
  throw StateError('Removal Commit did not commit: $processOutcome');
}
```

Process commits in exact order. Commit N+1 must never be applied before N. Task 6 owns production delivery ordering, confirmed per-device position, and stale-epoch fencing.

## Verify the result

On the removing device:

```dart
final status = await lifecycleInspectGroup(
  lifecycle: lifecycle,
  groupWire: group,
);

final remaining = await lifecycleListMemberDevices(
  lifecycle: lifecycle,
  groupWire: group,
);
```

For a full member removal, assert no returned entry has the removed account ID:

```dart
expect(
  remaining.any(
    (item) =>
        String.fromCharCodes(item.accountId) == 'family:hadi',
  ),
  isFalse,
);
```

For a single-device removal, assert the targeted leaf no longer exists and another expected device for that account still exists.

Also verify the Commit on at least one remaining recipient lifecycle. Local state alone does not prove delivery.

## What the removed member can still access

It is important to set the right expectation here: removal controls the future, not the past. It stops the removed device from following the group forward, but it cannot claw back content that device already held.

Removal changes future MLS membership. It does not erase plaintext or keys that were legitimately available before removal.

After a committed removal:

- the removed leaf must not process future group epochs;
- new object writes must use current post-removal authority;
- stale-epoch writes must be rejected;
- previously exported plaintext, screenshots, or backups cannot be revoked cryptographically;
- historical archive access follows the separately authorized history policy and key-grant design.

Do not promise retroactive deletion from a device that already possessed plaintext.

## Cancellation and ambiguous outcomes

Removal is a mutation, so its publication outcome is the definitive answer to "did the removal actually stick?" The three outcomes below tell you exactly how to proceed, and an `ambiguous` result in particular must be reconciled rather than retried, because a blind retry could target a different leaf after the epoch changes.

Removal is a mutation. Consume both result and outcome once.

- `notCommitted`: no removal Commit was durably published;
- `committed`: deliver the returned Commit to remaining devices;
- `ambiguous`: do not retry automatically and do not issue another removal. Reconcile that exact operation first.

```dart
final operationId = await removal.operationId();
```

Keep the operation ID for explicit reconciliation and audit correlation. Never infer publication from receiving `commit` alone.

## Expected failures

Handle these deliberately:

- `invalidHandle`: wrong, stale, foreign, or wrong-kind group capability;
- `closedHandle`: lifecycle or handle was already closed;
- `unauthorized`, `staleAuthorization`, or `revokedDeviceOrKey`: production authority rejected the action;
- `replayReservationConflict`: related state is already reserved or ambiguous;
- `malformedInput`: invalid account ID or input shape;
- `internalFailure`: MLS transition failed without a more specific safe code.

Provider detail and the exact authentication failure location must not be exposed to UI logs.

## End-to-end test: remove one device

This walkthrough proves single-device removal end to end: the targeted leaf disappears, the account's other device stays, and the removed device can no longer follow the group.

Use two devices for the same synthetic account plus one remaining administrator device.

1. Enroll all synthetic devices and join them to one group.
2. List current leaves and select exactly one device leaf.
3. Snapshot sealed state and group membership.
4. Call `lifecycleRemoveDevice` and assert `committed`.
5. Process the returned Commit on every remaining device.
6. Assert the removed leaf is absent and the account's other leaf remains.
7. Assert the removed lifecycle cannot process the next ordered group transition.
8. Assert remaining members can still perform a valid later operation.
9. Assert teardown returns workers, locks, handles, and permits to baseline.

## End-to-end test: remove a member

This walkthrough proves full-member removal end to end: every device for the account is gone, other accounts stay active, and the removed member does not reappear even after a restore.

Use one synthetic member with at least two devices.

1. Add both devices under the same account ID.
2. Call `lifecycleRemoveMember` with that account ID.
3. Assert one committed mutation returns one removal Commit.
4. Process the Commit on remaining devices.
5. Assert every leaf for the removed account is gone.
6. Assert another account remains active.
7. Assert a stale removed device cannot regain admission using an old handle or old epoch.
8. Assert a later valid operation from a remaining member succeeds.
9. Restore from sealed state and verify the member does not reappear.

## Security checklist

- [ ] Current Task 6 authority approves the removal action.
- [ ] Device leaf index was re-resolved immediately before removal.
- [ ] Full member removal uses `lifecycleRemoveMember`, not repeated UI guesses.
- [ ] Result and outcome were each consumed once.
- [ ] Ambiguous removal is reconciled, never automatically retried.
- [ ] Returned Commit is delivered in order to every remaining device.
- [ ] Removed devices are fenced from future epochs.
- [ ] The UI makes no retroactive-deletion promise.
- [ ] No plaintext, private key, raw handle, or provider diagnostic is logged.
