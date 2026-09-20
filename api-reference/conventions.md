# Conventions

These rules apply to every operation in this reference unless a module page states an explicit exception. They come directly from the contract's global rules (`x-rumpun-contract-rules`) and shared components.

## Envelopes

Every successful response is wrapped in a success envelope, and every error in an error envelope. The `data` shape varies by operation; the envelope shape does not.

### Success envelope

```json
{
  "data": {
    "id": "opq_01J8RUMPUNEXAMPLE000001",
    "version": 1,
    "state": "ACTIVE"
  },
  "meta": {
    "request_id": "01939f50-7c00-7000-8000-000000000001",
    "server_time": "2026-08-02T03:00:00Z",
    "api_version": "1.1"
  }
}
```

### List envelope

Collection responses add a `page` object with opaque cursor paging:

```json
{
  "data": [
    { "family_id": "fam_01J8RUMPUNEXAMPLE000001", "membership_state": "ACTIVE", "version": 4 }
  ],
  "page": { "next_cursor": null, "has_more": false },
  "meta": {
    "request_id": "01939f50-7c00-7000-8000-000000000001",
    "server_time": "2026-08-02T03:00:00Z",
    "api_version": "1.1"
  }
}
```

### Error envelope

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

`code` is the stable programmatic contract. `message` is human-facing and is **not** a contract — never branch on it. `retryable` tells you whether a retry can succeed. `details` carries structured, non-sensitive hints only. See [Errors](errors.md) for the full catalog.

## Headers

| Header | Direction | Required | Purpose |
|---|---|---|---|
| `X-Request-Id` | Request | Yes | UUID correlating the request across logs and the response `meta.request_id`. |
| `Idempotency-Key` | Request | On POST commands | UUID. The server retains the outcome for **24 hours** per actor, method, and path so a retry returns the original result instead of acting twice. |
| `If-Match` | Request | On mutations of existing aggregates | The expected aggregate version as a quoted integer, e.g. `"4"`. A stale value returns `412 AGGREGATE_VERSION_STALE`. |
| `X-Authorization-Epoch` | Request | Optional | The expected current authorization epoch for a protected mutation. It detects stale authority and **never grants permission**. |
| `Range` | Request | Optional | `bytes=<start>-<end>` for media content transfer over ciphertext. |
| `X-Chunk-Ciphertext-SHA256` | Request | On media chunk PUT | Base64url SHA-256 digest of the chunk's ciphertext body. A retry must reuse the same digest and byte length. |
| `Retry-After` | Response | On `429` | Seconds to wait before retrying a rate-limited request. |

`Authorization: Bearer <access_token>` is required on every non-public operation. Access tokens are short-lived and opaque; they are never used as an E2EE key.

## Identifiers

All resource identifiers are **opaque, type-prefixed, and non-sequential**. Never parse, infer meaning from, or derive private labels from an identifier. Common prefixes:

| Prefix | Resource |
|---|---|
| `dev_` | Device |
| `fam_` | Family |
| `mem_` | Membership |
| `ses_` | Session |
| `passkey_` | Passkey |
| `env_` | Key envelope |
| `rot_` | Key rotation |
| `rec_` | Recovery request |
| `person_` | Tree person |
| `union_` | Partner union |
| `pcr_` / `rel_` | Parent-child relation |
| `obj_` | Content object |
| `rep_` | Representation |
| `consent_` | Consent record |
| `deletion_` | Biometric/family deletion request |
| `upload_` | Media upload |
| `media_` | Completed media resource |
| `imp_` | GEDCOM import |
| `story_` / `vsession_` | Voice Story / session |
| `notif_` | Notification |
| `acctdel_` | Account deletion request |

Path identifiers match `^<prefix>_[A-Za-z0-9_-]{16,128}$` unless noted.

## Pagination

Collections use **opaque forward cursors**. Pass `?limit=<n>` (default `50`, maximum `100`) and `?cursor=<opaque>`. Read `page.next_cursor` and `page.has_more` from the response. Cursors are opaque and actor-bound; do not construct or mutate them.

## Idempotency

POST commands marked in the contract accept an `Idempotency-Key`. Replaying the same key within 24 hours returns the original outcome. If the request body or authorization context changed under the same key, the server rejects the replay (`IDEMPOTENCY_CONTEXT_CHANGED`) rather than silently acting again.

## Optimistic concurrency

Mutations of an existing aggregate require `If-Match` carrying the expected version. A stale version returns:

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

Re-read the aggregate, reapply your change on top of `latest_version`, and retry.

## Authorization model (ADR-091)

Existence is never authorization. For every protected read, mutation, job, webhook application, and support command, the server resolves **one server-authoritative context** that binds actor, session assurance, device, tenant, membership, effective capabilities, resource, operation, aggregate and scope versions, authorization epoch, idempotency identity, and any applicable external evidence.

Consequences you must handle as a client:

- Foreign resources are **concealed**, not merely forbidden — expect `404`-style concealment rather than a disclosure that a resource exists.
- Revocation, role change, device change, key rotation, Payment ownership transfer, break-glass transition, or policy change invalidates affected cached authority, queued work, and stale idempotency outcomes.
- A valid signature or decryptable ciphertext is never sufficient authorization on its own.

Stable ADR-091 failures: `AUTHORIZATION_CONTEXT_CHANGED`, `AUTHORIZATION_EPOCH_STALE`, `RESOURCE_TENANCY_MISMATCH`, `EXTERNAL_EVIDENCE_CONTEXT_INVALID`, `REAUTHENTICATION_REQUIRED`, `IDEMPOTENCY_CONTEXT_CHANGED`.

## Reserved routes

Post-MVP routes remain absent and return `404 ROUTE_NOT_AVAILABLE`. Do not assume a route exists because it appears in research material or an enum value.
