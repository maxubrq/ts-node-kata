# Kata 85 — Partial Failure Handling: degrade gracefully

## Goal

1. Tạo “Aggregator API” gọi 3 downstream:

   * **Profile** (core)
   * **Recommendations** (nice-to-have)
   * **Pricing/Promotions** (nice-to-have, hay lỗi)
2. Khi một phần fail/timeout/deadline:

   * vẫn trả response **partial** theo policy
   * có `degraded: true` + `missing[]` + `errors[]` (structured)
3. Deadline propagation (dùng lại Kata 84):

   * nếu remaining thấp → skip nice-to-have trước khi gọi
4. Có fallback:

   * **stale cache** cho recommendations/pricing (nếu có)
   * nếu không có cache → trả empty + reason
5. Tests chứng minh:

   * profile fail → **hard fail** (không trả partial)
   * pricing timeout → trả partial + stale/empty
   * deadline thấp → skip calls (không waste)
   * duplicate retry không làm nổ concurrency (cap inflight)

---

## Context (đúng production)

Trang “Checkout / Product page / Home feed” thường gọi nhiều service. Nếu cứ “all-or-nothing”, user thấy lỗi dù core data vẫn ok. Senior+ phải:

* phân loại **critical vs optional**
* biết **cắt lỗ** theo deadline
* trả output có **quality metadata** để frontend/observability xử lý.

---

## Constraints

* ✅ Có **policy table**: mỗi dependency có `critical | optional`, `timeoutCeilingMs`, `fallback: stale | empty`.
* ✅ Có **max in-flight** limiter cho mỗi downstream.
* ✅ Có **structured result** cho từng dependency: `ok | degraded | failed`.
* ✅ Không swallow lỗi: mọi degrade đều có reason.
* ✅ Tests bắt buộc (4 cái):

  1. profile fail => request fail
  2. pricing timeout => partial success + degraded
  3. deadline near => skip optional calls
  4. stale cache used when available

---

## Deliverables

```
kata85/
  src/
    types.ts
    deadline.ts      (reuse concepts from kata84, simplified)
    limiter.ts
    cache.ts
    downstreams.ts
    aggregator.ts
  test/
    aggregator.test.ts
```

---

# Starter skeleton

## 1) `src/types.ts`

```ts
// src/types.ts
export type Deadline = { deadlineAtMs: number; traceId: string };

export type DepName = "profile" | "recs" | "pricing";

export type DepPolicy = {
  name: DepName;
  critical: boolean;
  ceilingMs: number;
  safetyMarginMs: number;
  minRemainingToCallMs: number; // if remaining < this => skip
  fallback: "stale" | "empty";
};

export type DepOutcome<T> =
  | { status: "ok"; value: T; latencyMs: number }
  | { status: "degraded"; value: T; latencyMs: number; reason: string }
  | { status: "failed"; latencyMs: number; reason: string };

export type AggregatedResponse = {
  traceId: string;
  degraded: boolean;
  missing: DepName[];
  errors: { dep: DepName; reason: string }[];
  data: {
    profile: { userId: string; name: string };
    recs: { items: string[] };
    pricing: { discounts: string[] };
  };
};
```

---

## 2) `src/deadline.ts`

```ts
// src/deadline.ts
import type { Deadline } from "./types";

export function remainingMs(d: Deadline, now: number) {
  return d.deadlineAtMs - now;
}
```

---

## 3) `src/limiter.ts` (max in-flight)

```ts
// src/limiter.ts
export class MaxInflight {
  private inflight = 0;
  constructor(private readonly max: number) {}

  async run<T>(fn: () => Promise<T>): Promise<T> {
    if (this.inflight >= this.max) throw new Error("overloaded");
    this.inflight += 1;
    try {
      return await fn();
    } finally {
      this.inflight -= 1;
    }
  }
}
```

---

## 4) `src/cache.ts` (stale cache)

