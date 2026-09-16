# Manage Web lifecycle and groups

## Lifecycle ownership

Think of the lifecycle as the single long-lived owner of everything tied to one device. One `RumpunLifecycle` owns one device's Rust lifecycle, IndexedDB state, and exclusive Web Lock. Keep it in an application service, not a React component or short-lived request handler, so its ownership survives re-renders and does not get created or torn down repeatedly.

## Create a group

```ts
const groupId = new TextEncoder().encode("synthetic-family-group");
const create = lifecycle.createGroup(device, groupId);
const group = await create.result;

if (await create.outcome !== "Committed") {
  throw new Error("Group creation did not commit");
}
```

A group ID is authenticated identity, not current authorization.

## Find a restored group

```ts
const group = await lifecycle.findGroup(restoredDevice, groupId);
```

Use `findGroup()` after restore or reload. Never reconstruct a branded `GroupHandle` from bytes.

## Inspect current state

```ts
const status = await lifecycle.inspectGroup(group);
const roster = await lifecycle.listMemberDevices(group);

console.log(status.epoch, status.memberCount, roster.length);
```

These are non-secret reads. They do not grant authorization.

## Create a key package and join

Joining an MLS group is a short handshake. The joining device first publishes a key package (a signed bundle that advertises how to add it), a current member uses that key package to add it and receives back a Commit plus a Welcome, and the joining device then consumes the Welcome to enter the group at the new epoch. Deliver the Commit to existing members and the Welcome only to the newcomer.

Joining device:

```ts
const keyPackageOperation = lifecycle.createKeyPackage(device);
const keyPackage = await keyPackageOperation.result;
const keyPackageOutcome = await keyPackageOperation.outcome;
```

Existing member adds it, then the joining device consumes the returned Welcome:

```ts
const add = existingLifecycle.addMember(existingGroup, keyPackage);
const { commit, welcome } = await add.result;
const addOutcome = await add.outcome;

const join = lifecycle.joinWelcome(device, welcome);
const joinedGroup = await join.result;
const joinOutcome = await join.outcome;
```

Deliver key packages, Commits, and Welcomes unchanged over authenticated transport. Consume every terminal outcome.

## Cancellation

Cancelling asks the SDK to stop an in-flight mutation, but asking is not the same as knowing it did not happen. You must still read the terminal outcome to learn where state actually landed.

```ts
const operation = lifecycle.selfUpdate(group);
operation.cancel();
await operation.result.catch(() => undefined);
const outcome = await operation.outcome;
```

Cancellation is a request, not proof of non-commit. Reconcile if the terminal outcome is `Ambiguous`.

## Shutdown rule

Clean shutdown is what frees the device for the next tab or reload. Always call `dispose()` in `finally`. A second tab cannot own the same device until the first lifecycle releases its Web Lock.