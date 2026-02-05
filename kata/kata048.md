# Kata 48 — Streaming Upload: upload file không buffer toàn bộ

## Goal

1. Implement endpoint **upload streaming** trong Node/TS: nhận file và **stream thẳng** ra disk/object-store mock, **không buffer toàn bộ** vào RAM.
2. Đúng backpressure: khi downstream write chậm, upstream phải tự “chậm lại” (không OOM).
3. Có **limits** production-grade: max size, timeouts, max in-flight uploads, content-type sniffing tối thiểu, checksum (streaming hash).
4. Có **resume-safe semantics** tối thiểu (idempotency key / temp file + atomic rename).
5. Có **tests** chứng minh: không dùng nhiều memory, không đọc hết vào buffer, và xử lý abort/timeout đúng.

---

## Context

Toy upload hay làm:

* `await req.arrayBuffer()` / `multer memoryStorage()` → RAM chết khi file lớn
* thiếu limits → DOS by upload
* không cleanup temp files → disk đầy
* không hỗ trợ abort → treo socket, leak file handles

Bạn làm kata này để có “upload sống được” ở production.

---

## Constraints

* ❌ Không dùng middleware upload “đóng gói” kiểu multer memory storage.
* ✅ Bạn được dùng:

  * Node `http` hoặc Fastify/Express (khuyên Fastify vì stream-friendly, nhưng core phải là stream)
  * Node streams: `pipeline`, `Transform`, `fs.createWriteStream`
  * (Optional) `busboy` để parse multipart streaming (stretch)
* ✅ Must-have:

  * **maxBytes** hard limit (cắt giữa stream)
  * **timeout** (overall / idle)
  * **AbortSignal** / client abort handling
  * **temp file + atomic rename** để tránh file partial
  * **streaming hash** (sha256) trong lúc ghi
* ✅ Tests phải deterministic (mock clock nếu dùng timeout) và có ít nhất 1 test upload file “to” (stream) size lớn (vài MB) nhưng không tăng heap quá nhiều.

---

## Deliverables

1. `src/server.ts` — HTTP server + endpoint upload
2. `src/uploadPipeline.ts` — pipeline streaming: limit → hash → sink
3. `src/storage.ts` — local storage adapter (write temp + rename)
4. `test/upload.test.ts` — integration tests (supertest hoặc fetch)
5. `README.md` — semantics + failure modes + hardening

---

## Bài toán cụ thể (spec)

### Endpoint

* `POST /upload`
* **Mode core (dễ test, ít phụ thuộc):** body là raw bytes (`application/octet-stream`)

  * Header bắt buộc: `x-file-name`
  * Header optional: `x-idempotency-key`
* **Stretch:** hỗ trợ `multipart/form-data` với 1 field `file` bằng busboy (streaming).

### Output

* Trả JSON:

  * `fileId`, `size`, `sha256`, `storedPath` (hoặc key)
* Nếu reject:

  * 413 nếu vượt maxBytes
  * 408/504 nếu timeout
  * 499-ish (custom) nếu client abort
  * 409 nếu idempotency key conflict (tuỳ policy)

---

## Core architecture (đúng streaming)

Request stream → `ByteLimitTransform` → `HashTransform` → `StorageSink(fs.WriteStream)`
Dùng `pipeline()` để:

* tự propagate errors
* tự handle backpressure đúng cách

---

## Starter skeleton (nhìn vào làm)

### `src/uploadPipeline.ts`

