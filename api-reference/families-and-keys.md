# Families, membership, and keys

Base: `/api/v1`. Every operation here is authenticated and resolves ADR-091 authorization context server-side. Family names, narratives, labels, photos, and private meaning are always ciphertext; the server sees opaque tenancy plus approved operational metadata.

## Devices

Devices are cryptographic destinations and capability declarations. The service receives public keys and coarse capability flags — never private keys, local database keys, installed-app lists, or plaintext.

| Method | Path | Operation |
|---|---|---|
| GET | `/devices` | `listDevices` |
| POST | `/devices` | `registerDevice` |
| GET | `/devices/{device_id}` | `getDevice` |
| PATCH | `/devices/{device_id}` | `updateDevice` |
| DELETE | `/devices/{device_id}` | `deleteDevice` |
| POST | `/devices/{device_id}/revoke` | `revokeDevice` |
| POST | `/devices/{device_id}/public-keys/rotate` | `rotateDevicePublicKeys` |

Register device request:

```json
{
  "device_public_key": "base64url_x25519_public_key",
  "algorithm": "X25519",
  "signing_public_key": "base64url_ed25519_public_key",
  "signing_algorithm": "Ed25519",
  "capabilities": {
    "secure_storage": true,
    "passkey": true,
    "local_database": true,
    "chunked_media": true
  }
}
```

State model: `UNREGISTERED → REGISTERED → ENVELOPE_PENDING → READY`, plus `REVOKED`. `READY` means required service-side envelopes are available and acknowledged; it does not assert local decryption is active. `PATCH` requires `If-Match`. Revoke blocks future operations immediately; delete removes registration; neither erases plaintext a device already obtained.

Stable errors: `DEVICE_CAPABILITY_INSUFFICIENT`, `DEVICE_REVOKED`, `DEVICE_NOT_FOUND`, `DEVICE_KEY_VERSION_STALE`, `SIGNATURE_INVALID`, `AGGREGATE_VERSION_STALE`, `RATE_LIMITED`.

## Families

A family is an opaque service tenancy plus membership and cryptographic scope coordination.

| Method | Path | Operation |
|---|---|---|
| GET | `/families` | `listFamilies` |
| POST | `/families` | `createFamily` |
| GET | `/families/{family_id}` | `getFamily` |
| PATCH | `/families/{family_id}` | `updateFamily` |

Create requires a registered destination device and initial client-created envelopes. The backend validates structure, algorithms, recipient binding, signature, size, and version without unwrapping keys:

```json
{
  "device_id": "dev_01J8RUMPUNEXAMPLE000001",
  "encrypted_payload": "base64url_ciphertext_family_profile",
  "initial_envelopes": [
    {
      "scope": "TREE_PROFILE",
      "key_version": 1,
      "recipient_device_id": "dev_01J8RUMPUNEXAMPLE000001",
      "envelope_ciphertext": "base64url_envelope_ciphertext",
      "signature": "base64url_signature"
    },
    {
      "scope": "FAMILY",
      "key_version": 1,
      "recipient_device_id": "dev_01J8RUMPUNEXAMPLE000001",
      "envelope_ciphertext": "base64url_envelope_ciphertext",
      "signature": "base64url_signature"
    }
  ]
}
```

`listFamilies` is cursor-paginated and returns only operational state and encrypted profile references. Existing mutations require `If-Match`.

Stable errors: `FAMILY_STATE_INVALID`, `CAPABILITY_DENIED`, `DEVICE_CAPABILITY_INSUFFICIENT`, `INITIAL_ENVELOPE_REQUIRED`, `RECIPIENT_DEVICE_MISMATCH`, `SIGNATURE_INVALID`, `AGGREGATE_VERSION_STALE`.

## Memberships

Family membership, tree visibility, content access, invitation authority, key issuance, recovery approval, and deletion authority are **separate capabilities**. Kinship never substitutes for permission.

| Method | Path | Operation |
|---|---|---|
| GET | `/families/{family_id}/memberships` | `listMemberships` |
| GET | `/families/{family_id}/memberships/{membership_id}` | `getMembership` |
| POST | `/families/{family_id}/memberships/{membership_id}/revoke` | `revokeMembership` |

