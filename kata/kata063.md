# Kata 63 — JWT Pitfalls: issuer/audience, clock skew, rotation

## Goal

1. Viết `verifyJwt()` **đúng chuẩn production**:

   * Validate **issuer (`iss`)**
   * Validate **audience (`aud`)** (string hoặc array)
   * Validate thời gian: **`exp`, `nbf`, `iat`** với **clock skew allowance**
   * Validate **algorithm allowlist** (chặn alg confusion)
   * Validate **kid + JWKS rotation** (cache keys, refresh)
2. Trả về `Decision` (ok/deny) với **reason code** (không throw “bừa”).
3. Viết test mô phỏng các lỗi kinh điển:

   * sai `iss`, sai `aud`
   * expired nhưng trong skew window
   * `nbf` ở tương lai
   * `iat` quá xa tương lai (token from the future)
   * `alg=none` / hoặc đổi alg
   * `kid` không tồn tại / key rotated
4. Không log token / claims nhạy cảm.

---

## Context

Bạn có service nhận request với:
`Authorization: Bearer <JWT>`

Issuer (IdP) hợp lệ: `https://idp.example.com/`
Audience hợp lệ: `orders-api`

Bạn muốn chấp nhận clock skew ± **60s**.

JWKS endpoint (mô phỏng trong kata): `/.well-known/jwks.json`

---

## Constraints

* ✅ Dùng thư viện JWT phổ biến (gợi ý: `jose`) *hoặc* tự verify bằng crypto (khó hơn). Senior+ nên dùng `jose`.
* ✅ **Allowlist algorithms** (vd: chỉ `RS256`).
* ✅ Validate `iss`, `aud` *bắt buộc* (không optional).
* ✅ Time validation có `clockSkewSec`.
* ✅ Có **JWKS cache** + refresh khi gặp `kid` lạ (soft refresh).
* ✅ Output là `VerifyResult` với reason code (không throw ra ngoài middleware).
* ❌ Không được “decode rồi tin claims” trước khi verify signature.
* ❌ Không được log raw JWT hoặc full claims.

---

## Deliverables

Tạo `kata-63/`:

* `src/jwt/types.ts`
* `src/jwt/verify.ts`
* `src/jwt/jwks.ts` (cache + refresh)
* `src/authn.ts` (Bearer parsing + call verifyJwt)
* `src/logger.ts` (redaction)
* `test/kata63.verify.test.ts` (table-driven)
* `test/kata63.jwks-rotation.test.ts`

Tooling: `vitest`, `tsx`, `jose`.

---

## Spec: VerifyResult & reason codes

### `VerifyResult`

* `ok: true` → trả `claims` (subset safe)
* `ok: false` → trả `reason` + `httpStatus` (401)

### Reason codes (bạn phải dùng tối thiểu các cái này)

* `MISSING_TOKEN`
* `MALFORMED_AUTH_HEADER`
* `UNSUPPORTED_ALG`
* `SIGNATURE_INVALID`
* `KID_NOT_FOUND`
* `ISSUER_MISMATCH`
* `AUDIENCE_MISMATCH`
* `TOKEN_EXPIRED`
* `TOKEN_NOT_YET_VALID` (nbf)
* `TOKEN_IAT_IN_FUTURE`
* `TOKEN_MISSING_CLAIM` (iss/aud/exp/iat…)
* `JWKS_FETCH_FAILED`

---

## Starter skeleton (điền TODO)

### `src/jwt/types.ts`

```ts
export type JwtPolicy = {
  issuer: string;                 // required
  audience: string;               // required
  algs: readonly string[];        // allowlist, e.g. ["RS256"]
  clockSkewSec: number;           // e.g. 60
  maxIatFutureSec: number;        // e.g. 60 (iat cannot be too far in future)
};

export type SafeClaims = {
  sub: string;
  iss: string;
  aud: string | string[];
  iat: number;
  exp: number;
  nbf?: number;
  scope?: string;
  tenantId?: string;
  roles?: string[];
};

export type VerifyOk = { ok: true; claims: SafeClaims };
export type VerifyFail = {
  ok: false;
  reason:
    | "MISSING_TOKEN"
    | "MALFORMED_AUTH_HEADER"
    | "UNSUPPORTED_ALG"
    | "SIGNATURE_INVALID"
    | "KID_NOT_FOUND"
    | "ISSUER_MISMATCH"
    | "AUDIENCE_MISMATCH"
    | "TOKEN_EXPIRED"
    | "TOKEN_NOT_YET_VALID"
    | "TOKEN_IAT_IN_FUTURE"
    | "TOKEN_MISSING_CLAIM"
    | "JWKS_FETCH_FAILED";
  httpStatus: 401;
};

export type VerifyResult = VerifyOk | VerifyFail;
```

### `src/jwt/jwks.ts`

```ts
import type { KeyLike } from "jose";

export type JwksKey = { kid: string; key: KeyLike };

export type JwksClientOptions = {
  jwksUrl: string;
  cacheTtlMs: number;   // e.g. 5 min
};

export class JwksClient {
  private cache: Map<string, KeyLike> = new Map();
  private cacheUntil = 0;

  constructor(private readonly opts: JwksClientOptions) {}

  async getKey(kid: string): Promise<{ ok: true; key: KeyLike } | { ok: false; reason: "KID_NOT_FOUND" | "JWKS_FETCH_FAILED" }> {
    // TODO:
    // 1) if cache valid and has kid => return
    // 2) if cache expired OR missing kid => refresh JWKS (fetch)
    // 3) after refresh, if still missing => KID_NOT_FOUND
    return { ok: false, reason: "JWKS_FETCH_FAILED" };
  }

  private async refresh(): Promise<{ ok: true } | { ok: false; reason: "JWKS_FETCH_FAILED" }> {
    // TODO:
    // - fetch jwksUrl
    // - parse JWKS JSON
    // - import keys into KeyLike (jose importJWK)
    // - fill cache map kid->key
    // - update cacheUntil
    return { ok: false, reason: "JWKS_FETCH_FAILED" };
  }
}
```

