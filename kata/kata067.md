# Kata 67 — DoS by JSON: limit body, depth, arrays

## Goal

1. Chặn JSON DoS theo **4 lớp**:

   * **Body size limit** (bytes)
   * **Parse time limit / abort** (deadline)
   * **Depth limit** (nesting)
   * **Array/Object fanout limit** (số phần tử / số keys)
2. Trả lỗi chuẩn `413/400` với reason code rõ ràng, không leak body.
3. Viết tests:

   * payload quá lớn
   * payload sâu bất thường
   * payload array cực lớn
   * payload có rất nhiều keys
   * payload “gần ngưỡng” vẫn pass
4. Observability:

   * counter theo reason: `json_rejected_total{reason=...}`
   * log sampling (không log body)

---

## Context

Endpoint:

* `POST /v1/events` nhận JSON event từ client.

Thực tế production:

* attacker gửi body 50MB, hoặc JSON sâu 20k levels, hoặc array 1M items → CPU/RAM spike, event loop lag, OOM.

Bạn phải làm đúng: **reject càng sớm càng tốt**.

---

## Constraints

* ❌ Không parse JSON bằng `JSON.parse(reqBodyString)` cho mọi thứ (vì bạn phải giới hạn sớm).
* ✅ Phải có body size limit ở mức bytes **trước khi parse**.
* ✅ Sau parse, phải validate structure: depth + array length + object keys.
* ✅ Có `AbortController`/deadline cho request processing (parse/validate).
* ✅ Không log raw body.
* ✅ Có test chứng minh các guard hoạt động.

---

## Deliverables

Tạo `kata-67/`:

* `src/json/intake.ts` (read body with limit + timeout)
* `src/json/validate.ts` (depth/fanout checks)
* `src/errors.ts` (problem+json style)
* `src/metrics.ts` (counter mock)
* `src/server.ts`
* `test/kata67.size.test.ts`
* `test/kata67.depth.test.ts`
* `test/kata67.fanout.test.ts`
* `test/kata67.near-limit.test.ts`

Tooling: `vitest`, `supertest` (nếu express) hoặc node:http + fetch.

---

## Spec: Limits (bạn có thể giữ đúng số này)

* `MAX_BODY_BYTES = 64 * 1024` (64KB)
* `MAX_DEPTH = 20`
* `MAX_ARRAY_LENGTH = 1_000`
* `MAX_OBJECT_KEYS = 200`
* `MAX_TOTAL_NODES = 10_000` *(để chống “wide + deep” tổng hợp)*
* `MAX_PARSE_MS = 50` *(kata: mô phỏng; production có thể khác)*

### Error mapping

* body too large → `413` reason `BODY_TOO_LARGE`
* invalid json → `400` reason `INVALID_JSON`
* depth too deep → `400` reason `JSON_TOO_DEEP`
* array too large → `400` reason `ARRAY_TOO_LARGE`
* too many keys → `400` reason `OBJECT_TOO_MANY_KEYS`
* total nodes exceeded → `400` reason `JSON_TOO_COMPLEX`
* timeout → `408` (hoặc 400) reason `JSON_PARSE_TIMEOUT`

---

## Starter skeleton (điền TODO)

### `src/errors.ts`

```ts
export type Problem = {
  type: string;
  title: string;
  status: number;
  detail: string;
  reason:
    | "BODY_TOO_LARGE"
    | "INVALID_JSON"
    | "JSON_TOO_DEEP"
    | "ARRAY_TOO_LARGE"
    | "OBJECT_TOO_MANY_KEYS"
    | "JSON_TOO_COMPLEX"
    | "JSON_PARSE_TIMEOUT";
};

export function problem(status: number, reason: Problem["reason"], detail: string): Problem {
  return {
    type: "https://errors.example.com/json",
    title: "Invalid request body",
    status,
    reason,
    detail,
  };
}
```

### `src/json/intake.ts`

```ts
import { problem } from "../errors";

export type IntakeLimits = {
  maxBytes: number;
  maxParseMs: number;
};

export async function readJsonBody(
  req: NodeJS.ReadableStream,
  limits: IntakeLimits,
  signal?: AbortSignal,
): Promise<{ ok: true; value: unknown } | { ok: false; status: number; body: any }> {
  // TODO:
  // 1) read chunks, count bytes, if > maxBytes => stop reading, return 413
  // 2) enforce deadline maxParseMs (AbortController / timer)
  // 3) parse JSON safely:
  //    - collect into Buffer then JSON.parse (ok due to size cap)
  // 4) on parse error => 400 INVALID_JSON
  // 5) never echo request body in error
  return { ok: false, status: 500, body: problem(500, "INVALID_JSON", "TODO") };
}
```