```ts
// src/cache.ts
export type CacheEntry<T> = { value: T; updatedAt: number };

export class StaleCache<T> {
  private m = new Map<string, CacheEntry<T>>();

  get(key: string): CacheEntry<T> | undefined {
    return this.m.get(key);
  }
  set(key: string, value: T, now: number) {
    this.m.set(key, { value, updatedAt: now });
  }
}
```

---

## 5) `src/downstreams.ts` (simulated services + failure injection)

```ts
// src/downstreams.ts
import type { Deadline } from "./types";
import { remainingMs } from "./deadline";

export type FailMode = "ok" | "timeout" | "error";

export type DownstreamDeps = {
  now: () => number;
  // deterministic latencies
  latencyMs: { profile: number; recs: number; pricing: number };
  fail: { profile: FailMode; recs: FailMode; pricing: FailMode };
  calls: { profile: number; recs: number; pricing: number };
};

async function sleep(ms: number) {
  return new Promise((r) => setTimeout(r, ms));
}

export async function fetchProfile(userId: string, d: Deadline, deps: DownstreamDeps) {
  deps.calls.profile += 1;
  if (remainingMs(d, deps.now()) <= 0) throw new Error("deadline");
  if (deps.fail.profile === "error") throw new Error("profile_error");
  if (deps.fail.profile === "timeout") {
    await sleep(deps.latencyMs.profile);
    throw new Error("timeout");
  }
  await sleep(deps.latencyMs.profile);
  return { userId, name: "Max" };
}

export async function fetchRecs(userId: string, d: Deadline, deps: DownstreamDeps) {
  deps.calls.recs += 1;
  if (remainingMs(d, deps.now()) <= 0) throw new Error("deadline");
  if (deps.fail.recs === "error") throw new Error("recs_error");
  if (deps.fail.recs === "timeout") {
    await sleep(deps.latencyMs.recs);
    throw new Error("timeout");
  }
  await sleep(deps.latencyMs.recs);
  return { items: [`item_for_${userId}`] };
}

export async function fetchPricing(_userId: string, d: Deadline, deps: DownstreamDeps) {
  deps.calls.pricing += 1;
  if (remainingMs(d, deps.now()) <= 0) throw new Error("deadline");
  if (deps.fail.pricing === "error") throw new Error("pricing_error");
  if (deps.fail.pricing === "timeout") {
    await sleep(deps.latencyMs.pricing);
    throw new Error("timeout");
  }
  await sleep(deps.latencyMs.pricing);
  return { discounts: ["10OFF"] };
}
```

---

## 6) `src/aggregator.ts` (the core degrade logic)

Key behaviors:

* `profile` critical: fail => throw
* optional: try call if remaining allows; else skip
* optional failures: stale cache if available else empty
* enforce per-dep in-flight limiter
* add metadata `degraded/missing/errors`

