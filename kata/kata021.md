# Kata 21 — Idempotency Key

**POST thanh toán idempotent**: retry bao nhiêu lần cũng **không charge 2 lần**, và client nhận **cùng 1 kết quả**.

---

## Goal

Bạn build 1 endpoint giả lập:

`POST /payments` với body:

```ts
{ userId: string; amount: number; currency: "VND" | "USD"; idempotencyKey: string }
```

Yêu cầu:

1. **Same idempotencyKey + same semantic request** → trả lại **same response** (kể cả retry)
2. **Same idempotencyKey + different request** → **409 conflict**
3. **Concurrent requests** cùng key → chỉ **1** request được chạy charge; request khác phải:

   * chờ kết quả (preferred), hoặc
   * trả “in_progress” (acceptable stretch)
4. Có TTL để tránh store phình mãi (vd 24h)

---

## Constraints (production-grade)

* ❌ Không dựa vào “client tự đừng retry”
* ✅ Có “idempotency record” lưu:

  * request fingerprint
  * status: `in_progress | completed`
  * response (hoặc error)
  * timestamps + TTL
* ✅ Có **single-flight** theo key (lock/mutex hoặc atomic create)
* ✅ Không `any`, không `as` hack
* ✅ Không throw raw string; dùng error union (nối Kata 11)

---

## Deliverables

File `kata21.ts` gồm:

1. `IdempotencyStore` (in-memory) + TTL cleanup
2. `paymentHandler(req)` (POST /payments)
3. `fakeGatewayCharge()` (giả lập gọi payment gateway, có thể chậm / fail)
4. Demo 6 cases:

* first success
* retry success same response
* concurrent requests same key
* same key different amount → conflict
* gateway fail → retry returns same failure (hoặc policy bạn chọn, nhưng phải consistent)
* TTL expire → cho phép chạy lại

---

## Core design decision (bắt buộc ghi rõ trong code comment)

**Bạn chọn 1 trong 2 policy cho “gateway fail”:**

* **Policy A (recommended)**: nếu charge failed → lưu failure và retry trả lại same failure (tính “idempotent theo kết quả”)
* **Policy B**: failure không được cache lâu, retry sẽ attempt lại (rủi ro double charge nếu gateway actually processed).
  → Kata này mặc định **Policy A** (an toàn hơn).

---

## Starter skeleton (điền TODO)

