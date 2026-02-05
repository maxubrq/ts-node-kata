# Kata 57 — Debug Mode Safe

*Bật debug để điều tra nhanh, nhưng không lộ secrets/PII, không “mở cửa hậu” production.*

---

## 1) Goal (năng lực cần đạt)

Sau kata này bạn xây được một **Debug Mode** production-grade cho Node/TS service:

* Debug **bật/tắt được** (runtime) và **có TTL**.
* Debug **chỉ tăng signal cần thiết** (logs/metrics/traces), không dump bừa.
* Có **redaction** + **allowlist** fields: không lộ secrets/PII dù dev “lỡ tay”.
* Debug endpoints (nếu có) được **guard** chặt (auth + network + rate limit).
* Có **audit trail**: ai bật, lúc nào, ở env nào, lý do gì.
* Có **tests** chứng minh không leak secrets trong mọi mode.

---

## 2) Context (bài toán)

Service của bạn đã có:

* structured logging (kata 27)
* tracing context (kata 52)
* log sampling (kata 53)

Bạn sẽ thêm:

* Debug mode “normal / debug” + “trace tail” mode (optional)
* Một endpoint admin để bật debug (hoặc signal/env reload)

---

## 3) Constraints (Senior+ bắt buộc)

1. **Không** được có “DEBUG=true là log toàn bộ request body/headers” kiểu chết người.
2. Debug phải có **TTL auto-off** (vd 10 phút).
3. Debug chỉ được bật qua **1 trong các cơ chế an toàn**:

   * signed token (HMAC) + time bound
   * hoặc internal-only network + auth
   * hoặc config store + allowlist (tuỳ bạn)
4. Redaction:

   * Phải redact ít nhất: `authorization`, `cookie`, `set-cookie`, `x-api-key`, `password`, `secret`, `token`, `session`, `private_key`
   * PII: `email`, `phone`, `address` (tuỳ spec) — ít nhất phải có cơ chế allowlist/denylist.
5. Không log raw headers/body mặc định; debug cũng **chỉ log subset** (allowlist).
6. Debug mode không được phá performance quá mức:

   * Có rate limit / sampling riêng cho debug logs
7. Có tests: “bật debug” vẫn không leak secrets.

---

## 4) Deliverables (repo)

```
kata-57/
├─ src/
│  ├─ server.ts
│  ├─ debug/
│  │  ├─ state.ts        (debug state + TTL)
│  │  ├─ guard.ts        (auth + scope + rate limit)
│  │  ├─ redact.ts       (redaction/allowlist)
│  │  ├─ endpoints.ts    (/admin/debug on/off, status)
│  │  └─ logging.ts      (debug logging policy)
│  ├─ logger.ts          (pino + mixin)
│  └─ config.ts          (env + runtime config)
├─ test/
│  ├─ redact.spec.ts
│  ├─ debug-ttl.spec.ts
│  └─ integration.spec.ts
└─ README.md             (policy + threat model)
```

---

## 5) Debug policy (bạn implement đúng kiểu này)

### 5.1 Debug levels

* `NORMAL` (default)
* `DEBUG` (more logs, still safe)
* `TRACE_TAIL` (optional): keep logs cho tail requests (kết hợp kata 53/54)

### 5.2 What debug is allowed to do

In `DEBUG`:

* Increase baseline sampling (ví dụ từ 1% → 10%) **nhưng vẫn deterministic**
* Log thêm **safe context**:

  * `route`, `status_code`, `duration_ms`
  * dependency timing breakdown (db/payment duration)
  * error stack (ok) nhưng phải sanitize message nếu chứa secrets
* Add a debug tag: `debug=true`, `debug_reason`, `debug_expires_at`

Not allowed even in debug:

* Raw `Authorization` header
* Raw cookies
* Full request body
* Full response body
* Environment variables dump
* Stack trace của config loader nếu chứa secrets (phải redact)

---

## 6) Runtime toggle mechanism (safe)

Bạn chọn 1 cơ chế; kata yêu cầu implement ít nhất 1:

### Option A — Admin endpoint + signed token (khuyến nghị)

Endpoint:

* `POST /admin/debug/enable`
* Body: `{ ttl_seconds: 600, reason: "incident-123", level: "DEBUG" }`
* Header: `x-debug-token: <signed>`

Token format (gợi ý):

* `base64(payload).base64(hmac)`
  Payload chứa: `sub`, `scope=["debug:write"]`, `exp`, `env`, `service`

Guard:

* Validate HMAC bằng `DEBUG_HMAC_SECRET`
* Check `exp` còn hạn
* Check scope đúng
* Rate limit endpoint (vd 2/min)
* Log audit: `who`, `reason`, `ttl`, `level`, `trace_id`

### Option B — SIGUSR2 + allowlist (acceptable)

* On signal, toggle debug for TTL (still must have TTL)
* Only usable trong controlled ops; vẫn cần audit log.

---

## 7) Redaction system (phần quan trọng nhất)

### 7.1 Denylist keys (case-insensitive, partial match)

Must redact if key contains:

* `authorization`, `cookie`, `set-cookie`, `x-api-key`
* `password`, `passwd`, `secret`, `token`, `session`
* `private`, `key`, `credential`

### 7.2 Allowlist fields for request logging (debug only)

Request:

* `method`, `route`, `query` (nhưng redact query keys), `content-type`, `content-length`
* body: **chỉ allowlist** keys (vd: `order_id`, `amount`, `currency`) và truncate string length

### 7.3 Redaction output rules

* Replace with `"[REDACTED]"`.
* Truncate strings > 256 chars.
* Limit depth (max depth 5) + max array length (max 20) để chống DOS by JSON.

---

## 8) Integration với logger (pino)

Bạn làm `logger.debugSafe(obj, msg)`:

* Always run `redact(obj)`
* If NORMAL: drop hoặc sample thấp
* If DEBUG: keep theo policy, nhưng vẫn redact

Quan trọng:

* Logger phải không cho dev gọi nhầm `logger.info(req.headers)` mà leak.
  => Senior+ pattern: expose **safe wrappers** và lint rule/guard (stretch).

---

## 9) TTL auto-off + audit trail

`debug/state.ts`:

* Store:

  * `enabled: boolean`
  * `level`
  * `reason`
  * `enabled_by`
  * `expires_at`
* Timer auto disable
* Metrics:

  * `debug_mode_enabled` gauge (0/1)
  * `debug_mode_toggles_total{action="enable|disable", by}`

Audit log khi:

* enable/disable
* failed attempts (invalid token)

---

## 10) Tests (bắt buộc)

### 10.1 redact.spec.ts

Input object chứa:

* headers: authorization, cookie
* body: password, token, nested secrets
  Expect:
* tất cả bị `[REDACTED]`
* strings dài bị truncate
* depth limit works

### 10.2 debug-ttl.spec.ts

* Enable debug TTL=1s
* Wait >1s
* Assert debug auto-off

### 10.3 integration.spec.ts (quan trọng)

Scenario:

1. NORMAL mode:

* Call `/orders` với body có `password`, header `Authorization`
* Scrape captured logs
* Assert logs **không chứa** raw secret

2. DEBUG mode enabled:

* Enable debug with signed token
* Call `/orders` again
* Assert logs tăng signal (có debug fields)
* Assert secrets vẫn **không xuất hiện**

3. Attack:

* Enable debug với token sai / hết hạn
* Assert 401 + audit log `debug_enable_denied`

---

## 11) Definition of Done (pass/fail)

PASS khi:

* Debug toggle có TTL + audit trail.
* Debug logging tăng signal nhưng không leak secrets/PII (đã test).
* Redaction hoạt động với denylist + depth/size limits.
* Debug endpoints được guard (auth + rate limit).
* Có metrics cho debug mode.

---

## 12) Stretch (Senior++)

1. **Per-trace debug**: bật debug chỉ cho 1 `trace_id` trong 10 phút.
2. **Dynamic sampling integration**: debug tăng baseline sampling nhưng vẫn deterministic.
3. **Secret scanning test**: chạy regex scanner trên log output để fail CI nếu thấy patterns (JWT, API key prefix, `Bearer `).
4. **Break-glass workflow**: require 2-step approval (token từ 2 keys) trước khi bật debug prod.
