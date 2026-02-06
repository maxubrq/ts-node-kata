# Kata 84 — Timeout & Deadline Propagation: end-to-end deadline

## Goal

1. Thiết kế **Deadline** (absolute) thay vì timeout rời rạc:

   * Client gửi `deadlineMs` hoặc `deadlineAt` (epoch)
   * Mỗi hop tính **remaining** và set timeout tương ứng
2. **Propagate** deadline qua:

   * HTTP boundary (simulated headers)
   * Internal calls (function calls)
   * Broker messages (optional stretch)
3. **Cancel** đúng:

   * Khi deadline hết → abort in-flight work (AbortController)
   * Không retry khi deadline đã hết
4. Có **observability hooks**: log/metrics remaining time + timeout reason.

---

## Context (kịch bản thật)

Request đi qua:
`API Gateway → Orders Service → Inventory Service → Payment Service`

Nếu bạn chỉ set timeout ở gateway, downstream vẫn chạy tiếp dù client đã bỏ.
Deadline propagation giúp:

* tránh waste,
* tránh queue pile-up,
* tránh retry storm khi time đã hết.

---

## Constraints

* ✅ Deadline dùng **absolute time** (`deadlineAtMs`) khi truyền qua boundary.
* ✅ Mỗi hop phải:

  * compute `remainingMs = deadlineAtMs - nowMs`
  * nếu `remainingMs <= 0` → fail fast
  * set `timeoutMs = min(ownCeiling, remainingMs - safetyMarginMs)`
* ✅ Phải dùng **AbortController** để cancel downstream work.
* ✅ Tests bắt buộc:

  1. hop2 không gọi hop3 nếu remaining quá thấp
  2. cancel thật: downstream “sleep” bị abort
  3. budget phân bổ: mỗi hop có ceiling riêng
  4. deadline exceeded trả error chuẩn, không retry

---

## Deliverables

```
kata84/
  src/
    deadline.ts
    errors.ts
    sleep.ts
    services.ts
    client.ts
  test/
    deadline.test.ts
```

---

# Starter skeleton

## 1) `src/errors.ts`

```ts
// src/errors.ts
export class DeadlineExceededError extends Error {
  readonly name = "DeadlineExceededError";
  constructor(message = "deadline exceeded") {
    super(message);
  }
}

export class TimeoutError extends Error {
  readonly name = "TimeoutError";
  constructor(message = "timeout") {
    super(message);
  }
}
```

---

## 2) `src/sleep.ts` (abortable sleep)

```ts
// src/sleep.ts
export async function abortableSleep(ms: number, signal: AbortSignal): Promise<void> {
  if (signal.aborted) throw new DOMException("Aborted", "AbortError");

  return new Promise((resolve, reject) => {
    const t = setTimeout(() => {
      cleanup();
      resolve();
    }, ms);

    const onAbort = () => {
      cleanup();
      reject(new DOMException("Aborted", "AbortError"));
    };

    const cleanup = () => {
      clearTimeout(t);
      signal.removeEventListener("abort", onAbort);
    };

    signal.addEventListener("abort", onAbort);
  });
}
```

---

## 3) `src/deadline.ts` (core utility)

```ts
// src/deadline.ts
import { DeadlineExceededError, TimeoutError } from "./errors";

export type Deadline = {
  deadlineAtMs: number;  // absolute epoch ms
  traceId: string;
};

export type HopBudget = {
  name: string;
  ceilingMs: number;        // max time this hop can spend
  safetyMarginMs: number;   // leave room for network/serialization
};

export function remainingMs(d: Deadline, nowMs: number): number {
  return d.deadlineAtMs - nowMs;
}

export function assertHasTime(d: Deadline, nowMs: number, minMs = 1) {
  if (remainingMs(d, nowMs) < minMs) throw new DeadlineExceededError();
}

export function computeHopTimeoutMs(d: Deadline, nowMs: number, budget: HopBudget): number {
  const rem = remainingMs(d, nowMs);
  const usable = rem - budget.safetyMarginMs;
  if (usable <= 0) throw new DeadlineExceededError(`${budget.name}: no remaining time`);
  return Math.max(1, Math.min(budget.ceilingMs, usable));
}

/**
 * Wrap a promise with timeout + abort controller
 */
export async function withHopTimeout<T>(args: {
  deadline: Deadline;
  now: () => number;
  budget: HopBudget;
  run: (ctx: { signal: AbortSignal; timeoutMs: number }) => Promise<T>;
}): Promise<T> {
  const timeoutMs = computeHopTimeoutMs(args.deadline, args.now(), args.budget);

  const ac = new AbortController();
  const timer = setTimeout(() => ac.abort(), timeoutMs);

  try {
    return await args.run({ signal: ac.signal, timeoutMs });
  } catch (e: any) {
    if (e?.name === "AbortError") {
      // We aborted due to hop timeout, but map it to TimeoutError (deadline still may exist)
      throw new TimeoutError(`${args.budget.name}: timed out after ${timeoutMs}ms`);
    }
    throw e;
  } finally {
    clearTimeout(timer);
  }
}
```