### `src/jwt/verify.ts`

```ts
import { decodeProtectedHeader, jwtVerify } from "jose";
import type { JwtPolicy, VerifyResult, SafeClaims } from "./types";
import type { JwksClient } from "./jwks";

function hasAudience(aud: string | string[], expected: string): boolean {
  return Array.isArray(aud) ? aud.includes(expected) : aud === expected;
}

export async function verifyJwt(token: string, policy: JwtPolicy, jwks: JwksClient, nowSec = Math.floor(Date.now() / 1000)): Promise<VerifyResult> {
  // TODO:
  // 1) parse header safely: decodeProtectedHeader(token)
  // 2) enforce alg allowlist
  // 3) require kid for asymmetric algs
  // 4) jwks.getKey(kid)
  // 5) verify signature + standard claims using jose jwtVerify options:
  //    - issuer, audience, algorithms
  //    - clockTolerance for skew
  // 6) additional checks:
  //    - iat must exist; if iat > now + maxIatFutureSec => deny TOKEN_IAT_IN_FUTURE
  // 7) return safe subset claims
  //
  // IMPORTANT: don't log token, don't return raw payload
  return { ok: false, reason: "SIGNATURE_INVALID", httpStatus: 401 };
}

export function toSafeClaims(payload: any): SafeClaims | null {
  // TODO: validate required fields exist and types
  // required: sub, iss, aud, iat, exp
  return null;
}
```

### `src/authn.ts`

```ts
import type { JwtPolicy, VerifyResult } from "./jwt/types";
import { verifyJwt } from "./jwt/verify";
import type { JwksClient } from "./jwt/jwks";

export async function authenticateBearer(authHeader: string | undefined, policy: JwtPolicy, jwks: JwksClient): Promise<VerifyResult> {
  if (!authHeader) return { ok: false, reason: "MISSING_TOKEN", httpStatus: 401 };
  const m = authHeader.match(/^Bearer\s+(.+)$/i);
  if (!m) return { ok: false, reason: "MALFORMED_AUTH_HEADER", httpStatus: 401 };
  const token = m[1].trim();
  if (!token) return { ok: false, reason: "MALFORMED_AUTH_HEADER", httpStatus: 401 };
  return verifyJwt(token, policy, jwks);
}
```

---

## Tests (table-driven) — bắt buộc

Bạn sẽ tạo token trong test bằng `jose` với private key generate runtime.

### `test/kata63.verify.test.ts` — cases tối thiểu

1. ✅ **Happy path**: iss/aud đúng, exp tương lai, iat hợp lệ.
2. ❌ **ISS mismatch** → `ISSUER_MISMATCH`
3. ❌ **AUD mismatch** → `AUDIENCE_MISMATCH`
4. ❌ **Expired**:

   * exp < now - skew → `TOKEN_EXPIRED`
   * exp < now nhưng within skew → **allow** (đây là điểm kata)
5. ❌ **NBF in future**:

   * nbf > now + skew → `TOKEN_NOT_YET_VALID`
   * within skew → allow
6. ❌ **IAT in future**:

   * iat > now + maxIatFutureSec → `TOKEN_IAT_IN_FUTURE`
7. ❌ **Unsupported alg**:

   * header alg = `none` hoặc `HS256` khi allowlist chỉ RS256 → `UNSUPPORTED_ALG`
8. ❌ **Signature invalid**:

   * ký bằng key khác → `SIGNATURE_INVALID`

### `test/kata63.jwks-rotation.test.ts`

* Setup JWKS server mock:

  * JWKS v1 chứa kid=`k1`
  * Issue token kid=`k1` ok
  * Rotate JWKS sang kid=`k2` (k1 removed)
  * Token mới kid=`k2`:

    * lần đầu jwks cache chưa có → refresh → ok
  * Token cũ kid=`k1`:

    * nếu cache còn TTL và còn key: tùy policy bạn chọn:

      * **Option A (strict)**: vẫn ok cho đến TTL hết
      * **Option B (paranoid)**: refresh nếu signature fail / kid revoked list
  * Bạn chọn 1 option và ghi rõ trong README.

---

## Checks (Definition of Done)

Bạn pass kata nếu:

1. Không có đường nào “accept token” mà thiếu `iss` hoặc `aud`.
2. Có allowlist algorithm; `alg=none` không thể lọt.
3. Clock skew hoạt động đúng (expired/nbf within tolerance).
4. `iat` future được chặn.
5. JWKS cache + refresh khi kid lạ hoạt động (rotation test).
6. Trả về **reason codes đúng** (không throw raw error ra ngoài).
7. Không log token/claims nhạy cảm (audit log test).

---

## Stretch (Senior+ thật sự)

1. **Replay protection (jti)**
   Nếu token có `jti`, lưu vào cache TTL ngắn để chặn replay trong window (đặc biệt với endpoints nhạy cảm).

2. **DPoP / proof-of-possession (conceptual)**
   Không cần implement full, nhưng thêm hook `validateProof(req, claims)` để mở đường.

3. **Key pinning & kid allowlist per issuer**
   Chặn trường hợp attacker đưa token từ issuer khác nhưng same kid format.

4. **Split reasons for 401 vs 403**
   AuthN fail → 401, AuthZ fail → 403. Kata này chỉ AuthN, nhưng bạn thêm mapping để future-proof.

5. **Observability**
   Counter `authn_fail_total{reason=...}` + `jwks_refresh_total{result=...}`.