```ts
import { Transform, Writable } from "node:stream";
import { createHash } from "node:crypto";

export class MaxBytesExceededError extends Error { name = "MaxBytesExceededError"; }
export class ClientAbortedError extends Error { name = "ClientAbortedError"; }

export class ByteLimitTransform extends Transform {
  private seen = 0;
  constructor(private maxBytes: number) {
    super();
  }

  _transform(chunk: any, _enc: BufferEncoding, cb: (err?: Error | null, data?: any) => void) {
    this.seen += chunk.length;
    if (this.seen > this.maxBytes) {
      cb(new MaxBytesExceededError(`Max bytes exceeded: ${this.maxBytes}`));
      return;
    }
    cb(null, chunk);
  }

  bytesSeen() { return this.seen; }
}

export class Sha256Transform extends Transform {
  private h = createHash("sha256");
  private size = 0;

  _transform(chunk: any, _enc: BufferEncoding, cb: (err?: Error | null, data?: any) => void) {
    this.h.update(chunk);
    this.size += chunk.length;
    cb(null, chunk);
  }

  digestHex() { return this.h.digest("hex"); }
  bytesSeen() { return this.size; }
}
```

### `src/storage.ts` (temp + atomic rename)

```ts
import { createWriteStream, promises as fsp } from "node:fs";
import { dirname, join } from "node:path";
import { randomUUID } from "node:crypto";
import { Writable } from "node:stream";

export type StoredFile = {
  fileId: string;
  finalPath: string;
  tmpPath: string;
};

export class LocalStorage {
  constructor(private baseDir: string) {}

  async prepare(fileName: string): Promise<StoredFile & { sink: Writable }> {
    const fileId = randomUUID();
    const safeName = fileName.replace(/[^a-zA-Z0-9._-]/g, "_");
    const finalPath = join(this.baseDir, `${fileId}_${safeName}`);
    const tmpPath = finalPath + ".tmp";

    await fsp.mkdir(dirname(finalPath), { recursive: true });

    const sink = createWriteStream(tmpPath, { flags: "wx" }); // fail if exists
    return { fileId, finalPath, tmpPath, sink };
  }

  async commit(tmpPath: string, finalPath: string) {
    // atomic on same filesystem
    await fsp.rename(tmpPath, finalPath);
  }

  async abort(tmpPath: string) {
    // best effort cleanup
    await fsp.rm(tmpPath, { force: true });
  }
}
```

### `src/server.ts` (raw streaming upload)

```ts
import http from "node:http";
import { pipeline } from "node:stream/promises";
import { ByteLimitTransform, Sha256Transform, MaxBytesExceededError, ClientAbortedError } from "./uploadPipeline";
import { LocalStorage } from "./storage";

const MAX_BYTES = 20 * 1024 * 1024; // 20MB (kata)
const REQ_TIMEOUT_MS = 30_000;

const storage = new LocalStorage("./uploads");

function json(res: http.ServerResponse, status: number, obj: any) {
  const body = Buffer.from(JSON.stringify(obj));
  res.writeHead(status, { "content-type": "application/json", "content-length": body.length });
  res.end(body);
}

export function createServer() {
  return http.createServer(async (req, res) => {
    if (req.method === "POST" && req.url === "/upload") {
      const fileName = req.headers["x-file-name"];
      if (!fileName || Array.isArray(fileName)) return json(res, 400, { error: "x-file-name required" });

      // basic content-type check (optional strict)
      const ct = req.headers["content-type"] ?? "application/octet-stream";
      if (typeof ct !== "string") return json(res, 400, { error: "bad content-type" });

      // timeout (overall)
      req.setTimeout(REQ_TIMEOUT_MS, () => {
        req.destroy(new Error("RequestTimeout"));
      });

      const limiter = new ByteLimitTransform(MAX_BYTES);
      const hasher = new Sha256Transform();

      const prepared = await storage.prepare(fileName);

      const onAborted = () => {
        // node emits 'aborted' on IncomingMessage in some cases; also 'close'
        prepared.sink.destroy(new ClientAbortedError("client aborted"));
      };

      req.on("aborted", onAborted);
      req.on("close", () => {
        // if closed early and not ended, treat as abort
        if (!req.readableEnded) prepared.sink.destroy(new ClientAbortedError("client closed"));
      });

      try {
        await pipeline(req, limiter, hasher, prepared.sink);

        const size = hasher.bytesSeen();
        const sha256 = hasher.digestHex();

        await storage.commit(prepared.tmpPath, prepared.finalPath);

        return json(res, 201, {
          fileId: prepared.fileId,
          size,
          sha256,
          storedPath: prepared.finalPath,
        });
      } catch (e) {
        // cleanup tmp
        await storage.abort(prepared.tmpPath);

        const err = e instanceof Error ? e : new Error(String(e));

        if (err instanceof MaxBytesExceededError) return json(res, 413, { error: "too_large" });
        if (err.name === "RequestTimeout") return json(res, 408, { error: "timeout" });
        if (err instanceof ClientAbortedError || err.name === "ClientAbortedError") return json(res, 499, { error: "client_aborted" });

        return json(res, 500, { error: "internal", detail: err.name });
      } finally {
        req.off("aborted", onAborted);
      }
    }

    json(res, 404, { error: "not_found" });
  });
}

if (require.main === module) {
  const s = createServer();
  s.listen(3000, () => console.log("listening on :3000"));
}
```

