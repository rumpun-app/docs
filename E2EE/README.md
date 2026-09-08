# Rumpun E2EE SDK documentation

Rumpun E2EE SDK is the privacy and cryptographic foundation for the proprietary Rumpun family archive. One authoritative Rust core is exposed through thin Web/TypeScript and Flutter/native adapters.

> **Current status: `NOT_PRODUCTION_SAFE`.** Use synthetic data only. Production
> use requires accepted production-authority and release criteria; development
> conformance never grants production authorization.

## Documentation map

### Platform guides

- [Flutter and native](Flutter/README.md): setup, lifecycle, groups, members, devices, object crypto, testing, API reference, and error catalog.
- [Web and TypeScript](Web/README.md): setup, browser lifecycle, Web Locks, membership, object crypto, testing, API reference, and contract catalog.

### Shared reference

- [Shared SDK concepts](Shared/concepts.md): opaque capabilities, mutation outcomes, persistence, object encryption, journals, errors, and security boundaries.

## Quick path

1. Choose [Flutter](Flutter/README.md) or [Web](Web/README.md).
2. Complete that platform's getting-started guide.
3. Follow lifecycle and membership guides before object encryption.
4. Use the platform API and error references while implementing.
5. Run the platform end-to-end suite with the exact pinned SDK revision.

## Canonical SDK documentation

This site is an application-integration guide. The SDK repository owns the
normative security and adapter contracts. The links below are pinned to the SDK
documentation reorganization at [`92081b3`](https://github.com/rumpun-app/rumpun-e2ee-sdk/commit/92081b3):

- [SDK documentation index](https://github.com/rumpun-app/rumpun-e2ee-sdk/blob/92081b3/docs/README.md)
- [Flutter/native quickstart](https://github.com/rumpun-app/rumpun-e2ee-sdk/blob/92081b3/docs/guides/flutter-quickstart.md)
- [Web/WASM quickstart](https://github.com/rumpun-app/rumpun-e2ee-sdk/blob/92081b3/docs/guides/web-quickstart.md)
- [MLS mapping](https://github.com/rumpun-app/rumpun-e2ee-sdk/blob/92081b3/docs/specifications/mls-mapping.md)
- [Object-layer specification](https://github.com/rumpun-app/rumpun-e2ee-sdk/blob/92081b3/docs/specifications/object-layer-spec.md)
- [`NOT_PRODUCTION_SAFE` removal criteria](https://github.com/rumpun-app/rumpun-e2ee-sdk/blob/92081b3/docs/specifications/not-production-safe-removal-criteria.md)

When upgrading the SDK revision, update this reference set and revalidate the
application against the public generated adapter API. Do not infer API or
security semantics from a historical task record.

## Architecture

```text
Flutter app -> generated Dart/FRB -> native actor -> Rust lifecycle
Web app     -> TypeScript wrapper -> WASM adapter -> Rust lifecycle

Rust lifecycle -> sealed local persistence
Application backend -> ciphertext and approved operational metadata only
```

## SDK responsibilities

- MLS device and group lifecycle.
- Client-side object encryption and authenticated decryption.
- Lifecycle-owned epoch and key-version journal.
- Sealed local persistence with rollback detection.
- Opaque capabilities, deterministic admission, cancellation, and cleanup.
- Stable cross-platform errors and publication outcomes.

## Application and backend prohibitions

- Never construct or expose CEKs, KWKs, exporter output, private keys, raw handles, or authority booleans.
- Never infer authorization from valid ciphertext or an MLS signature alone.
- Never replay an `Ambiguous` mutation automatically.
- Never store plaintext family content in logs, analytics, crash reports, or backend transport.
- Never bypass generated adapters with direct-core calls as production evidence.

## Production readiness

Production use additionally requires current authorization and authenticated
transport, supported production persistence and secure-key providers,
exact-head release evidence, independent security review, and explicit approval
to remove `NOT_PRODUCTION_SAFE`.

## Source repositories

- SDK source: <https://github.com/rumpun-app/rumpun-e2ee-sdk>
- Production authority tracker: <https://github.com/rumpun-app/rumpun-e2ee-sdk/issues/82>
