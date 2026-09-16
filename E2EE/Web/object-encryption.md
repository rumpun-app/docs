# Encrypt, decrypt, and re-key Web objects

> **Status: `NOT_PRODUCTION_SAFE`.** Ordinary release builds currently return `UnsupportedProtocol` before protected object admission until Task 6 is accepted. Never add fallback WebCrypto.

## Define the expected content identity

Before encrypting or decrypting, you declare what the content is: which family, object, scope, and version it belongs to. These fields become authenticated identity that the SDK verifies, so they act as a fingerprint of the intended content rather than a display label you can change freely.

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

Encrypting hands your plaintext to the Rust core, which protects it with a fresh content-encryption key (CEK) and returns only the public encrypted bundle. You never see or handle the CEK yourself.

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

Decrypting checks that the bundle really matches the content identity you expected, then returns plaintext only after that check passes. Supply the expected identity from your own application state, never copied out of the bundle, so a swapped or tampered bundle cannot masquerade as the object you meant to open.

```ts
const opened = await lifecycle.decryptObjectVersionV1(
  group,
  content,
  bundle,
);
```

Supply `content` independently from application state. Never trust the bundle to identify which object the caller intended to open. Authentication failure returns no partial plaintext.

## Same-scope re-key

Re-keying (also called re-wrapping) adds a new wrapped copy of the same content key under different group key material, for example after the group's keys evolve. It never touches the encrypted content itself: only the wrapping changes. Because it is additive, keeping the old wrap alongside the new one preserves access for anyone who still needs it.

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

The bundle is only useful, and only verifiable, as a whole. Persist all of its fields together so the SDK can re-authenticate everything on decrypt; storing a subset breaks that guarantee.

Persist these fields together:

- `contentContext`
- `contentNonce`
- `ciphertext`
- `wrapContext`
- `wrappedCekNonce`
- `wrappedCekCiphertext`

Never store plaintext, CEK, KWK, exporter output, private keys, raw lifecycle/group handles, caller-created authority flags, or unverified key-version mappings.

## Bounds

The Rust core enforces these exact limits and rejects anything outside them. Do not try to pre-trim, pad, or normalize input in TypeScript to fit; pass values through as-is and let the core validate.

- Plaintext: `0..=16,777,216` bytes.
- Family, object, and scope IDs: `1..=255` bytes each.
- Object version: `1..=u64::MAX`, represented as `bigint`.
- Schema version: exactly `1`.
- Ciphersuite: exactly `0x0001`.
- Content and wrapped-CEK nonces: exactly 12 bytes.
- Wrapped-CEK ciphertext: exactly 48 bytes.

Do not truncate, normalize, or retry malformed input in TypeScript.

## Error handling

Object operations throw a typed `RumpunSdkError` you branch on by `code`. Handle each case for what it means rather than papering over it: notably, never reveal which authenticated field failed, and keep the production gate closed on `UnsupportedProtocol`.

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