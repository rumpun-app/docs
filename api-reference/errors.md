# Errors

Every error response uses one envelope. `code` is the stable, programmatic contract; `message` is human-facing and may change, so never branch on it.

```json
{
  "error": {
    "code": "AGGREGATE_VERSION_STALE",
    "message": "Request could not be completed.",
    "request_id": "01939f50-7c00-7000-8000-000000000001",
    "retryable": true,
    "details": { "latest_version": 4 }
  }
}
```

- `code` — stable machine-readable code. Branch on this.
- `message` — generic, non-sensitive, not a contract.
- `request_id` — echoes the request's `X-Request-Id` for correlation.
- `retryable` — whether a retry can succeed.
- `details` — structured, non-sensitive hints (e.g. `latest_version`, `retry_after_seconds`). Field-level detail is `{ "field": "opaque" }` when disclosure would leak content.

## Standard HTTP mapping

These come from the shared `responses` component and appear across modules:

| Status | Code | Meaning | Retryable |
|---|---|---|---|
| 400 | `REQUEST_MALFORMED` | Request could not be parsed or was structurally invalid | No |
| 401 | `AUTHENTICATION_REQUIRED` | Invalid or expired service session | No |
| 403 | `CAPABILITY_DENIED` | Actor lacks the explicit capability | No |
| 404 | `RESOURCE_NOT_FOUND` | Missing or **deliberately concealed** resource | No |
| 409 | `DOMAIN_STATE_CONFLICT` | State or idempotency conflict | No |
| 412 | `AGGREGATE_VERSION_STALE` | Stale `If-Match` aggregate version (`details.latest_version`) | Yes |
| 422 | `DOMAIN_INVARIANT_VIOLATED` | Domain or schema validation failed | No |
| 429 | `RATE_LIMITED` | Rate limit exceeded (`Retry-After` header, `details.retry_after_seconds`) | Yes, after wait |

A `404 RESOURCE_NOT_FOUND` may mean the resource is concealed from you rather than absent. Do not infer existence from the distinction between "not found" and "forbidden" — concealment is intentional.

## Authorization context codes (ADR-091)

Returned by protected operations when server-authoritative authority does not match the request:

| Code | Meaning |
|---|---|
| `AUTHORIZATION_CONTEXT_CHANGED` | Resolved authority changed between validation and commit |
| `AUTHORIZATION_EPOCH_STALE` | Expected authorization epoch is behind the current one |
| `RESOURCE_TENANCY_MISMATCH` | Resource belongs to a different tenant/family/payment account |
| `EXTERNAL_EVIDENCE_CONTEXT_INVALID` | Required external evidence does not bind to this context |
| `REAUTHENTICATION_REQUIRED` | A short-lived, operation-bound strong re-auth receipt is required |
| `IDEMPOTENCY_CONTEXT_CHANGED` | Idempotency key reused with a changed body or authority |

## Recurring concurrency and reserved codes

| Code | Meaning |
|---|---|
| `AGGREGATE_VERSION_STALE` | `If-Match` version is behind the aggregate |
| `SCOPE_VERSION_STALE` | Client scope/key version is behind the active version |
| `ROUTE_NOT_AVAILABLE` | Reserved Post-MVP route; returns `404` |

## Module-specific codes

Each module page lists its own stable codes. Grouped for reference:

- **Authentication / Passkeys / Account:** `ACCOUNT_STATE_INVALID`, `EMAIL_VERIFICATION_REQUIRED`, `IDENTITY_LINK_CONFLICT`, `HEALTHY_LOGIN_METHOD_REQUIRED`, `PASSKEY_CHALLENGE_EXPIRED`, `PASSKEY_CHALLENGE_REPLAYED`, `PASSKEY_ORIGIN_INVALID`, `PASSKEY_CREDENTIAL_INVALID`, `PASSKEY_NOT_FOUND`.
- **Devices:** `DEVICE_CAPABILITY_INSUFFICIENT`, `DEVICE_REVOKED`, `DEVICE_NOT_FOUND`, `DEVICE_KEY_VERSION_STALE`, `SIGNATURE_INVALID`.
- **Families / Memberships:** `FAMILY_STATE_INVALID`, `INITIAL_ENVELOPE_REQUIRED`, `RECIPIENT_DEVICE_MISMATCH`, `MEMBERSHIP_NOT_FOUND`, `MEMBERSHIP_REVOKED`, `MEMBERSHIP_APPROVAL_REQUIRED`, `KEY_ROTATION_REQUIRED`.
- **Invitations / Claims:** `INVITATION_EXPIRED`, `INVITATION_REVOKED`, `INVITATION_ALREADY_USED`, `INVITATION_DEVICE_REQUIRED`, `INVITATION_STATE_INVALID`, `CLAIM_CONFLICT`, `CLAIM_EXPIRED`, `CLAIM_ALREADY_REVIEWED`, `CLAIM_NOT_REVIEWABLE`, `DESTINATION_DEVICE_MISMATCH`.
- **Envelopes / Rotations / Recovery:** `KEY_VERSION_INVALID`, `SCOPE_NOT_GRANTED`, `ENVELOPE_EXPIRED`, `ENVELOPE_ALREADY_ACKNOWLEDGED`, `ROTATION_INCOMPLETE`, `ENVELOPE_COVERAGE_INCOMPLETE`, `RECIPIENT_NOT_ELIGIBLE`, `ROTATION_ALREADY_COMPLETED`, `RECOVERY_THRESHOLD_NOT_MET`, `RECOVERY_DELAY_ACTIVE`, `RECOVERY_VETOED`, `RECOVERY_EXPIRED`, `RECOVERY_SCOPE_NOT_PREVIOUSLY_HELD`, `RECOVERY_KEY_VERSION_UNAVAILABLE`, `APPROVAL_REPLAYED`, `REQUESTER_CANNOT_APPROVE`.
- **Tree / Content / Voice Stories:** `STRUCTURAL_CONFLICT`, `HIDDEN_RELATION`, `RELATION_TYPE_INVALID`, `RELATION_PERIOD_INVALID`, `KINSHIP_LENS_NOT_AVAILABLE`, `PROJECTION_PARTIAL`, `CONTENT_TYPE_NOT_ALLOWED`, `PLAINTEXT_FIELD_PROHIBITED`, `VISIBILITY_SCOPE_INVALID`, `KEY_SCOPE_MISMATCH`, `CONTENT_POLICY_REJECTED`, `REPRESENTATION_KIND_INVALID`, `VOICE_STORY_STATE_INVALID`, `VOICE_STORY_SESSION_ORDER_INVALID`, `MEDIA_NOT_READY`.
- **Consent / Biometric deletion:** `CONSENT_REQUIRED`, `CONSENT_WITHDRAWN`, `CONSENT_SCOPE_MISMATCH`, `CONSENT_VERSION_UNSUPPORTED`, `WITHDRAWAL_TOKEN_INVALID`, `SUBJECT_AUTHORITY_REQUIRED`, `PROHIBITED_PROCESSING`, `SUBJECT_SCOPE_INVALID`, `DELETION_ALREADY_COMPLETED`, `REDACTION_REQUIRED`, `DERIVATIVE_PURGE_INCOMPLETE`, `OBJECT_NOT_ACCESSIBLE`.
- **GEDCOM import:** `PLAINTEXT_GEDCOM_PROHIBITED`, `DNA_EXTENSION_REJECTED`, `GEDCOM_IMPORT_RESOURCE_LIMIT`, `IMPORT_BATCH_OUT_OF_ORDER`, `IMPORT_CHECKPOINT_INVALID`, `IMPORT_BATCH_DIGEST_MISMATCH`, `GRAPH_VERSION_CONFLICT`, `IMPORT_CANCELLED`.
- **Media:** `UPLOAD_EXPIRED`, `UPLOAD_INCOMPLETE`, `CHUNK_INDEX_INVALID`, `CHUNK_DIGEST_MISMATCH`, `CHUNK_ALREADY_EXISTS_DIFFERENT`, `MANIFEST_INVALID`, `CIPHERTEXT_SIZE_MISMATCH`, `RANGE_NOT_SATISFIABLE`, `MEDIA_NOT_READY`.
- **Sync / Family deletion:** `SYNC_CURSOR_INVALID`, `SYNC_CURSOR_EXPIRED`, `CIPHERTEXT_REF_INVALID`, `CHANGE_BATCH_TOO_LARGE`, `TOMBSTONE_CONFLICT`, `BOOTSTRAP_PROJECTION_INVALID`, `DELETION_QUORUM_NOT_MET`, `DELETION_COOLING_OFF`, `DELETION_ALREADY_COMPLETED`, `DELETION_CANCELLED`, `DELETION_REJECTED`, `LEGAL_HOLD_RESTRICTED`, `PURGE_INCOMPLETE`.
- **Notifications:** `NOTIFICATION_NOT_FOUND`, `NOTIFICATION_OWNERSHIP_MISMATCH`, `NOTIFICATION_ACTION_NOT_ALLOWED`, `NOTIFICATION_SECURITY_PREFERENCE_IMMUTABLE`.
- **Roles / break-glass (ADR-092/093):** `ROLE_ASSIGNMENT_FORBIDDEN`, `ROLE_VERSION_STALE`, `DELEGATION_CEILING_EXCEEDED`, `SELF_ESCALATION_FORBIDDEN`, `FOUR_EYES_APPROVAL_REQUIRED`, `LAST_ELIGIBLE_OWNER_REQUIRED`, `SYSTEM_ROLE_ASSIGNMENT_APPROVAL_REQUIRED`, `SYSTEM_ROLE_ASSIGNMENT_SELF_APPROVAL_FORBIDDEN`, `BREAK_GLASS_DISABLED`, `BREAK_GLASS_APPROVAL_REQUIRED`, `BREAK_GLASS_EXPIRED`, `TEMPLATE_IMPACT_ANALYSIS_REQUIRED`.

## Handling guidance

- Never surface `details.field: "opaque"` or a raw code to end users; map to friendly copy locally.
- On `AGGREGATE_VERSION_STALE` / `SCOPE_VERSION_STALE`, re-read and re-apply rather than force-writing.
- On `429`, honor `Retry-After` before retrying.
- On any ADR-091 code, re-resolve authority (which may mean re-authenticating or refreshing membership/role state) rather than replaying blindly.
