# Rumpun E2EE SDK

Rumpun E2EE SDK is the privacy and cryptographic foundation for the proprietary Rumpun family archive. It provides one Rust core with generated adapters for Web/TypeScript and native/Flutter.

> **Current status: `NOT_PRODUCTION_SAFE`.** Development conformance evidence does not grant production authority. Do not use the SDK with real family data until Task 6 production authorization and Gate G release review are accepted.

## Documentation

- [Flutter documentation hub](Flutter/README.md): ordered guides, complete table of contents, every callable API, public type, outcome, and error.
- [Web and TypeScript documentation hub](Web/README.md): complete table of contents, every callable API, public type, outcome, error, constant, and contract helper.
- [Core concepts](concepts.md): architecture, handles, mutations, outcomes, persistence, and security boundaries.
- [Cross-platform object encryption](object-encryption.md): Web/TypeScript and Flutter examples, bundle fields, bounds, storage, and failure rules.
- [Flutter and native overview](flutter-native.md): generated FRB setup, lifecycle operations, outcomes, cancellation, and cleanup.

## Flutter quick path

1. [Install and initialize](Flutter/getting-started.md).
2. [Manage lifecycle and groups](Flutter/lifecycle-and-groups.md).
3. [Manage family members and devices](Flutter/members-and-devices.md).
4. [Encrypt, decrypt, and re-key objects](Flutter/object-encryption.md).
5. Use the [complete API reference](Flutter/api-reference.md) and [types, outcomes, and errors](Flutter/types-outcomes-errors.md).
6. [Test the integration end-to-end](Flutter/end-to-end-testing.md).

## Web quick path

1. [Install and initialize the Web SDK](web-typescript.md).
2. Use the [complete Web API reference](Web/api-reference.md).
3. Handle public types, outcomes, errors, constants, and versions with the [Web contract reference](Web/types-outcomes-errors.md).
4. Follow the [cross-platform object encryption guide](object-encryption.md).
5. Re-check examples against `packages/sdk-ts/examples/quickstart.ts` at the exact SDK revision used by the application.

## What the SDK owns

- MLS device and group lifecycle.
- Client-side object encryption and authenticated decryption.
- Lifecycle-owned epoch and key-version journal.
- Sealed local persistence with rollback detection.
- Opaque handles, deterministic admission, cancellation, and cleanup.
- Stable cross-platform errors and publication outcomes.

## What applications and backends must not do

- Never construct or expose CEKs, KWKs, exporter output, private keys, raw handles, or authority booleans.
- Never infer authorization from valid ciphertext or an MLS signature alone.
- Never replay an `Ambiguous` mutation automatically.
- Never store plaintext family content in logs, analytics, crash reports, or backend transport.
- Never bypass generated adapters with direct-core calls as production evidence.

## Platform model

```text
Flutter app -> generated Dart/FRB -> native actor -> Rust lifecycle
Web app     -> TypeScript wrapper -> WASM adapter -> Rust lifecycle

Rust lifecycle -> sealed local persistence
Application backend -> ciphertext and approved operational metadata only
```

## Production readiness

Production use additionally requires:

1. Task 6 current authorization, registry, trusted-time, and authenticated transport.
2. Supported production persistence and secure-key providers.
3. Exact-head release evidence and forbidden-material scans.
4. Independent cryptography and application-security review.
5. Explicit authorization to remove `NOT_PRODUCTION_SAFE`.

## Source repositories

- SDK source: <https://github.com/rumpun-app/rumpun-e2ee-sdk>
- Production authority tracker: <https://github.com/rumpun-app/rumpun-e2ee-sdk/issues/82>
