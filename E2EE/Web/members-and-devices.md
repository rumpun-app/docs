# Manage Web family members and devices

## Inspect the roster first

```ts
const roster = await lifecycle.listMemberDevices(group);
```

Each entry contains `accountId`, `leafIndex`, and `signaturePublicKey`. Leaf indexes are epoch-sensitive; refresh immediately before removing one device.

## Add a member or device

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

```ts
const remove = lifecycle.removeMember(group, accountId);
const commit = await remove.result;
const outcome = await remove.outcome;
```

Do not interpret a returned Commit as proof of publication. Handle the terminal outcome first.

## Remove one device

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

If removal returns `Ambiguous`, fence further mutations and reconcile the exact operation ID:

```ts
const reconciled = await lifecycle.reconcile(remove.operationId);
```

Never retry the removal automatically. That can target a different leaf after the epoch changes.