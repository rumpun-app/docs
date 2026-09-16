# Manage Web family members and devices

Adding and removing people or devices changes who can read the group's content, so each change is a protected mutation that produces an MLS Commit the rest of the group must apply in order. Read the roster before you act, and always handle the terminal outcome.

## Inspect the roster first

Start by reading the current members and devices so you act on live state rather than a stale UI cache.

```ts
const roster = await lifecycle.listMemberDevices(group);
```

Each entry contains `accountId`, `leafIndex`, and `signaturePublicKey`. The `leafIndex` is the device's slot in the current MLS tree, and that numbering shifts whenever the group changes (a new epoch), so a value read earlier can point at a different device now. Leaf indexes are epoch-sensitive; refresh immediately before removing one device.

## Add a member or device

Adding is a two-party step: the newcomer publishes a key package describing itself, then a current member uses that key package to admit it and produces the Commit and Welcome to distribute.

The joining device creates a key package:

```ts
const kp = joiningLifecycle.createKeyPackage(joiningDevice);
const keyPackage = await kp.result;
if (await kp.outcome !== "Committed") throw new Error("Key package failed");
```

A current member adds it:

```ts
const add = lifecycle.addMember(group, keyPackage);
const { commit, welcome } = await add.result;
const outcome = await add.outcome;
```

Deliver `commit` to current members in order and `welcome` only to the joining device.

## Remove every device for one account

Removing a whole account evicts every device it owns from the group in one Commit. As with any mutation, the returned Commit is not proof that the change was published; the terminal outcome is.

```ts
const remove = lifecycle.removeMember(group, accountId);
const commit = await remove.result;
const outcome = await remove.outcome;
```

Do not interpret a returned Commit as proof of publication. Handle the terminal outcome first.

## Remove one device

To remove a single device you must target its exact current `leafIndex`. Resolve it fresh right before the call, because the index can move as the group's epoch advances.

```ts
const current = await lifecycle.listMemberDevices(group);
const target = current.find((entry) => matchesDevice(entry));
if (!target) throw new Error("Device no longer exists");

const remove = lifecycle.removeDevice(group, target.leafIndex);
const commit = await remove.result;
const outcome = await remove.outcome;
```

Never use a cached UI array index as `leafIndex`.

## Deliver and process the Commit

Every membership change advances the group by one Commit, and each remaining member must apply those Commits in the exact order they were produced. Applying them out of order, skipping one, or applying one twice fails closed.

For each remaining member:

```ts
const process = peerLifecycle.processOrderedCommit(peerGroup, commit);
await process.result;
const processOutcome = await process.outcome;
```

For multiple missing Commits:

```ts
const catchUp = lifecycle.catchUpSequentially(group, commits);
await catchUp.result;
const outcome = await catchUp.outcome;
```

Do not sort, skip, deduplicate, or parallelize ordered Commits.

## Verify removal

Refresh the roster and group status after the Commit is applied. A removed device must fail future protected operations; do not rely only on UI state.

## Ambiguous removal

`Ambiguous` means the SDK cannot prove whether the removal was saved or not. Do not guess and do not retry: pause further mutations and reconcile the exact operation ID so you learn the real outcome first.

If removal returns `Ambiguous`, fence further mutations and reconcile the exact operation ID:

```ts
const reconciled = await lifecycle.reconcile(remove.operationId);
```

Never retry the removal automatically. That can target a different leaf after the epoch changes.