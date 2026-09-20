# Encrypted media

Base: **`/media/v1`** — media object transfer uses its own server base, not the core `/api/v1`. Media files, chunks, manifests, thumbnails, waveforms, posters, and renditions are created and encrypted on trusted clients. Object storage receives **ciphertext only**.

> **Implementation status: parked.** The end-to-end encrypted media path (contract → store → download → decrypt → play → cache) is not yet proven across the three repos. The media chain is parked at Gate 5F under the RUM-124 stage program and is **not** production-ready. Treat the shapes below as the pinned 1.3.0 contract surface, not a certification that E2E media works today. See [Implementation status](implementation-status.md).

## What changed in 1.3.0

The 1.3.0 media contract splits sizing and integrity metadata out of the create call and **removes `encrypted_payload` from the media upload flow entirely**. If you are porting from an older integration that posted an `encrypted_payload`, that field is no longer accepted by any media operation. Instead:

- **Create** declares the object binding, the encrypted manifest, and immutable ciphertext sizing/integrity metadata.
- **Chunks** are uploaded as raw binary ciphertext bodies with a per-chunk digest header.
- **Complete** re-states the manifest digest and the exact chunk-count and ciphertext-size expectations.

## Operations

| Method | Media path | Operation |
|---|---|---|
| POST | `/uploads` | `createMediaUpload` |
| GET / DELETE | `/uploads/{upload_id}` | `getMediaUpload`, `deleteMediaUpload` |
| PUT | `/uploads/{upload_id}/chunks/{chunk_index}` | `putMediaChunk` |
| POST | `/uploads/{upload_id}/complete` | `completeMediaUpload` |
| GET | `/media/{media_id}/manifest` | `getMediaManifest` |
| GET | `/media/{media_id}/content` | `getMediaContent` |
| DELETE | `/media/{media_id}` | `deleteMedia` |

## 1. Create the upload

`createMediaUpload` requires the opaque content object, the encrypted manifest, and immutable ciphertext sizing and integrity metadata. **`encrypted_payload` is not accepted.**

```bash
curl -sS -X POST https://media.rumpun.example/media/v1/uploads \
  -H 'Authorization: Bearer opaque_access_token' \
  -H 'Content-Type: application/json' \
  -H 'X-Request-Id: 01939f50-7c00-7000-8000-000000000001' \
  -H 'Idempotency-Key: 01939f50-7c00-7000-8000-000000000002' \
  -d '{
    "object_id": "obj_01J8RUMPUNEXAMPLE000001",
    "encrypted_manifest": "base64url_ciphertext_manifest",
    "total_chunks": 8,
    "ciphertext_size": 8388608,
    "chunk_size": 1048576,
    "ciphertext_sha256": "base64url_digest"
  }'
```

All six fields are required. Response:

```json
{
  "data": {
    "upload_id": "upload_01J8RUMPUNEXAMPLE001",
    "state": "UPLOADING",
    "received_chunks": 0,
    "total_chunks": 8,
    "expires_at": "2026-08-03T03:00:00Z"
  },
  "meta": {
    "request_id": "01939f50-7c00-7000-8000-000000000001",
    "server_time": "2026-08-02T03:00:00Z",
    "api_version": "1.1"
  }
}
```

## 2. Upload each chunk

`PUT /uploads/{upload_id}/chunks/{chunk_index}` sends a **binary ciphertext body** — there is no JSON request body. The required `X-Chunk-Ciphertext-SHA256` header carries the base64url SHA-256 digest of that binary body. The chunk index must be within manifest bounds (`0`–`999999`). A retry is idempotent only when the digest and byte count match the existing chunk.

```bash
curl -sS -X PUT "https://media.rumpun.example/media/v1/uploads/upload_01J8RUMPUNEXAMPLE001/chunks/0" \
  -H 'Authorization: Bearer opaque_access_token' \
  -H 'Content-Type: application/octet-stream' \
  -H 'X-Request-Id: 01939f50-7c00-7000-8000-000000000003' \
  -H 'X-Chunk-Ciphertext-SHA256: base64url_chunk_digest' \
  --data-binary @chunk-0.bin
```

The service checks upload ownership, expiry, chunk index, maximum size, digest, object binding, and duplicate consistency. It never validates media plaintext.

## 3. Complete the upload

`completeMediaUpload` requires the digest of the manifest supplied at creation plus the exact chunk-count and ciphertext-size expectations. It does **not** repeat `encrypted_manifest` and does **not** accept `encrypted_payload`.

```bash
curl -sS -X POST "https://media.rumpun.example/media/v1/uploads/upload_01J8RUMPUNEXAMPLE001/complete" \
  -H 'Authorization: Bearer opaque_access_token' \
  -H 'Content-Type: application/json' \
  -H 'X-Request-Id: 01939f50-7c00-7000-8000-000000000004' \
  -d '{
    "manifest_digest": "base64url_digest",
    "expected_chunks": 8,
    "expected_ciphertext_size": 8388608
  }'
```

Completion verifies all expected ciphertext chunks and the encrypted manifest structure before creating a media resource:

```json
{
  "data": {
    "media_id": "media_01J8RUMPUNEXAMPLE0001",
    "state": "READY",
    "total_chunks": 8,
    "ciphertext_size": 8388608
  },
  "meta": {
    "request_id": "01939f50-7c00-7000-8000-000000000001",
    "server_time": "2026-08-02T03:00:00Z",
    "api_version": "1.1"
  }
}
```

## Playback

`GET /media/{media_id}/content` supports HTTP `Range` (`bytes=<start>-<end>`) over ciphertext. Signed object-storage URLs are short-lived, audience-bound, method-bound, and never expose internal bucket paths.

## Rules

- Nonces and encryption parameters follow the canonical media protocol; the client owns all encryption.
- No server-side thumbnailing, transcription, OCR, face detection, or semantic analysis.
- Interrupted uploads support resume and cancellation; low-resource clients may pause and continue from an encrypted checkpoint.
- Private-vault media is not active in MVP.

## Stable errors

`UPLOAD_EXPIRED`, `UPLOAD_INCOMPLETE`, `CHUNK_INDEX_INVALID`, `CHUNK_DIGEST_MISMATCH`, `CHUNK_ALREADY_EXISTS_DIFFERENT`, `MANIFEST_INVALID`, `CIPHERTEXT_SIZE_MISMATCH`, `RANGE_NOT_SATISFIABLE`, `MEDIA_NOT_READY`, `RATE_LIMITED`.
