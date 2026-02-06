# Kata 81 — Retry Budget: Limited retries + jitter (anti-storm)

## Goal (năng lực phải đạt)

1. Thiết kế cơ chế retry **có ngân sách (budget)** thay vì “cố cho chạy”.
2. Retry phải:

   * có **deadline** / **time budget** (không retry vô hạn theo thời gian)
   * có **attempt budget** (không vượt số lần)
   * có **jitter** (tránh thundering herd)
   * có **classify** (chỉ retry lỗi đáng retry)
   * tôn trọng **Retry-After** khi có
3. Có **observability hooks** (counter + reason) để nhìn thấy retry đang làm gì.

### Context (kịch bản production)

Bạn gọi downstream (HTTP / DB proxy / gRPC gateway). Downstream bị:

* timeout ngẫu nhiên,
* 503/429,
* connection reset,
* hoặc trả 400 do bạn sai.

Nếu bạn retry “bừa”, hệ thống sẽ tự DDOS chính nó. Senior+ phải chứng minh: **retry là controlled risk**.

---

## Constraints (ép trade-off đúng)

* ❌ Không dùng thư viện retry có sẵn (`p-retry`, `async-retry`, …).
* ✅ Phải có **decorrelated jitter** hoặc **full jitter** (không được “sleep fixed 100ms”).
* ✅ Phải có **global retry budget** (token bucket / leaky bucket) *và* **per-request budget**.
* ✅ Phải có **deadline propagation**: retry không được vượt quá `deadlineMs`.
* ✅ Phải có test cho:

  1. jitter hoạt động (không cùng delay),
  2. budget cạn thì stop,
  3. non-retryable error không retry,
  4. 429 + Retry-After được tôn trọng.
* ✅ Không được “nuốt lỗi”: phải trả về **structured result** (ok / error + attempts + timings).

---

## Deliverables (những file phải tạo)

```
kata81/
  src/
    retryBudget.ts
    classify.ts
    sleep.ts
    demoClient.ts
  test/
    retryBudget.test.ts
  README.md (optional nhưng nên có)
```

---

## Definition of Done (Checks)

* Có `createRetryBudget()` (global) + `withRetry()` (per-call).
* `withRetry()` trả về `RetryOutcome<T>` chứa: attempts, totalDelayMs, elapsedMs, stopReason.
* Delay dùng jitter, base/backoff hợp lý.
* Có `metrics` callback hook: `onAttempt`, `onRetryScheduled`, `onGiveUp`.
* Tests xanh.

---

# Starter skeleton (copy là chạy)

## 1) `src/classify.ts`

Phân loại lỗi: retryable vs non-retryable. Senior+ phải “rõ ràng lý do”.

```ts
// src/classify.ts
export type RetryDecision =
  | { retry: true; reason: "timeout" | "network" | "http_5xx" | "http_429" | "http_503" }
  | { retry: false; reason: "bad_request" | "unauthorized" | "forbidden" | "not_found" | "conflict" | "unknown" };

export type HttpLikeError = {
  kind: "http";
  status: number;
  retryAfterMs?: number; // parsed from Retry-After if present
  message?: string;
};

export type TimeoutError = { kind: "timeout"; message?: string };
export type NetworkError = { kind: "network"; code?: string; message?: string };

export type DownstreamError = HttpLikeError | TimeoutError | NetworkError | { kind: "unknown"; message?: string };

export function classify(err: DownstreamError): RetryDecision {
  if (err.kind === "timeout") return { retry: true, reason: "timeout" };
  if (err.kind === "network") return { retry: true, reason: "network" };
  if (err.kind === "http") {
    const s = err.status;
    if (s === 429) return { retry: true, reason: "http_429" };
    if (s === 503) return { retry: true, reason: "http_503" };
    if (s >= 500 && s <= 599) return { retry: true, reason: "http_5xx" };
    if (s === 400) return { retry: false, reason: "bad_request" };
    if (s === 401) return { retry: false, reason: "unauthorized" };
    if (s === 403) return { retry: false, reason: "forbidden" };
    if (s === 404) return { retry: false, reason: "not_found" };
    if (s === 409) return { retry: false, reason: "conflict" };
    return { retry: false, reason: "unknown" };
  }
  return { retry: false, reason: "unknown" };
}
```

## 2) `src/sleep.ts`

