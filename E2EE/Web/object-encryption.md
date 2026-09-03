# Encrypt, decrypt, and re-key Web objects

> **Status: `NOT_PRODUCTION_SAFE`.** Ordinary release builds currently return `UnsupportedProtocol` before protected object admission until Task 6 is accepted. Never add fallback WebCrypto.

## Define the expected content identity

```ts
import {
  RumpunSdkError,
  type EncryptedObjectBundleV1,
  type ObjectContentContextV1,
} from "@rumpun/sdk-ts";

const utf8 = new TextEncoder();
const content: ObjectContentContextV1 = {
  familyId: utf8.encode("synthetic-family"),
  objectId: utf8.encode("story-0001"),
  scopeId: utf8.encode("family-archive"),
  objectVersion: 1n,
  schemaVersion: 1,
};
```

Identity fields are authenticated identifiers, not display labels or authorization claims. Use `bigint` for every object version.

## Encrypt

```ts
const plaintext = utf8.encode("Synthetic family story");
const bundle = await lifecycle.encryptObjectVersionV1(
  group,
  content,
  plaintext,
);
```

Rust generates the CEK and nonces, constructs canonical AAD, encrypts content, resolves wrapping material, and wraps the CEK. TypeScript does none of that.

## Decrypt

```ts
const opened = await lifecycle.decryptObjectVersionV1(
  group,
  content,
  bundle,
);
```

Supply `content` independently from application state. Never trust the bundle to identify which object the caller intended to open. Authentication failure returns no partial plaintext.

## Same-scope re-key

The current TypeScript surface keeps wrap contexts opaque. Transport the Rust-returned source context intact and obtain the destination key version only through the authorized lifecycle flow.

```ts
const sourceWrap = bundle.wrapContext;
const destinationWrap = authorizedDestinationWrapContext;

const rekeyed = await lifecycle.rewrapObjectCekV1(
  group,
  content,
  sourceWrap,
  bundle,
  destinationWrap,
);
```

Content context, content nonce, and content ciphertext must remain byte-identical. Re-keying is additive; retain the old wrap when historical access is required.

## Store the complete bundle

Persist these fields together:

- `contentContext`
- `contentNonce`
- `ciphertext`
- `wrapContext`
- `wrappedCekNonce`
- `wrappedCekCiphertext`

Never store plaintext, CEK, KWK, exporter output, private keys, raw lifecycle/group handles, caller-created authority flags, or unverified key-version mappings.

## Bounds

- Plaintext: `0..=16,777,216` bytes.
- Family, object, and scope IDs: `1..=255` bytes each.
- Object version: `1..=u64::MAX`, represented as `bigint`.
- Schema version: exactly `1`.
- Ciphersuite: exactly `0x0001`.
- Content and wrapped-CEK nonces: exactly 12 bytes.
- Wrapped-CEK ciphertext: exactly 48 bytes.

Do not truncate, normalize, or retry malformed input in TypeScript.

## Error handling

```ts
try {
  await lifecycle.decryptObjectVersionV1(group, content, bundle);
} catch (error) {
  if (!(error instanceof RumpunSdkError)) throw error;

  switch (error.code) {
    case "AuthenticationFailure":
      // Do not reveal which authenticated field failed.
      break;
    case "UnsupportedProtocol":
      // Keep the production gate closed.
      break;
    case "InvalidHandle":
    case "ClosedHandle":
      // Restore and resolve a fresh capability.
      break;
    default:
      throw error;
  }
}
```