# Rumpun E2EE SDK for Web and TypeScript

A focused path for integrating the Rumpun WebAssembly adapter into a browser application.

> **Current status: `NOT_PRODUCTION_SAFE`.** Use synthetic data only. Production authorization remains unavailable until Task 6 and Gate G release acceptance.

## Complete table of contents

### Guides

1. [Install, initialize, enroll, restore, and dispose](../web-typescript.md)
2. [Encrypt, decrypt, and re-key objects](../object-encryption.md)
3. [Core concepts and security boundaries](../concepts.md)

### API reference

4. [All callable Web SDK APIs](api-reference.md)
   - lifecycle opening and disposal
   - enrollment and restoration
   - group creation, lookup, and inspection
   - member and device mutations
   - ordered Commit processing
   - reconciliation
   - object encryption, decryption, and CEK rewrap
   - low-level contract-vector APIs
5. [All public types, outcomes, errors, constants, and contract helpers](types-outcomes-errors.md)
   - opaque branded values
   - mutation control and operation IDs
   - DTOs and object contexts
   - publication outcomes
   - complete stable error-code catalog
   - version, ABI, handle, and contract registries

## Choose what you need

- **New integration:** start with the Web and TypeScript guide.
- **Looking up a method signature:** use the callable API reference.
- **Handling outcomes or errors:** use the types, outcomes, and errors reference.
- **Encrypting application data:** use the object encryption guide.
- **Testing contract vectors:** use the low-level API section, never production application code.

## Runtime model

```text
Browser application
  -> thin TypeScript wrapper
  -> generated WASM boundary
  -> authoritative Rust lifecycle
  -> WebCrypto-backed secure operations
  -> IndexedDB sealed persistence
  -> Web Lock per device identity
```

TypeScript does not implement MLS, cryptography, AAD construction, authorization, sealing, or state serialization.

## Golden rules

- Await every mutation's `result` and `outcome`.
- Never serialize, decode, log, or reconstruct lifecycle, device, or group handles.
- Never add fallback WebCrypto or JavaScript cryptography.
- Never treat ciphertext, a signature, or a group ID as authorization.
- Never replay an `Ambiguous` mutation automatically.
- Dispose the lifecycle before another tab or reload opens the same device identity.
- Keep plaintext, keys, raw handles, and provider diagnostics out of logs and analytics.

## Supported development environment

The browser must provide WebAssembly, WebCrypto, IndexedDB, and Web Locks. Chromium development evidence is not a blanket production claim for every browser, operating system, or storage mode.

## Source of truth

The reviewed public surface and executable example live in:

- `packages/sdk-ts/src/index.ts`
- `packages/sdk-ts/src/contract.ts`
- `packages/sdk-ts/examples/quickstart.ts`
- `packages/sdk-ts/tests/`

Always verify these docs against the exact SDK revision used by the application.