Cho phép test fake timers.

```ts
// src/sleep.ts
export type Sleeper = (ms: number) => Promise<void>;

export const realSleep: Sleeper = (ms) =>
  new Promise((resolve) => setTimeout(resolve, ms));
```

## 3) `src/retryBudget.ts` (core)

### Thiết kế

* **Global budget**: token bucket `capacity`, refill theo thời gian.
* **Per-request**: `maxAttempts`, `deadlineMs`, `maxTotalDelayMs`.
* **Backoff + jitter**: dùng **decorrelated jitter** (AWS style) để tránh đồng bộ.

```ts
// src/retryBudget.ts
import type { Sleeper } from "./sleep";
import type { DownstreamError } from "./classify";
import { classify } from "./classify";

export type StopReason =
  | "success"
  | "non_retryable"
  | "attempts_exhausted"
  | "deadline_exceeded"
  | "delay_budget_exhausted"
  | "global_budget_exhausted";

export type RetryOutcome<T> =
  | {
      ok: true;
      value: T;
      attempts: number;
      elapsedMs: number;
      totalDelayMs: number;
      stopReason: "success";
    }
  | {
      ok: false;
      error: DownstreamError;
      attempts: number;
      elapsedMs: number;
      totalDelayMs: number;
      stopReason: Exclude<StopReason, "success">;
      lastDecision?: ReturnType<typeof classify>;
    };

export type RetryMetrics = {
  onAttempt?: (info: { attempt: number }) => void;
  onRetryScheduled?: (info: { attempt: number; delayMs: number; reason: string }) => void;
  onGiveUp?: (info: { attempts: number; stopReason: StopReason; reason?: string }) => void;
};

export type RetryPolicy = {
  maxAttempts: number;        // includes initial attempt
  deadlineMs: number;         // total wall-clock budget
  maxTotalDelayMs: number;    // sum of sleeps cannot exceed this
  baseDelayMs: number;        // e.g. 50
  maxDelayMs: number;         // e.g. 2_000
  jitter: "decorrelated" | "full";
};

export type RetryBudgetConfig = {
  capacity: number;      // max tokens
  refillPerSecond: number; // tokens added per second
};

export type RetryBudget = {
  tryConsume: (tokens?: number) => boolean;
  snapshot: () => { tokens: number; capacity: number };
};

export function createRetryBudget(cfg: RetryBudgetConfig, now: () => number = () => Date.now()): RetryBudget {
  let tokens = cfg.capacity;
  let lastRefillAt = now();

  function refill() {
    const t = now();
    const elapsedMs = t - lastRefillAt;
    if (elapsedMs <= 0) return;
    const add = (elapsedMs / 1000) * cfg.refillPerSecond;
    if (add > 0) {
      tokens = Math.min(cfg.capacity, tokens + add);
      lastRefillAt = t;
    }
  }

  return {
    tryConsume(tokensNeeded = 1) {
      refill();
      if (tokens >= tokensNeeded) {
        tokens -= tokensNeeded;
        return true;
      }
      return false;
    },
    snapshot() {
      refill();
      return { tokens, capacity: cfg.capacity };
    },
  };
}

// Jitter strategies
function fullJitter(base: number, cap: number): number {
  const max = Math.min(cap, base);
  return Math.floor(Math.random() * (max + 1));
}

// Decorrelated jitter (AWS): sleep = min(cap, random(base, prev*3))
function decorrelatedJitter(base: number, cap: number, prev: number): number {
  const min = base;
  const max = Math.min(cap, Math.max(base, prev * 3));
  const r = min + Math.random() * (max - min);
  return Math.floor(r);
}

export async function withRetry<T>(args: {
  run: (ctx: { attempt: number }) => Promise<T>;
  policy: RetryPolicy;
  budget: RetryBudget;
  sleep: Sleeper;
  now?: () => number;
  metrics?: RetryMetrics;
  // Optional: treat only idempotent operations as retryable
  idempotent: boolean;
}): Promise<RetryOutcome<T>> {
  const now = args.now ?? (() => Date.now());
  const startedAt = now();

  let attempt = 0;
  let totalDelayMs = 0;
  let prevDelay = args.policy.baseDelayMs;

  while (attempt < args.policy.maxAttempts) {
    attempt += 1;
    args.metrics?.onAttempt?.({ attempt });

    try {
      const value = await args.run({ attempt });
      return {
        ok: true,
        value,
        attempts: attempt,
        elapsedMs: now() - startedAt,
        totalDelayMs,
        stopReason: "success",
      };
    } catch (e) {
      const err = e as DownstreamError;
      const decision = classify(err);

      // If not idempotent, be conservative: only retry safe classes (timeout/network/503/5xx) if you accept risk.
      if (!args.idempotent && decision.retry) {
        // Senior+ rule: default give up unless explicitly allowed.
        return {
          ok: false,
          error: err,
          attempts: attempt,
          elapsedMs: now() - startedAt,
          totalDelayMs,
          stopReason: "non_retryable",
          lastDecision: { retry: false, reason: "unknown" },
        };
      }

      if (!decision.retry) {
        args.metrics?.onGiveUp?.({ attempts: attempt, stopReason: "non_retryable", reason: decision.reason });
        return {
          ok: false,
          error: err,
          attempts: attempt,
          elapsedMs: now() - startedAt,
          totalDelayMs,
          stopReason: "non_retryable",
          lastDecision: decision,
        };
      }

      if (attempt >= args.policy.maxAttempts) {
        args.metrics?.onGiveUp?.({ attempts: attempt, stopReason: "attempts_exhausted", reason: decision.reason });
        return {
          ok: false,
          error: err,
          attempts: attempt,
          elapsedMs: now() - startedAt,
          totalDelayMs,
          stopReason: "attempts_exhausted",
          lastDecision: decision,
        };
      }

      const elapsedMs = now() - startedAt;
      if (elapsedMs >= args.policy.deadlineMs) {
        args.metrics?.onGiveUp?.({ attempts: attempt, stopReason: "deadline_exceeded", reason: decision.reason });
        return {
          ok: false,
          error: err,
          attempts: attempt,
          elapsedMs,
          totalDelayMs,
          stopReason: "deadline_exceeded",
          lastDecision: decision,
        };
      }

      // Global retry budget token: every retry "costs" 1 token.
      if (!args.budget.tryConsume(1)) {
        args.metrics?.onGiveUp?.({ attempts: attempt, stopReason: "global_budget_exhausted", reason: decision.reason });
        return {
          ok: false,
          error: err,
          attempts: attempt,
          elapsedMs: now() - startedAt,
          totalDelayMs,
          stopReason: "global_budget_exhausted",
          lastDecision: decision,
        };
      }

      // Decide delay: honor Retry-After when present (429) but still cap.
      let delayMs: number;
      if (err.kind === "http" && err.status === 429 && typeof err.retryAfterMs === "number") {
        delayMs = Math.min(args.policy.maxDelayMs, Math.max(0, err.retryAfterMs));
      } else {
        const base = Math.min(args.policy.maxDelayMs, args.policy.baseDelayMs * Math.pow(2, attempt - 1));
        delayMs =
          args.policy.jitter === "full"
            ? fullJitter(base, args.policy.maxDelayMs)
            : decorrelatedJitter(args.policy.baseDelayMs, args.policy.maxDelayMs, prevDelay);
      }

      // Per-request delay budget
      if (totalDelayMs + delayMs > args.policy.maxTotalDelayMs) {
        args.metrics?.onGiveUp?.({
          attempts: attempt,
          stopReason: "delay_budget_exhausted",
          reason: decision.reason,
        });
        return {
          ok: false,
          error: err,
          attempts: attempt,
          elapsedMs: now() - startedAt,
          totalDelayMs,
          stopReason: "delay_budget_exhausted",
          lastDecision: decision,
        };
      }

      // Deadline check before sleeping
      const remaining = args.policy.deadlineMs - (now() - startedAt);
      if (delayMs > remaining) {
        args.metrics?.onGiveUp?.({ attempts: attempt, stopReason: "deadline_exceeded", reason: decision.reason });
        return {
          ok: false,
          error: err,
          attempts: attempt,
          elapsedMs: now() - startedAt,
          totalDelayMs,
          stopReason: "deadline_exceeded",
          lastDecision: decision,
        };
      }

      args.metrics?.onRetryScheduled?.({ attempt, delayMs, reason: decision.reason });
      await args.sleep(delayMs);
      totalDelayMs += delayMs;
      prevDelay = delayMs;
      continue;
    }
  }

  // Should be unreachable, but keep safe:
  return {
    ok: false,
    error: { kind: "unknown", message: "unreachable" },
    attempts: attempt,
    elapsedMs: now() - startedAt,
    totalDelayMs,
    stopReason: "attempts_exhausted",
  };
}
```

