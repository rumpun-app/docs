# Authentication and accounts

Service authentication proves access to the Rumpun service. It never unlocks family ciphertext, creates archive keys, or recovers a Family Key. Password, Google OIDC, and passkey are login methods for the **service account** only; local archive unlock is a separate client-side state machine.

Base: `/api/v1`. Register, login, and passkey authentication-option retrieval are public (`security: []`); everything else needs a bearer session.

## Copy-paste: register a service account

`registerAccount` is public. The body carries the email and password **in plaintext over TLS** — these are service credentials, not family content, so they are not encrypted at the application layer. `locale` is optional.

```bash
curl -sS -X POST https://api.rumpun.example/api/v1/auth/register \
  -H 'Content-Type: application/json' \
  -H 'X-Request-Id: 01939f50-7c00-7000-8000-000000000001' \
  -H 'Idempotency-Key: 01939f50-7c00-7000-8000-000000000002' \
  -d '{
    "email": "dewi@example.invalid",
    "password": "example-not-a-real-secret",
    "locale": "id-ID"
  }'
```

Successful response (`201`):

```json
{
  "data": {
    "account_state": "EMAIL_VERIFICATION_REQUIRED",
    "email_verified": false
  },
  "meta": {
    "request_id": "01939f50-7c00-7000-8000-000000000001",
    "server_time": "2026-08-02T03:00:00Z",
    "api_version": "1.1"
  }
}
```

## Copy-paste: log in with password

`loginWithPassword` is public and takes `email` and `password`. It does not require an `Idempotency-Key`.

```bash
curl -sS -X POST https://api.rumpun.example/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -H 'X-Request-Id: 01939f50-7c00-7000-8000-000000000001' \
  -d '{
    "email": "dewi@example.invalid",
    "password": "example-not-a-real-secret"
  }'
```

Successful response (`200`):

```json
{
  "data": {
    "access_token": "opaque_access_token",
    "expires_in": 900,
    "session_id": "ses_01J8RUMPUNEXAMPLE000001",
    "account_state": "ACTIVE",
    "email_verified": true,
    "assurance": "PASSWORD",
    "device_state": "REGISTERED"
  },
  "meta": {
    "request_id": "01939f50-7c00-7000-8000-000000000001",
    "server_time": "2026-08-02T03:00:00Z",
    "api_version": "1.1"
  }
}
```

Use the returned `access_token` as `Authorization: Bearer <access_token>` on subsequent calls. `device_state: READY` means the service completed its device and envelope state; it does **not** mean the local archive is decrypted.

> **Request shape.** In the 1.3.0 contract these public auth operations reference the generic `Command` schema, and the authoritative synthetic examples send `email`, `password`, and (for register) `locale`. When a request carries a device binding, `device_id` matches `^dev_[A-Za-z0-9_-]{16,128}$`. Passwords, reset tokens, and verification tokens never enter logs or analytics.

## Authentication operations

| Method | Path | Operation | Notes |
|---|---|---|---|
| POST | `/auth/register` | `registerAccount` | Public. |
| POST | `/auth/email/verify` | `verifyEmail` | |
| POST | `/auth/email/verify/resend` | `resendEmailVerification` | |
| POST | `/auth/login` | `loginWithPassword` | Public. |
| POST | `/auth/password/forgot` | `requestPasswordReset` | Response does not reveal account existence. |
| POST | `/auth/password/reset` | `resetPassword` | Restores service access only; never touches archive keys. |
| POST | `/auth/password/change` | `changePassword` | |
| POST | `/auth/password/add` | `addPassword` | |
| DELETE | `/auth/password` | `removePassword` | Requires another healthy login method. |
| POST | `/auth/logout` | `logout` | |
| POST | `/auth/logout-all` | `logoutAll` | |
| GET | `/auth/sessions` | `listSessions` | Cursor-paginated; operational metadata only. |
| DELETE | `/auth/sessions/{session_id}` | `revokeSession` | |
| POST | `/auth/google/start` | `startGoogleLogin` | |
| POST | `/auth/google/callback` | `completeGoogleLogin` | |
| POST | `/auth/google/link` | `linkGoogleIdentity` | Requires recent strong reauthentication. |
| DELETE | `/auth/google/link` | `unlinkGoogleIdentity` | Rejected if it removes the final healthy method. |

## Google OIDC rules

- Identity key is `(issuer, subject)`, never email alone. A matching email does not auto-link accounts.
- Linking and unlinking require recent strong reauthentication.
- Unlinking is rejected when it would remove the final healthy login method.
- A Google outage must not break an existing password or passkey method.

## Passkeys

Passkeys establish service-account assurance and may support local device unlock. The private key stays in the platform authenticator; server verification never unlocks family content.

| Method | Path | Operation |
|---|---|---|
| POST | `/auth/passkeys/options/register` | `getPasskeyRegistrationOptions` |
| POST | `/auth/passkeys/register` | `registerPasskey` |
| POST | `/auth/passkeys/options/authenticate` | `getPasskeyAuthenticationOptions` |
| POST | `/auth/passkeys/authenticate` | `authenticatePasskey` |
| GET | `/auth/passkeys` | `listPasskeys` |
| DELETE | `/auth/passkeys/{passkey_id}` | `deletePasskey` |

Challenges are one-time, audience-bound, origin-bound, and short-lived. Deleting the last healthy login method is rejected. A passkey success can satisfy service assurance while local database unlock still fails or remains pending.

## Account lifecycle

Service-account deletion is a request-and-confirm lifecycle, separate from family deletion.

| Method | Path | Operation |
|---|---|---|
| POST | `/account/deletion-requests` | `requestAccountDeletion` |
| GET | `/account/deletion-requests/{deletion_request_id}` | `getAccountDeletionRequest` |
| POST | `/account/deletion-requests/{deletion_request_id}/cancel` | `cancelAccountDeletion` |

## Stable errors

| Code | Meaning | Retry |
|---|---|---|
| `AUTHENTICATION_REQUIRED` | Missing or invalid service session | No |
| `ACCOUNT_STATE_INVALID` | Operation blocked by account lifecycle | No |
| `EMAIL_VERIFICATION_REQUIRED` | Verification must finish first | No |
| `IDENTITY_LINK_CONFLICT` | External identity already linked elsewhere | No |
| `HEALTHY_LOGIN_METHOD_REQUIRED` | Removal would lock the account | No |
| `PASSKEY_CHALLENGE_EXPIRED` | Passkey challenge no longer valid | No |
| `PASSKEY_CHALLENGE_REPLAYED` | Challenge already consumed | No |
| `PASSKEY_ORIGIN_INVALID` | Origin or RP ID mismatch | No |
| `PASSKEY_CREDENTIAL_INVALID` | Credential assertion cannot be verified | No |
| `RATE_LIMITED` | Authentication bucket exceeded | Yes, after `Retry-After` |

Public auth endpoints use `security: []`, auth errors never reveal login methods or account existence, and no endpoint accepts a password together with Family Keys, recovery phrases, or decrypted content. In the UI, keep "Masuk ke akun" (sign in) and "Buka arsip" (open the archive) as separate actions.