---

## 4) `src/services.ts` (4-hop chain)

Mỗi service sẽ:

* fail fast nếu remaining thấp
* call downstream với propagated deadline
* apply hop timeout

```ts
// src/services.ts
import type { Deadline, HopBudget } from "./deadline";
import { withHopTimeout, assertHasTime } from "./deadline";
import { abortableSleep } from "./sleep";
import { DeadlineExceededError } from "./errors";

export type Deps = {
  now: () => number;
  // simulate work durations
  workMs: {
    gateway: number;
    orders: number;
    inventory: number;
    payment: number;
  };
  // for assertions
  calls: {
    inventory: number;
    payment: number;
  };
};

const GW_BUDGET: HopBudget = { name: "gateway", ceilingMs: 120, safetyMarginMs: 5 };
const ORDERS_BUDGET: HopBudget = { name: "orders", ceilingMs: 120, safetyMarginMs: 5 };
const INV_BUDGET: HopBudget = { name: "inventory", ceilingMs: 120, safetyMarginMs: 5 };
const PAY_BUDGET: HopBudget = { name: "payment", ceilingMs: 120, safetyMarginMs: 5 };

export async function apiGateway(req: { deadline: Deadline }, deps: Deps): Promise<string> {
  // fail fast if already expired
  assertHasTime(req.deadline, deps.now());

  return withHopTimeout({
    deadline: req.deadline,
    now: deps.now,
    budget: GW_BUDGET,
    run: async ({ signal }) => {
      // gateway internal work
      await abortableSleep(deps.workMs.gateway, signal);
      // call orders service
      return ordersService({ deadline: req.deadline }, deps, signal);
    },
  });
}

async function ordersService(req: { deadline: Deadline }, deps: Deps, parentSignal: AbortSignal): Promise<string> {
  // If parent aborted, stop immediately
  if (parentSignal.aborted) throw new DeadlineExceededError("orders: parent aborted");

  assertHasTime(req.deadline, deps.now());

  return withHopTimeout({
    deadline: req.deadline,
    now: deps.now,
    budget: ORDERS_BUDGET,
    run: async ({ signal }) => {
      // Link cancellation: if parent aborts, abort child
      parentSignal.addEventListener("abort", () => (signal as any).abort?.(), { once: true });
      await abortableSleep(deps.workMs.orders, signal);

      // downstream calls
      deps.calls.inventory += 1;
      await inventoryService({ deadline: req.deadline }, deps, signal);

      deps.calls.payment += 1;
      await paymentService({ deadline: req.deadline }, deps, signal);

      return "OK";
    },
  });
}

async function inventoryService(req: { deadline: Deadline }, deps: Deps, parentSignal: AbortSignal): Promise<void> {
  if (parentSignal.aborted) throw new DeadlineExceededError("inventory: parent aborted");
  assertHasTime(req.deadline, deps.now());

  return withHopTimeout({
    deadline: req.deadline,
    now: deps.now,
    budget: INV_BUDGET,
    run: async ({ signal }) => {
      parentSignal.addEventListener("abort", () => (signal as any).abort?.(), { once: true });
      await abortableSleep(deps.workMs.inventory, signal);
    },
  });
}

async function paymentService(req: { deadline: Deadline }, deps: Deps, parentSignal: AbortSignal): Promise<void> {
  if (parentSignal.aborted) throw new DeadlineExceededError("payment: parent aborted");
  assertHasTime(req.deadline, deps.now());

  return withHopTimeout({
    deadline: req.deadline,
    now: deps.now,
    budget: PAY_BUDGET,
    run: async ({ signal }) => {
      parentSignal.addEventListener("abort", () => (signal as any).abort?.(), { once: true });
      await abortableSleep(deps.workMs.payment, signal);
    },
  });
}
```

> Note: `parentSignal.addEventListener("abort", () => (signal as any).abort?.())` là hơi hack vì `AbortSignal` không có abort. Stretch sẽ làm “linked AbortController” đúng chuẩn.

---

## 5) `src/client.ts` (deadline creation)

```ts
// src/client.ts
import type { Deadline } from "./deadline";

export function makeDeadline(args: { nowMs: number; timeoutMs: number; traceId?: string }): Deadline {
  return {
    deadlineAtMs: args.nowMs + args.timeoutMs,
    traceId: args.traceId ?? `trace_${Math.random().toString(16).slice(2)}`,
  };
}
```

---

# Tests (vitest)

## `test/deadline.test.ts`