### `src/json/validate.ts`

```ts
import { problem } from "../errors";

export type ShapeLimits = {
  maxDepth: number;
  maxArrayLength: number;
  maxObjectKeys: number;
  maxTotalNodes: number;
};

export function validateJsonShape(value: unknown, limits: ShapeLimits):
  | { ok: true }
  | { ok: false; status: 400; body: any } {
  // TODO:
  // Traverse value (DFS) and compute:
  // - depth
  // - array length per node
  // - object keys per node
  // - total nodes visited
  //
  // Rules:
  // - primitives ok
  // - arrays: if length > maxArrayLength => ARRAY_TOO_LARGE
  // - objects: if keys > maxObjectKeys => OBJECT_TOO_MANY_KEYS
  // - if depth > maxDepth => JSON_TOO_DEEP
  // - if total nodes > maxTotalNodes => JSON_TOO_COMPLEX
  //
  return { ok: true };
}
```

### `src/metrics.ts`

```ts
type Reason =
  | "BODY_TOO_LARGE"
  | "INVALID_JSON"
  | "JSON_TOO_DEEP"
  | "ARRAY_TOO_LARGE"
  | "OBJECT_TOO_MANY_KEYS"
  | "JSON_TOO_COMPLEX"
  | "JSON_PARSE_TIMEOUT";

const counters = new Map<Reason, number>();

export function incRejected(reason: Reason) {
  counters.set(reason, (counters.get(reason) ?? 0) + 1);
}

export function snapshotCounters() {
  return Object.fromEntries(counters.entries());
}
```

### `src/server.ts` (minimal)

* `POST /v1/events`:

  1. `readJsonBody(req, limits, reqAbortSignal)`
  2. `validateJsonShape(value, shapeLimits)`
  3. if ok return 202

* Log:

  * `reason` + `content-length` + `request_id`
  * không log body

---

## Tests — bắt buộc

### 1) Size limit (`test/kata67.size.test.ts`)

* send JSON string > 64KB → expect 413 + reason `BODY_TOO_LARGE`
* send 63KB → pass parse stage (có thể fail shape tùy payload)

### 2) Depth limit (`test/kata67.depth.test.ts`)

* generate nested object depth=25:

  * `{a:{a:{a:...}}}`
* expect 400 reason `JSON_TOO_DEEP`

### 3) Fanout (`test/kata67.fanout.test.ts`)

* array length 2000 → `ARRAY_TOO_LARGE`
* object với 300 keys → `OBJECT_TOO_MANY_KEYS`

### 4) Total nodes (`test/kata67.near-limit.test.ts`)

* tạo payload vừa đủ depth/array/key nhưng tổng nodes > 10k → `JSON_TOO_COMPLEX`
* tạo payload “gần ngưỡng” nhưng <= 10k → pass

### 5) Timeout simulation (optional nhưng khuyến nghị)

* inject “slow parser” bằng cách thêm `await new Promise(r=>setTimeout(r, 60))` trong `readJsonBody` khi test mode
* expect reason `JSON_PARSE_TIMEOUT`

---

## Checks (Definition of Done)

Bạn pass kata nếu:

1. Body size limit chặn trước parse (413).
2. Invalid JSON trả 400 `INVALID_JSON`.
3. Depth/array/keys/total nodes đều bị chặn đúng reason code.
4. Không log body; logs chỉ có metadata.
5. Counters tăng đúng theo reason (test snapshot).
6. Payload hợp lệ gần ngưỡng vẫn pass.

---

## Stretch (Senior+ thực chiến)

1. **Streaming JSON parser** (hard mode)

   * Dùng streaming parser để fail sớm theo depth/array ngay khi đọc (không cần buffer).
2. **Adaptive limits per route**

   * `/v1/events` 64KB, `/v1/batch` 256KB nhưng stricter shape.
3. **Backpressure + slowloris defense**

   * limit read rate, set socket timeout, abort nếu body drip-feed.
4. **Cost-based model**

   * tính “cost” = nodes + string bytes + number of arrays; cap theo budget.
5. **Observability SLO**

   * alert nếu `json_rejected_total` tăng đột biến hoặc event loop lag tăng.