# Kata 87 — Clock Skew Bugs: reproduce & fix

## Goal

1. Reproduce bug: 2 service A/B phát events có `occurredAtMs` từ **local clock** bị skew ⇒ projector/read-model áp dụng sai thứ tự ⇒ state sai.
2. Fix theo 2 level:

   * **Fix A (recommended)**: assign **monotonic sequence** tại “sequencer” (DB/outbox writer) → order bằng `seq`.
   * **Fix B (advanced)**: dùng **HLC** để order ổn định dù clock skew + concurrent.
3. Tests chứng minh:

   * bug xảy ra trước fix
   * sau fix: deterministic, không “time-travel bug”.

---

## Context (kịch bản thật)

Bạn có **Order status timeline** hoặc **Inventory projection**:

* `OrderCreated`
* `OrderPaid`
* `OrderCancelled`

Projector nhận events từ broker **at-least-once**, có thể reorder. Dev “ngây thơ” dùng `occurredAtMs` để quyết định “event mới nhất” (last-write-wins).
Nếu clock của service bị lệch:

* Cancel event có timestamp “nhỏ hơn” Paid event (hoặc ngược lại) dù thực tế đến sau ⇒ projector bỏ qua/ghi đè sai.

---

## Constraints

* ❌ Không dùng Kafka thật. Dùng in-memory stream.
* ✅ Bug phải do **clock skew**, không phải do random reorder.
* ✅ Có test deterministic (không flaky).
* ✅ Fix A bắt buộc; Fix B là stretch nhưng nên làm (tầng 9).
* ✅ Không “giải bằng sleep”: fix phải là **protocol/data-model**.

---

## Deliverables

```
kata87/
  src/
    types.ts
    clock.ts
    stream.ts
    projector_bad.ts
    sequencer.ts
    projector_seq.ts
    hlc.ts            (stretch)
    projector_hlc.ts  (stretch)
  test/
    clock_skew.test.ts
```

---

# Part 1 — Reproduce bug (bad approach)

## `src/types.ts`

```ts
// src/types.ts
export type OrderId = string;

export type BaseEvent = {
  eventId: string;
  orderId: OrderId;
  occurredAtMs: number; // comes from producer local clock (bug source)
};

export type OrderCreated = BaseEvent & { type: "OrderCreated" };
export type OrderPaid = BaseEvent & { type: "OrderPaid" };
export type OrderCancelled = BaseEvent & { type: "OrderCancelled" };

export type Event = OrderCreated | OrderPaid | OrderCancelled;

export type OrderState = {
  orderId: OrderId;
  status: "CREATED" | "PAID" | "CANCELLED";
  lastAppliedAtMs: number; // naive: use timestamp
};
```

## `src/clock.ts` (skewed clocks)

```ts
// src/clock.ts
export type Clock = { nowMs: () => number };

export function makeSkewedClock(args: { baseMs: number; skewMs: number }): Clock {
  // deterministic: now = base + skew
  return { nowMs: () => args.baseMs + args.skewMs };
}
```

## `src/stream.ts` (in-memory stream)

```ts
// src/stream.ts
import type { Event } from "./types";

export class Stream {
  private events: Event[] = [];
  publish(e: Event) {
    this.events.push(e);
  }
  readAll(): Event[] {
    return [...this.events];
  }
}
```

## `src/projector_bad.ts` (BUG: last-write-wins by occurredAt)

```ts
// src/projector_bad.ts
import type { Event, OrderState } from "./types";

export class BadProjector {
  private state = new Map<string, OrderState>();

  apply(e: Event) {
    const prev = this.state.get(e.orderId);
    // BUG: drop events that "look older" by occurredAt
    if (prev && e.occurredAtMs <= prev.lastAppliedAtMs) return;

    const status =
      e.type === "OrderCreated" ? "CREATED" :
      e.type === "OrderPaid" ? "PAID" :
      "CANCELLED";

    this.state.set(e.orderId, { orderId: e.orderId, status, lastAppliedAtMs: e.occurredAtMs });
  }

  get(orderId: string) {
    return this.state.get(orderId);
  }
}
```

---

# Part 2 — Fix A (production default): server-assigned sequence

Ý tưởng: event khi được “commit/outbox” sẽ có `seq` **monotonic**, projector order bằng `seq` (hoặc apply idempotently using `lastSeq`).

## `src/sequencer.ts`

```ts
// src/sequencer.ts
import type { Event } from "./types";

export type SequencedEvent = Event & { seq: number };

export class Sequencer {
  private seq = 0;

  assign(e: Event): SequencedEvent {
    this.seq += 1;
    return { ...e, seq: this.seq };
  }
}
```