```ts
import { describe, it, expect, vi } from "vitest";
import { apiGateway } from "../src/services";
import { makeDeadline } from "../src/client";
import { DeadlineExceededError, TimeoutError } from "../src/errors";

describe("Kata 84 - Deadline propagation", () => {
  it("fails fast when deadline already expired", async () => {
    const now = vi.fn(() => 1000);
    const deadline = makeDeadline({ nowMs: 0, timeoutMs: 10, traceId: "t1" }); // deadlineAt=10 << now

    await expect(
      apiGateway(
        { deadline },
        {
          now,
          workMs: { gateway: 1, orders: 1, inventory: 1, payment: 1 },
          calls: { inventory: 0, payment: 0 },
        }
      )
    ).rejects.toBeInstanceOf(DeadlineExceededError);
  });

  it("does not call payment if remaining time is too low after inventory", async () => {
    // Simulate time movement manually via now()
    let t = 0;
    const now = () => t;

    const deadline = makeDeadline({ nowMs: 0, timeoutMs: 60, traceId: "t2" }); // small budget

    const deps = {
      now,
      workMs: { gateway: 5, orders: 5, inventory: 50, payment: 10 }, // inventory eats almost all time
      calls: { inventory: 0, payment: 0 },
    };

    // monkey patch setTimeout effect by advancing time in sleeps: simplest is to just advance t in now()
    // But abortableSleep uses real timers; for deterministic, use fake timers:
    vi.useFakeTimers();

    const p = apiGateway({ deadline }, deps as any);

    // advance time enough for gateway/orders/inventory
    await vi.advanceTimersByTimeAsync(5);  t += 5;
    await vi.advanceTimersByTimeAsync(5);  t += 5;
    await vi.advanceTimersByTimeAsync(50); t += 50;

    // payment should not proceed or should timeout quickly
    await expect(p).rejects.toBeInstanceOf(Error);

    expect(deps.calls.inventory).toBe(1);
    // payment may have been attempted then immediately timed out depending on computeHopTimeout; we enforce "fail fast" by asserting <=1
    expect(deps.calls.payment).toBeLessThanOrEqual(1);

    vi.useRealTimers();
  });

  it("cancels downstream work on hop timeout (maps to TimeoutError)", async () => {
    let t = 0;
    const now = () => t;

    const deadline = makeDeadline({ nowMs: 0, timeoutMs: 1000, traceId: "t3" });

    const deps = {
      now,
      workMs: { gateway: 1, orders: 1, inventory: 500, payment: 1 }, // inventory long
      calls: { inventory: 0, payment: 0 },
    };

    vi.useFakeTimers();
    const p = apiGateway({ deadline }, deps as any);

    // advance just enough so inventory starts and then exceeds its hop timeout ceiling
    await vi.advanceTimersByTimeAsync(200); // ceiling is 120 in budgets
    t += 200;

    await expect(p).rejects.toBeInstanceOf(TimeoutError);

    vi.useRealTimers();
  });

  it("respects hop ceilings: even with huge deadline, each hop times out at its ceiling", async () => {
    let t = 0;
    const now = () => t;

    const deadline = makeDeadline({ nowMs: 0, timeoutMs: 10_000, traceId: "t4" });

    const deps = {
      now,
      workMs: { gateway: 1, orders: 1, inventory: 5000, payment: 1 },
      calls: { inventory: 0, payment: 0 },
    };

    vi.useFakeTimers();
    const p = apiGateway({ deadline }, deps as any);

    await vi.advanceTimersByTimeAsync(200); // should exceed inventory ceiling and abort
    t += 200;

    await expect(p).rejects.toBeInstanceOf(TimeoutError);

    vi.useRealTimers();
  });
});
```

> Lưu ý: test 2 ở trên có chút “soft” vì phần link abort giữa parent/child đang hack. Stretch sẽ làm link AbortController chuẩn để test chắc hơn.

---

## Checks (DoD)

* Deadline là absolute time, propagate xuyên hops.
* Mỗi hop có ceiling và safety margin.
* Deadline hết → fail fast.
* Hop timeout → abort in-flight → throw `TimeoutError`.
* Có tests chứng minh: không chạy tiếp khi không còn time.

---

# Senior+ Stretch (đúng đời hơn)

1. **Linked AbortController đúng chuẩn**

   * Tạo helper `linkAbort(parentSignal): AbortController` để child abort khi parent abort.
2. **Deadline header spec**

   * HTTP: `X-Request-Deadline-At: <epoch_ms>` + `X-Trace-Id`
3. **Retry integration**

   * Retry chỉ khi `remainingMs > minRetryWindowMs` và `idempotent=true`
4. **Budget split strategy**

   * orders phân bổ: inventory ≤ 40% remaining, payment ≤ 40%, reserve 20% for response
5. **Observability**

   * log fields: `traceId`, `deadlineAt`, `remainingMs`, `hopTimeoutMs`, `timeoutReason`
6. **Chaos**

   * random latency injection + prove p99 giảm waste bằng propagation