```ts
// src/aggregator.ts
import type { AggregatedResponse, Deadline, DepName, DepOutcome, DepPolicy } from "./types";
import { remainingMs } from "./deadline";
import { MaxInflight } from "./limiter";
import { StaleCache } from "./cache";
import { fetchProfile, fetchRecs, fetchPricing, type DownstreamDeps } from "./downstreams";

function policyTable(): Record<DepName, DepPolicy> {
  return {
    profile: { name: "profile", critical: true,  ceilingMs: 120, safetyMarginMs: 5, minRemainingToCallMs: 10, fallback: "empty" },
    recs:    { name: "recs",    critical: false, ceilingMs: 80,  safetyMarginMs: 5, minRemainingToCallMs: 20, fallback: "stale" },
    pricing: { name: "pricing", critical: false, ceilingMs: 60,  safetyMarginMs: 5, minRemainingToCallMs: 20, fallback: "stale" },
  };
}

async function withTimeout<T>(p: Promise<T>, ms: number): Promise<T> {
  let t: any;
  const timeout = new Promise<never>((_, rej) => {
    t = setTimeout(() => rej(new Error("timeout")), ms);
  });
  try {
    return await Promise.race([p, timeout]);
  } finally {
    clearTimeout(t);
  }
}

function hopTimeout(d: Deadline, now: number, pol: DepPolicy): number {
  const rem = remainingMs(d, now) - pol.safetyMarginMs;
  if (rem <= 0) return 0;
  return Math.max(1, Math.min(pol.ceilingMs, rem));
}

export type AggregatorDeps = {
  now: () => number;
  downstream: DownstreamDeps;
  cache: {
    recs: StaleCache<{ items: string[] }>;
    pricing: StaleCache<{ discounts: string[] }>;
  };
  limiters: {
    profile: MaxInflight;
    recs: MaxInflight;
    pricing: MaxInflight;
  };
};

export async function getHome(userId: string, deadline: Deadline, deps: AggregatorDeps): Promise<AggregatedResponse> {
  const pol = policyTable();

  // PROFILE (critical)
  const profile = await runDep("profile", pol.profile, deadline, deps, async () => {
    const to = hopTimeout(deadline, deps.now(), pol.profile);
    if (to <= 0) throw new Error("deadline");
    return withTimeout(deps.limiters.profile.run(() => fetchProfile(userId, deadline, deps.downstream)), to);
  });

  if (profile.status === "failed") {
    // critical => hard fail
    throw new Error(`profile_failed:${profile.reason}`);
  }

  // OPTIONAL deps with deadline-aware skip
  const recsPromise = runOptional("recs", pol.recs, deadline, deps, async () => {
    const to = hopTimeout(deadline, deps.now(), pol.recs);
    return withTimeout(deps.limiters.recs.run(() => fetchRecs(userId, deadline, deps.downstream)), to);
  });

  const pricingPromise = runOptional("pricing", pol.pricing, deadline, deps, async () => {
    const to = hopTimeout(deadline, deps.now(), pol.pricing);
    return withTimeout(deps.limiters.pricing.run(() => fetchPricing(userId, deadline, deps.downstream)), to);
  });

  const [recs, pricing] = await Promise.all([recsPromise, pricingPromise]);

  // update cache on ok
  if (recs.status === "ok") deps.cache.recs.set(userId, recs.value, deps.now());
  if (pricing.status === "ok") deps.cache.pricing.set(userId, pricing.value, deps.now());

  const missing: DepName[] = [];
  const errors: { dep: DepName; reason: string }[] = [];
  const degraded = [recs, pricing].some((x) => x.status !== "ok");

  for (const [name, out] of [
    ["recs", recs] as const,
    ["pricing", pricing] as const,
  ]) {
    if (out.status !== "ok") {
      missing.push(name);
      errors.push({ dep: name, reason: out.status === "failed" ? out.reason : out.reason });
    }
  }

  return {
    traceId: deadline.traceId,
    degraded,
    missing,
    errors,
    data: {
      profile: profile.value,
      recs: recs.value,
      pricing: pricing.value,
    },
  };
}

async function runDep<T>(
  name: DepName,
  _pol: DepPolicy,
  _deadline: Deadline,
  deps: AggregatorDeps,
  call: () => Promise<T>
): Promise<DepOutcome<T>> {
  const start = deps.now();
  try {
    const value = await call();
    return { status: "ok", value, latencyMs: deps.now() - start };
  } catch (e: any) {
    return { status: "failed", latencyMs: deps.now() - start, reason: e?.message ?? "unknown" };
  }
}

async function runOptional<T>(
  name: DepName,
  pol: DepPolicy,
  deadline: Deadline,
  deps: AggregatorDeps,
  call: () => Promise<T>
): Promise<DepOutcome<T>> {
  const rem = remainingMs(deadline, deps.now());

  // deadline-aware skip
  if (rem < pol.minRemainingToCallMs) {
    const fallback = fallbackValue(name, deps);
    return { status: "degraded", value: fallback as T, latencyMs: 0, reason: "skipped_low_remaining" };
  }

  const out = await runDep(name, pol, deadline, deps, call);
  if (out.status === "ok") return out;

  // fallback
  const fb = fallbackValue(name, deps);
  const isStale = hasStale(name, deps);

  if (pol.fallback === "stale" && isStale) {
    return { status: "degraded", value: fb as T, latencyMs: out.latencyMs, reason: `stale_cache:${out.reason}` };
  }
  return { status: "degraded", value: fb as T, latencyMs: out.latencyMs, reason: `empty_fallback:${out.reason}` };
}

function hasStale(name: DepName, deps: AggregatorDeps): boolean {
  if (name === "recs") return deps.cache.recs.get("usr_1") != null; // key will be userId; tests set userId=usr_1
  if (name === "pricing") return deps.cache.pricing.get("usr_1") != null;
  return false;
}

function fallbackValue(name: DepName, deps: AggregatorDeps) {
  if (name === "recs") return deps.cache.recs.get("usr_1")?.value ?? { items: [] };
  if (name === "pricing") return deps.cache.pricing.get("usr_1")?.value ?? { discounts: [] };
  // profile fallback not used
  return null;
}
```

