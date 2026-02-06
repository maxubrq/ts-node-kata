# Kata 77 — Anti-corruption Layer: Adapter cho External API

## Goal

1. Thiết kế **ACL (Anti-corruption Layer)** để:

   * map **external DTO → internal canonical model**
   * normalize data (units, timezone, enums, naming)
   * translate error + retry/timeout policy
2. Domain/App **không import** external SDK/types/DTO.
3. Có **contract tests** cho adapter (fixture / recorded response) để chống “drift”.
4. Có **failure model** rõ: rate limit, timeout, invalid payload, schema change.

---

## Context (bài toán)

Bạn tích hợp một dịch vụ ngoài: **Payment Provider X** để “charge” (thu tiền).

External API (giả lập) rất “bẩn”:

* trả tiền dạng **string decimal** `"12.34"` (USD) thay vì cents
* status lộn xộn: `"ok" | "OK" | "success" | "paid"`
* errors trả kiểu: `{ error_code: "E_RATE", error_message: "...", retry_after: "2s" }`
* đôi lúc thiếu field hoặc thêm field mới bất ngờ

Bạn muốn internal system sạch:

* internal amount luôn là **integer cents**
* status là enum chuẩn: `PAID | DECLINED | PENDING`
* errors là taxonomy chuẩn: `RateLimited | Timeout | BadRequest | ProviderDown | InvalidResponse`

---

## Constraints

* ✅ **No leakage**: `src/domain` + `src/application` **không được** import từ `src/infra/external/**`
* ✅ ACL nằm ở `src/infra/acl/paymentProviderX/` và expose qua **port** `PaymentGateway`
* ✅ Parse don’t validate: external JSON → parse into strict internal types
* ✅ Policy:

  * **Timeout** per request
  * **Retry** có jitter + budget (chỉ retry lỗi “transient”: timeout, 5xx, rate limit)
  * **Circuit breaker** optional (stretch)
* ✅ Contract tests:

  * adapter phải pass các fixture responses (valid/invalid/schema change)
  * tests fail nếu mapping sai (units/enums)
* ✅ Observability:

  * log/metric ở boundary (không log secrets)

---

## Deliverables (repo shape)

```
kata-77/
  src/
    domain/
      money.ts
    application/
      ports/
        paymentGateway.ts
      usecases/
        chargeCustomer.ts
    infra/
      acl/
        paymentProviderX/
          externalTypes.ts          // external DTO (only here)
          mapper.ts                 // translate external -> internal
          client.ts                 // http client wrapper
          adapter.ts                // implements PaymentGateway
          fixtures/                 // json fixtures for contract tests
    test/
      acl.contract.test.ts
      acl.failure-model.test.ts
      usecase.chargeCustomer.test.ts
      arch.no-leakage.test.ts
  README.md
```

---

## Internal model (Canonical)

### `src/domain/money.ts`

```ts
export type MoneyCents = number;

export function moneyCents(n: number): MoneyCents {
  if (!Number.isInteger(n) || n < 0) throw new Error("MoneyCents must be a non-negative integer");
  return n;
}
```

### `src/application/ports/paymentGateway.ts`

```ts
import type { MoneyCents } from "../../domain/money";

export type ChargeRequest = {
  idempotencyKey: string;
  customerId: string;
  amountCents: MoneyCents;
  currency: "USD";
};

export type PaymentStatus = "PAID" | "DECLINED" | "PENDING";

export type ChargeResult =
  | { ok: true; providerChargeId: string; status: PaymentStatus; processedAt: string }
  | { ok: false; error: PaymentError };

export type PaymentError =
  | { type: "RateLimited"; retryAfterMs?: number }
  | { type: "Timeout" }
  | { type: "BadRequest"; message: string }
  | { type: "ProviderDown" }
  | { type: "InvalidResponse"; message: string };

export interface PaymentGateway {
  charge(req: ChargeRequest): Promise<ChargeResult>;
}
```

---

## External API (giả lập bẩn)

### `src/infra/acl/paymentProviderX/externalTypes.ts`

```ts
// External request DTO
export type XChargeRequest = {
  customer_id: string;
  amount: string;     // "12.34" USD decimal string
  currency: "USD";
};

// External response DTO (bẩn, inconsistent)
export type XChargeResponse =
  | { status: "ok" | "OK" | "success" | "paid"; charge_id: string; processed_at: string }
  | { status: "pending"; charge_id: string; processed_at: string }
  | { status: "failed"; error_code: string; error_message: string; retry_after?: string };

// External error might be HTTP-level, etc.
```

