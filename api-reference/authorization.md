# Authorization and roles

Base: `/api/v1`. Authorization governance is defined by the accepted, release-blocking security ADRs **ADR-091** (authorization context), **ADR-092** (family roles and effective capabilities), and **ADR-093** (privileged system-role governance). These are mandatory for MVP v1.1.

## Authorization context (ADR-091)

Every protected read, mutation, job, outbox consumer, webhook application, support command, and system operation resolves **one server-authoritative context** binding actor, session assurance, device, tenant, membership, effective capabilities, resource, operation, aggregate and scope versions, authorization epoch, idempotency identity, and applicable external evidence.

Rules a client must respect:

- **Existence is not authorization.** Repository and query boundaries enforce family and Payment-account isolation; foreign resources are concealed.
- Revocation, role change, device change, key rotation, Payment ownership transfer, break-glass transition, or policy change invalidates affected cached authority, queued work, and stale idempotency outcomes.
- Clients may send an expected `X-Authorization-Epoch` on protected mutations, but the server resolves authoritative values. The header detects stale authority and **never grants permission**. A successful governed authority change returns the new epoch.
- Strong re-authentication receipts are short-lived, single-purpose, operation-bound, and phishing-resistant. They are mandatory for Owner assignment, protected System Role assignment transitions, high-risk role changes, and break-glass transitions.

Stable ADR-091 failures: `AUTHORIZATION_CONTEXT_CHANGED`, `AUTHORIZATION_EPOCH_STALE`, `RESOURCE_TENANCY_MISMATCH`, `EXTERNAL_EVIDENCE_CONTEXT_INVALID`, `REAUTHENTICATION_REQUIRED`, `IDEMPOTENCY_CONTEXT_CHANGED`.

## Family roles (ADR-092)

Family users assign only fixed, immutable, system-defined role templates — they cannot author arbitrary permissions.

| Method | Path | Operation |
|---|---|---|
| GET | `/families/{family_id}/role-templates` | `listFamilyRoleTemplates` |
| GET | `/families/{family_id}/memberships/{membership_id}/effective-capabilities` | `getMembershipEffectiveCapabilities` |
| POST | `/families/{family_id}/memberships/{membership_id}/role-assignments` | `assignFamilyRole` |
| GET | `/families/{family_id}/role-assignment-history` | `listFamilyRoleAssignmentHistory` |

`FAMILY_OWNER` assignment requires operation-bound phishing-resistant re-authentication; ownership transfer additionally requires destination acceptance. The last eligible active owner is protected (`LAST_ELIGIBLE_OWNER_REQUIRED`). Successful assignment increments the family authorization epoch and never issues an envelope, grants keyholder qualification, or transfers Payment ownership.

## System roles and break-glass (ADR-093)

Privileged role and permission governance is versioned, four-eyes-gated, and separated from family plaintext and keys.

| Method | Path | Operation |
|---|---|---|
| GET | `/system/permissions` | `listSystemPermissions` |
| GET / POST | `/system/roles` | `listSystemRoles`, `createSystemRole` |
| GET | `/system/roles/{role_id}/versions/{version}` | `getSystemRoleVersion` |
| POST | `/system/roles/{role_id}/versions` | `createSystemRoleVersion` |
| POST | `/system/role-changes/{change_id}/approve` | `approveSystemRoleChange` |
| POST | `/system/role-changes/{change_id}/activate` | `activateSystemRoleChange` |
| POST | `/system/role-changes/{change_id}/rollback` | `rollbackSystemRoleChange` |
| GET / POST | `/system/role-assignments` | `listSystemRoleAssignments`, `createSystemRoleAssignment` |
| POST | `/system/role-assignments/{assignment_id}/approve` | `approveSystemRoleAssignment` |
| POST | `/system/role-assignments/{assignment_id}/activate` | `activateSystemRoleAssignment` |
| POST | `/system/role-assignments/{assignment_id}/revoke` | `revokeSystemRoleAssignment` |
| GET | `/system/family-role-templates` | `listSystemFamilyRoleTemplates` |
| POST | `/system/family-role-templates/{template_id}/versions` | `createFamilyRoleTemplateVersion` |
| POST | `/system/break-glass/requests` | `requestBreakGlassAccess` |
| GET | `/system/break-glass/requests/{request_id}` | `getBreakGlassRequest` |
| POST | `/system/break-glass/requests/{request_id}/approve` | `approveBreakGlassAccess` |
| POST | `/system/break-glass/requests/{request_id}/activate` | `activateBreakGlassAccess` |
| POST | `/system/break-glass/requests/{request_id}/revoke` | `revokeBreakGlassAccess` |

System Role assignments follow create (`PENDING_APPROVAL`) → distinct-actor approve → activate → revoke. Activation increments the authorization epoch. Delegation ceilings, immutable versions, separation of duties, strong re-authentication, tamper-evident audit, and rollback are mandatory. Break-glass binds an incident reference, narrow capabilities, scope, reason, short expiry, and phishing-resistant re-authentication; expiry is automatic; E2EE plaintext and key access remain impossible. Every Family Role Template version must include impact analysis or fail with `TEMPLATE_IMPACT_ANALYSIS_REQUIRED`.

Stable failures: `ROLE_ASSIGNMENT_FORBIDDEN`, `ROLE_VERSION_STALE`, `DELEGATION_CEILING_EXCEEDED`, `SELF_ESCALATION_FORBIDDEN`, `FOUR_EYES_APPROVAL_REQUIRED`, `AUTHORIZATION_EPOCH_STALE`, `RESOURCE_TENANCY_MISMATCH`, `REAUTHENTICATION_REQUIRED`, `LAST_ELIGIBLE_OWNER_REQUIRED`, `SYSTEM_ROLE_ASSIGNMENT_APPROVAL_REQUIRED`, `SYSTEM_ROLE_ASSIGNMENT_SELF_APPROVAL_FORBIDDEN`, `BREAK_GLASS_DISABLED`, `BREAK_GLASS_APPROVAL_REQUIRED`, `BREAK_GLASS_EXPIRED`, `TEMPLATE_IMPACT_ANALYSIS_REQUIRED`.

No role — family or system — can grant server access to Family Keys, private keys, recovery shares, or family plaintext.
