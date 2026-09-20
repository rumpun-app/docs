# Tree, content, and family data

Base: `/api/v1`. Every payload of family meaning is ciphertext; the server stores minimum pseudonymous topology and approved operational metadata only.

## Tree and kinship

The server stores only the minimum pseudonymous topology needed for authorized traversal and integrity. Names, biographies, private labels, and narratives stay encrypted.

| Method | Path | Operation |
|---|---|---|
| GET / POST | `/families/{family_id}/people` | `listPeople`, `createPerson` |
| GET / PATCH / DELETE | `/families/{family_id}/people/{person_id}` | `getPerson`, `updatePerson`, `deletePerson` |
| POST | `/families/{family_id}/partner-unions` | `createPartnerUnion` |
| PATCH / DELETE | `/families/{family_id}/partner-unions/{union_id}` | `updatePartnerUnion`, `deletePartnerUnion` |
| POST | `/families/{family_id}/parent-child-relations` | `createParentChildRelation` |
| PATCH / DELETE | `/families/{family_id}/parent-child-relations/{relation_id}` | `updateParentChildRelation`, `deleteParentChildRelation` |
| GET | `/families/{family_id}/tree` | `getTreeProjection` |
| POST | `/families/{family_id}/kinship/resolve` | `resolveKinship` |
| POST | `/families/{family_id}/tree/validate` | `validateTreeMutation` |

Person create:

```json
{
  "encrypted_payload": "base64url_ciphertext_profile",
  "device_id": "dev_01J8RUMPUNEXAMPLE000001",
  "expected_version": 1
}
```

Partner unions support historical, concurrent, ended, unknown, and disputed states — there is no universal "primary spouse". Parent-child relations support `BIOLOGICAL`, `ADOPTIVE`, `FOSTER`, `GUARDIAN`, `CAREGIVER`, `STEP`, `UNKNOWN`. Projections declare `partial` and `coverage` (e.g. `AUTHORIZED_SUBSET`); hidden edges are never exposed through path details, timing, counts, or error differences. Mutations require `If-Match` where an aggregate exists.

Stable errors: `STRUCTURAL_CONFLICT`, `HIDDEN_RELATION`, `RELATION_TYPE_INVALID`, `RELATION_PERIOD_INVALID`, `KINSHIP_LENS_NOT_AVAILABLE`, `PROJECTION_PARTIAL`, `CAPABILITY_DENIED`, `AGGREGATE_VERSION_STALE`.

## Content objects

One encrypted container model for family stories and related representations. Canonical types: `story`, `media`, `note`, `message`, `tribute`, `capsule`, `source_record`. Facets such as `recipe`, `travel`, or `tradition` are encrypted topic data, never object types.

| Method | Path | Operation |
|---|---|---|
| GET / POST | `/families/{family_id}/objects` | `listContentObjects`, `createContentObject` |
| GET / PATCH / DELETE | `/families/{family_id}/objects/{object_id}` | `getContentObject`, `updateContentObject`, `deleteContentObject` |
| POST | `/families/{family_id}/objects/{object_id}/representations` | `addRepresentation` |
| DELETE | `/families/{family_id}/objects/{object_id}/representations/{representation_id}` | `deleteRepresentation` |

Create example (note the client generates a random content key per object and the server never sees plaintext title, narrative, topic, transcript, caption, or content key):

```json
{
  "type": "story",
  "encrypted_payload": {
    "ciphertext_ref": "blob_ciphertext_example",
    "cipher": "AES-256-GCM",
    "wrapped_content_key": "base64url_wrapped_key_example",
    "key_scope": "FAMILY",
    "key_version": 3,
    "nonce": "base64url_nonce_example",
    "aad_version": 1,
    "ciphertext_sha256": "base64url_digest_example"
  },
  "context_refs": { "people": ["person_01J8RUMPUNEXAMPLE0001"], "events": [], "places": [] },
  "topics_ciphertext": "base64url_ciphertext_topics",
  "provenance_ciphertext": "base64url_ciphertext_provenance",
  "visibility_scope": "FAMILY",
  "policy_version": 1
}
```

Existing mutations require `If-Match`. Visibility and derived data never become broader than their sources.

Stable errors: `CONTENT_TYPE_NOT_ALLOWED`, `PLAINTEXT_FIELD_PROHIBITED`, `VISIBILITY_SCOPE_INVALID`, `KEY_SCOPE_MISMATCH`, `KEY_VERSION_INVALID`, `CONTENT_POLICY_REJECTED`, `REPRESENTATION_KIND_INVALID`, `CAPABILITY_DENIED`, `AGGREGATE_VERSION_STALE`.

## Voice Stories (Cerita Suara)

A product experience for a canonical `story` with encrypted audio and optional encrypted transcript representations. It is not a separate content-object type. A session references one completed Media ciphertext object in the same family.

