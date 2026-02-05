# Kata 70 — Security Logging: Minimal Viable Audit Trail

## Goal

1. Thiết kế **audit event schema** chuẩn hoá (JSON) cho security-relevant actions.
2. Implement `audit.log(event)`:

   * luôn có `request_id`, `trace_id` (nếu có), `actor`, `tenant`, `action`, `target`, `outcome`, `reason`
   * **redact** secrets/PII nhạy cảm
   * chống **log forging** (strip control chars)
3. Persist audit events theo strategy tối thiểu:

   * **append-only** file hoặc SQLite (kata-friendly)
   * rotate file theo ngày/size
4. Expose truy vấn tối thiểu (admin-only):

   * `GET /v1/audit?tenantId=...&action=...&from=...&to=...`
5. Tests:

   * event schema validation
   * redaction
   * tamper-evident chain (stretch)
   * “incident reconstruction” test: from logs, reconstruct a timeline.

---

## Context

Bạn cần audit trail cho các action:

* login success/fail
* token refresh/revoke
* password change
* role change / permission change
* refund order (money)
* config change (feature flag, webhook URL)
* suspicious request blocked (SSRF blocked, CSRF denied)

**Mục tiêu**: khi có sự cố, bạn trả lời được:

* User nào/actor nào đã làm gì?
* Có bao nhiêu lần fail?
* Từ IP nào/UA nào?
* Trên tenant nào?
* Outcome và reason?

---

## Constraints

* ❌ Không log raw token, password, session cookie, API key.
* ❌ Không log full request body cho endpoints nhạy cảm.
* ✅ Tất cả audit log phải là **structured JSON**, 1 event/line (NDJSON).
* ✅ Có **event_id** và **timestamp** (UTC).
* ✅ Có **reason codes** (machine-readable).
* ✅ Audit log phải **append-only** (không update).
* ✅ Có **retention/rotation** (tối thiểu by size hoặc by day).
* ✅ Có test chứng minh redaction + no control chars.

---

## Deliverables

Tạo `kata-70/`:

* `src/audit/schema.ts`
* `src/audit/redact.ts`
* `src/audit/logger.ts`
* `src/audit/store.ts` (append-only + rotate)
* `src/server.ts` (sample routes generating audit)
* `test/kata70.schema.test.ts`
* `test/kata70.redaction.test.ts`
* `test/kata70.appendonly.test.ts`
* `test/kata70.incident.test.ts`
* `README.md` (policy: what we log / what we never log)

Tooling: `vitest`. Store: file NDJSON (khuyến nghị) để đơn giản.

---

## Spec — Audit Event Schema (Minimal Viable)

### Core fields (bắt buộc)

```ts
type AuditEvent = {
  v: 1;
  event_id: string;        // uuid
  ts: string;              // ISO UTC
  request_id: string;      // from middleware
  trace_id?: string;       // optional
  actor: {
    type: "user" | "service" | "system" | "anonymous";
    id_hash?: string;      // hash(sub/userId), not raw
    roles?: string[];
  };
  tenant?: { id: string };

  action: string;          // e.g. "auth.login", "order.refund"
  target?: {
    type: string;          // e.g. "order"
    id?: string;           // safe id (orderId ok)
  };

  outcome: "ALLOW" | "DENY" | "FAIL";
  reason: string;          // machine-readable reason code
  severity: "INFO" | "WARN" | "HIGH";

  network?: {
    ip: string;
    ua_hash?: string;      // hash user-agent
  };

  metadata?: Record<string, unknown>; // sanitized, no secrets/PII
};
```

### Reason codes (tối thiểu)

* AuthN/AuthZ:

  * `LOGIN_SUCCESS`, `LOGIN_FAIL_BAD_CREDENTIALS`, `TOKEN_INVALID`, `AUTHZ_DENY`
* Security blocks:

  * `CSRF_DENY`, `SSRF_BLOCKED`, `RATE_LIMITED`, `JSON_REJECTED`
* Money/config:

  * `REFUND_SUCCESS`, `REFUND_FAIL`, `ROLE_CHANGED`, `CONFIG_CHANGED`

### Severity mapping

* `INFO`: normal success (login success, read allowed)
* `WARN`: deny / repeated fails / blocks
* `HIGH`: money movement, role change, config change

---

## Starter skeleton (điền TODO)

### `src/audit/redact.ts`

```ts
const SECRET_KEYS = [
  "password",
  "token",
  "access_token",
  "refresh_token",
  "authorization",
  "cookie",
  "apiKey",
  "secret",
];

export function stripControlChars(s: string): string {
  // TODO: remove/escape \r \n \t and other CTLs
  return s.replace(/\r/g, "\\r").replace(/\n/g, "\\n").replace(/\t/g, "\\t");
}

export function redactObject(input: any): any {
  // TODO:
  // - deep clone with limits
  // - if key matches SECRET_KEYS (case-insensitive) => "[REDACTED]"
  // - if value is string containing JWT-like pattern => redact (stretch)
  // - strip control chars in strings
  return input;
}
```

### `src/audit/schema.ts`

