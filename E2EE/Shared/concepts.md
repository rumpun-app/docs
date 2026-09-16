# Shared E2EE SDK concepts

These rules apply equally to the Web/TypeScript and Flutter/native adapters.

> **Status: `NOT_PRODUCTION_SAFE`.** Development conformance is not production authorization. Use synthetic data until Task 6 and Gate G release acceptance.

This page is the shared vocabulary that the Web/TypeScript and Flutter/native guides both build on. Read it once to get the mental model, then the platform guides will reuse these same terms with concrete code. Each section below opens with a short plain-language note on what the concept is and why it matters, followed by the precise rules you must follow.

## One Rust core, thin adapters

**Why this matters:** keeping all cryptography in a single Rust core means there is exactly one place that has to be correct about security. The adapters are deliberately "dumb" translators so no security decision is ever duplicated or second-guessed in TypeScript or Dart.

The Rust core owns cryptography, MLS state, key-version mapping, persistence semantics, authorization consumption, and lifecycle transitions. TypeScript and Dart translate typed values only. They must not implement fallback cryptography, construct AAD, infer outcomes, or manufacture authority.

## Opaque capabilities

**Why this matters:** you never hold real keys or internal state; instead the SDK gives you opaque capabilities (handles) that let you act without exposing anything sensitive. This keeps secrets inside the core and makes accidental leaks (through logs, storage, or serialization) far less likely.

> A capability is like a coat-check ticket: it lets you act on the coat without being the coat, and it is worthless the moment it is stale or you have left the building. Treat handles the same way, use them, do not inspect or keep them.

Devices, groups, lifecycles, and operations are represented by opaque capabilities bound to an owner, kind, generation, and lifecycle registry.

- Never serialize, persist, inspect, fabricate, or log handle bytes.
- Discard capabilities after disposal, shutdown, reload, or process restart.
- Resolve fresh capabilities through verified restoration.
- Expect stale, foreign, wrong-kind, and closed capabilities to fail.

## Mutations and publication outcomes

**Why this matters:** a change succeeding locally is not the same as that change being durably published. The SDK separates the two so you never assume a mutation was saved when it was not. The publication outcome is the definitive answer to "did this actually stick?", and handling `Ambiguous` correctly (by reconciling rather than retrying) is what prevents duplicate or lost changes.

A protected mutation returns a typed result and a separate terminal outcome:

- `NotCommitted`: non-commit is proven and durable state remains reusable.
- `Committed`: durable publication is proven.
- `Ambiguous`: neither commit nor non-commit can be proven.

Consume result and outcome exactly once. A successful result alone is not publication proof. Never automatically retry `Ambiguous`; reconcile its exact operation ID.

## Reads

**Why this matters:** reads only surface plaintext after the SDK has fully verified it, so a cancelled or failed read can never hand you half-decrypted content. Reads do not change anything, so unlike mutations they carry no publication outcome.

Authenticated reads return a result without a publication outcome. Plaintext is released only after complete authentication. Cancellation, teardown, or authentication failure must release no partial plaintext.

## Persistence and restore

**Why this matters:** the SDK saves its state as sealed opaque bytes plus a rollback reference so that a device can safely resume later and detect if someone tried to roll it back to an older state. Writing both pieces together (or neither) is what keeps that protection intact. Restore brings state back and hands you fresh handles; it is not the same as reconciling an in-flight mutation.

The SDK persists opaque sealed state and an independent rollback reference atomically. Storage must publish both or neither and must not add mutable plaintext metadata beside sealed bytes.

Restore validates durable state and issues fresh process-local capabilities. Restore is not reconciliation and must not downgrade an object-capable group.

## Object encryption

**Why this matters:** this is how a single piece of family content gets protected. The SDK encrypts the content with a fresh per-object key, then wraps that key with the group's key material so only members can unwrap it. You only ever receive the public encrypted bundle, never the raw keys. The byte-for-byte preservation rule below is what lets the SDK re-key a group (for example after membership changes) without re-encrypting or altering the content itself.

The SDK composes object encryption internally:

1. Generate a fresh content-encryption key and nonce.
2. Encrypt content with canonical content AAD.
3. Resolve lifecycle-owned wrapping material.
4. Wrap the content key with canonical wrap AAD.
5. Return only the public encrypted bundle.

Same-scope re-key preserves content context, content nonce, and content ciphertext byte-for-byte. Only wrapping metadata, wrapping nonce, wrapped-key ciphertext, and key version may change.

## Journal and key versions

**Why this matters:** as groups change over time, many key versions can exist. The journal is the authoritative record of which key applies to which state, so the SDK looks up the right key rather than guessing. A key version quoted on an incoming bundle is only a hint to look up, never permission to create a new mapping.

The authoritative journal binds `(lineage_id, MLS epoch)` to a monotonic key version. A bundle key version is only a lookup candidate; it never authorizes creation of a mapping. Unknown mappings fail before key derivation.

Compile-time-isolated authenticated fixtures may support development tests. Production mapping and Commit authority belong to Task 6.

## Errors and retryability

**Why this matters:** every platform reports the same set of errors, so you can handle failures consistently. Most errors are not safe to retry blindly; only a narrow set (trusted-time and secure-key unavailability) is retryable, and even then only as a deliberate retry once you know the previous outcome. Error details are also deliberately kept from crossing the boundary so secrets never leak through messages or stack traces.

Adapters expose the same stable error taxonomy. Only trusted-time and secure-key unavailability are retryable by contract, and only as a deliberate retry after the previous terminal outcome is known.

Provider text, stack traces, plaintext, secret material, and raw capabilities must not cross the adapter boundary.

## Security boundary

**Why this matters:** this is the line that makes the archive end-to-end encrypted. Backends only ever see ciphertext and a little approved metadata, never plaintext or key material. Just as important, a valid signature or decryptable ciphertext proves the data is intact, but it does not prove the sender is currently allowed to do what they are asking; membership and permission are checked separately.

Backends may receive ciphertext and minimum approved operational metadata. They must never receive family plaintext, CEKs, KWKs, exporter output, private keys, or recovery shares.

Cryptographic validity proves integrity, not current membership or permission. Production operations remain unavailable until Task 6 verifies current account, session, device, family, scope, capability, versions, freshness, and replay identity.