## 4) `src/demoClient.ts` (fake downstream)

Dùng để chạy thủ công + demo retry.

```ts
// src/demoClient.ts
import { withRetry, createRetryBudget } from "./retryBudget";
import { realSleep } from "./sleep";
import type { DownstreamError } from "./classify";

export async function flakyDownstream(): Promise<string> {
  const r = Math.random();
  if (r < 0.2) throw { kind: "timeout" } satisfies DownstreamError;
  if (r < 0.4) throw { kind: "network", code: "ECONNRESET" } satisfies DownstreamError;
  if (r < 0.6) throw { kind: "http", status: 503 } satisfies DownstreamError;
  if (r < 0.7) throw { kind: "http", status: 400 } satisfies DownstreamError;
  return "OK";
}

async function main() {
  const budget = createRetryBudget({ capacity: 10, refillPerSecond: 2 });

  const outcome = await withRetry({
    run: async () => flakyDownstream(),
    idempotent: true,
    policy: {
      maxAttempts: 5,
      deadlineMs: 1500,
      maxTotalDelayMs: 1200,
      baseDelayMs: 50,
      maxDelayMs: 400,
      jitter: "decorrelated",
    },
    budget,
    sleep: realSleep,
    metrics: {
      onAttempt: ({ attempt }) => console.log("attempt", attempt),
      onRetryScheduled: ({ delayMs, reason }) => console.log("retry in", delayMs, "reason", reason),
      onGiveUp: ({ stopReason, reason }) => console.log("give up", stopReason, reason),
    },
  });

  console.log(outcome);
}

if (require.main === module) {
  main().catch(console.error);
}
```