Membership is created through invitation and claim workflows, not a generic endpoint. Capability vocabulary is explicit and server-enforced: `family.view`, `tree.view`, `tree.edit`, `content.create`, `content.view`, `content.manage_own`, `membership.invite`, `membership.review`, `key.envelope.issue`, `key.rotate`, `recovery.approve`, `family.delete.request`, `family.delete.approve`. Holding a capability never means the server can decrypt the protected scope. Revocation may return `202` because it can require envelope invalidation and key-rotation orchestration.

Stable errors: `CAPABILITY_DENIED`, `MEMBERSHIP_NOT_FOUND`, `MEMBERSHIP_REVOKED`, `MEMBERSHIP_APPROVAL_REQUIRED`, `KEY_ROTATION_REQUIRED`, `AGGREGATE_VERSION_STALE`, `DOMAIN_STATE_CONFLICT`.

## Invitations

Invitations are blind entry points. Before authentication and approval, the service discloses no family identity, tree position, member count, or content.

| Method | Path | Operation | Security |
|---|---|---|---|
| POST | `/families/{family_id}/invitations` | `createInvitation` | Bearer + `membership.invite` |
| GET | `/invitations/{token}/preview` | `previewInvitation` | Public |
| POST | `/invitations/{token}/accept` | `acceptInvitation` | Bearer |

The public preview is deliberately sparse (`expires_at`, `inviter_display_name`, `state`). Expired, invalid, revoked, or concealed tokens return uniform responses that resist enumeration. Accepting an invitation alone never opens the archive; key envelopes arrive in a separate transition.

Stable errors: `INVITATION_EXPIRED`, `INVITATION_REVOKED`, `INVITATION_ALREADY_USED`, `INVITATION_DEVICE_REQUIRED`, `INVITATION_STATE_INVALID`, `RESOURCE_NOT_FOUND`, `RATE_LIMITED`.

## Blind claims

A blind claim requests association with a tree identity without exposing the tree beforehand. Approval confirms membership state only; decryption still requires valid envelopes.

| Method | Path | Operation |
|---|---|---|
| POST | `/invitations/{token}/claim` | `submitBlindClaim` |
| GET | `/families/{family_id}/claims` | `listClaims` |
| POST | `/families/{family_id}/claims/{claim_id}/approve` | `approveClaim` |
| POST | `/families/{family_id}/claims/{claim_id}/reject` | `rejectClaim` |

The claim payload is client-encrypted; the service never receives a plaintext name, relationship answer, birth date, or tree position. Review sequence: `PENDING_REVIEW → APPROVED_ENVELOPE_PENDING → ACTIVE`, with terminal `REJECTED`, `EXPIRED`, `CONFLICT`, `CANCELLED`. Reviewers need `membership.review`; approval is idempotent.

Stable errors: `MEMBERSHIP_APPROVAL_REQUIRED`, `CLAIM_CONFLICT`, `CLAIM_EXPIRED`, `CLAIM_ALREADY_REVIEWED`, `CLAIM_NOT_REVIEWABLE`, `DESTINATION_DEVICE_MISMATCH`, `CAPABILITY_DENIED`, `AGGREGATE_VERSION_STALE`.

## Key envelopes

An envelope is ciphertext wrapping a client-generated scope key for one destination device. The service validates metadata and signatures but cannot unwrap it.

| Method | Path | Operation |
|---|---|---|
| GET | `/families/{family_id}/envelopes` | `listEnvelopes` |
| POST | `/families/{family_id}/envelopes` | `issueEnvelope` |
| POST | `/families/{family_id}/envelopes/{envelope_id}/acknowledge` | `acknowledgeEnvelope` |
| DELETE | `/families/{family_id}/envelopes/{envelope_id}` | `revokeEnvelope` |

Issue requires `key.envelope.issue` and an idempotency key:

```json
{
  "recipient_device_id": "dev_01J8RUMPUNEXAMPLE000001",
  "scope": "FAMILY",
  "scope_id": "fam_01J8RUMPUNEXAMPLE000001",
  "key_version": 3,
  "algorithm": "X25519+HKDF-SHA256+A256KW",
  "envelope_ciphertext": "base64url_envelope_ciphertext",
  "sender_device_id": "dev_01J8RUMPUNEXAMPLE000002",
  "signature": "base64url_ed25519_signature",
  "expires_at": "2026-08-09T03:00:00Z"
}
```

