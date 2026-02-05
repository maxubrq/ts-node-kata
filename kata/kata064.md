# Kata 64 — CSRF/CORS Correct: rule set + tests

## Goal

1. Implement **CORS middleware** đúng chuẩn:

   * allowlist origin (không dùng `*` bừa bãi khi credentials)
   * xử lý **preflight OPTIONS** (methods/headers)
   * set đúng headers: `Access-Control-Allow-Origin`, `-Methods`, `-Headers`, `-Credentials`, `-Max-Age`, `Vary: Origin`
   * chặn origin “null”, origin lạ, origin bị spoof (string tricks)
2. Implement **CSRF protection** đúng cho app dùng **cookie auth**:

   * double-submit cookie hoặc synchronizer token (chọn 1; kata gợi ý double-submit)
   * enforce cho **unsafe methods**: POST/PUT/PATCH/DELETE
   * check **Origin/Referer** khi thích hợp
3. Viết test:

   * CORS preflight & simple requests
   * credentials + allowed origin
   * disallowed origin
   * CSRF token missing/invalid
   * bypass attempts (wrong content-type, same-site, null origin)

---

## Context

Service chạy ở `https://api.example.com`.

Frontend hợp lệ:

* `https://app.example.com`
* `https://admin.example.com`
  Dev:
* `http://localhost:3000`

Auth dùng cookie:

* `session=<opaque>` (HttpOnly, Secure, SameSite=Lax hoặc None tùy scenario)

Endpoints:

* `GET /v1/profile` (safe)
* `POST /v1/orders` (unsafe, cần CSRF)
* `POST /v1/orders/:id/refund` (unsafe, cần CSRF)

---

## Constraints

* ❌ Không dùng `Access-Control-Allow-Origin: *` khi `Access-Control-Allow-Credentials: true`.
* ❌ Không “reflect origin” bừa bãi (Origin nào cũng echo lại).
* ✅ Origin parsing phải **an toàn** (dùng `new URL(origin)`; reject invalid).
* ✅ `Vary: Origin` phải được set khi response phụ thuộc origin.
* ✅ CSRF: không dựa duy nhất vào CORS (CORS không thay thế CSRF).
* ✅ Có test table-driven cho policy.
* ✅ Không log cookie/session/csrf token.

---

## Deliverables

Tạo `kata-64/`:

* `src/cors/policy.ts`
* `src/cors/middleware.ts`
* `src/csrf/tokens.ts`
* `src/csrf/middleware.ts`
* `src/server.ts` (mini app)
* `test/kata64.cors.test.ts`
* `test/kata64.csrf.test.ts`

Bạn có thể dùng `express` hoặc `node:http`. Test dùng `vitest` + `supertest` (nếu express).

---

## Spec 1 — CORS Rule Set

### CORS options

* `allowedOrigins`: exact match list
* `allowedMethods`: `GET,POST,PUT,PATCH,DELETE`
* `allowedHeaders`: allowlist + echo requested headers (nhưng phải intersect allowlist)
* `allowCredentials`: true (vì cookie auth)
* `maxAgeSec`: 600 (10 phút)

### Rules

1. Nếu **không có Origin**: không set CORS headers (đây là same-origin / non-browser client).
2. Nếu Origin **không nằm allowlist**: không set allow headers, và với preflight trả **403**.
3. Nếu request là **OPTIONS preflight**:

   * Require `Access-Control-Request-Method`
   * Validate method thuộc allowMethods
   * Validate `Access-Control-Request-Headers` (nếu có) nằm trong allowHeaders
   * Return `204` (no content) + headers allow
4. Nếu **credentials=true**:

   * `Access-Control-Allow-Origin` phải là origin cụ thể
   * `Access-Control-Allow-Credentials: true`
5. Luôn set `Vary: Origin` (và `Vary: Access-Control-Request-Headers` cho preflight nếu bạn echo).

---

## Spec 2 — CSRF (Double-submit cookie)

### Strategy

* Server cấp token CSRF qua endpoint: `GET /v1/csrf`:

  * Set cookie: `csrf_token=<random>` (NOT HttpOnly, để JS đọc)
  * Đồng thời return JSON `{ csrfToken: "<same>" }` (optional)
* Client gửi token trong header cho unsafe requests:

  * `X-CSRF-Token: <token>`