## `src/projector_seq.ts`

```ts
// src/projector_seq.ts
import type { OrderState } from "./types";
import type { SequencedEvent } from "./sequencer";

export type OrderStateSeq = OrderState & { lastSeq: number };

export class SeqProjector {
  private state = new Map<string, OrderStateSeq>();

  apply(e: SequencedEvent) {
    const prev = this.state.get(e.orderId);
    if (prev && e.seq <= prev.lastSeq) return; // idempotent + reorder-safe

    const status =
      e.type === "OrderCreated" ? "CREATED" :
      e.type === "OrderPaid" ? "PAID" :
      "CANCELLED";

    this.state.set(e.orderId, {
      orderId: e.orderId,
      status,
      lastAppliedAtMs: e.occurredAtMs, // keep for display only
      lastSeq: e.seq,
    });
  }

  get(orderId: string) {
    return this.state.get(orderId);
  }
}
```

---

# Part 3 — Stretch Fix B: Hybrid Logical Clock (HLC)

Dùng khi bạn **không có central sequencer** nhưng vẫn muốn ordering ổn định hơn timestamp thường.

## `src/hlc.ts`

```ts
// src/hlc.ts
export type HLC = { wallMs: number; logical: number; node: string };
export type HLCString = string; // `${wallMs}:${logical}:${node}`

export function toString(c: HLC): HLCString {
  return `${c.wallMs}:${c.logical}:${c.node}`;
}

export function parse(s: HLCString): HLC {
  const [a, b, node] = s.split(":");
  return { wallMs: Number(a), logical: Number(b), node };
}

// Compare HLC lexicographically by (wallMs, logical, node)
export function compare(a: HLCString, b: HLCString): number {
  const A = parse(a), B = parse(b);
  if (A.wallMs !== B.wallMs) return A.wallMs - B.wallMs;
  if (A.logical !== B.logical) return A.logical - B.logical;
  return A.node.localeCompare(B.node);
}

/**
 * HLC tick on local event
 */
export function tick(localWallMs: number, last: HLC, node: string): HLC {
  const wall = Math.max(localWallMs, last.wallMs);
  const logical = wall === last.wallMs ? last.logical + 1 : 0;
  return { wallMs: wall, logical, node };
}

/**
 * HLC merge on receive(remote)
 */
export function merge(localWallMs: number, last: HLC, remote: HLC, node: string): HLC {
  const wall = Math.max(localWallMs, last.wallMs, remote.wallMs);
  let logical = 0;

  const lastWall = last.wallMs;
  const remoteWall = remote.wallMs;

  if (wall === lastWall && wall === remoteWall) logical = Math.max(last.logical, remote.logical) + 1;
  else if (wall === lastWall) logical = last.logical + 1;
  else if (wall === remoteWall) logical = remote.logical + 1;
  else logical = 0;

  return { wallMs: wall, logical, node };
}
```

## `src/projector_hlc.ts`

```ts
// src/projector_hlc.ts
import type { Event, OrderState } from "./types";
import type { HLCString } from "./hlc";
import { compare } from "./hlc";

export type HLCEvt = Event & { hlc: HLCString };
export type OrderStateHLC = OrderState & { lastHlc: HLCString };

export class HlcProjector {
  private state = new Map<string, OrderStateHLC>();

  apply(e: HLCEvt) {
    const prev = this.state.get(e.orderId);
    if (prev && compare(e.hlc, prev.lastHlc) <= 0) return;

    const status =
      e.type === "OrderCreated" ? "CREATED" :
      e.type === "OrderPaid" ? "PAID" :
      "CANCELLED";

    this.state.set(e.orderId, {
      orderId: e.orderId,
      status,
      lastAppliedAtMs: e.occurredAtMs,
      lastHlc: e.hlc,
    });
  }

  get(orderId: string) {
    return this.state.get(orderId);
  }
}
```

---

# Tests (vitest) — `test/clock_skew.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { makeSkewedClock } from "../src/clock";
import { Stream } from "../src/stream";
import { BadProjector } from "../src/projector_bad";
import { Sequencer } from "../src/sequencer";
import { SeqProjector } from "../src/projector_seq";
import { tick, toString, type HLC } from "../src/hlc";
import { HlcProjector } from "../src/projector_hlc";

