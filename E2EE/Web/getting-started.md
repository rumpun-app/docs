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

```ts
import {
  RumpunLifecycle,
  RumpunSdkError,
  type PublicationOutcome,
} from "@rumpun/sdk-ts";

const deviceId = crypto.getRandomValues(new Uint8Array(32));
const lifecycle = await RumpunLifecycle.open(deviceId);
```

Persist the stable application device ID through the approved device-identity layer, not logs or analytics. `open()` acquires an exclusive Web Lock; a competing tab fails with `ReplayReservationConflict` instead of silently queueing.

## Enroll

```ts
const accountId = new TextEncoder().encode("synthetic-account");
const operation = lifecycle.enroll(accountId);

const device = await operation.result;
const outcome: PublicationOutcome = await operation.outcome;

if (outcome !== "Committed") {
  throw new Error(`Enrollment did not commit: ${outcome}`);
}
```

Always consume both `result` and `outcome`.

## Dispose

```ts
try {
  // Use the lifecycle.
} finally {
  await lifecycle.dispose();
}
```

Disposal drains operations, closes native state, releases the Web Lock, and invalidates every capability minted by the lifecycle.

## Restore after reload

Dispose the old lifecycle before opening the same device identity again:

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