```ts
import { z } from "zod";

export const AuditEventSchema = z.object({
  v: z.literal(1),
  event_id: z.string().min(8),
  ts: z.string(),
  request_id: z.string().min(6),
  trace_id: z.string().optional(),
  actor: z.object({
    type: z.enum(["user","service","system","anonymous"]),
    id_hash: z.string().optional(),
    roles: z.array(z.string()).optional(),
  }),
  tenant: z.object({ id: z.string().min(1) }).optional(),
  action: z.string().min(3),
  target: z.object({
    type: z.string().min(1),
    id: z.string().optional(),
  }).optional(),
  outcome: z.enum(["ALLOW","DENY","FAIL"]),
  reason: z.string().min(2),
  severity: z.enum(["INFO","WARN","HIGH"]),
  network: z.object({
    ip: z.string().min(3),
    ua_hash: z.string().optional(),
  }).optional(),
  metadata: z.record(z.any()).optional(),
});

export type AuditEvent = z.infer<typeof AuditEventSchema>;
```

### `src/audit/store.ts`

```ts
import fs from "node:fs/promises";
import path from "node:path";

export type AuditStoreOptions = {
  dir: string;
  maxBytes: number; // rotate by size, e.g. 5MB
};

export class AuditStore {
  private currentFile: string | null = null;

  constructor(private readonly opts: AuditStoreOptions) {}

  async append(line: string): Promise<void> {
    // TODO:
    // - ensure dir
    // - choose current file (date-based) e.g. audit-YYYY-MM-DD.ndjson
    // - rotate if size > maxBytes => audit-YYYY-MM-DD-<n>.ndjson
    // - append line + "\n"
  }

  async query(filter: { tenantId?: string; action?: string; from?: string; to?: string; limit: number }): Promise<any[]> {
    // TODO (kata-minimal):
    // - read recent files only (today + yesterday) or all in dir (ok for kata)
    // - parse NDJSON lines
    // - filter fields
    // - return newest first, cap limit
    return [];
  }
}
```

### `src/audit/logger.ts`

```ts
import crypto from "node:crypto";
import { AuditEventSchema, type AuditEvent } from "./schema";
import { redactObject } from "./redact";
import type { AuditStore } from "./store";

export type AuditContext = {
  request_id: string;
  trace_id?: string;
  actor: { type: AuditEvent["actor"]["type"]; id?: string; roles?: string[] };
  tenantId?: string;
  ip?: string;
  userAgent?: string;
};

function sha256(s: string) {
  return crypto.createHash("sha256").update(s).digest("hex");
}

export class AuditLogger {
  constructor(private readonly store: AuditStore) {}

  async log(ctx: AuditContext, e: Omit<AuditEvent, "v"|"event_id"|"ts"|"request_id"|"trace_id"|"actor"|"tenant"|"network">) {
    // TODO:
    // - build event with mandatory fields
    // - actor.id_hash = sha256(actor.id) (do not store raw)
    // - ua_hash = sha256(userAgent)
    // - redact metadata
    // - validate with zod
    // - write NDJSON line
  }
}
```

---

## Sample routes to generate audit (server)

Trong `src/server.ts`, tạo tối thiểu:

1. `POST /v1/login`

   * success → `auth.login` outcome ALLOW reason `LOGIN_SUCCESS`
   * fail → outcome DENY reason `LOGIN_FAIL_BAD_CREDENTIALS` severity WARN
2. `POST /v1/orders/:id/refund`

   * allow/deny (giả lập authz) → audit reason `REFUND_SUCCESS` hoặc `AUTHZ_DENY`
3. `GET /v1/audit` (admin-only giả lập)

   * query store theo filter

**Quan trọng**: dùng `request_id` middleware (random per request). Không log password.

---

## Tests — bắt buộc

### `test/kata70.schema.test.ts`

* log một event hợp lệ → store append được
* event thiếu `reason` → zod reject

### `test/kata70.redaction.test.ts`

* metadata chứa `{ password:"x", token:"abc", nested:{authorization:"Bearer ..."} }`
* output NDJSON phải có `"[REDACTED]"`, không có raw value
* string có `\n` phải bị escape thành `\\n`

### `test/kata70.appendonly.test.ts`

* append 3 events → file có 3 dòng
* không có update/overwrite (size tăng dần)

### `test/kata70.incident.test.ts` (đúng chất Senior+)

* simulate:

  * 3 lần login fail từ cùng IP
  * 1 lần login success
  * 1 refund deny
* query theo tenant/action/time → reconstruct timeline:

  * assert ordering + reason codes + severity mapping đúng

---

## Checks (Definition of Done)

Bạn pass kata nếu:

1. Audit event schema chuẩn, validate được.
2. Không leak secrets/PII: không password/token/cookie/raw userId/raw UA.
3. Logs chống forging: không có newline thật trong output lines.
4. Store append-only + rotation basic.
5. Query endpoint trả được events theo filter.
6. Tests cover schema + redaction + incident reconstruction.

---

## Stretch (Senior+ “forensics-grade”)

1. **Tamper-evident chain**

   * mỗi event thêm `prev_hash` và `hash = sha256(prev_hash + line)`
   * giúp phát hiện sửa log (không ngăn được xoá, nhưng phát hiện).
2. **Rate-limited audit**

   * audit log itself không được DoS (buffer + drop policy).
3. **PII minimization**

   * hash với salt rotation; store salt in secret manager.
4. **Retention & export**

   * keep 30 ngày local, export S3/WORM (chỉ note).
5. **Alert hooks**

   * nếu `LOGIN_FAIL` > N/minute hoặc `AUTHZ_DENY` spike → emit security alert event.