# Kata 24 — Rate Limiting (Token Bucket, in-memory) + Tests

Mục tiêu: implement **token bucket** chuẩn production (trong phạm vi 1 process) với **fake clock** để test deterministic.

---

## Goal

Bạn build limiter:

* Per-key (ví dụ theo `userId` hoặc `ip`)
* Token bucket:

  * `capacity` (burst)
  * `refillPerSecond`
* `allow(key, cost=1)` trả:

  * `allowed: boolean`
  * `remainingTokens`
  * `retryAfterMs` (nếu bị chặn)
* Có test:

  1. burst đúng
  2. refill đúng theo thời gian
  3. nhiều key độc lập
  4. cost > 1
  5. edge: time jump lớn

---

## Constraints

* ❌ Không dựa vào `setTimeout` trong test
* ✅ Dùng clock inject (`nowMs(): number`)
* ✅ Không `any`
* ✅ Không để token drift âm / NaN
* ✅ Có cleanup TTL để map không phình mãi

---

## Spec

### Types

```ts
type AllowResult = {
  allowed: boolean;
  remaining: number;      // integer-ish (floor)
  retryAfterMs?: number;  // only when !allowed
};

type TokenBucketConfig = {
  capacity: number;         // burst
  refillPerSecond: number;  // tokens/s
  ttlMs?: number;           // default 10 minutes
};
```

Rule:

* Bucket starts full (`tokens = capacity`)
* Refill continuously: `tokens += (elapsedMs/1000)*refillPerSecond`, capped at `capacity`
* Consume if `tokens >= cost`
* If not enough: `retryAfterMs = ceil(((cost - tokens)/refillPerSecond)*1000)`

---

## Starter skeleton (điền TODO)

