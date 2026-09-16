# Rumpun E2EE SDK documentation

Rumpun E2EE SDK is the privacy and cryptographic foundation for the proprietary Rumpun family archive. One authoritative Rust core is exposed through thin Web/TypeScript and Flutter/native adapters.

**What this SDK does in plain terms:** it lets a family archive store and share content that only family members can read. Photos, notes, and other family content are encrypted on the device before they ever leave it, so the servers and backend see ciphertext (scrambled bytes) rather than the real content. The SDK handles the hard cryptographic parts for you: who is in the family group, which keys unlock which content, and whether a change was actually saved. Your job as a developer is to call the SDK and pass its typed values around, not to touch keys or encryption yourself.

> **Current status: `NOT_PRODUCTION_SAFE`.** Use synthetic data only until Task 6 production authorization and Gate G release acceptance.

## Documentation map

### Platform guides

- [Flutter and native](Flutter/README.md): setup, lifecycle, groups, members, devices, object crypto, testing, API reference, and error catalog.
- [Web and TypeScript](Web/README.md): setup, browser lifecycle, Web Locks, membership, object crypto, testing, API reference, and contract catalog.

### Shared reference

- [Shared SDK concepts](Shared/concepts.md): opaque capabilities, mutation outcomes, persistence, object encryption, journals, errors, and security boundaries.

## Core concepts at a glance

New to the SDK? These are the recurring terms you will meet everywhere in the docs. Each entry gives the exact term and a one-sentence plain-language gloss. See [Shared SDK concepts](Shared/concepts.md) for the precise rules behind each one.

- **MLS**: the group-messaging security protocol that decides who is currently in a family group and derives the shared keys the group uses, so that adding or removing a member changes the keys accordingly.
- **Opaque capability / handle**: a token the SDK hands you to act on a device, group, lifecycle, or operation without ever seeing its internal bytes, like a coat-check ticket that lets you claim your coat without being the coat, and that is worthless once it is stale or you have left.
- **Publication outcome**: the separate, terminal answer to "was this change actually saved?", which is exactly one of `NotCommitted` (proven not saved, safe to reuse prior state), `Committed` (proven saved), or `Ambiguous` (cannot be proven either way, must be reconciled, never blindly retried).
- **Object encryption / CEK**: the process of encrypting a piece of content with a fresh content-encryption key (CEK), which the SDK then wraps with group key material so only members can unwrap it; you get back only the public encrypted bundle.
- **Journal & key version**: the authoritative log that binds each `(lineage_id, MLS epoch)` to a monotonically increasing key version, so the SDK always knows which key applies and never guesses.
- **`NOT_PRODUCTION_SAFE`**: the current status marker meaning the SDK is not yet authorized for real family data; use synthetic data only until Task 6 production authorization and Gate G release acceptance.

## Quick path

1. Choose [Flutter](Flutter/README.md) or [Web](Web/README.md).
2. Complete that platform's getting-started guide.
3. Follow lifecycle and membership guides before object encryption.
4. Use the platform API and error references while implementing.
5. Run the platform end-to-end suite with the exact pinned SDK revision.

## Architecture

```text
Flutter app -> generated Dart/FRB -> native actor -> Rust lifecycle
Web app     -> TypeScript wrapper -> WASM adapter -> Rust lifecycle

Rust lifecycle -> sealed local persistence
Application backend -> ciphertext and approved operational metadata only
```

Read the diagram as two thin paths into one shared core:

- **Your app (Flutter or Web)** is where you write feature code. It never performs cryptography itself; it only asks the SDK to do things and reads back typed values.
- **The adapter layer** (generated Dart/FRB for Flutter, the TypeScript wrapper plus WASM for Web) translates between your language's types and the Rust core. It carries typed values across the boundary and nothing more, no fallback crypto and no home-grown key handling.
- **The Rust lifecycle (the core)** is the single source of cryptographic truth. All real encryption, MLS group state, key-version bookkeeping, and authorization live here. There is exactly one core so there is exactly one place that can be right or wrong about security, which makes the system far easier to reason about and audit.
- **Sealed local persistence** is where the core stores its own state as opaque sealed bytes on the device.
- **The application backend** only ever sees ciphertext and a small amount of approved operational metadata. It never sees family plaintext or key material.

The split exists so that adapters stay thin and interchangeable while the sensitive logic stays in one audited place. If a rule about keys or authorization needs to change, it changes in the core, not in two separate app languages.

## SDK responsibilities

- MLS device and group lifecycle.
- Client-side object encryption and authenticated decryption.
- Lifecycle-owned epoch and key-version journal.
- Sealed local persistence with rollback detection.
- Opaque capabilities, deterministic admission, cancellation, and cleanup.
- Stable cross-platform errors and publication outcomes.

## Application and backend prohibitions

These rules all exist for one reason: to keep plaintext and key material on the client and out of your application code and backend. Follow every one of them exactly.

- Never construct or expose CEKs, KWKs, exporter output, private keys, raw handles, or authority booleans.
- Never infer authorization from valid ciphertext or an MLS signature alone.
- Never replay an `Ambiguous` mutation automatically.
- Never store plaintext family content in logs, analytics, crash reports, or backend transport.
- Never bypass generated adapters with direct-core calls as production evidence.

## Production readiness

Production use additionally requires Task 6 current authorization and authenticated transport, supported production persistence and secure-key providers, exact-head release evidence, independent security review, and explicit approval to remove `NOT_PRODUCTION_SAFE`.

## Source repositories

- SDK source: <https://github.com/rumpun-app/rumpun-e2ee-sdk>
- Production authority tracker: <https://github.com/rumpun-app/rumpun-e2ee-sdk/issues/82>