```ts
/* kata21.ts */

// ---------- Result ----------
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };
const ok = <T>(value: T): Result<T, never> => ({ ok: true, value });
const err = <E>(error: E): Result<never, E> => ({ ok: false, error });

// ---------- Errors ----------
type AppError =
  | { type: "validation"; field: string; message: string }
  | { type: "conflict"; message: string }
  | { type: "rate_limited"; retryAfterMs: number }
  | { type: "downstream_unavailable"; service: string }
  | { type: "timeout"; operation: string; ms: number };

type HttpRequest = { path: "/payments"; body: unknown };
type HttpResponse = { status: number; body: unknown };

// ---------- Domain ----------
type Currency = "VND" | "USD";

type PaymentRequest = {
  userId: string;
  amount: number;
  currency: Currency;
  idempotencyKey: string;
};

type PaymentSuccess = {
  paymentId: string;
  status: "succeeded";
  chargedAmount: number;
  currency: Currency;
};

type PaymentFailure = {
  status: "failed";
  reason: "insufficient_funds" | "gateway_unavailable" | "unknown";
};

type PaymentResponse = PaymentSuccess | PaymentFailure;

// ---------- Helpers: safe parsing ----------
function isRecord(x: unknown): x is Record<string, unknown> {
  return typeof x === "object" && x !== null;
}

function parsePaymentRequest(body: unknown): Result<PaymentRequest, AppError> {
  if (!isRecord(body)) return err({ type: "validation", field: "body", message: "must be object" });

  const userId = body.userId;
  const amount = body.amount;
  const currency = body.currency;
  const idempotencyKey = body.idempotencyKey;

  if (typeof userId !== "string" || userId.length === 0)
    return err({ type: "validation", field: "userId", message: "must be non-empty string" });

  if (typeof amount !== "number" || !Number.isFinite(amount) || amount <= 0)
    return err({ type: "validation", field: "amount", message: "must be number > 0" });

  if (currency !== "VND" && currency !== "USD")
    return err({ type: "validation", field: "currency", message: "must be VND|USD" });

  if (typeof idempotencyKey !== "string" || idempotencyKey.length < 8)
    return err({ type: "validation", field: "idempotencyKey", message: "must be string length >= 8" });

  return ok({ userId, amount, currency, idempotencyKey });
}

// ---------- Fingerprint ----------
// Must ensure "same key but different request" can be detected.
// For kata: stable string fingerprint.
function fingerprintRequest(r: PaymentRequest): string {
  // TODO: pick canonical order and stringify
  // Example: `${userId}|${amount}|${currency}`
  return `${r.userId}|${r.amount}|${r.currency}`;
}

// ---------- Idempotency store ----------
type IdemStatus = "in_progress" | "completed";

type IdemRecord = {
  key: string;
  fingerprint: string;
  status: IdemStatus;
  createdAt: number;
  updatedAt: number;
  expiresAt: number;

  // When in progress: promise for completion (single-flight)
  inFlight?: Promise<PaymentResponse>;

  // When completed: stored response
  response?: PaymentResponse;
};

class IdempotencyStore {
  private byKey = new Map<string, IdemRecord>();

  constructor(private opts: { ttlMs: number }) {}

  cleanup(now: number) {
    for (const [k, rec] of this.byKey) {
      if (rec.expiresAt <= now) this.byKey.delete(k);
    }
  }

  get(key: string, now: number): IdemRecord | undefined {
    this.cleanup(now);
    return this.byKey.get(key);
  }

  // Atomically create record if absent.
  // In real DB: INSERT ... ON CONFLICT DO NOTHING.
  createIfAbsent(key: string, fingerprint: string, now: number): IdemRecord {
    this.cleanup(now);
    const existing = this.byKey.get(key);
    if (existing) return existing;

    const rec: IdemRecord = {
      key,
      fingerprint,
      status: "in_progress",
      createdAt: now,
      updatedAt: now,
      expiresAt: now + this.opts.ttlMs,
    };
    this.byKey.set(key, rec);
    return rec;
  }

  setInFlight(key: string, p: Promise<PaymentResponse>, now: number) {
    const rec = this.byKey.get(key);
    if (!rec) return;
    rec.inFlight = p;
    rec.updatedAt = now;
  }

  complete(key: string, response: PaymentResponse, now: number) {
    const rec = this.byKey.get(key);
    if (!rec) return;
    rec.status = "completed";
    rec.response = response;
    rec.inFlight = undefined;
    rec.updatedAt = now;
  }
}

// ---------- Fake payment gateway (downstream) ----------
function sleep(ms: number): Promise<void> {
  return new Promise((r) => setTimeout(r, ms));
}

async function fakeGatewayCharge(r: PaymentRequest): Promise<PaymentResponse> {
  // simulate latency
  await sleep(80);

  // deterministic-ish failure for kata (so retries are stable)
  if (r.amount % 7 === 0) return { status: "failed", reason: "insufficient_funds" };
  if (r.amount % 11 === 0) return { status: "failed", reason: "gateway_unavailable" };

  return {
    paymentId: "pay_" + Math.random().toString(16).slice(2),
    status: "succeeded",
    chargedAmount: r.amount,
    currency: r.currency,
  };
}

// ---------- Handler ----------
const store = new IdempotencyStore({ ttlMs: 10_000 }); // 10s for demo; prod could be 24h

export async function postPayments(req: HttpRequest): Promise<HttpResponse> {
  const parsed = parsePaymentRequest(req.body);
  if (!parsed.ok) return toHttp(parsed.error);

  const r = parsed.value;
  const now = Date.now();
  const fp = fingerprintRequest(r);

  // 1) create record or get existing
  const rec = store.createIfAbsent(r.idempotencyKey, fp, now);

  // 2) conflict if same key but different request fingerprint
  if (rec.fingerprint !== fp) {
    return toHttp({ type: "conflict", message: "Idempotency-Key reused with different request" });
  }

  // 3) if completed, return stored response
  if (rec.status === "completed" && rec.response) {
    return { status: 200, body: rec.response };
  }

  // 4) if in progress and has inflight promise, await it (single-flight)
  if (rec.status === "in_progress" && rec.inFlight) {
    const res = await rec.inFlight;
    return { status: 200, body: res };
  }

  // 5) start charge exactly once per key
  const inFlight = (async () => {
    // Policy A: store both success and failure as completed result
    const res = await fakeGatewayCharge(r);
    store.complete(r.idempotencyKey, res, Date.now());
    return res;
  })();

  store.setInFlight(r.idempotencyKey, inFlight, now);

  const finalRes = await inFlight;
  return { status: 200, body: finalRes };
}

// ---------- Error mapping ----------
function toHttp(e: AppError): HttpResponse {
  switch (e.type) {
    case "validation":
      return { status: 400, body: { error: e.type, field: e.field, message: e.message } };
    case "conflict":
      return { status: 409, body: { error: e.type, message: e.message } };
    case "rate_limited":
      return { status: 429, body: { error: e.type, retryAfterMs: e.retryAfterMs } };
    case "downstream_unavailable":
      return { status: 503, body: { error: e.type, service: e.service } };
    case "timeout":
      return { status: 504, body: { error: e.type, operation: e.operation, ms: e.ms } };
    default: {
      const _x: never = e;
      return _x;
    }
  }
}

// ---------- Demo ----------
async function demo() {
  // Case 1: first success
  const req1: HttpRequest = {
    path: "/payments",
    body: { userId: "u1", amount: 100, currency: "VND", idempotencyKey: "idem_aaaa1111" },
  };
  const a1 = await postPayments(req1);
  console.log("case1 first:", a1);

  // Case 2: retry same key same request -> same response (succeeded or failed)
  const a2 = await postPayments(req1);
  console.log("case2 retry:", a2);

  // Case 3: concurrent same key -> only one charge; both get same response
  const req2: HttpRequest = {
    path: "/payments",
    body: { userId: "u1", amount: 101, currency: "VND", idempotencyKey: "idem_bbbb2222" },
  };
  const [c1, c2] = await Promise.all([postPayments(req2), postPayments(req2)]);
  console.log("case3 concurrent:", c1, c2);

  // Case 4: same key different amount -> conflict
  const req3a: HttpRequest = {
    path: "/payments",
    body: { userId: "u1", amount: 120, currency: "VND", idempotencyKey: "idem_cccc3333" },
  };
  const req3b: HttpRequest = {
    path: "/payments",
    body: { userId: "u1", amount: 121, currency: "VND", idempotencyKey: "idem_cccc3333" },
  };
  console.log("case4 first:", await postPayments(req3a));
  console.log("case4 conflict:", await postPayments(req3b));

  // Case 5: deterministic failure (amount % 7 === 0) -> retry returns same failure
  const req4: HttpRequest = {
    path: "/payments",
    body: { userId: "u1", amount: 98, currency: "VND", idempotencyKey: "idem_dddd4444" },
  };
  console.log("case5 fail first:", await postPayments(req4));
  console.log("case5 fail retry:", await postPayments(req4));

  // Case 6: TTL expire -> allow new attempt
  await sleep(11_000);
  console.log("case6 after ttl:", await postPayments(req4));
}

if (process.argv[1]?.includes("kata21")) {
  demo().catch((e) => {
    console.error("DEMO FAILED:", e);
    process.exitCode = 1;
  });
}
```

