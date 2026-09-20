# Implementation status

This page records what the **contract** defines versus what is **proven working end-to-end**, so you never assume a surface is production-ready when it is not. The contract describing an operation does not mean the full feature is live and interoperable across the web, mobile, and backend repos.

> **Overall product status:** the E2EE archive is `NOT_PRODUCTION_SAFE`. Use synthetic data only. See the [E2EE SDK space](../E2EE/README.md).

## Pins

| Item | Value |
|---|---|
| Contract version | 1.3.0 |
| `openapi-contract` commit | `cc5e167` |
| Paths / operations | 147 / 170 |
| Core base | `/api/v1` |
| Media base | `/media/v1` |
| Accepted security ADRs | ADR-091, ADR-092, ADR-093 (release blocking) |
| Active contract modules | Core, Tree, Memory, Media, Notification, Payment, Authorization |

## Status by area

| Area | Contract | Status |
|---|---|---|
| Authentication, passkeys, sessions, account lifecycle | 1.3.0 | Contract defined. Service auth boundary is stable; it never unlocks the archive. |
| Families, memberships, invitations, claims | 1.3.0 | Contract defined. |
| Key envelopes, key rotations, recovery | 1.3.0 | Contract defined. Client owns all key material; server validates metadata only. |
| Tree, kinship, content objects | 1.3.0 | Contract defined. |
| Voice Stories | 1.3.0 | Contract defined; depends on Media readiness for playback. |
| Consent, biometric deletion, GEDCOM import | 1.3.0 | Contract defined. |
| **Encrypted media (upload/chunk/complete/content)** | 1.3.0 | **Parked at Gate 5F.** Contract surface is pinned, but the end-to-end path is not yet proven across repos. Not production-ready. |
| Synchronization, notifications, family deletion | 1.3.0 | Contract defined. |
| Payment and subscriptions | 1.3.0 (active MVP) | Contract defined; provider-hosted checkout, verified webhooks required. |
| Authorization context and roles (ADR-091/092/093) | 1.3.0 | Accepted and release-blocking; enforced server-side. |

## Media is parked — read before integrating

The audio-media end-to-end goal (contract → store ciphertext → download with byte-range → decrypt → play on a second device → replay from persistent cache) is being delivered in staged order under the RUM-124 program, and the chain is **parked at Gate 5F**. What this means for you:

- The 1.3.0 media shapes on the [Encrypted media](media.md) page are the authoritative contract, including the removal of `encrypted_payload` and the create/chunk/complete flow.
- Interoperability of the three repos (web-app, mobile-app, backend) on the media path is **not yet certified**. Encryption-before-upload has round-trip and mock-network coverage, but full E2E media has not been demonstrated.
- Do **not** claim, ship, or document that encrypted media playback works end-to-end until that gate is cleared.

## What "contract defined" does and does not mean

- **Does mean:** the request/response shapes, headers, identifiers, and stable error codes are fixed in the pinned contract and safe to implement against.
- **Does not mean:** a running backend, verified CI on the consuming repos, or a security certification. The contract is the machine agreement; runtime behavior and cross-repo interoperability are verified separately in the consuming repositories.

If prose in this reference ever conflicts with the OpenAPI document, stop and treat the OpenAPI document as authoritative.