```ts
/* kata24.ts */

type AllowResult = {
  allowed: boolean;
  remaining: number;
  retryAfterMs?: number;
};

type TokenBucketConfig = {
  capacity: number;
  refillPerSecond: number;
  ttlMs?: number;
};

type Clock = { nowMs(): number };

type BucketState = {
  tokens: number;      // fractional allowed internally
  lastRefillMs: number;
  expiresAtMs: number; // for cleanup
};

function clampInt(n: number, min: number, max: number): number {
  if (!Number.isFinite(n)) return min;
  return Math.max(min, Math.min(max, Math.trunc(n)));
}

export class TokenBucketLimiter {
  private readonly capacity: number;
  private readonly refillPerSecond: number;
  private readonly ttlMs: number;
  private readonly clock: Clock;

  private buckets = new Map<string, BucketState>();

  constructor(cfg: TokenBucketConfig, clock: Clock) {
    this.capacity = clampInt(cfg.capacity, 1, 1_000_000);
    // refillPerSecond can be fractional, but must be > 0
    if (!(typeof cfg.refillPerSecond === "number") || !Number.isFinite(cfg.refillPerSecond) || cfg.refillPerSecond <= 0) {
      throw new Error("refillPerSecond must be a finite number > 0");
    }
    this.refillPerSecond = cfg.refillPerSecond;

    this.ttlMs = clampInt(cfg.ttlMs ?? 10 * 60_000, 1_000, 24 * 60 * 60_000);
    this.clock = clock;
  }

  allow(key: string, cost: number = 1): AllowResult {
    const now = this.clock.nowMs();
    if (typeof key !== "string" || key.length === 0) throw new Error("key must be non-empty string");
    if (!(typeof cost === "number") || !Number.isFinite(cost) || cost <= 0) throw new Error("cost must be finite > 0");

    this.cleanup(now);

    const st = this.getOrCreate(key, now);
    this.refill(st, now);

    if (st.tokens >= cost) {
      st.tokens -= cost;
      st.expiresAtMs = now + this.ttlMs;

      return {
        allowed: true,
        remaining: Math.floor(st.tokens),
      };
    }

    // not enough tokens
    const needed = cost - st.tokens; // > 0
    const retryAfterMs = Math.ceil((needed / this.refillPerSecond) * 1000);

    st.expiresAtMs = now + this.ttlMs;

    return {
      allowed: false,
      remaining: Math.floor(st.tokens),
      retryAfterMs,
    };
  }

  private getOrCreate(key: string, now: number): BucketState {
    const existing = this.buckets.get(key);
    if (existing) return existing;

    const st: BucketState = {
      tokens: this.capacity,        // starts full
      lastRefillMs: now,
      expiresAtMs: now + this.ttlMs,
    };
    this.buckets.set(key, st);
    return st;
  }

  private refill(st: BucketState, now: number): void {
    // TODO:
    // - compute elapsedMs = max(0, now - lastRefillMs)
    // - add tokens = elapsedMs/1000 * refillPerSecond
    // - cap at capacity
    // - set lastRefillMs = now
  }

  private cleanup(now: number): void {
    // TODO: delete expired buckets
    // tip: iterate map and delete where expiresAtMs <= now
  }
}

// ---------- Tests (no timers) ----------
class FakeClock implements Clock {
  private t: number;
  constructor(startMs: number) { this.t = startMs; }
  nowMs(): number { return this.t; }
  advance(ms: number) { this.t += ms; }
}

function assert(cond: unknown, msg: string) {
  if (!cond) throw new Error("ASSERT FAIL: " + msg);
}

function testBurstAndBlock() {
  const clk = new FakeClock(0);
  const lim = new TokenBucketLimiter({ capacity: 3, refillPerSecond: 1 }, clk);

  assert(lim.allow("u1").allowed, "1 allowed");
  assert(lim.allow("u1").allowed, "2 allowed");
  assert(lim.allow("u1").allowed, "3 allowed");

  const r4 = lim.allow("u1");
  assert(!r4.allowed, "4th should block");
  assert(typeof r4.retryAfterMs === "number" && r4.retryAfterMs >= 1, "retryAfter present");
}

function testRefill() {
  const clk = new FakeClock(0);
  const lim = new TokenBucketLimiter({ capacity: 2, refillPerSecond: 2 }, clk); // 2 tokens/s

  lim.allow("u1"); // consume 1 -> 1 left
  lim.allow("u1"); // consume 1 -> 0 left
  assert(!lim.allow("u1").allowed, "should block at 0");

  clk.advance(250); // +0.5 tokens
  assert(!lim.allow("u1").allowed, "still block at 0.5 tokens if cost=1");

  clk.advance(250); // total 500ms -> +1 token
  assert(lim.allow("u1").allowed, "allowed after 1 token refilled");
}

function testIndependentKeys() {
  const clk = new FakeClock(0);
  const lim = new TokenBucketLimiter({ capacity: 1, refillPerSecond: 1 }, clk);

  assert(lim.allow("a").allowed, "a first ok");
  assert(!lim.allow("a").allowed, "a second blocked");
  assert(lim.allow("b").allowed, "b independent ok");
}

function testCost() {
  const clk = new FakeClock(0);
  const lim = new TokenBucketLimiter({ capacity: 5, refillPerSecond: 1 }, clk);

  assert(lim.allow("u1", 3).allowed, "cost 3 ok");
  const r = lim.allow("u1", 3);
  assert(!r.allowed, "cost 3 blocked when insufficient");
  assert(typeof r.retryAfterMs === "number", "retryAfter present");

  // tokens left should be 2 - (we consumed 3 from 5)
  // allow cost=2 should succeed now
  assert(lim.allow("u1", 2).allowed, "cost 2 ok");
}

function testTimeJumpCapsAtCapacity() {
  const clk = new FakeClock(0);
  const lim = new TokenBucketLimiter({ capacity: 2, refillPerSecond: 1 }, clk);

  lim.allow("u1"); // 1 left
  clk.advance(10_000); // huge jump -> should cap at 2
  const r = lim.allow("u1", 2);
  assert(r.allowed, "should cap refill at capacity");
}

function run() {
  testBurstAndBlock();
  testRefill();
  testIndependentKeys();
  testCost();
  testTimeJumpCapsAtCapacity();
  console.log("ALL TESTS PASSED");
}

run();
```

---

## Tasks bạn phải làm (cụ thể)

1. Implement `refill()` đúng công thức, không cho token vượt `capacity`
2. Implement `cleanup()` theo `expiresAtMs` (TTL)
3. Verify tests pass, và tự thêm 1 test:

* `retryAfterMs` gần đúng (ví dụ capacity 1, refill 1/s, thiếu 1 token → ~1000ms)

---

## Definition of Done

* Burst hoạt động đúng (capacity)
* Refill đúng theo thời gian (fractional ok, remaining floor)
* Multi-key không ảnh hưởng nhau
* `retryAfterMs` hợp lý
* TTL cleanup giúp map không phình vô hạn

---

## Stretch (Senior+)

1. Trả thêm headers kiểu HTTP:

* `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`

2. “Sliding window by cost”: mỗi request cost theo endpoint (POST = 5, GET = 1)
3. Distributed store (Redis) với atomic script (Lua) — không cần làm ở kata này, nhưng thiết kế interface từ đầu