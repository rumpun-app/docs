# Rumpun REST API reference

This space documents the Rumpun backend REST API as defined by the machine contract in [`rumpun-app/openapi-contract`](https://github.com/rumpun-app/openapi-contract). The OpenAPI document is authoritative; this reference is a human-readable companion. If prose here ever conflicts with the contract, the contract wins and implementation must stop until the discrepancy is resolved.

> **Pinned source.** This reference is written against contract version **1.3.0** (147 paths / 170 operations) at `openapi-contract` commit `cc5e167`. Consumers must pull, validate, and pin the exact contract commit before implementing against it.

## What this API is

The Rumpun backend is the service tier for an end-to-end-encrypted family archive. It authorizes opaque scopes, validates approved operational metadata, and stores or routes **ciphertext**. It never receives family plaintext or key material.

Two boundaries recur throughout this reference and must never be blurred:

- **Service authentication** proves access to the Rumpun service (register, login, sessions, passkeys). It never unlocks family content and never creates, unwraps, rotates, or recovers an archive key.
- **Archive unlock** is a separate client-side state machine handled by the E2EE SDK. A successful login does not mean the archive is open.

See the [E2EE SDK space](../E2EE/README.md) for the cryptographic client that produces the ciphertext this API stores.

## What data crosses the wire

| Data | Direction | Form |
|---|---|---|
| Service credentials (email, password, OIDC assertion, passkey) | Client → server | Plaintext over TLS. Never persisted in logs or analytics. |
| Family content (names, stories, photos, transcripts, media) | Client → server | **Ciphertext only.** Encrypted on-device by the SDK before upload. |
| Key envelopes, wrapped keys, signatures | Client → server | Opaque ciphertext the server validates but cannot unwrap. |
| Approved operational metadata (versions, states, coarse capabilities) | Both | Plaintext, minimized, non-content. |
| Minimum Payment data | Both | Plaintext, minimized. No PAN/CVV/PIN/OTP/credentials. |

Family plaintext, Family Keys, private keys, recovery shares, search terms, GEDCOM, transcripts, prompts, and decrypted media are prohibited on the wire.

## Base URLs

| Base | Purpose | Default security |
|---|---|---|
| `/api/v1` | Core API (all modules except Media object transfer) | `bearerAuth` (opaque access token) |
| `/media/v1` | Encrypted media upload, chunk, complete, and content transfer | `bearerAuth` |

A small number of endpoints are explicitly public (`security: []`): account register, login, invitation preview, and passkey authentication-option retrieval. Every other operation resolves server-authoritative authorization context (ADR-091).

## How to read this reference

- Start with [Conventions](conventions.md) for envelopes, headers, identifiers, pagination, idempotency, concurrency, and the authorization model. Every module page assumes it.
- [Authentication and accounts](auth.md) covers the copy-paste-ready public entry points and service-session lifecycle.
- The module pages describe each operation group, its request and response shapes, and its stable error codes.
- [Errors](errors.md) is the consolidated catalog of the error envelope and stable codes.
- [Implementation status](implementation-status.md) records which areas are live and which are parked, so you never assume a surface is production-ready when it is not.

## Modules at a glance

| Group | Modules | Base |
|---|---|---|
| Core identity and access | Authentication, Passkeys, Devices, Account Lifecycle | `/api/v1` |
| Families and membership | Families, Memberships, Invitations, Blind Claims | `/api/v1` |
| Keys and recovery | Key Envelopes, Key Rotations, Recovery | `/api/v1` |
| Tree and content | Tree and Kinship, Content Objects, Voice Stories, Consent, Biometric Deletion, GEDCOM Import | `/api/v1` |
| Media | Encrypted Media | `/media/v1` |
| Lifecycle and sync | Family Deletion, Synchronization, Notifications | `/api/v1` |
| Payment | Payment, Payments, and Subscriptions | `/api/v1` |
| Authorization governance | Family Roles, System Roles, Authorization Context (ADR-091/092/093) | `/api/v1` |

## Example policy

Every example in this reference is synthetic. Email addresses use the reserved `example.invalid` domain, identifiers are opaque and non-sequential, and any ciphertext placeholder is not decryptable. Do not paste real family data, real credentials, or real keys into requests while testing.