Envelope ciphertext is never logged, indexed, inspected, deduplicated, or sent to analytics. Family Keys, content keys, private keys, recovery shares, and plaintext scope names inside the payload are prohibited. Acknowledgement confirms the client obtained and locally validated the envelope; it does not prove decryption.

Stable errors: `RECIPIENT_DEVICE_MISMATCH`, `DEVICE_REVOKED`, `KEY_VERSION_INVALID`, `SCOPE_NOT_GRANTED`, `SIGNATURE_INVALID`, `ENVELOPE_EXPIRED`, `ENVELOPE_ALREADY_ACKNOWLEDGED`, `CAPABILITY_DENIED`, `RATE_LIMITED`.

## Key rotations

Key rotation protects **future** access after membership, device, or policy changes. It never claims retroactive erasure of plaintext already obtained.

| Method | Path | Operation |
|---|---|---|
| POST | `/families/{family_id}/key-rotations` | `startKeyRotation` |
| GET | `/families/{family_id}/key-rotations/{rotation_id}` | `getKeyRotation` |
| POST | `/families/{family_id}/key-rotations/{rotation_id}/envelopes` | `addRotationEnvelopes` |
| POST | `/families/{family_id}/key-rotations/{rotation_id}/complete` | `completeKeyRotation` |

State model: `REQUESTED → COLLECTING_ENVELOPES → COVERAGE_READY → COMPLETED`, plus `BLOCKED`, `CANCELLED`, `EXPIRED`, `FAILED_SAFE`. Completion requires `key.rotate`, coverage of all required eligible recipients (unless policy permits staged activation), monotonic target versions, and updates active scope versions atomically.

Stable errors: `ROTATION_INCOMPLETE`, `ENVELOPE_COVERAGE_INCOMPLETE`, `KEY_VERSION_INVALID`, `RECIPIENT_NOT_ELIGIBLE`, `SIGNATURE_INVALID`, `ROTATION_ALREADY_COMPLETED`, `AGGREGATE_VERSION_STALE`, `CAPABILITY_DENIED`.

## Recovery

Recovery is M-of-N approval by authorized family keyholders, bound to a verified destination device, requested scope, prior access, key version, nonce, delay, veto window, and expiry. **Rumpun has no master recovery key.**

| Method | Path | Operation |
|---|---|---|
| POST | `/recovery/requests` | `createRecoveryRequest` |
| GET | `/recovery/requests/{recovery_id}` | `getRecoveryRequest` |
| POST | `/recovery/requests/{recovery_id}/approvals` | `approveRecovery` |
| POST | `/recovery/requests/{recovery_id}/veto` | `vetoRecovery` |
| POST | `/recovery/requests/{recovery_id}/cancel` | `cancelRecovery` |
| POST | `/recovery/requests/{recovery_id}/finalize` | `finalizeRecovery` |
| GET | `/recovery/requests/{recovery_id}/envelopes` | `listRecoveryEnvelopes` |

State model: `REQUESTED → AWAITING_APPROVALS → THRESHOLD_MET_DELAY → READY_TO_FINALIZE → COMPLETED`, plus `VETOED`, `CANCELLED`, `EXPIRED`, `SCOPE_UNAVAILABLE`, `FAILED_SAFE`. Approvers need `recovery.approve` and cannot be the requester. Finalization cannot deliver a scope or version the requester never held; a password reset cannot satisfy recovery; support cannot override quorum.

Stable errors: `RECOVERY_THRESHOLD_NOT_MET`, `RECOVERY_DELAY_ACTIVE`, `RECOVERY_VETOED`, `RECOVERY_EXPIRED`, `RECOVERY_SCOPE_NOT_PREVIOUSLY_HELD`, `RECOVERY_KEY_VERSION_UNAVAILABLE`, `DESTINATION_DEVICE_MISMATCH`, `APPROVAL_REPLAYED`, `REQUESTER_CANNOT_APPROVE`, `SIGNATURE_INVALID`.
