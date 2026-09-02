# Web and TypeScript guide

> **Status: `NOT_PRODUCTION_SAFE`.** Use synthetic data only until production authority and release review are accepted.

## Prerequisites

Use a browser with WebAssembly, WebCrypto, IndexedDB, and Web Locks. Build the WASM package from the SDK repository:

```bash
cd packages/sdk-ts
npm ci
npm run build:wasm
npx tsc --noEmit
```

The TypeScript package is a thin wrapper. Do not add WebCrypto object encryption or adapter-side AAD construction.

## Open a lifecycle

```ts
import { RumpunLifecycle, RumpunSdkError } from "@rumpun/sdk-ts";

const deviceId = new TextEncoder().encode("synthetic-device-01");
const lifecycle = await RumpunLifecycle.open(deviceId);
```

Opening acquires the device's exclusive Web Lock. A competing lifecycle fails rather than silently queueing.

## Enroll

```ts
const accountId = new TextEncoder().encode("synthetic-account");
const enroll = lifecycle.enroll(accountId);

const device = await enroll.result;
const outcome = await enroll.outcome;
if (outcome !== "Committed") {
  throw new Error(`Enrollment did not commit: ${outcome}`);
}
```

Always consume both the result and outcome exactly once.

## Create and inspect a group

```ts
const groupId = new TextEncoder().encode("synthetic-family-group");
const create = lifecycle.createGroup(device, groupId);
const group = await create.result;

if (await create.outcome !== "Committed") {
  throw new Error("Group creation did not commit");
}

const status = await lifecycle.inspectGroup(group);
console.log(status.epoch, status.memberCount);
```

Production object-capable group creation remains unavailable until current authority is verified. Do not treat a generic group or caller-supplied labels as object authority.

## Restore after reload

Dispose the active lifecycle before opening a fresh lifecycle over the same durable device state:

```ts
await lifecycle.dispose();

const restoredLifecycle = await RumpunLifecycle.open(deviceId);
const restore = restoredLifecycle.restore();
const restoredDevice = await restore.result;

if (await restore.outcome !== "Committed") {
  throw new Error("Restore did not commit");
}
```

Old handles are invalid after reload or restore. Resolve fresh group handles from the restored device.

## Handle errors

```ts
try {
  const status = await restoredLifecycle.inspectGroup(group); // old handle
  console.log(status);
} catch (error) {
  if (error instanceof RumpunSdkError) {
    switch (error.code) {
      case "InvalidHandle":
      case "ClosedHandle":
        // Resolve a fresh handle. Never reconstruct one.
        break;
      case "ReplayReservationConflict":
        // Another lifecycle or mutation owns the state transition.
        break;
      case "SecureKeyUnavailable":
      case "TrustedTimeUnavailable":
        // Explicit retry may be offered after the prior outcome is known.
        break;
      default:
        throw error;
    }
  } else {
    throw error;
  }
}
```

Never log encrypted object inputs together with plaintext, secrets, raw handles, or provider diagnostics.

## Ambiguous mutations

If and only if a mutation returns `Ambiguous`, reconcile that exact operation ID through the supported authority flow. Do not call restore as a substitute and do not automatically replay the mutation.

## Dispose

```ts
await restoredLifecycle.dispose();
```

Disposal invalidates handles, zeroizes owned in-memory state where supported, drains operations, and releases the Web Lock.

## Current limitations

- no production authorization until Task 6;
- no offline mutation queue or reconnect auto-replay;
- no production claim based only on Chromium development evidence;
- no raw key export or backend plaintext processing;
- unsupported platforms must fail closed.

Authoritative runnable source lives in `packages/sdk-ts/examples/quickstart.ts` in the SDK repository. Re-check it against the exact SDK revision before copying code into an application.
