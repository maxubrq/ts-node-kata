# Kata 69 — File Upload Security: sniff MIME, limit size, safe store

## Goal

1. Implement endpoint `POST /v1/uploads` nhận **multipart/form-data** (1 file) hoặc raw binary (tuỳ bạn chọn).
2. Enforce security controls:

   * **Max size (bytes)**: reject sớm trong stream (413)
   * **Allowed types**: xác định bằng **magic bytes** (sniff), không tin `Content-Type`
   * **Disallow dangerous types** (exe/script/html/svg nếu bạn chọn)
   * **Filename safety**: không dùng filename user để lưu (tránh traversal/overwrite)
   * **Store strategy**: content-addressed hoặc random ID, thư mục phân tán
   * **Quarantine flow** (optional): lưu tạm, scan hook, rồi promote
3. Return response có:

   * `uploadId`
   * `detectedMime`
   * `size`
   * `sha256`
4. Tests cover:

   * size over limit
   * spoofed content-type
   * polyglot-ish simple case
   * path traversal filename
   * duplicate upload (same sha) → dedupe

---

## Context

User upload avatar / receipt. Attacker cố upload:

* file cực lớn → OOM / disk fill
* file “.png” nhưng thực chất là HTML/script
* filename `../../etc/passwd`
* file có bytes gây crash sniff/parser

Bạn build pipeline “boring nhưng đúng”.

---

## Constraints

* ✅ Không buffer toàn bộ file trước khi check size (phải stream + count).
* ✅ MIME phải dựa trên **sniff** (magic bytes) ít nhất vài loại phổ biến.
* ✅ Không lưu theo filename gốc.
* ✅ Không serve file trực tiếp từ upload dir như static web root (chỉ note trong README).
* ✅ Có atomic write (write temp → rename).
* ✅ Có unit/integration tests.
* ❌ Không log raw file content.

---

## Deliverables

Tạo `kata-69/`:

* `src/upload/limits.ts`
* `src/upload/sniff.ts`
* `src/upload/stream.ts` (size limit + hashing)
* `src/upload/store.ts` (atomic write + dedupe)
* `src/upload/policy.ts`
* `src/server.ts`
* `test/kata69.size.test.ts`
* `test/kata69.mime.test.ts`
* `test/kata69.store.test.ts`
* `test/kata69.security.test.ts`
* `README.md`

Tooling gợi ý: `fastify` (upload tốt) hoặc `express + busboy`. Test: `vitest` + `supertest`.

---

## Spec: Limits & Policy

### Limits

* `MAX_BYTES = 5 * 1024 * 1024` (5MB)
* `MAX_FILES = 1`
* `SNIFF_BYTES = 4096` (đọc tối đa 4KB đầu để sniff)

### Allowed types (ví dụ)

* `image/png`
* `image/jpeg`
* `application/pdf`

### Blocked

* `text/html`, `image/svg+xml`, `application/x-msdownload`, `application/x-sh`, unknown

### Storage

* Base dir: `./data/uploads`
* Use **content-addressed**:

  * compute `sha256` while streaming
  * final path: `data/uploads/sha256/<first2>/<sha256>`
* Metadata JSON sidecar:

  * `.../<sha256>.json` chứa `{mime,size,uploadedAt,originalName}` (originalName sanitized)

### Response

```json
{
  "uploadId": "<sha256>",
  "size": 12345,
  "detectedMime": "image/png",
  "sha256": "<hex>",
  "deduped": true|false
}
```

---

## Starter skeleton (điền TODO)

### `src/upload/sniff.ts`

```ts
export type Detected =
  | { ok: true; mime: "image/png" | "image/jpeg" | "application/pdf" }
  | { ok: false; mime: "unknown" };

export function sniffMime(buf: Buffer): Detected {
  // TODO: magic bytes
  // PNG: 89 50 4E 47 0D 0A 1A 0A
  // JPEG: FF D8 FF
  // PDF: 25 50 44 46 2D ("%PDF-")
  return { ok: false, mime: "unknown" };
}
```

### `src/upload/stream.ts`

