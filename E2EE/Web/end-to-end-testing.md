# Test the Web SDK end-to-end

Real-browser end-to-end tests matter because the guarantees you care about here, exclusive device ownership, durable persistence, clean cancellation, and cross-tab behavior, only exist in a real browser with real WASM and real storage. Unit mocks can stub those away and quietly pass while the real integration is broken, which is exactly the failure you are trying to catch.

Use a real browser, generated WASM, real IndexedDB, WebCrypto, and Web Locks. Unit mocks cannot prove lifecycle ownership, persistence, cancellation, or cross-tab behavior.

## Run the verified suite

```bash
cd packages/sdk-ts
npm ci
npm run build:wasm
npx tsc --noEmit
npx playwright test
```

Run the cross-platform integration package separately when validating Web/Flutter exchange:

```bash
cd packages/sdk-ts-integration
npm ci
npx playwright test
```

## Minimum application journey

This is the end-to-end path a meaningful test should walk, from opening a device through membership changes to teardown. Each numbered step exercises a guarantee that mocks cannot.

A useful E2E test must:

1. Open a lifecycle with a unique synthetic device ID.
2. Enroll and assert both result and `Committed` outcome.
3. Create a group and inspect epoch and roster.
4. Dispose the lifecycle and verify the competing Web Lock is released.
5. Open a fresh lifecycle, restore, and resolve a fresh group handle.
6. Add and remove a synthetic member/device, delivering Commits in order.
7. Exercise cancellation and assert both result error and terminal outcome.
8. Exercise explicit ambiguity reconciliation without replaying the mutation.
9. Verify stale, foreign, wrong-kind, and closed handles fail.
10. Tear down all lifecycles even when assertions fail.

## Cross-tab ownership

This proves only one tab can own a device at a time, and that ownership is released cleanly on disposal. Open one lifecycle, then attempt the same device ID in a second page. The second page must fail with `ReplayReservationConflict`; it must not queue. After disposal, a fresh page must be able to open and restore.

## Persistence and rollback

This confirms state survives a genuine reload and that tampering or rolling back durable data fails closed instead of silently resetting. Test a real reload against IndexedDB. Corrupt or roll back one durable component and assert fail-closed behavior (`PersistenceCorruption` or `RollbackDetected`) without silent reset.

## Object crypto matrix

This is the grid of encrypt/decrypt cases to cover, including cross-platform exchange and deliberate tampering, so you prove both correct decryption and correct rejection. Release artifacts should never fake success here.

Development integration builds should cover:

- encrypt then decrypt on Web;
- Web producer to Flutter consumer and the reverse;
- independently supplied expected content identity;
- `bigint` versions above JavaScript's safe integer range;
- wrong identity, nonce, ciphertext, key version, ciphersuite, and wrapped CEK;
- same-scope re-key with byte-identical retained content fields;
- consumer and joiner behavior after key evolution;
- teardown and cancellation with no plaintext release.

Ordinary release artifacts should assert `UnsupportedProtocol`, not fake successful encryption.

## Security assertions

These checks make sure nothing secret leaked into what you shipped or logged. Scan built artifacts and logs for plaintext fixtures, raw keys, CEKs, KWKs, exporter output, authority booleans, test-only imports, and integration-only authenticated mapping fixtures.

## Evidence rules

Record the exact SDK commit, browser version, operating system, WASM toolchain, first failing step, and concrete result. Chromium evidence alone does not authorize every browser or production deployment.

## Source suites

The authoritative scenarios live under:

- `packages/sdk-ts/tests/`
- `packages/sdk-ts-integration/tests/`
- `.github/workflows/`

Keep copied application tests aligned with the exact SDK revision.