# Kata 89 — Schema Evolution: backward/forward compatibility

## Goal

1. Thiết kế event theo chuẩn:

   * **Envelope** có `eventType`, `schemaVersion`, `eventId`, `occurredAt`, `producer`, `traceId`
   * Payload có nguyên tắc: **additive**, không rename/remove bừa
2. Implement:

   * **Upcaster**: v1 → v2 (consumer hiện đại đọc event cũ)
   * **Downcaster**: v2 → v1 (producer mới vẫn publish “compatible subset” khi cần)
3. Compat tests:

   * Consumer v2 đọc được v1
   * Consumer v1 đọc được v2 (bỏ qua field lạ)
   * Rollout order:

     * **Producer v2 first** (consumer v1 vẫn sống)
     * **Consumer v2 first** (producer v1 vẫn sống)
4. “Poison prevention”: mọi schema mismatch được convert thành **degraded** hoặc **DLQ with reason** (không crash loop).

---

## Context (scenario cụ thể)

Event: **`OrderPaid`**

### V1 payload

```json
{ "orderId": "...", "userId": "...", "amountCents": 1000 }
```

### V2 changes (realistic)

* Add `currency` (default `"USD"`)
* Replace `userId` with `customerId` (không được break!)
* Add `paymentMethod` (optional)

=> Bạn phải evolve mà:

* consumer cũ vẫn chạy
* consumer mới đọc được cả v1 và v2

---

## Constraints

* ✅ Không dùng Avro/Protobuf registry để né bài. Dùng JSON + runtime validation (Zod hoặc hand-rolled).
* ✅ Must support:

  * Unknown fields ignored (forward compatibility)
  * Missing fields have defaults (backward compatibility)
* ✅ Có “compat matrix tests”:

  * (P1→C1), (P1→C2), (P2→C1), (P2→C2)
* ✅ Có policy: khi không upcast được → đưa về Kata 88 (quarantine/DLQ), không loop.
* ✅ Không dùng `any`.

---

## Deliverables

```
kata89/
  src/
    envelope.ts
    schemas.ts
    upcaster.ts
    downcaster.ts
    consumer_v1.ts
    consumer_v2.ts
    producer_v1.ts
    producer_v2.ts
    compat_harness.ts
  test/
    compat.test.ts
```

---

# 1) `src/envelope.ts`

```ts
// src/envelope.ts
export type EventEnvelope<TPayload> = {
  eventType: "OrderPaid";
  schemaVersion: 1 | 2;
  eventId: string;
  occurredAtMs: number;
  producer: string; // service name/version
  traceId: string;
  payload: TPayload;
};

export type AnyOrderPaidEnvelope =
  | EventEnvelope<OrderPaidV1Payload> & { schemaVersion: 1 }
  | EventEnvelope<OrderPaidV2Payload> & { schemaVersion: 2 };

export type OrderPaidV1Payload = {
  orderId: string;
  userId: string;
  amountCents: number;
};

export type OrderPaidV2Payload = {
  orderId: string;
  customerId: string;   // renamed conceptually from userId
  amountCents: number;
  currency: "USD" | "VND" | "EUR";
  paymentMethod?: "CARD" | "COD" | "BANK";
};
```

---

# 2) `src/schemas.ts` (runtime parsing without breaking)

Bạn có thể dùng Zod. Nếu không muốn thêm deps, mình đưa hand-rolled parser kiểu “Parse don’t validate”.