```ts
import crypto from "node:crypto";
import { sniffMime } from "./sniff";

export type StreamResult =
  | { ok: true; sha256: string; size: number; sniffedMime: string; tempPath: string }
  | { ok: false; status: number; reason: "BODY_TOO_LARGE" | "SNIFF_FAILED" | "STREAM_ERROR" };

export async function consumeFileStream(
  stream: NodeJS.ReadableStream,
  opts: { maxBytes: number; sniffBytes: number; tempDir: string },
): Promise<StreamResult> {
  // TODO:
  // - stream bytes to temp file (write stream)
  // - count bytes; if exceed maxBytes => abort + delete temp + 413
  // - keep first sniffBytes in memory for sniffMime
  // - compute sha256 incrementally (hash.update(chunk))
  // - return sha256 hex, size, sniffedMime, tempPath
  return { ok: false, status: 500, reason: "STREAM_ERROR" };
}
```

### `src/upload/policy.ts`

```ts
export type UploadPolicy = {
  allowedMimes: readonly string[];
};

export function enforceMime(policy: UploadPolicy, detectedMime: string): { ok: true } | { ok: false; status: 415; reason: "UNSUPPORTED_MEDIA_TYPE" } {
  // TODO
  return { ok: false, status: 415, reason: "UNSUPPORTED_MEDIA_TYPE" };
}
```

### `src/upload/store.ts`

```ts
import fs from "node:fs/promises";
import path from "node:path";

export type StoreResult =
  | { ok: true; uploadId: string; finalPath: string; deduped: boolean }
  | { ok: false; status: 500; reason: "STORE_FAILED" };

export async function storeBySha256(
  tempPath: string,
  sha256: string,
  baseDir: string,
): Promise<StoreResult> {
  // TODO:
  // - finalDir = baseDir/sha256/aa (first2)
  // - finalPath = .../<sha256>
  // - if exists => delete temp, deduped=true
  // - else atomic move: ensure dir, rename temp->final
  // - return uploadId=sha256
  return { ok: false, status: 500, reason: "STORE_FAILED" };
}

export async function writeMetadata(finalPath: string, meta: any): Promise<void> {
  // TODO: write JSON sidecar atomically too
}
```

---

## Server wiring (minimal)

`POST /v1/uploads`:

1. Parse multipart, lấy file stream (1 file). (fastify-multipart hoặc busboy)
2. `consumeFileStream(...)`
3. `sniffMime` từ bytes đầu → `enforceMime`
4. `storeBySha256(...)`
5. `writeMetadata(...)`
6. Return response JSON

**Quan trọng**:

* Không dùng `originalFilename` làm path.
* Log chỉ: `uploadId`, `size`, `mime`, `deduped`, `request_id`.

---

## Tests — bắt buộc

### `test/kata69.size.test.ts`

* upload file > 5MB → 413 `BODY_TOO_LARGE`
* upload file nhỏ vừa đủ → ok

### `test/kata69.mime.test.ts`

* gửi header `Content-Type: image/png` nhưng body bắt đầu bằng `<html>` → 415
* body có PNG magic bytes → ok
* PDF `%PDF-` → ok
* unknown random bytes → 415

### `test/kata69.security.test.ts`

* filename: `../../evil.png` → vẫn lưu an toàn (không traverse) và metadata `originalName` được sanitize (no slashes)
* confirm server không trả file raw path ngoài base dir

### `test/kata69.store.test.ts`

* upload cùng nội dung 2 lần:

  * lần 1 `deduped=false`
  * lần 2 `deduped=true`
* verify final file tồn tại đúng path theo sha prefix

---

## Checks (Definition of Done)

Bạn pass kata nếu:

1. Size limit enforced trong stream (không OOM, không buffer full).
2. MIME sniff dựa magic bytes, chặn spoofed `Content-Type`.
3. Store dùng sha hoặc random ID, không dùng filename user.
4. Atomic write + dedupe.
5. Metadata sidecar ghi an toàn.
6. Tests cover size + mime spoof + traversal + dedupe.

---

## Stretch (Senior+ “upload pipeline thật”)

1. **Quarantine & promote**

   * lưu vào `quarantine/` trước
   * hook `scanFile(tempPath)` (mock) trả clean/dirty
   * clean → move to final; dirty → delete + audit log
2. **Zip bomb / decompression bomb awareness**

   * nếu allow zip, phải limit extracted size (chưa làm ở kata này).
3. **Image re-encode**

   * với avatar: decode + re-encode (strip metadata) để phá polyglot.
4. **Content-Disposition hardening**

   * Khi download, set `Content-Disposition: attachment; filename="..."` safe + `X-Content-Type-Options: nosniff`.
5. **Rate limiting per user / IP**

   * kết hợp kata 24/46.
6. **Storage abstraction**

   * interface store local/S3; với S3 dùng presigned URL + server-side validation.