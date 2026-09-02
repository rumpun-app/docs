# Core concepts

## One Rust core, thin adapters

The Rust core owns cryptography, MLS state, key-version mapping, persistence meaning, authorization consumption, and lifecycle transitions. TypeScript and Dart translate typed values only. They must not implement fallback cryptography, construct AAD, infer outcomes, or manufacture authority.

## Opaque handles

Devices, groups, lifecycles, and operations are represented by opaque handles. A handle is process-local and bound to its owner, kind, generation, and lifecycle registry.

Treat handles as capabilities:

- do not serialize or persist them;
- do not inspect or fabricate their bytes;
- discard them after shutdown or process restart;
- obtain fresh handles through a verified restore;
- expect stale, foreign, wrong-kind, and closed handles to fail.

## Mutations and publication outcomes

A protected mutation returns a typed result and a separate terminal publication outcome:

- `NotCommitted`: no durable publication occurred;
- `Committed`: the sealed state transition completed;
- `Ambiguous`: the caller cannot prove whether publication completed.

Consume both exactly once. Never interpret a successful result alone as publication proof, and never automatically retry `Ambiguous` work. Reconcile the exact operation through the supported authority path.

## Reads

Authenticated reads return a result but have no publication outcome. Plaintext is released only after complete authentication. Cancellation, teardown, or authentication failure must release no plaintext.

## Persistence

The SDK persists an opaque sealed state record and an independent rollback reference atomically. The storage layer must publish both or neither. It must not add mutable plaintext metadata beside sealed bytes.

Restore validates the sealed state and rollback reference, then issues new process-local handles. Restore is not reconciliation and must not silently downgrade an object-capable group to a generic group.

## Object encryption

The SDK composes object encryption internally:

1. Generate a fresh content-encryption key and nonce.
2. Encrypt content with canonical content AAD.
3. Resolve lifecycle-owned wrapping material.
4. Wrap the content key with canonical wrap AAD.
5. Return the public encrypted bundle only.

Same-scope re-key preserves content context, content nonce, and content ciphertext byte-for-byte. Only wrapping metadata, wrapping nonce, wrapped-key ciphertext, and key version may change.

## Journal and key versions

The authoritative journal binds `(lineage_id, MLS epoch)` to a monotonic key version. A bundle's key version is a lookup candidate, never authority to create a mapping. Unknown mappings fail before key derivation.

Development tests may use compile-time-isolated authenticated mapping fixtures. Production mapping and commit authority belongs to Task 6.

## Error handling

Adapters return stable typed codes. Handle malformed or unsupported input, authentication or authorization failure, replay or sequence conflict, invalid or closed handles, cancellation, resource limits, persistence corruption, rollback detection, and secure-key or trusted-time unavailability.

Provider messages, stack traces, secret material, and plaintext must never cross the adapter boundary.

## Security boundary

The backend may receive ciphertext and minimum approved operational metadata. It must never receive family plaintext, CEKs, KWKs, exporter output, private keys, or recovery shares.

Cryptographic validity proves integrity, not current membership or permission. Production operations remain unavailable until Task 6 verifies current account, session, device, family, scope, capability, versions, freshness, and replay identity.