```ts
// src/schemas.ts
import type { AnyOrderPaidEnvelope, OrderPaidV1Payload, OrderPaidV2Payload } from "./envelope";

export type ParseResult<T> = { ok: true; value: T } | { ok: false; error: string };

function isObject(x: unknown): x is Record<string, unknown> {
  return typeof x === "object" && x !== null;
}

function readStr(o: Record<string, unknown>, k: string): string | undefined {
  const v = o[k];
  return typeof v === "string" ? v : undefined;
}

function readNum(o: Record<string, unknown>, k: string): number | undefined {
  const v = o[k];
  return typeof v === "number" ? v : undefined;
}

export function parseEnvelope(x: unknown): ParseResult<AnyOrderPaidEnvelope> {
  if (!isObject(x)) return { ok: false, error: "not_object" };

  const eventType = readStr(x, "eventType");
  const schemaVersion = x["schemaVersion"];
  const eventId = readStr(x, "eventId");
  const occurredAtMs = readNum(x, "occurredAtMs");
  const producer = readStr(x, "producer");
  const traceId = readStr(x, "traceId");
  const payload = x["payload"];

  if (eventType !== "OrderPaid") return { ok: false, error: "wrong_eventType" };
  if (schemaVersion !== 1 && schemaVersion !== 2) return { ok: false, error: "bad_schemaVersion" };
  if (!eventId || !occurredAtMs || !producer || !traceId) return { ok: false, error: "missing_meta" };
  if (!isObject(payload)) return { ok: false, error: "payload_not_object" };

  if (schemaVersion === 1) {
    const p = parsePayloadV1(payload);
    if (!p.ok) return p as any;
    return {
      ok: true,
      value: {
        eventType: "OrderPaid",
        schemaVersion: 1,
        eventId,
        occurredAtMs,
        producer,
        traceId,
        payload: p.value,
      },
    };
  }

  const p = parsePayloadV2(payload);
  if (!p.ok) return p as any;
  return {
    ok: true,
    value: {
      eventType: "OrderPaid",
      schemaVersion: 2,
      eventId,
      occurredAtMs,
      producer,
      traceId,
      payload: p.value,
    },
  };
}

export function parsePayloadV1(p: Record<string, unknown>): ParseResult<OrderPaidV1Payload> {
  const orderId = readStr(p, "orderId");
  const userId = readStr(p, "userId");
  const amountCents = readNum(p, "amountCents");
  if (!orderId || !userId || amountCents == null) return { ok: false, error: "v1_missing_fields" };
  return { ok: true, value: { orderId, userId, amountCents } };
}

export function parsePayloadV2(p: Record<string, unknown>): ParseResult<OrderPaidV2Payload> {
  const orderId = readStr(p, "orderId");
  const customerId = readStr(p, "customerId");
  const amountCents = readNum(p, "amountCents");
  const currency = readStr(p, "currency");
  const paymentMethod = readStr(p, "paymentMethod");

  if (!orderId || !customerId || amountCents == null) return { ok: false, error: "v2_missing_fields" };

  const cur = (currency ?? "USD") as any;
  if (cur !== "USD" && cur !== "VND" && cur !== "EUR") return { ok: false, error: "v2_bad_currency" };

  const pm = paymentMethod as any;
  if (pm != null && pm !== "CARD" && pm !== "COD" && pm !== "BANK") return { ok: false, error: "v2_bad_paymentMethod" };

  return { ok: true, value: { orderId, customerId, amountCents, currency: cur, paymentMethod: pm } };
}
```

---

# 3) `src/upcaster.ts` (v1 → v2)

```ts
// src/upcaster.ts
import type { AnyOrderPaidEnvelope, EventEnvelope, OrderPaidV2Payload } from "./envelope";

export type UpcastResult =
  | { ok: true; value: EventEnvelope<OrderPaidV2Payload> & { schemaVersion: 2 } }
  | { ok: false; error: string };

export function upcastToV2(e: AnyOrderPaidEnvelope): UpcastResult {
  if (e.schemaVersion === 2) return { ok: true, value: e };

  // v1 -> v2 mapping:
  // userId => customerId
  // currency default USD
  const v2: EventEnvelope<OrderPaidV2Payload> & { schemaVersion: 2 } = {
    ...e,
    schemaVersion: 2,
    payload: {
      orderId: e.payload.orderId,
      customerId: e.payload.userId,
      amountCents: e.payload.amountCents,
      currency: "USD",
    },
  };

  return { ok: true, value: v2 };
}
```

---

# 4) `src/downcaster.ts` (v2 → v1 “compat subset”)

Ý tưởng: producer v2 vẫn có thể publish **dual** hoặc publish v2 nhưng giữ field v1 mirror. Bài này chọn kiểu **downcast for legacy consumers**.

```ts
// src/downcaster.ts
import type { AnyOrderPaidEnvelope, EventEnvelope, OrderPaidV1Payload } from "./envelope";

export function downcastToV1(e: AnyOrderPaidEnvelope): EventEnvelope<OrderPaidV1Payload> & { schemaVersion: 1 } {
  if (e.schemaVersion === 1) return e;

  return {
    eventType: "OrderPaid",
    schemaVersion: 1,
    eventId: e.eventId,
    occurredAtMs: e.occurredAtMs,
    producer: e.producer,
    traceId: e.traceId,
    payload: {
      orderId: e.payload.orderId,
      userId: e.payload.customerId,
      amountCents: e.payload.amountCents,
    },
  };
}
```

