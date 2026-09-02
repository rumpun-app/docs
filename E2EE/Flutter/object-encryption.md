# Encrypt, decrypt, and re-key objects

> **Status: `NOT_PRODUCTION_SAFE`.** The API shape is final for development evidence, but ordinary release builds return `unsupportedProtocol` until Task 6 enables production authorization. Never add fallback crypto in Dart.

## Before you start

You need:

- a live `lifecycle`;
- a live, owned, object-capable `group`;
- synthetic plaintext;
- generated DTO and operation imports.

```dart
import 'dart:typed_data';
import 'package:rumpun_sdk_dart/rumpun_sdk_dart.dart' as sdk;
import 'package:rumpun_sdk_dart/src/rust/api.dart' as api;
import 'package:rumpun_sdk_dart/src/rust/api/api_operations.dart'
    as operations;
```

## Encode object versions safely

The generated Flutter boundary carries `u64` values as exactly 8 big-endian bytes.

```dart
Uint8List u64(int value) {
  if (value <= 0) throw ArgumentError.value(value);

  final bytes = Uint8List(8);
  ByteData.sublistView(bytes).setUint64(0, value, Endian.big);
  return bytes;
}
```

Do not use decimal strings, `toString()`, platform-endian values, or floating-point numbers.

## Define the expected content identity

```dart
final content = api.ObjectContentContextV1(
  familyId: Uint8List.fromList('synthetic-family'.codeUnits),
  objectId: Uint8List.fromList('story-0001'.codeUnits),
  scopeId: Uint8List.fromList('family-archive'.codeUnits),
  objectVersion: u64(1),
  schemaVersion: 1,
);
```

These fields are authenticated identity. They are not display names and do not grant authorization.

## Encrypt

```dart
final plaintext = Uint8List.fromList(
  'Synthetic family story'.codeUnits,
);

final encrypted = await operations.encryptObjectVersionV1(
  lifecycle: lifecycle,
  versionWire: sdk.CallVersion.forFfi().toWireBytes(),
  scopeGroupHandle: group,
  content: content,
  plaintext: plaintext,
);

final bundle = encrypted.bundle;
```

Rust owns the full operation:

1. Validate handles, versions, bounds, and object identity.
2. Resolve current lifecycle-owned wrapping material.
3. Generate a fresh CEK and nonce.
4. Construct canonical content and wrap AAD.
5. Encrypt content and wrap the CEK.
6. Return only the public encrypted bundle.

The bundle contains encrypted data and authenticated metadata, never CEK, KWK, exporter output, or private keys.

## Decrypt

Supply `content` again as the independently expected identity. Never derive the expected identity from the bundle itself.

```dart
final opened = await operations.decryptObjectVersionV1(
  lifecycle: lifecycle,
  versionWire: sdk.CallVersion.forFfi().toWireBytes(),
  scopeGroupHandle: group,
  expectedContentContext: content,
  bundle: bundle,
);
```

Plaintext is returned only after complete authentication. Any family, object, scope, object-version, schema, key-version, nonce, ciphertext, or wrapped-CEK mismatch must return an error with no partial plaintext.

## Re-key without re-encrypting content

Re-keying changes the CEK wrap for the same scope. It does not encrypt the object content again.

The destination key version must come from the authorized lifecycle or approved mapping flow. It must not be invented by UI code.

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
```

Construct the additional wrapped representation:

```dart
final rekeyedBundle = api.EncryptedObjectBundleV1(
  contentContext: rewrapped.retainedContentContext,
  contentNonce: rewrapped.retainedContentNonce,
  ciphertext: rewrapped.retainedCiphertext,
  wrapContext: rewrapped.addedWrapContext,
  wrappedCekNonce: rewrapped.addedWrappedCekNonce,
  wrappedCekCiphertext: rewrapped.addedWrappedCekCiphertext,
);
```

The following fields must be byte-identical to the original:

```text
content context
content nonce
content ciphertext
```

Only these may change:

```text
wrap context
key version
wrapped-CEK nonce
wrapped-CEK ciphertext
```

Keep the old wrap when historical decryption is required. Re-keying is additive, not replacement.

## Store the encrypted bundle

Persist all bundle fields together:

```text
contentContext
contentNonce
ciphertext
wrapContext
wrappedCekNonce
wrappedCekCiphertext
```

Never persist:

- plaintext beside the bundle;
- CEK, KWK, exporter output, or private key;
- lifecycle, device, or group handles;
- an unverified journal mapping;
- caller-created authorization flags.

## Handle errors

```dart
try {
  final opened = await operations.decryptObjectVersionV1(
    lifecycle: lifecycle,
    versionWire: sdk.CallVersion.forFfi().toWireBytes(),
    scopeGroupHandle: group,
    expectedContentContext: content,
    bundle: bundle,
  );
} on api.FfiError catch (error) {
  switch (error.code) {
    case api.FfiErrorCode.authenticationFailure:
      // Do not reveal which authenticated field failed.
      break;
    case api.FfiErrorCode.unsupportedProtocol:
      // Closed production gate. Never use fallback crypto.
      break;
    case api.FfiErrorCode.invalidHandle:
    case api.FfiErrorCode.closedHandle:
      // Resolve a fresh capability through the lifecycle.
      break;
    default:
      rethrow;
  }
}
```

## Bounds

- plaintext: `0..=16,777,216` bytes;
- family, object, and scope IDs: `1..=255` UTF-8 bytes each;
- object version: `1..=u64::MAX`;
- schema version: exactly `1`;
- ciphersuite: exactly `0x0001`;
- content nonce: exactly 12 bytes;
- wrapped-CEK nonce: exactly 12 bytes;
- wrapped-CEK ciphertext: exactly 48 bytes.

Do not truncate, normalize, or retry invalid inputs in Dart. Let the Rust boundary reject them before cryptographic work.

## Checklist

- [ ] Expected content context comes from application state, not the bundle.
- [ ] Object and key versions use lossless 8-byte big-endian encoding.
- [ ] All encrypted bundle fields are stored together.
- [ ] Re-key retains content nonce and ciphertext byte-for-byte.
- [ ] No fallback cryptography exists in Dart.
- [ ] Plaintext and keys never enter logs or analytics.
- [ ] `unsupportedProtocol` remains fail-closed until Task 6 acceptance.
