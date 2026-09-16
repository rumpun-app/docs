# Install and initialize the Web SDK

> **Status: `NOT_PRODUCTION_SAFE`.** Use synthetic data only.

## Prerequisites

Use a browser with WebAssembly, WebCrypto, IndexedDB, and Web Locks.

```bash
cd packages/sdk-ts
npm ci
npm run build:wasm
npx tsc --noEmit
npm test
```

The TypeScript package is a thin wrapper. Never add JavaScript or WebCrypto fallback cryptography.

## Open a lifecycle

A lifecycle is your handle to one device's Rust core. Opening it initializes WASM and claims exclusive ownership of that device identity so nothing else can act as the same device at the same time.

```ts
import {
  RumpunLifecycle,
  RumpunSdkError,
  type PublicationOutcome,
} from "@rumpun/sdk-ts";

const deviceId = crypto.getRandomValues(new Uint8Array(32));
const lifecycle = await RumpunLifecycle.open(deviceId);
```

Persist the stable application device ID through the approved device-identity layer, not logs or analytics. `open()` acquires an exclusive Web Lock (a browser primitive that lets only one tab hold a named lock at a time); a competing tab fails with `ReplayReservationConflict`, the error that signals the call is failing fast rather than silently queueing whenever a competing owner still holds that device, including a stale lock.

## Enroll

Enrollment registers this device under a family account so it can join groups. It is a protected mutation, so it returns two things: a `result` (the value produced) and an `outcome` (the separate, terminal answer to "was this change actually saved?"). You must read both.

```ts
const accountId = new TextEncoder().encode("synthetic-account");
const operation = lifecycle.enroll(accountId);

const device = await operation.result;
const outcome: PublicationOutcome = await operation.outcome;

if (outcome !== "Committed") {
  throw new Error(`Enrollment did not commit: ${outcome}`);
}
```

Always consume both `result` and `outcome`. To "consume both result and outcome" means awaiting each promise exactly once: the `result` gives you the produced value, and the `outcome` tells you whether it durably committed. A successful `result` on its own is not proof that the change was published.

## Dispose

Disposing shuts the lifecycle down cleanly and, just as importantly, releases the Web Lock so another tab or a later reload can open the same device. Always do this in a `finally` block so it runs even if your code throws.

```ts
try {
  // Use the lifecycle.
} finally {
  await lifecycle.dispose();
}
```

Disposal drains operations, closes native state, releases the Web Lock, and invalidates every capability minted by the lifecycle.

## Restore after reload

After a page reload the previous lifecycle is gone, so you open the device again and restore its sealed state to resume where you left off. Dispose the old lifecycle before opening the same device identity again, otherwise the Web Lock is still held:

```ts
const restoredLifecycle = await RumpunLifecycle.open(deviceId);
const restore = restoredLifecycle.restore();
const restoredDevice = await restore.result;

if (await restore.outcome !== "Committed") {
  throw new Error("Restore did not commit");
}
```

Resolve fresh group handles after restore. Never retain old handles across reloads.

## Next

Continue with [Lifecycle and groups](lifecycle-and-groups.md).