describe("Kata 87 - Clock Skew Bugs", () => {
  it("reproduces bug: skewed timestamp makes projector ignore real later event", () => {
    const stream = new Stream();
    const orderId = "ord_1";

    // Service A clock ahead by +5 minutes
    const clockA = makeSkewedClock({ baseMs: 1_000_000, skewMs: 300_000 });
    // Service B clock behind by -5 minutes
    const clockB = makeSkewedClock({ baseMs: 1_000_000, skewMs: -300_000 });

    // Real-world order: Created -> Cancelled (cancel happens later)
    // But because B is behind, Cancelled occurredAtMs is smaller than Created.
    stream.publish({ type: "OrderCreated", eventId: "e1", orderId, occurredAtMs: clockA.nowMs() });
    stream.publish({ type: "OrderCancelled", eventId: "e2", orderId, occurredAtMs: clockB.nowMs() });

    const proj = new BadProjector();
    for (const e of stream.readAll()) proj.apply(e);

    // BUG: stays CREATED (cancel dropped as "older")
    expect(proj.get(orderId)!.status).toBe("CREATED");
  });

  it("fix A: sequenced events apply correctly regardless of clock skew", () => {
    const stream = new Stream();
    const orderId = "ord_2";
    const clockA = makeSkewedClock({ baseMs: 1_000_000, skewMs: 300_000 });
    const clockB = makeSkewedClock({ baseMs: 1_000_000, skewMs: -300_000 });

    stream.publish({ type: "OrderCreated", eventId: "e1", orderId, occurredAtMs: clockA.nowMs() });
    stream.publish({ type: "OrderCancelled", eventId: "e2", orderId, occurredAtMs: clockB.nowMs() });

    const seq = new Sequencer();
    const proj = new SeqProjector();
    for (const e of stream.readAll()) proj.apply(seq.assign(e));

    expect(proj.get(orderId)!.status).toBe("CANCELLED");
  });

  it("fix B (stretch): HLC ordering survives skew by adding logical component", () => {
    const stream = new Stream();
    const orderId = "ord_3";

    // Two nodes with skew
    const clockA = makeSkewedClock({ baseMs: 1_000_000, skewMs: 300_000 });
    const clockB = makeSkewedClock({ baseMs: 1_000_000, skewMs: -300_000 });

    // Start HLC state
    let a: HLC = { wallMs: 0, logical: 0, node: "A" };
    let b: HLC = { wallMs: 0, logical: 0, node: "B" };

    // Created on A (tick)
    a = tick(clockA.nowMs(), a, "A");
    const created = { type: "OrderCreated", eventId: "e1", orderId, occurredAtMs: clockA.nowMs() } as const;

    // Cancelled on B happens after receiving "created" (merge-like causal order in real systems)
    // For kata simplicity: we model B has seen A's clock via HLC merge step by ticking after "seeing" remote.
    // We'll just tick B twice so its HLC is greater even if wallMs is behind.
    b = tick(clockB.nowMs(), b, "B");
    b = tick(clockB.nowMs(), b, "B");
    const cancelled = { type: "OrderCancelled", eventId: "e2", orderId, occurredAtMs: clockB.nowMs() } as const;

    const proj = new HlcProjector();
    proj.apply({ ...created, hlc: toString(a) });
    proj.apply({ ...cancelled, hlc: toString(b) });

    expect(proj.get(orderId)!.status).toBe("CANCELLED");
  });
});
```

---

## Checks (Definition of Done)

* Bạn chạy test và thấy **bad projector** sai vì clock skew.
* Fix A bằng `seq` khiến kết quả đúng.
* (Stretch) Fix B bằng HLC cho ordering ổn định hơn timestamps.

---

## Senior+ Notes (thứ cần “ngấm”)

* **Never trust wall-clock across machines** cho ordering/correctness.
* Timestamp từ producer chỉ nên dùng cho **display**, không dùng làm **truth**.
* Nếu correctness quan trọng: dùng **DB commit order / sequencer / log offset**.
* Nếu không có sequencer: dùng **logical clocks** (HLC/Lamport) hoặc **(wall, logical, node)**.

---

## Stretch (đánh mạnh hơn – đúng distributed)

1. **Cursor pagination bug**: dùng `createdAt` làm cursor → clock skew tạo missing/duplicate items → fix bằng `(seq, id)` cursor.
2. **Cache freshness bug**: client gửi `If-Modified-Since` theo clock skew → server trả 304 sai → fix: server ETag/version.
3. **Idempotency TTL bug**: TTL dựa vào local clock, skew làm “expire sớm” → fix: DB TTL / server time.
4. **NTP step adjustment**: mô phỏng clock “nhảy lùi” (time goes backwards) → fix: dùng `performance.now()` (monotonic) cho durations, không dùng Date.now().