| Method | Path | Operation |
|---|---|---|
| GET / POST | `/families/{family_id}/voice-stories` | `listVoiceStories`, `createVoiceStory` |
| GET / PATCH | `/families/{family_id}/voice-stories/{voice_story_id}` | `getVoiceStory`, `updateVoiceStory` |
| GET / POST | `/families/{family_id}/voice-stories/{voice_story_id}/sessions` | `listVoiceStorySessions`, `createVoiceStorySession` |
| GET / PATCH | `/families/{family_id}/voice-stories/{voice_story_id}/sessions/{session_id}` | `getVoiceStorySession`, `updateVoiceStorySession` |
| POST | `/families/{family_id}/voice-stories/{voice_story_id}/publish` | `publishVoiceStory` |
| POST | `/families/{family_id}/voice-stories/{voice_story_id}/archive` | `archiveVoiceStory` |
| GET | `/families/{family_id}/voice-stories/{voice_story_id}/playback` | `getVoiceStoryPlayback` |

Lifecycle: `DRAFT`, `RECORDING`, `READY_FOR_REVIEW`, `PUBLISHED_WITHIN_FAMILY`, `ARCHIVED`, `REDACTED`, `DELETED_PENDING`. Recording requires narrator consent; publishing requires active consent and at least one complete session. No cloud transcription in MVP. Playback returns authorized metadata and Media references; ciphertext bytes are served by the Media Range contract.

Stable errors: `CONSENT_REQUIRED`, `CONSENT_WITHDRAWN`, `MEDIA_NOT_READY`, `PLAINTEXT_FIELD_PROHIBITED`, `RESOURCE_TENANCY_MISMATCH`, `VOICE_STORY_STATE_INVALID`, `VOICE_STORY_SESSION_ORDER_INVALID`, `AUTHORIZATION_CONTEXT_CHANGED`, `AUTHORIZATION_EPOCH_STALE`.

## Consent

Family relationship is never consent. Consent is purpose-, audience-, and representation-specific, versioned, withdrawable, and evidenced without exposing plaintext to the service.

| Method | Path | Operation |
|---|---|---|
| POST | `/families/{family_id}/consents` | `recordConsent` |
| GET | `/families/{family_id}/consents/{consent_id}` | `getConsent` |
| POST | `/families/{family_id}/consents/{consent_id}/withdraw` | `withdrawConsent` |

The service stores an encrypted evidence reference, minimum receipt metadata, policy version, and timestamp — never the oral consent plaintext. A narrator without an account can be issued a privacy-preserving withdrawal token.

Stable errors: `CONSENT_REQUIRED`, `CONSENT_WITHDRAWN`, `CONSENT_SCOPE_MISMATCH`, `CONSENT_VERSION_UNSUPPORTED`, `WITHDRAWAL_TOKEN_INVALID`, `SUBJECT_AUTHORITY_REQUIRED`, `PROHIBITED_PROCESSING`, `AGGREGATE_VERSION_STALE`.

## Biometric deletion and redaction

A voice, face, or other biometric subject can request deletion or redaction. Subject rights are not blocked by family-owner quorum.

| Method | Path | Operation |
|---|---|---|
| POST | `/biometric-deletion/requests` | `requestBiometricDeletion` |
| GET | `/biometric-deletion/requests/{deletion_id}` | `getBiometricDeletion` |
| POST | `/biometric-deletion/requests/{deletion_id}/complete` | `completeBiometricDeletion` |

Requests accept either a bearer session or an approved withdrawal token where the OpenAPI security alternatives allow it. Lifecycle: `REQUESTED → VALIDATING_SCOPE → LOCAL_REDACTION_REQUIRED → PURGE_PENDING → COMPLETED`. Completion must cover derivatives (renditions, transcripts, thumbnails, waveforms, indexes, caches, future envelopes), not just the original file.

Stable errors: `WITHDRAWAL_TOKEN_INVALID`, `SUBJECT_SCOPE_INVALID`, `DELETION_ALREADY_COMPLETED`, `REDACTION_REQUIRED`, `DERIVATIVE_PURGE_INCOMPLETE`, `OBJECT_NOT_ACCESSIBLE`, `SIGNATURE_INVALID`, `RATE_LIMITED`.

## GEDCOM import

All plaintext GEDCOM parsing, encoding repair, DNA-extension rejection, duplicate detection, and privacy review happen locally. The service accepts encrypted staged batches and approved pseudonymous topology mutations only. **There is no plaintext `.ged` upload endpoint.**

| Method | Path | Operation |
|---|---|---|
| POST | `/families/{family_id}/imports` | `createImport` |
| GET | `/families/{family_id}/imports/{import_id}` | `getImport` |
| POST | `/families/{family_id}/imports/{import_id}/batches` | `addImportBatch` |
| POST | `/families/{family_id}/imports/{import_id}/checkpoint` | `saveImportCheckpoint` |
| POST | `/families/{family_id}/imports/{import_id}/activate` | `activateImport` |
| POST | `/families/{family_id}/imports/{import_id}/cancel` | `cancelImport` |

Activation requires the expected family graph version; a changed graph returns `409 GRAPH_VERSION_CONFLICT`, preserves encrypted staging, and requires local re-review. Import-only in MVP; export is Post-MVP.