> Core done: streaming, limit, hash, temp+rename, timeout, abort cleanup.

---

## Test plan bắt buộc (integration + correctness)

### Test 1 — Upload success, file exists, hash matches

* Start server on random port
* Send `POST /upload` với body 2MB random bytes stream
* Expect 201, returned `size==bytes`, sha256 đúng (tính lại ở test), file tồn tại

### Test 2 — Exceed MAX_BYTES returns 413 and no final file

* Send body > MAX_BYTES (stream)
* Expect 413
* Assert: không có final file, tmp cleaned (glob `.tmp` empty)

### Test 3 — Backpressure sanity (write slow)

* Thay storage sink bằng Writable chậm (delay mỗi chunk)
* Upload stream lớn
* Expect request không OOM, completes eventually
* Assert: `process.memoryUsage().heapUsed` không tăng vượt ngưỡng (đặt margin hợp lý, ví dụ < +30MB cho file vài chục MB)

### Test 4 — Client abort cleans tmp

* Bắt đầu upload, viết một ít rồi abort socket
* Expect server trả 499 hoặc connection closed
* Assert tmp file bị rm

### Test 5 — Timeout triggers cleanup

* Set REQ_TIMEOUT_MS nhỏ (vd 50ms), làm sink chậm
* Expect 408, tmp cleaned

> Dùng `node:stream` để tạo Readable từ generator thay vì Buffer lớn để tránh test tự buffer.

---

## Checks (Definition of Done)

* Không có chỗ nào đọc toàn bộ body vào memory (`arrayBuffer`, `Buffer.concat` toàn file…).
* `pipeline()` được dùng để đảm bảo backpressure.
* MaxBytes enforced giữa stream (413).
* Abort/timeout không để lại tmp file.
* Hash được tính streaming, size đúng.
* Có ít nhất 1 test “file vài MB” mà heap không tăng bậy.

---

## README bắt buộc (10 dòng, đúng chỗ đau)

1. Vì sao upload buffering giết RAM
2. Dataflow pipeline (req → limit → hash → sink)
3. Backpressure hoạt động thế nào trong pipeline
4. Size limit và vì sao enforce ở stream, không dựa Content-Length
5. Temp file + atomic rename để tránh partial file
6. Abort/timeout cleanup để tránh disk leak
7. Security basics: filename sanitize, content-type minimal, max size
8. Observability: log bytes/sha, duration, abort reason
9. Failure modes: slowloris, partial writes, disk full
10. Stretch: multipart streaming + resumable + AV scan hook

---

## Stretch (Senior+ thật)

Chọn 1:

1. **Multipart streaming** bằng `busboy` (không buffer), support field metadata.
2. **Idempotency**: `x-idempotency-key` → nếu key đã commit thì return same result; nếu đang upload thì 409/202.
3. **Content sniff**: đọc 512 bytes đầu bằng Transform để detect mime (không tin header) + allowlist.
4. **Object storage adapter**: stream sang “S3 mock” (Writable) + multipart upload simulation.
5. **Rate-limit / max in-flight uploads**: integrate Bulkhead (44) hoặc limiter (41) theo route `/upload`.