---

# 5) Producers

## `src/producer_v1.ts`

```ts
// src/producer_v1.ts
import type { EventEnvelope, OrderPaidV1Payload } from "./envelope";

export function produceV1(args: {
  eventId: string;
  nowMs: number;
  traceId: string;
  orderId: string;
  userId: string;
  amountCents: number;
}): EventEnvelope<OrderPaidV1Payload> & { schemaVersion: 1 } {
  return {
    eventType: "OrderPaid",
    schemaVersion: 1,
    eventId: args.eventId,
    occurredAtMs: args.nowMs,
    producer: "payments@v1",
    traceId: args.traceId,
    payload: { orderId: args.orderId, userId: args.userId, amountCents: args.amountCents },
  };
}
```

## `src/producer_v2.ts`

```ts
// src/producer_v2.ts
import type { EventEnvelope, OrderPaidV2Payload } from "./envelope";

export function produceV2(args: {
  eventId: string;
  nowMs: number;
  traceId: string;
  orderId: string;
  customerId: string;
  amountCents: number;
  currency?: "USD" | "VND" | "EUR";
  paymentMethod?: "CARD" | "COD" | "BANK";
}): EventEnvelope<OrderPaidV2Payload> & { schemaVersion: 2 } {
  return {
    eventType: "OrderPaid",
    schemaVersion: 2,
    eventId: args.eventId,
    occurredAtMs: args.nowMs,
    producer: "payments@v2",
    traceId: args.traceId,
    payload: {
      orderId: args.orderId,
      customerId: args.customerId,
      amountCents: args.amountCents,
      currency: args.currency ?? "USD",
      paymentMethod: args.paymentMethod,
    },
  };
}
```

---

# 6) Consumers

## `src/consumer_v1.ts` (legacy, ignores unknown fields)

Giả lập consumer v1: chỉ parse envelope v1. Nếu nhận v2, nó **không crash** mà “degrade”: bỏ qua hoặc DLQ (tùy policy). Ở đây ta chọn: **ignore v2** (forward compatible by ignoring).

```ts
// src/consumer_v1.ts
import type { AnyOrderPaidEnvelope } from "./envelope";

export type ConsumeV1Result =
  | { ok: true; applied: true }
  | { ok: true; applied: false; reason: string };

export class ConsumerV1 {
  applied: string[] = [];

  consume(e: AnyOrderPaidEnvelope): ConsumeV1Result {
    if (e.schemaVersion !== 1) {
      // forward compatibility strategy: ignore newer version
      return { ok: true, applied: false, reason: "ignored_newer_schema" };
    }
    this.applied.push(`${e.payload.orderId}:${e.payload.amountCents}`);
    return { ok: true, applied: true };
  }
}
```

## `src/consumer_v2.ts` (modern, upcasts v1)

```ts
// src/consumer_v2.ts
import type { AnyOrderPaidEnvelope } from "./envelope";
import { upcastToV2 } from "./upcaster";

export class ConsumerV2 {
  applied: string[] = [];

  consume(e: AnyOrderPaidEnvelope) {
    const u = upcastToV2(e);
    if (!u.ok) return { ok: false as const, error: u.error };

    // always v2 from here
    const v2 = u.value;
    this.applied.push(`${v2.payload.orderId}:${v2.payload.currency}:${v2.payload.amountCents}:${v2.payload.customerId}`);
    return { ok: true as const };
  }
}
```

---

# 7) Compat harness

## `src/compat_harness.ts`

```ts
// src/compat_harness.ts
import type { AnyOrderPaidEnvelope } from "./envelope";
import { parseEnvelope } from "./schemas";

export function wireThroughJSON(e: unknown): AnyOrderPaidEnvelope {
  // simulate serialize/deserialize over broker
  const raw = JSON.parse(JSON.stringify(e));
  const parsed = parseEnvelope(raw);
  if (!parsed.ok) throw new Error(`parse_failed:${parsed.error}`);
  return parsed.value;
}
```

---

# Tests — `test/compat.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { produceV1 } from "../src/producer_v1";
import { produceV2 } from "../src/producer_v2";
import { wireThroughJSON } from "../src/compat_harness";
import { ConsumerV1 } from "../src/consumer_v1";
import { ConsumerV2 } from "../src/consumer_v2";
import { downcastToV1 } from "../src/downcaster";