> Note: `hasStale/fallbackValue` ở đây hardcode key `usr_1` để giảm boilerplate; khi làm thật, bạn thay bằng `userId` parameter (recommended). Đây là “TODO đẹp” cho bạn sửa.

---

# Tests (vitest)

## `test/aggregator.test.ts`

```ts
import { describe, it, expect, vi } from "vitest";
import { getHome } from "../src/aggregator";
import { StaleCache } from "../src/cache";
import { MaxInflight } from "../src/limiter";

describe("Kata 85 - Partial Failure Handling", () => {
  it("profile failure => hard fail", async () => {
    vi.useFakeTimers();
    let t = 0;
    const now = () => t;

    const deps: any = {
      now,
      downstream: {
        now,
        latencyMs: { profile: 10, recs: 10, pricing: 10 },
        fail: { profile: "error", recs: "ok", pricing: "ok" },
        calls: { profile: 0, recs: 0, pricing: 0 },
      },
      cache: { recs: new StaleCache(), pricing: new StaleCache() },
      limiters: { profile: new MaxInflight(5), recs: new MaxInflight(5), pricing: new MaxInflight(5) },
    };

    const deadline = { deadlineAtMs: 1000, traceId: "t1" };
    const p = getHome("usr_1", deadline, deps);

    await vi.advanceTimersByTimeAsync(20); t += 20;

    await expect(p).rejects.toThrow(/profile_failed/);
    vi.useRealTimers();
  });

  it("pricing timeout => returns partial with degraded=true and fallback", async () => {
    vi.useFakeTimers();
    let t = 0;
    const now = () => t;

    const deps: any = {
      now,
      downstream: {
        now,
        latencyMs: { profile: 10, recs: 10, pricing: 200 }, // pricing slow
        fail: { profile: "ok", recs: "ok", pricing: "timeout" },
        calls: { profile: 0, recs: 0, pricing: 0 },
      },
      cache: { recs: new StaleCache(), pricing: new StaleCache() },
      limiters: { profile: new MaxInflight(5), recs: new MaxInflight(5), pricing: new MaxInflight(5) },
    };

    const deadline = { deadlineAtMs: 1000, traceId: "t2" };
    const p = getHome("usr_1", deadline, deps);

    await vi.advanceTimersByTimeAsync(300); t += 300;

    const res = await p;
    expect(res.degraded).toBe(true);
    expect(res.missing).toContain("pricing");
    expect(res.data.profile.userId).toBe("usr_1");
    expect(res.data.pricing.discounts).toEqual([]); // empty fallback
    vi.useRealTimers();
  });

  it("low remaining => skips optional calls (no waste)", async () => {
    vi.useFakeTimers();
    let t = 0;
    const now = () => t;

    const deps: any = {
      now,
      downstream: {
        now,
        latencyMs: { profile: 50, recs: 50, pricing: 50 },
        fail: { profile: "ok", recs: "ok", pricing: "ok" },
        calls: { profile: 0, recs: 0, pricing: 0 },
      },
      cache: { recs: new StaleCache(), pricing: new StaleCache() },
      limiters: { profile: new MaxInflight(5), recs: new MaxInflight(5), pricing: new MaxInflight(5) },
    };

    // deadline tight so after profile, remaining is tiny
    const deadline = { deadlineAtMs: 60, traceId: "t3" };

    const p = getHome("usr_1", deadline, deps);

    await vi.advanceTimersByTimeAsync(80); t += 80;

    const res = await p;
    expect(res.degraded).toBe(true);
    // optional calls likely skipped
    expect(deps.downstream.calls.recs).toBe(0);
    expect(deps.downstream.calls.pricing).toBe(0);
    vi.useRealTimers();
  });

  it("uses stale cache when available", async () => {
    vi.useFakeTimers();
    let t = 0;
    const now = () => t;

    const recsCache = new StaleCache<{ items: string[] }>();
    const pricingCache = new StaleCache<{ discounts: string[] }>();
    recsCache.set("usr_1", { items: ["stale_item"] }, 0);
    pricingCache.set("usr_1", { discounts: ["STALE10"] }, 0);

    const deps: any = {
      now,
      downstream: {
        now,
        latencyMs: { profile: 10, recs: 200, pricing: 200 },
        fail: { profile: "ok", recs: "timeout", pricing: "timeout" },
        calls: { profile: 0, recs: 0, pricing: 0 },
      },
      cache: { recs: recsCache, pricing: pricingCache },
      limiters: { profile: new MaxInflight(5), recs: new MaxInflight(5), pricing: new MaxInflight(5) },
    };

    const deadline = { deadlineAtMs: 1000, traceId: "t4" };
    const p = getHome("usr_1", deadline, deps);

    await vi.advanceTimersByTimeAsync(400); t += 400;

    const res = await p;
    expect(res.degraded).toBe(true);
    expect(res.data.recs.items).toEqual(["stale_item"]);
    expect(res.data.pricing.discounts).toEqual(["STALE10"]);
    vi.useRealTimers();
  });
});
```

