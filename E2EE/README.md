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
5. Run the platform end-to-end suite against the SDK revision accepted by the
   application release.

## Documentation ownership

`docs/E2EE/` is an **application-implementation guide**. It owns instructions
for integrating published SDK surfaces into a Flutter or Web application: app
lifecycle ownership, calling patterns, synthetic integration tests, and safe
application/backend boundaries. It does not define SDK behavior.

Technical SDK documentation is maintained separately and is the source for
cryptographic semantics, formats, security invariants, public adapter contracts,
platform support, authorization, and production readiness. If this guide and
the accepted SDK contract disagree, the SDK contract wins.

`E2EE/` has a **no-external-SDK-link rule**: it must contain no hyperlink or URL
to the SDK source or its technical documentation. Keep this guide self-contained
for application implementation. When the SDK revision changes, revalidate the
application against its public generated adapter API and update only the
application-facing calling guidance that changes.

Do not duplicate or reinterpret SDK security invariants, formats, error
semantics, authorization rules, or release criteria here. Escalate a semantic
question to the SDK maintainers; historical task records are audit context,
never a source of current application behavior.

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
