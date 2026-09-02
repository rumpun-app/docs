# Rumpun E2EE SDK

Rumpun E2EE SDK is the privacy and cryptographic foundation for the proprietary Rumpun family archive. It provides one Rust core with generated adapters for Web/TypeScript and native/Flutter.

> **Current status: `NOT_PRODUCTION_SAFE`.** Development conformance evidence does not grant production authority. Do not use the SDK with real family data until Task 6 production authorization and Gate G release review are accepted.

## Documentation

- [Core concepts](concepts.md): architecture, handles, mutations, outcomes, persistence, and security boundaries.
- [Web and TypeScript guide](web-typescript.md): build, initialize, enroll, create or restore state, and handle errors.
- [Flutter and native guide](flutter-native.md): generated FRB setup, lifecycle operations, outcomes, cancellation, and cleanup.

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