Stable errors: `PLAINTEXT_GEDCOM_PROHIBITED`, `DNA_EXTENSION_REJECTED`, `GEDCOM_IMPORT_RESOURCE_LIMIT`, `IMPORT_BATCH_OUT_OF_ORDER`, `IMPORT_CHECKPOINT_INVALID`, `IMPORT_BATCH_DIGEST_MISMATCH`, `GRAPH_VERSION_CONFLICT`, `STRUCTURAL_CONFLICT`, `IMPORT_CANCELLED`, `RATE_LIMITED`.

## Synchronization

Sync exchanges authorized ciphertext references, versions, tombstones, and scope versions. The encrypted local database is never mirrored as server-readable content.

| Method | Path | Operation |
|---|---|---|
| GET | `/sync/changes` | `listSyncChanges` |
| POST | `/sync/push` | `pushSyncChanges` |
| POST | `/sync/acknowledge` | `acknowledgeSyncChanges` |
| GET | `/sync/bootstrap` | `bootstrapSync` |

A change record carries `resource_type`, `resource_id`, `aggregate_version`, `operation`, `ciphertext_ref`, `scope_version`, and `tombstone` — never plaintext title, name, topic, query, or transcript. Clients tolerate duplicate delivery, retries, interruption, and reordering within documented bounds. Bootstrap is a bounded minimum projection, not a plaintext dump.

Stable errors: `SYNC_CURSOR_INVALID`, `SYNC_CURSOR_EXPIRED`, `AGGREGATE_VERSION_STALE`, `SCOPE_VERSION_STALE`, `CIPHERTEXT_REF_INVALID`, `CHANGE_BATCH_TOO_LARGE`, `TOMBSTONE_CONFLICT`, `BOOTSTRAP_PROJECTION_INVALID`, `RATE_LIMITED`.

## Notifications

In-app notification history and transactional email are the active MVP channels. Push, SMS, WhatsApp Business, and marketing are outside MVP.

| Method | Path | Operation |
|---|---|---|
| GET | `/notifications` | `listNotifications` |
| GET | `/notifications/unread-count` | `getNotificationUnreadCount` |
| POST | `/notifications/{notification_id}/read` | `markNotificationRead` |
| POST | `/notifications/read-all` | `markAllNotificationsRead` |
| POST | `/notifications/{notification_id}/archive` | `archiveNotification` |
| GET | `/notification-preferences` | `getNotificationPreferences` |
| PATCH | `/notification-preferences` | `updateNotificationPreferences` |

Notification possession is never authorization; `action_code` comes from a closed allowlist and `target_ref` is opaque, and target modules re-authorize every action. Family names, story titles, transcripts, filenames, media content, key material, tokens, signed URLs, and stack traces are prohibited in payloads. Security-critical notifications cannot be disabled.

Stable errors: `NOTIFICATION_NOT_FOUND`, `NOTIFICATION_OWNERSHIP_MISMATCH`, `NOTIFICATION_ACTION_NOT_ALLOWED`, `NOTIFICATION_SECURITY_PREFERENCE_IMMUTABLE`, `AUTHORIZATION_CONTEXT_CHANGED`, `AUTHORIZATION_EPOCH_STALE`, `RESOURCE_TENANCY_MISMATCH`.

## Family deletion

Family deletion is an irreversible, quorum-controlled lifecycle with a cooling-off period. It is not ordinary CRUD and is distinct from leaving a family or deleting one contribution.

| Method | Path | Operation |
|---|---|---|
| POST | `/families/{family_id}/deletion-requests` | `createFamilyDeletionRequest` |
| GET | `/families/{family_id}/deletion-requests/{deletion_id}` | `getFamilyDeletionRequest` |
| POST | `/families/{family_id}/deletion-requests/{deletion_id}/approvals` | `approveFamilyDeletion` |
| POST | `/families/{family_id}/deletion-requests/{deletion_id}/reject` | `rejectFamilyDeletion` |
| POST | `/families/{family_id}/deletion-requests/{deletion_id}/cancel` | `cancelFamilyDeletion` |
| GET | `/families/{family_id}/deletion-requests/{deletion_id}/purge-status` | `getFamilyPurgeStatus` |

State model: `REQUESTED → AWAITING_QUORUM → COOLING_OFF → PURGE_SCHEDULED → PURGING → COMPLETED`, plus `REJECTED`, `CANCELLED`, `LEGAL_HOLD_RESTRICTED`, `FAILED_SAFE`. Create requires `family.delete.request` and recent strong assurance. Destructive confirmation is mandatory in the UI.

Stable errors: `DELETION_QUORUM_NOT_MET`, `DELETION_COOLING_OFF`, `DELETION_ALREADY_COMPLETED`, `DELETION_CANCELLED`, `DELETION_REJECTED`, `LEGAL_HOLD_RESTRICTED`, `PURGE_INCOMPLETE`, `CAPABILITY_DENIED`, `AGGREGATE_VERSION_STALE`.
