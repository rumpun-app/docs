# Shared E2EE SDK concepts

These rules apply equally to the Web/TypeScript and Flutter/native adapters.

> **Status: `NOT_PRODUCTION_SAFE`.** Development conformance is not production authorization. Use synthetic data until Task 6 and Gate G release acceptance.

## One Rust core, thin adapters

The Rust core owns cryptography, MLS state, key-version mapping, persistence semantics, authorization consumption, and lifecycle transitions. TypeScript and Dart translate typed values only. They must not implement fallback cryptography, construct AAD, infer outcomes, or manufacture authority.

## Opaque capabilities

Devices, groups, lifecycles, and operations are represented by opaque capabilities bound to an owner, kind, generation, and lifecycle registry.

- Never serialize, persist, inspect, fabricate, or log handle bytes.
- Discard capabilities after disposal, shutdown, reload, or process restart.
- Resolve fresh capabilities through verified restoration.
- Expect stale, foreign, wrong-kind, and closed capabilities to fail.

## Mutations and publication outcomes

A protected mutation returns a typed result and a separate terminal outcome:

- `NotCommitted`: non-commit is proven and durable state remains reusable.
- `Committed`: durable publication is proven.
- `Ambiguous`: neither commit nor non-commit can be proven.

Consume result and outcome exactly once. A successful result alone is not publication proof. Never automatically retry `Ambiguous`; reconcile its exact operation ID.

## Reads

Authenticated reads return a result without a publication outcome. Plaintext is released only after complete authentication. Cancellation, teardown, or authentication failure must release no partial plaintext.

## Persistence and restore

The SDK persists opaque sealed state and an independent rollback reference atomically. Storage must publish both or neither and must not add mutable plaintext metadata beside sealed bytes.

Restore validates durable state and issues fresh process-local capabilities. Restore is not reconciliation and must not downgrade an object-capable group.

## Object encryption

The SDK composes object encryption internally:

1. Generate a fresh content-encryption key and nonce.
2. Encrypt content with canonical content AAD.
3. Resolve lifecycle-owned wrapping material.
4. Wrap the content key with canonical wrap AAD.
5. Return only the public encrypted bundle.

Same-scope re-key preserves content context, content nonce, and content ciphertext byte-for-byte. Only wrapping metadata, wrapping nonce, wrapped-key ciphertext, and key version may change.

## Journal and key versions

The authoritative journal binds `(lineage_id, MLS epoch)` to a monotonic key version. A bundle key version is only a lookup candidate; it never authorizes creation of a mapping. Unknown mappings fail before key derivation.

Compile-time-isolated authenticated fixtures may support development tests. Production mapping and Commit authority belong to Task 6.

## Errors and retryability

Adapters expose the same stable error taxonomy. Only trusted-time and secure-key unavailability are retryable by contract, and only as a deliberate retry after the previous terminal outcome is known.

Provider text, stack traces, plaintext, secret material, and raw capabilities must not cross the adapter boundary.

## Security boundary

Backends may receive ciphertext and minimum approved operational metadata. They must never receive family plaintext, CEKs, KWKs, exporter output, private keys, or recovery shares.

Cryptographic validity proves integrity, not current membership or permission. Production operations remain unavailable until Task 6 verifies current account, session, device, family, scope, capability, versions, freshness, and replay identity.