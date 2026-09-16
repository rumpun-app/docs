# Flutter end-to-end testing

Use this guide to test the complete native path from generated Dart bindings to the Rust lifecycle and sealed local persistence.

> **Status: `NOT_PRODUCTION_SAFE`.** Tests must use synthetic data and an isolated temporary state directory. A passing development test does not grant production authority.

## What counts as end-to-end

End-to-end here means the test drives the whole real stack, from your Dart call, across `flutter_rust_bridge`, through the native actor and the Rust lifecycle, down to sealed files on disk, and back out as a typed result and publication outcome. That full composed path matters because the guarantees you care about (real cryptography, durable persistence, clean cancellation, and correct outcomes) only exist when every layer runs for real. A mock can stub a layer away and quietly pass while the real integration is broken, which is exactly the failure you are trying to catch.

A valid test traverses the real composed path:

```text
Flutter test
  -> public/generated Dart API
  -> flutter_rust_bridge
  -> native bridge cdylib
  -> native actor and worker
  -> ProductionMlsLifecycle
  -> object service and cryptography
  -> sealed filesystem state
  -> typed result/error and publication outcome
```

Direct Rust-core tests, mocked operation results, manually planted outcomes, or a Dart-only crypto implementation are not end-to-end evidence.

## 1. Verify prerequisites

First confirm your machine matches the verified toolchain, so a failure is a real defect and not a version mismatch.

From the E2EE SDK repository:

```bash
rustc --version
flutter --version
dart --version
```

Development baseline:

```text
Rust 1.91.1
Flutter 3.47.0
Dart 3.13.0
flutter_rust_bridge_codegen 2.12.0
Linux x86_64
```

## 2. Run the public release-path quickstart

The fastest way to confirm the whole path works is the bundled harness, which builds the release bridge and runs one composed test through the public generated API for you.

The repository includes a harness that builds the release bridge, creates a fresh state directory, initializes FRB, and runs the public generated API:

```bash
FLUTTER_BIN="$(command -v flutter)" \
  scripts/gate-g-g6-quickstart-harness.sh
```

Expected result:

```text
PASS: 1  FAIL: 0
G6 quickstart passed through the public generated API.
```

The harness runs only:

```text
packages/sdk-dart/test/g6_quickstart_test.dart
```

with the exact test name:

```text
g6:quickstart-end-to-end
```

It covers lifecycle creation, enrollment, operation identity, result/outcome consumption, deterministic cancellation, group creation, inspection, pre-admission rejection, reconciliation failure, disposal, restore, stale/wrong-kind/cross-owner handles, and resource bounds.

## 3. Run one test manually

When you need more control than the harness gives, run the same test yourself: build the bridge, point the environment at an isolated state directory, and invoke the test directly.

Build the bridge and create isolated state:

```bash
cargo build -p rumpun-sdk-bridge --release

WORK="$(mktemp -d)"
export RUMPUN_STATE_DIR="$WORK/state"
export RUMPUN_BRIDGE_CDYLIB="$PWD/target/release/librumpun_sdk_bridge.so"
mkdir -p "$RUMPUN_STATE_DIR"
```

Run the test:

```bash
cd packages/sdk-dart
flutter pub get
flutter test test/g6_quickstart_test.dart \
  --plain-name "g6:quickstart-end-to-end"
```

Clean up afterward:

```bash
rm -rf "$WORK"
```

Never point tests at an application's real state directory.

## 4. Write a composed Flutter test

To write your own end-to-end test, follow this skeleton: it loads the exact native library the runner supplies, skips cleanly when that environment is absent, and always tears the lifecycle down. Skipping when the environment is missing is what stops a test from silently running only part of the path.

Use the generated bridge and public operations. Initialize the exact cdylib supplied by the runner:

```dart
import 'dart:io';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_rust_bridge/flutter_rust_bridge_for_generated_io.dart';
import 'package:rumpun_sdk_dart/rumpun_sdk_dart.dart';

void main() {
  final bridge = Platform.environment['RUMPUN_BRIDGE_CDYLIB'];
  final state = Platform.environment['RUMPUN_STATE_DIR'];

  if (bridge == null || state == null) {
    test(
      'composed test requires the native runner',
      () {},
      skip: 'set RUMPUN_BRIDGE_CDYLIB and RUMPUN_STATE_DIR',
    );
    return;
  }

  setUpAll(() async {
    await RustLib.init(
      externalLibrary: ExternalLibrary.open(bridge),
    );
  });

  tearDownAll(RustLib.dispose);

  test('synthetic lifecycle round trip', () async {
    final lifecycle = await lifecycleCreate();
    try {
      // Use public/generated operations here.
    } finally {
      await lifecycleShutdown(lifecycle: lifecycle);
    }
  });
}
```

A plain `flutter test` without the integration environment must skip explicitly. It must never silently run half of the path.

## 5. Test encrypt and decrypt

Object encryption is gated in ordinary builds, so to exercise it you build the compile-time-isolated integration bridge, point the environment at it, and run the composed object test that provisions a group, encrypts, re-keys, decrypts, and restores.

Protected object operations need the compile-time-isolated integration bridge until Task 6 provides production authority. Generate and build that bridge first:

```bash
codegen="$(scripts/bootstrap-frb-codegen.sh "$RUNNER_TEMP/frb-codegen")"

(
  cd crates/rumpun-sdk-bridge
  RUSTFLAGS='--cfg=task_5e_integration --check-cfg=cfg(task_5e_integration)' \
    "$codegen" generate --config-file flutter_rust_bridge.yaml
)

RUSTFLAGS='--cfg=task_5e_integration --check-cfg=cfg(task_5e_integration)' \
  cargo build -p rumpun-sdk-bridge --profile task-5e-integration
```

Set the bridge path:

```bash
export RUMPUN_BRIDGE_CDYLIB="$PWD/target/task-5e-integration/librumpun_sdk_bridge.so"
export RUMPUN_STATE_DIR="$(mktemp -d)"
```

Run the composed object test:

```bash
cd packages/sdk-dart
flutter test test/task_5e_integration_composed_test.dart \
  --plain-name "provision, encrypt, advance, rewrap, decrypt, and restore"
```

The test must prove:

1. A Rust-owned object-capable group is provisioned.
2. Encrypt returns a typed public bundle.
3. Decrypt returns the exact synthetic plaintext.
4. Re-key preserves content context, nonce, and ciphertext byte-for-byte.
5. Old and new authorized wraps behave as required.
6. Restore uses sealed state and fresh handles.
7. No CEK, KWK, exporter output, or authority object crosses Dart.

## 6. Test negative and attack paths

Proving the SDK accepts valid input is only half the job; you also prove it rejects bad input safely. This step runs the attack matrix to confirm malformed, tampered, and stale inputs fail closed and release no plaintext.

Run generated-adapter attack coverage:

```bash
flutter test test/task_5e_integration_composed_attack_test.dart \
  --plain-name "Gate F generated object attack, bounds, replay, and teardown matrix"
```

At minimum, verify:

- malformed and oversized input fails before crypto;
- wrong family, object, scope, version, schema, or ciphersuite is rejected;
- nonce, ciphertext, tag, and wrapped-CEK tampering releases no plaintext;
- wrong-kind, stale, closed, and foreign handles fail with stable codes;
- rejected work leaves durable state unchanged;
- a later valid operation still succeeds;
- cleanup returns workers, permits, locks, and operation entries to baseline.

Never loosen a test to accept two unrelated error codes. Fix the nondeterminism or contract mismatch instead.

## 7. Test cancellation and isolate teardown

These tests prove that cancelling or abruptly exiting an isolate leaves no leaked workers and that opaque handles cannot be smuggled across isolates. Use a fresh state directory per scenario so one test cannot contaminate another.

Run the focused isolate tests with a fresh state directory per scenario:

```bash
flutter test test/f8_dart_isolate_tests.dart \
  --plain-name \
  "dart_object_isolate_cleanup object-round-trip-then-clean-shutdown-leaks-zero-workers"

flutter test test/f8_dart_isolate_tests.dart \
  --plain-name \
  "dart_object_isolate_abrupt_exit pending-object-operation-is-rust-drained-after-isolate-exit"

flutter test test/f8_dart_isolate_tests.dart \
  --plain-name \
  "dart_object_foreign_isolate_rejects_handles opaque-handle-transfer-refused-foreign-isolate-stays-clean"
```

Use Rust-owned terminal acknowledgements. Sleeps and timer polling are not deterministic cleanup proof.

## 8. Test restore correctly

Restore is easy to test wrong, so this step spells out the honest sequence: finish the original mutation, shut down, reopen over the same sealed state, and prove old handles fail while fresh ones work.

For a restart scenario:

1. Complete and verify the original mutation outcome.
2. Shut down the first lifecycle.
3. Create a fresh lifecycle over the same isolated state directory.
4. Call restore.
5. Assert old handles fail.
6. Resolve fresh handles.
7. Perform a new valid operation.

Restore is not reconciliation. Do not use ordinary restore as evidence that an ambiguous mutation was reconciled.

## 9. Assert both result and outcome

Every mutation test must check two things, not one: the produced result and the separate publication outcome. A successful result alone is never proof that the change was published, so assert both.

For every mutation:

```dart
final operation = await lifecycleCreateKeyPackage(
  lifecycle: lifecycle,
  deviceWire: device,
);

final result = await operation.awaitResult();
final outcome = await operation.awaitOutcome();

expect(result, isNotEmpty);
expect(outcome, PublicationOutcome.committed);
```

For deterministic pre-effect cancellation:

```dart
expect(await exactError(operation.awaitResult()), FfiErrorCode.canceled);
expect(await operation.awaitOutcome(), PublicationOutcome.notCommitted);
```

A generic successful result is never publication proof. Consume each future once.

## 10. Check durable-state invariance

To prove a rejected call wrote nothing, snapshot the sealed files before it and compare bytes afterward. Byte equality is the observation you can make from the application side without ever decoding the sealed records.

Capture only the opaque sealed files in the test-owned state directory before a rejected call, then compare them afterward:

```dart
Map<String, List<int>> durableSnapshot(String root) {
  final snapshot = <String, List<int>>{};
  for (final entry in Directory(root).listSync()) {
    if (entry is File) {
      final name = entry.uri.pathSegments.last;
      if (name.startsWith('pair_') || name == 'slot.bin') {
        snapshot[name] = entry.readAsBytesSync();
      }
    }
  }
  return snapshot;
}
```

Do not decode sealed records in Dart. Byte equality is the application-visible no-write observation.

## 11. Run the CI-equivalent Flutter suite

Finally, run what CI runs so your local pass matches the pipeline's verdict before you push.

Before opening or updating a PR, run:

```bash
cargo fmt --all --check
cd packages/sdk-dart
flutter pub get
dart analyze lib
flutter test
```

Then run the exact focused composed commands from:

```text
.github/workflows/task-5e-flutter.yml
```

CI additionally regenerates FRB bindings and rejects a diff, builds the ordinary release bridge separately, executes native actor regressions, runs the Dart corpus, validates exactly 12 canonical IDs, and uploads an exact-SHA report.

## Failure triage

When a test fails, report:

```text
exact full commit SHA
first failing command
first causal error
expected and observed stable code/outcome
durable state before/after
cleanup observation
whether later valid admission succeeded
```

Do not report only "CI is red," and do not push an unvalidated partial checkpoint.

## End-to-end checklist

- [ ] Test uses public/generated Dart APIs.
- [ ] Native bridge was generated and built with the intended configuration.
- [ ] State directory is fresh and test-owned.
- [ ] Test uses synthetic data only.
- [ ] Every mutation checks result and outcome.
- [ ] Reads return plaintext only after full authentication.
- [ ] Rejections prove no durable change and later admission where applicable.
- [ ] Cleanup uses an explicit terminal acknowledgement.
- [ ] Restore uses fresh handles and is not called reconciliation.
- [ ] No secret, plaintext, raw handle, or provider diagnostic is logged.
- [ ] Exact source SHA and toolchain are recorded.