describe("Kata 89 - Schema evolution compatibility matrix", () => {
  it("P1 -> C1 works (baseline)", () => {
    const c1 = new ConsumerV1();
    const e1 = wireThroughJSON(
      produceV1({ eventId: "e1", nowMs: 1, traceId: "t", orderId: "o1", userId: "u1", amountCents: 1000 })
    );
    const r = c1.consume(e1);
    expect(r.ok).toBe(true);
    expect(r.applied).toBe(true);
    expect(c1.applied.length).toBe(1);
  });

  it("P1 -> C2 works via upcast (backward compatible)", () => {
    const c2 = new ConsumerV2();
    const e1 = wireThroughJSON(
      produceV1({ eventId: "e2", nowMs: 1, traceId: "t", orderId: "o2", userId: "u2", amountCents: 2000 })
    );
    const r = c2.consume(e1);
    expect(r.ok).toBe(true);
    expect(c2.applied[0]).toContain("USD"); // default currency from upcast
    expect(c2.applied[0]).toContain("u2");  // userId mapped to customerId
  });

  it("P2 -> C1: legacy ignores newer schema (forward compatible behavior)", () => {
    const c1 = new ConsumerV1();
    const e2 = wireThroughJSON(
      produceV2({ eventId: "e3", nowMs: 1, traceId: "t", orderId: "o3", customerId: "c3", amountCents: 3000, currency: "VND" })
    );
    const r = c1.consume(e2);
    expect(r.ok).toBe(true);
    expect(r.applied).toBe(false);
    expect(c1.applied.length).toBe(0);
  });

  it("P2 -> C1 works if producer provides downcast for legacy", () => {
    const c1 = new ConsumerV1();
    const e2 = wireThroughJSON(
      produceV2({ eventId: "e4", nowMs: 1, traceId: "t", orderId: "o4", customerId: "c4", amountCents: 4000, currency: "EUR" })
    );
    const legacy = wireThroughJSON(downcastToV1(e2));
    const r = c1.consume(legacy);
    expect(r.ok).toBe(true);
    expect(r.applied).toBe(true);
    expect(c1.applied[0]).toBe("o4:4000");
  });

  it("P2 -> C2 works (baseline v2)", () => {
    const c2 = new ConsumerV2();
    const e2 = wireThroughJSON(
      produceV2({ eventId: "e5", nowMs: 1, traceId: "t", orderId: "o5", customerId: "c5", amountCents: 5000, currency: "USD", paymentMethod: "CARD" })
    );
    const r = c2.consume(e2);
    expect(r.ok).toBe(true);
    expect(c2.applied[0]).toContain("CARD");
  });
});
```

---

## Checks (Definition of Done)

* Compat matrix tests xanh.
* Consumer v2 đọc v1 (upcast).
* Consumer v1 không chết khi thấy v2 (ignore hoặc downcast path).
* Có quy ước schema:

  * additive changes OK
  * rename = add new field + keep old (deprecate later)
  * remove field = two-phase rollout (expand/contract)

---

## Senior+ Rules of Event Evolution (để tránh poison thật)

1. **Add, don’t change**: thêm field mới, giữ field cũ trong một thời gian.
2. **Rename = add alias**: `userId` + `customerId` song song, consumer prefer new, fallback old.
3. **Never rely on required new fields immediately**: required chỉ khi rollout xong toàn bộ consumers.
4. **Unknown fields must be ignored** (forward compatibility).
5. **Version in envelope** + `upcaster` in consumer side là “default pattern”.
6. **Compat tests are part of CI** (bảo hiểm rẻ nhất).

---

## Stretch (Senior++ thật sự)

1. **3 versions**: v1 → v2 → v3, với deprecate window, prove you can drop v1 safely (contract tests + metrics).
2. **Schema registry lite**: keep JSON Schema files + validate + compatibility check (backward/forward) in CI.
3. **Consumer poison policy**: nếu parse fail → send to quarantine (Kata 88) kèm “schemaVersion seen” + “producer”.
4. **Dual-publish strategy**: producer v2 publish both v1 and v2 eventType (or same type, diff topic) with correlation id.
5. **Field-level defaults** by domain: currency default from user region (but deterministic).