* Server validate:

  * cookie `csrf_token` tồn tại
  * header `X-CSRF-Token` tồn tại
  * **exact match**
* Apply cho methods: `POST, PUT, PATCH, DELETE`
* Skip CSRF cho:

  * `GET, HEAD, OPTIONS`
  * endpoints public idempotent nếu bạn muốn (nhưng kata: chỉ skip safe methods)

### Origin/Referer check (defense-in-depth)

* Nếu có header `Origin`:

  * must be allowlisted origin **hoặc** cùng site (tùy thiết kế)
  * Nếu Origin thiếu (một số case): fallback check `Referer` (optional)
* Kata yêu cầu: implement **Origin check cho unsafe methods khi có Origin**.

---

## Starter skeleton (điền TODO)

### `src/cors/policy.ts`

```ts
export type CorsPolicy = {
  allowedOrigins: readonly string[];
  allowedMethods: readonly string[];
  allowedHeaders: readonly string[];
  allowCredentials: boolean;
  maxAgeSec: number;
};

export function isAllowedOrigin(origin: string, policy: CorsPolicy): boolean {
  // TODO: exact match only (no wildcard), after normalization
  return policy.allowedOrigins.includes(origin);
}

export function normalizeOrigin(origin: string): string | null {
  // TODO:
  // - parse by new URL(origin)
  // - only allow http/https
  // - return `${protocol}//${host}` (no path)
  // - reject invalid / null / file / chrome-extension etc
  return null;
}
```

### `src/cors/middleware.ts`

```ts
import type { CorsPolicy } from "./policy";
import { isAllowedOrigin, normalizeOrigin } from "./policy";

export type Req = { method: string; headers: Record<string, string | undefined> };
export type Res = { statusCode: number; setHeader: (k: string, v: string) => void; end: (b?: any) => void };
export type Next = () => Promise<void> | void;

export function cors(policy: CorsPolicy) {
  return async (req: Req, res: Res, next: Next) => {
    const rawOrigin = req.headers["origin"];
    if (!rawOrigin) return next();

    const origin = normalizeOrigin(rawOrigin);
    if (!origin || !isAllowedOrigin(origin, policy)) {
      // Preflight should be rejected explicitly
      if (req.method.toUpperCase() === "OPTIONS") {
        res.statusCode = 403;
        res.end();
        return;
      }
      return next();
    }

    // TODO: set Vary: Origin (append-safe)
    res.setHeader("Vary", "Origin");

    // TODO: set ACAO and credentials
    res.setHeader("Access-Control-Allow-Origin", origin);
    if (policy.allowCredentials) res.setHeader("Access-Control-Allow-Credentials", "true");

    if (req.method.toUpperCase() !== "OPTIONS") {
      return next();
    }

    // Preflight
    const reqMethod = req.headers["access-control-request-method"];
    if (!reqMethod) {
      res.statusCode = 400;
      res.end();
      return;
    }
    // TODO: validate method
    // TODO: validate headers
    // TODO: set Allow-Methods/Allow-Headers/Max-Age
    res.statusCode = 204;
    res.end();
  };
}
```

### `src/csrf/tokens.ts`

```ts
import crypto from "node:crypto";

export function generateCsrfToken(): string {
  return crypto.randomBytes(32).toString("base64url");
}

export function timingSafeEqual(a: string, b: string): boolean {
  // TODO: constant-time compare
  // (Hint: use Buffer + crypto.timingSafeEqual, but handle length mismatch safely)
  return false;
}
```

### `src/csrf/middleware.ts`

```ts
import type { CorsPolicy } from "../cors/policy";
import { normalizeOrigin, isAllowedOrigin } from "../cors/policy";
import { timingSafeEqual } from "./tokens";

export type Req = { method: string; headers: Record<string, string | undefined>; cookies?: Record<string, string> };
export type Res = { statusCode: number; end: (b?: any) => void };
export type Next = () => Promise<void> | void;

const UNSAFE = new Set(["POST", "PUT", "PATCH", "DELETE"]);