---

## ACL Mapper (trái tim của kata)

### `src/infra/acl/paymentProviderX/mapper.ts`

```ts
import type { ChargeRequest, ChargeResult, PaymentStatus, PaymentError } from "../../../application/ports/paymentGateway";
import type { XChargeRequest, XChargeResponse } from "./externalTypes";

export function toXChargeRequest(req: ChargeRequest): XChargeRequest {
  // TODO: cents -> decimal string with 2 digits
  // e.g. 1234 -> "12.34"
  return { customer_id: req.customerId, amount: "TODO", currency: req.currency };
}

function normalizeStatus(s: string): PaymentStatus | null {
  const v = s.toLowerCase();
  if (v === "ok" || v === "success" || v === "paid") return "PAID";
  if (v === "pending") return "PENDING";
  if (v === "failed") return "DECLINED";
  return null;
}

export function parseRetryAfterMs(x?: string): number | undefined {
  // TODO: parse "2s" | "1500ms" | undefined
  return undefined;
}

export function fromXChargeResponse(x: unknown): ChargeResult {
  // TODO: strict parse:
  // - must be object
  // - must have status string
  // - if success: require charge_id, processed_at
  // - if failed: require error_code, error_message
  // - unknown shapes => InvalidResponse
  const bad = (message: string): ChargeResult => ({ ok: false, error: { type: "InvalidResponse", message } });
  return bad("TODO");
}

export function mapXErrorToPaymentError(args: {
  kind: "HTTP_TIMEOUT" | "HTTP_429" | "HTTP_5XX" | "HTTP_4XX";
  body?: unknown;
}): PaymentError {
  // TODO:
  // - timeout => Timeout
  // - 429 => RateLimited (try parse retry_after from body)
  // - 5xx => ProviderDown
  // - 4xx => BadRequest (extract message)
  return { type: "ProviderDown" };
}
```

---

## HTTP Client wrapper (policy: timeout + retry)

### `src/infra/acl/paymentProviderX/client.ts`

```ts
export type HttpResponse = { status: number; body: unknown };

export interface HttpClient {
  postJson(path: string, body: unknown, opts: { timeoutMs: number; headers: Record<string, string> }): Promise<HttpResponse>;
}

export type RetryPolicy = {
  maxAttempts: number;      // e.g. 3
  baseDelayMs: number;      // e.g. 100
  jitterMs: number;         // e.g. 50
};

export async function withRetry<T>(
  fn: (attempt: number) => Promise<T>,
  shouldRetry: (err: unknown) => boolean,
  policy: RetryPolicy,
  sleep: (ms: number) => Promise<void>
): Promise<T> {
  let lastErr: unknown;
  for (let i = 1; i <= policy.maxAttempts; i++) {
    try {
      return await fn(i);
    } catch (e) {
      lastErr = e;
      if (i === policy.maxAttempts || !shouldRetry(e)) break;
      const delay = policy.baseDelayMs * i + Math.floor(Math.random() * policy.jitterMs);
      await sleep(delay);
    }
  }
  throw lastErr;
}
```

---

## Adapter implements Port

### `src/infra/acl/paymentProviderX/adapter.ts`

