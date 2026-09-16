# Rumpun E2EE SDK for Web and TypeScript

A focused path for integrating the WebAssembly adapter into a browser application.

**In plain terms:** your browser app talks to a thin TypeScript wrapper, the wrapper calls a WebAssembly (WASM) build of the Rust core, and that Rust core does all the real cryptography. The TypeScript layer stays deliberately thin so that no security decision is ever made twice: every key, encryption step, and authorization check lives in the one audited Rust core, and TypeScript only carries typed values back and forth.

> **Current status: `NOT_PRODUCTION_SAFE`.** Use synthetic data only. Production authorization remains unavailable until Task 6 and Gate G release acceptance.

## Complete table of contents

### Guides

1. [Install and initialize](getting-started.md)
2. [Manage lifecycle and groups](lifecycle-and-groups.md)
3. [Manage family members and devices](members-and-devices.md)
4. [Encrypt, decrypt, and re-key objects](object-encryption.md)
5. [Test Web end-to-end](end-to-end-testing.md)
6. [Understand shared security concepts](../Shared/concepts.md)

### API reference

7. [All callable Web SDK APIs](api-reference.md)
8. [All public types, outcomes, errors, constants, and contract helpers](types-outcomes-errors.md)

## Choose what you need

- **New integration:** start with installation, then lifecycle and groups.
- **Inviting or removing people/devices:** use member and device management.
- **Encrypting application data:** use object encryption.
- **Looking up a signature:** use the callable API reference.
- **Handling outcomes or errors:** use the types and contract reference.
- **Writing browser, cross-tab, or parity tests:** use end-to-end testing.

## Runtime model

This is the chain a request travels through, from your app down to storage. Each arrow is one thin hop, and the authoritative decisions happen only in the Rust lifecycle near the bottom.

```text
Browser application
  -> thin TypeScript wrapper
  -> generated WASM boundary
  -> authoritative Rust lifecycle
  -> IndexedDB sealed persistence
  -> exclusive Web Lock per device identity
```

TypeScript does not implement MLS, cryptography, AAD construction, authorization, sealing, or state serialization. (MLS is the group-messaging security protocol that decides who is currently in a family group and derives the group's shared keys.)

## Golden rules

These are the non-negotiable habits that keep your integration safe. Follow every one exactly; each maps to a rule enforced by the Rust core.

- Await every mutation's `result` and `outcome`.
- Never serialize, decode, log, or reconstruct lifecycle, device, or group handles.
- Never add fallback WebCrypto or JavaScript cryptography.
- Never treat ciphertext, a signature, or a group ID as authorization.
- Never retry an `Ambiguous` mutation automatically.
- Dispose before another tab or reload opens the same device identity.
- Keep plaintext, keys, raw handles, and provider diagnostics out of logs and analytics.

## Source of truth

When the docs and the code disagree, the code wins. These are the files to check against the exact SDK revision your application pins.

- `packages/sdk-ts/src/index.ts`
- `packages/sdk-ts/src/contract.ts`
- `packages/sdk-ts/examples/quickstart.ts`
- `packages/sdk-ts/tests/`
- `packages/sdk-ts-integration/tests/`

Always verify documentation against the exact SDK revision used by the application.