export function csrfProtect(corsPolicyForOriginCheck: CorsPolicy) {
  return async (req: Req, res: Res, next: Next) => {
    const m = req.method.toUpperCase();
    if (!UNSAFE.has(m)) return next();

    // Defense-in-depth: Origin check when present
    const rawOrigin = req.headers["origin"];
    if (rawOrigin) {
      const origin = normalizeOrigin(rawOrigin);
      if (!origin || !isAllowedOrigin(origin, corsPolicyForOriginCheck)) {
        res.statusCode = 403;
        res.end({ error: "CSRF_ORIGIN_DENY" });
        return;
      }
    }

    const cookieToken = req.cookies?.["csrf_token"];
    const headerToken = req.headers["x-csrf-token"];

    // TODO: deny if missing
    // TODO: timing-safe compare
    if (!cookieToken || !headerToken) {
      res.statusCode = 403;
      res.end({ error: "CSRF_TOKEN_MISSING" });
      return;
    }

    if (!timingSafeEqual(cookieToken, headerToken)) {
      res.statusCode = 403;
      res.end({ error: "CSRF_TOKEN_INVALID" });
      return;
    }

    return next();
  };
}
```

---

## Server wiring (minimal)

### `src/server.ts` requirements

* `GET /v1/csrf`:

  * generate token
  * set cookie `csrf_token=...; Secure; SameSite=Lax` (dev có thể allow non-secure)
  * return `{ csrfToken }`
* `GET /v1/profile`:

  * returns ok
* `POST /v1/orders`:

  * behind `csrfProtect(...)`
* `OPTIONS /*`:

  * handled by CORS middleware

---

## Tests — bắt buộc (table-driven)

### `test/kata64.cors.test.ts`

Cases tối thiểu:

1. **No Origin header**:

   * `GET /v1/profile` → không có `Access-Control-Allow-Origin`
2. **Allowed Origin simple request**:

   * Origin `https://app.example.com`
   * `GET` → `ACAO` đúng, `ACC=true`, `Vary: Origin`
3. **Disallowed Origin simple request**:

   * Origin `https://evil.com`
   * `GET` → không set ACAO/ACC
4. **Preflight allowed**:

   * `OPTIONS /v1/orders`
   * Origin allowed
   * `Access-Control-Request-Method: POST`
   * `Access-Control-Request-Headers: content-type,x-csrf-token`
   * Expect `204`, headers allow đúng
5. **Preflight method not allowed**:

   * request-method `TRACE` → `403` hoặc `400` (bạn chọn, nhưng phải consistent)
6. **Preflight headers not allowed**:

   * request-headers includes `x-evil` → reject

### `test/kata64.csrf.test.ts`

Cases tối thiểu:

1. **GET does not require CSRF**:

   * `GET /v1/profile` ok
2. **POST missing token**:

   * `POST /v1/orders` (Origin allowed) nhưng thiếu cookie/header → 403 `CSRF_TOKEN_MISSING`
3. **POST header mismatch**:

   * cookie `csrf_token=a`, header `X-CSRF-Token=b` → 403 `CSRF_TOKEN_INVALID`
4. **POST ok**:

   * cookie/header match → 200
5. **Unsafe request with disallowed Origin**:

   * Origin `https://evil.com` → 403 `CSRF_ORIGIN_DENY`

---

## Checks (Definition of Done)

Bạn pass kata nếu:

1. CORS:

   * Không bao giờ dùng `*` với credentials
   * Preflight trả đúng allow headers/methods, status 204
   * `Vary: Origin` có mặt
   * Disallowed origin không được “reflect”
2. CSRF:

   * Unsafe methods bắt buộc token match
   * Origin check hoạt động (defense-in-depth)
   * Compare timing-safe
3. Tests cover các pitfalls chính.

---

## Stretch (Senior+ “đốt production” level)

1. **SameSite nuance drill**

   * Mô phỏng `SameSite=Lax` vs `None; Secure` và ghi rõ khi nào cần cái nào (SPA cross-site).
2. **Trusted origins by environment**

   * Policy load từ config schema + test “prod must not allow localhost”.
3. **CORS: Vary append-safe**

   * Nếu đã có `Vary` từ upstream, bạn phải append chứ không overwrite.
4. **CSRF exemptions**

   * Exempt endpoint `POST /webhooks/*` (machine-to-machine) nhưng enforce signature.
5. **Fetch metadata headers**

   * Dùng `Sec-Fetch-Site` / `Sec-Fetch-Mode` như extra signal (không dựa hoàn toàn).