```ts
import type { PaymentGateway, ChargeRequest, ChargeResult } from "../../../application/ports/paymentGateway";
import type { HttpClient } from "./client";
import { toXChargeRequest, fromXChargeResponse, mapXErrorToPaymentError } from "./mapper";
import { withRetry } from "./client";

export type PaymentProviderXAdapterDeps = {
  http: HttpClient;
  apiKey: string;           // secret
  timeoutMs: number;        // e.g. 1500
  retry: { maxAttempts: number; baseDelayMs: number; jitterMs: number };
  sleep: (ms: number) => Promise<void>;
  log: (obj: Record<string, unknown>, msg: string) => void;
};

export class PaymentProviderXAdapter implements PaymentGateway {
  constructor(private deps: PaymentProviderXAdapterDeps) {}

  async charge(req: ChargeRequest): Promise<ChargeResult> {
    const xReq = toXChargeRequest(req);

    const call = async () => {
      const res = await this.deps.http.postJson(
        "/charge",
        xReq,
        {
          timeoutMs: this.deps.timeoutMs,
          headers: {
            "authorization": `Bearer ${this.deps.apiKey}`,
            "idempotency-key": req.idempotencyKey,
          },
        }
      );

      if (res.status >= 200 && res.status < 300) {
        return fromXChargeResponse(res.body);
      }

      if (res.status === 429) throw { kind: "HTTP_429", body: res.body };
      if (res.status >= 500) throw { kind: "HTTP_5XX", body: res.body };
      throw { kind: "HTTP_4XX", body: res.body };
    };

    const shouldRetry = (e: any) => {
      return e?.kind === "HTTP_429" || e?.kind === "HTTP_5XX" || e?.kind === "HTTP_TIMEOUT";
    };

    try {
      const result = await withRetry(
        async () => {
          const r = await call();
          // NOTE: retry only on transport-level errors; if ok:false from provider, don't retry
          return r;
        },
        shouldRetry,
        this.deps.retry,
        this.deps.sleep
      );

      // observability: log decision but don't log apiKey
      this.deps.log({ ok: result.ok }, "payment.charge result");
      return result;
    } catch (e: any) {
      const err = mapXErrorToPaymentError({ kind: e?.kind ?? "HTTP_5XX", body: e?.body });
      this.deps.log({ err: err.type }, "payment.charge failed");
      return { ok: false, error: err };
    }
  }
}
```

---

## Use-case uses the port (clean)

### `src/application/usecases/chargeCustomer.ts`

```ts
import type { PaymentGateway, ChargeRequest, ChargeResult } from "../ports/paymentGateway";

export function makeChargeCustomer(deps: { gateway: PaymentGateway }) {
  return async function chargeCustomer(req: ChargeRequest): Promise<ChargeResult> {
    // application policy: could add idempotency store here too (optional)
    return deps.gateway.charge(req);
  };
}
```

---

## Tests (bắt buộc)

### 1) Contract tests với fixtures: `test/acl.contract.test.ts`

Bạn tạo fixtures:

* `success_ok.json` `{ "status":"ok","charge_id":"ch_1","processed_at":"2026-02-06T00:00:00Z" }`
* `success_paid_upper.json` `{ "status":"PAID","charge_id":"ch_2","processed_at":"..." }` *(để chứng minh normalize robust — dù spec external “bẩn”)*
* `failed_rate.json` `{ "status":"failed","error_code":"E_RATE","error_message":"slow down","retry_after":"2s" }`
* `invalid_missing_charge_id.json` `{ "status":"ok","processed_at":"..." }`

Test:

* `fromXChargeResponse(fixture)` returns đúng `PAID/PENDING/DECLINED`
* invalid fixture => `InvalidResponse`

### 2) Failure model: `test/acl.failure-model.test.ts`

Mock `HttpClient`:

* timeout (throw `{ kind:"HTTP_TIMEOUT" }`) => `Timeout`
* 429 => `RateLimited` + parse retryAfterMs
* 5xx => `ProviderDown`
* 4xx => `BadRequest` (extract message if present)

### 3) Use-case stays clean: `test/usecase.chargeCustomer.test.ts`

* inject fake gateway: ensure use-case returns exactly gateway result
* confirms no coupling to external

### 4) No leakage architecture test: `test/arch.no-leakage.test.ts`

Scan imports:

* fail nếu `src/application/**` hoặc `src/domain/**` import path contains `/infra/acl/` hoặc external types.

---

## Checks (Definition of Done)

* ✅ Domain/app chỉ biết `PaymentGateway` port + canonical types.
* ✅ Adapter handles unit conversion + enum normalization + strict parse.
* ✅ Retry/timeout applied đúng chỗ (chỉ retry transient).
* ✅ Contract tests lock mapping against drift.
* ✅ Schema change/invalid payload → `InvalidResponse` (không crash, không leak undefined).

---

## Stretch (Senior+ → rất cao)

1. **Schema evolution hardening**

   * adapter accepts v1/v2 payload (e.g. `chargeId` renamed), nhưng internal không đổi
2. **Circuit breaker**

   * mở khi lỗi liên tiếp, half-open thử lại, expose metrics
3. **Hedged requests**

   * nếu p95 latency cao: sau X ms bắn request thứ 2 (budgeted), lấy first win
4. **PII/Secret redaction**

   * structured logging: never log `customerId` raw, hash/shorten
5. **Contract test via recorded HTTP**

   * tạo “tape” file: request+response pair, replay in test; fail nếu request shape drift