---

## Tasks bạn phải làm

1. **Fingerprint chuẩn** trong `fingerprintRequest()`

   * phải đủ để phát hiện “same key but different request”
2. Đảm bảo **single-flight**: concurrent gọi chỉ charge **1 lần**

   * (hint) `rec.inFlight` phải được set trước khi `await`
3. TTL cleanup chạy đúng, case 6 pass
4. (Stretch) integrate **timeout + abort**:

   * wrap `fakeGatewayCharge` bằng `withTimeout` (Kata 13)
   * nếu timeout: vẫn complete record với failure `{status:"failed", reason:"gateway_unavailable"}` hoặc map sang AppError timeout (nhưng phải consistent)

---

## Definition of Done

* Retry cùng key trả cùng response (byte-for-byte về shape + values, trừ khi bạn cố tình không cache paymentId—nhưng kata yêu cầu cache)
* Same key different request → 409
* Concurrent same key → 1 charge
* TTL expire → cho phép attempt lại

---

## Stretch (rất production)

1. Idempotency key scope theo **(userId + key)** để tránh cross-user collisions
2. Store persistence thật (Redis/Postgres) với atomic insert + row lock
3. “In-progress response”: trả 202 + retry-after thay vì await
4. Attach metadata (requestId) bằng AsyncLocalStorage (Kata 12)
5. Exactly-once-ish: lưu `gatewayRequestId` để reconcile với gateway