---

# Tests (vitest) — bắt buộc có

## `test/retryBudget.test.ts`

```ts
// test/retryBudget.test.ts
import { describe, it, expect, vi } from "vitest";
import { withRetry, createRetryBudget } from "../src/retryBudget";
import type { DownstreamError } from "../src/classify";

describe("Kata 81 - Retry Budget", () => {
  it("does not retry on non-retryable (400)", async () => {
    const budget = createRetryBudget({ capacity: 10, refillPerSecond: 10 }, () => 0);

    let calls = 0;
    const outcome = await withRetry({
      idempotent: true,
      run: async () => {
        calls++;
        throw { kind: "http", status: 400 } satisfies DownstreamError;
      },
      policy: {
        maxAttempts: 5,
        deadlineMs: 1000,
        maxTotalDelayMs: 1000,
        baseDelayMs: 10,
        maxDelayMs: 50,
        jitter: "full",
      },
      budget,
      sleep: async () => {},
      now: () => 0,
    });

    expect(calls).toBe(1);
    expect(outcome.ok).toBe(false);
    if (!outcome.ok) expect(outcome.stopReason).toBe("non_retryable");
  });

  it("honors Retry-After on 429", async () => {
    const now = vi.fn(() => 0);
    const sleep = vi.fn(async (_ms: number) => {});
    const budget = createRetryBudget({ capacity: 10, refillPerSecond: 10 }, now);

    let calls = 0;
    const outcome = await withRetry({
      idempotent: true,
      run: async () => {
        calls++;
        if (calls === 1) {
          throw { kind: "http", status: 429, retryAfterMs: 123 } satisfies DownstreamError;
        }
        return "ok";
      },
      policy: {
        maxAttempts: 3,
        deadlineMs: 1000,
        maxTotalDelayMs: 1000,
        baseDelayMs: 10,
        maxDelayMs: 500,
        jitter: "decorrelated",
      },
      budget,
      sleep,
      now,
    });

    expect(outcome.ok).toBe(true);
    expect(sleep).toHaveBeenCalledTimes(1);
    expect(sleep).toHaveBeenCalledWith(123);
  });

  it("stops when global budget exhausted", async () => {
    const t = { v: 0 };
    const now = () => t.v;
    const budget = createRetryBudget({ capacity: 1, refillPerSecond: 0 }, now); // only 1 retry token

    const sleep = async (ms: number) => {
      t.v += ms;
    };

    let calls = 0;
    const outcome = await withRetry({
      idempotent: true,
      run: async () => {
        calls++;
        throw { kind: "http", status: 503 } satisfies DownstreamError;
      },
      policy: {
        maxAttempts: 10,
        deadlineMs: 10_000,
        maxTotalDelayMs: 10_000,
        baseDelayMs: 10,
        maxDelayMs: 50,
        jitter: "full",
      },
      budget,
      sleep,
      now,
    });

    // attempt 1 fails, 1 retry token allows scheduling ONE retry, next failure should stop
    expect(calls).toBeGreaterThanOrEqual(2);
    expect(outcome.ok).toBe(false);
    if (!outcome.ok) expect(outcome.stopReason).toBe("global_budget_exhausted");
  });

  it("uses jitter: two runs should not produce identical delay sequences (probabilistic)", async () => {
    // Make Math.random deterministic-ish but different per run
    const originalRandom = Math.random;

    const collect = async (seed: number) => {
      let x = seed;
      Math.random = () => {
        // simple LCG for predictability
        x = (1103515245 * x + 12345) % 2 ** 31;
        return x / 2 ** 31;
      };

      const t = { v: 0 };
      const now = () => t.v;
      const budget = createRetryBudget({ capacity: 10, refillPerSecond: 0 }, now);
      const delays: number[] = [];

      const outcome = await withRetry({
        idempotent: true,
        run: async () => {
          throw { kind: "network", code: "ECONNRESET" } satisfies DownstreamError;
        },
        policy: {
          maxAttempts: 4,
          deadlineMs: 10_000,
          maxTotalDelayMs: 10_000,
          baseDelayMs: 20,
          maxDelayMs: 200,
          jitter: "decorrelated",
        },
        budget,
        sleep: async (ms) => {
          delays.push(ms);
          t.v += ms;
        },
        now,
      });

      expect(outcome.ok).toBe(false);
      return delays.join(",");
    };

    const a = await collect(1);
    const b = await collect(2);

    Math.random = originalRandom;
    expect(a).not.toBe(b);
  });

  it("stops when deadline exceeded (no sleeping past deadline)", async () => {
    const t = { v: 0 };
    const now = () => t.v;
    const budget = createRetryBudget({ capacity: 10, refillPerSecond: 0 }, now);

    const outcome = await withRetry({
      idempotent: true,
      run: async () => {
        throw { kind: "timeout" } satisfies DownstreamError;
      },
      policy: {
        maxAttempts: 10,
        deadlineMs: 30,       // very tight
        maxTotalDelayMs: 1000,
        baseDelayMs: 20,
        maxDelayMs: 200,
        jitter: "full",
      },
      budget,
      sleep: async (ms) => {
        t.v += ms; // simulate time pass
      },
      now,
    });

    expect(outcome.ok).toBe(false);
    if (!outcome.ok) expect(outcome.stopReason).toBe("deadline_exceeded");
  });
});
```

