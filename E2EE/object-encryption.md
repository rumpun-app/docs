# Object encryption, decryption, and re-keying

> **Status: `NOT_PRODUCTION_SAFE`.** These are the final public API shapes, but ordinary release builds currently fail protected object operations with `UnsupportedProtocol` until Task 6 provides production authorization. Use only the repository's compile-time-isolated integration harness with synthetic data.

## Data model

One encrypted object version contains:

- `contentContext`: family, object, scope, object version, and schema version;
- `contentNonce`: exactly 12 bytes;
- `ciphertext`: encrypted content plus the authentication tag;
- `wrapContext`: the same content identity plus key version and ciphersuite;
- `wrappedCekNonce`: exactly 12 bytes;
- `wrappedCekCiphertext`: exactly 48 bytes.

CEKs, KWKs, and MLS exporter output never appear in this bundle. Persist or transport the complete typed bundle without dropping, renaming, or reinterpreting fields.

## Web and TypeScript

### Define the content context

Use `bigint` for `objectVersion`. Identity byte strings are cryptographic identifiers, not display labels or authorization claims.

```ts
import {
  RumpunLifecycle,
  RumpunSdkError,
  type EncryptedObjectBundleV1,
  type GroupHandle,
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

The `group` below must be a live, owned object-capable `GroupHandle`. A generic group ID does not grant object authority.

### Encrypt

```ts
const plaintext = utf8.encode("Synthetic family story");

let bundle: EncryptedObjectBundleV1;
try {
  bundle = await lifecycle.encryptObjectVersionV1(
    group,
    content,
    plaintext,
  );
} catch (error) {
  if (error instanceof RumpunSdkError && error.code === "UnsupportedProtocol") {
    // Expected in an ordinary release build until Task 6 is accepted.
  }
  throw error;
}
```

The Rust lifecycle generates the CEK and nonces, constructs canonical AAD, encrypts the content, resolves the current journal mapping, and wraps the CEK. TypeScript does none of those jobs.

### Decrypt

Always supply the independently expected content context. Do not trust the bundle to tell the application which object it belongs to.

```ts
const opened = await lifecycle.decryptObjectVersionV1(
  group,
  content,
  bundle,
);

console.log(new TextDecoder().decode(opened));
```

Wrong family, object, scope, object version, schema, key version, nonce, ciphertext, or wrapped CEK fails without returning partial plaintext.

### Same-scope re-key

The destination wrap context must stay in the same family, object, scope, and object version. Obtain the authorized destination key version from the lifecycle or approved protocol flow, never from arbitrary application input.

```ts
const sourceWrap = bundle.wrapContext as {
  familyId: Uint8Array;
  objectId: Uint8Array;
  scopeId: Uint8Array;
  objectVersion: string;
  schemaVersion: number;
  keyVersion: string;
  ciphersuite: number;
};

const destinationWrap = {
  ...sourceWrap,
  keyVersion: authorizedDestinationKeyVersion.toString(),
};

const rekeyed = await lifecycle.rewrapObjectCekV1(
  group,
  content,
  sourceWrap,
  bundle,
  destinationWrap,
);
```

After re-keying, these fields must remain byte-identical:

```ts
assertBytesEqual(rekeyed.contentNonce, bundle.contentNonce);
assertBytesEqual(rekeyed.ciphertext, bundle.ciphertext);
```

Only wrap context, wrapped-CEK nonce, wrapped-CEK ciphertext, and key version may change. Cross-scope or recipient delivery does not use this operation.

## Flutter and native

The generated object operations currently live in the generated operations library:

```dart
import 'dart:typed_data';
import 'package:rumpun_sdk_dart/rumpun_sdk_dart.dart' as sdk;
import 'package:rumpun_sdk_dart/src/rust/api.dart' as api;
import 'package:rumpun_sdk_dart/src/rust/api/api_operations.dart' as operations;
```

### Encode a lossless u64

The generated Dart boundary represents object and key versions as 8-byte big-endian values.

```dart
Uint8List u64(int value) {
  if (value <= 0) throw ArgumentError.value(value);
  final bytes = Uint8List(8);
  ByteData.sublistView(bytes).setUint64(0, value, Endian.big);
  return bytes;
}
```

### Define the content context

```dart
final content = api.ObjectContentContextV1(
  familyId: Uint8List.fromList('synthetic-family'.codeUnits),
  objectId: Uint8List.fromList('story-0001'.codeUnits),
  scopeId: Uint8List.fromList('family-archive'.codeUnits),
  objectVersion: u64(1),
  schemaVersion: 1,
);
```

### Encrypt

```dart
final encrypted = await operations.encryptObjectVersionV1(
  lifecycle: lifecycle,
  versionWire: sdk.CallVersion.forFfi().toWireBytes(),
  scopeGroupHandle: group,
  content: content,
  plaintext: Uint8List.fromList('Synthetic family story'.codeUnits),
);