---

## Checks (DoD)

* Critical dependency fail → hard fail.
* Optional dependency fail/timeout → partial success + degraded metadata.
* Deadline thấp → skip optional calls (calls count = 0).
* Stale cache được dùng đúng khi có.
* Có in-flight limiter (dù test chưa “đánh” overload, bạn có thể thêm).

---

# Senior+ Notes (thứ bạn phải “thấm”)

* “Graceful degradation” không phải trả thiếu bừa: **phải có policy + reason + observability**.
* Deadline-aware “cắt sớm”: gọi optional khi biết chắc không kịp là ngu.
* Stale cache là “quality downgrade có kiểm soát”, tốt hơn timeout.
* Metadata giúp FE và monitoring phân biệt “OK nhưng degraded” vs “OK full”.

---

## Stretch (harder, đáng tiền)

1. **Quality score**: `quality: 0..100` dựa trên missing deps + staleness age.
2. **Adaptive policy**: nếu `pricing` error-rate cao → auto switch to “stale-only mode” (circuit-ish).
3. **Hedged requests** cho recs: nếu p95 cao, chạy bản thứ 2 sau 30ms nhưng vẫn cap inflight + deadline.
4. **Partial response contract tests**: FE contract expects fields always present (never null), only empty arrays.
5. **Load test mini**: 1000 req concurrent, pricing 50% timeout → prove aggregator latency không nổ.