---

# Senior+ Notes (đúng “tầng 9”)

Những điểm phân biệt Senior+ ở bài này:

* **Global budget**: bạn đang quản lý *hành vi hệ thống*, không chỉ 1 request.
* **Retry-After**: tôn trọng server signaling, tránh đánh nhau.
* **Deadline**: retry không vượt quá “time you can afford”.
* **Idempotency gate**: non-idempotent mà retry bừa là tạo bug business (double charge, duplicate order).

---

## Stretch (cấp cao hơn nữa — “đụng distributed thật”)

1. **Retry Budget theo error-rate**: nếu `downstream_error_rate > X%` thì giảm `refillPerSecond` (adaptive budget).
2. **Per-downstream partition budget**: budget riêng cho `inventory-service` vs `payment-service` (tránh 1 service kéo sập cả app).
3. **Hedged requests (tail latency)**: attempt thứ 2 chạy song song sau `p95` nhưng vẫn chịu budget (chỉ cho idempotent).
4. **Retry budget propagation**: gửi header `X-Retry-Budget-Remaining` giữa services để upstream không “đốt” downstream.
5. **Proof bằng chaos mini**: tạo 1000 requests đồng thời, downstream 503 50%, chứng minh:

   * không herd (delay phân tán),
   * retry không vượt budget,
   * latency không nổ vô hạn (deadline cắt).