final bundle = encrypted.bundle;
```

The ordinary release build currently returns `FfiErrorCode.unsupportedProtocol` before protected admission. Never work around that error in Dart.

### Decrypt

```dart
final opened = await operations.decryptObjectVersionV1(
  lifecycle: lifecycle,
  versionWire: sdk.CallVersion.forFfi().toWireBytes(),
  scopeGroupHandle: group,
  expectedContentContext: content,
  bundle: bundle,
);
```

The returned `Uint8List` exists only after complete authentication. Do not log it or retain it longer than the application needs.

### Same-scope re-key

```dart
final destination = api.ObjectWrapContextV1(
  familyId: bundle.wrapContext.familyId,
  objectId: bundle.wrapContext.objectId,
  scopeId: bundle.wrapContext.scopeId,
  objectVersion: bundle.wrapContext.objectVersion,
  schemaVersion: bundle.wrapContext.schemaVersion,
  keyVersion: authorizedDestinationKeyVersionWire,
  ciphersuite: bundle.wrapContext.ciphersuite,
);

final rewrapped = await operations.rewrapObjectCekV1(
  lifecycle: lifecycle,
  versionWire: sdk.CallVersion.forFfi().toWireBytes(),
  scopeGroupHandle: group,
  expectedContentContext: content,
  sourceWrapContext: bundle.wrapContext,
  bundle: bundle,
  destinationWrapContext: destination,
);

final rekeyedBundle = api.EncryptedObjectBundleV1(
  contentContext: rewrapped.retainedContentContext,
  contentNonce: rewrapped.retainedContentNonce,
  ciphertext: rewrapped.retainedCiphertext,
  wrapContext: rewrapped.addedWrapContext,
  wrappedCekNonce: rewrapped.addedWrappedCekNonce,
  wrappedCekCiphertext: rewrapped.addedWrappedCekCiphertext,
);
```

Verify that `retainedContentContext`, `retainedContentNonce`, and `retainedCiphertext` exactly equal the original content fields.

## Storage and transport

Store the public encrypted bundle as one versioned record. Keep its content and wrap contexts intact. Never store:

- plaintext beside the bundle;
- CEK, KWK, exporter output, or private key;
- process-local lifecycle or group handles;
- caller-created authority flags;
- an unverified key-version mapping.

Backend transport may carry ciphertext and approved operational metadata only. Authorization is separate from cryptographic validity.

## Important bounds

- plaintext: `0..=16,777,216` bytes;
- family, object, and scope IDs: `1..=255` UTF-8 bytes each;
- object version: `1..=u64::MAX`;
- schema version: `1`;
- ciphersuite: `0x0001`;
- content nonce and wrapped-CEK nonce: exactly 12 bytes;
- wrapped-CEK ciphertext: exactly 48 bytes.

All bounds are checked before cryptographic work. Do not pre-truncate or silently normalize rejected values in an adapter.

## Failure rules

- Treat `AuthenticationFailure` as coarse: do not reveal which authenticated field failed.
- Treat `UnsupportedProtocol` as a closed gate, not a signal to use fallback crypto.
- Resolve stale or closed handles by obtaining fresh capabilities through the lifecycle.
- Never automatically retry a mutation with an unknown or ambiguous outcome.
- Never use restore as a substitute for reconciliation.
