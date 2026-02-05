# Kata 50 — Thundering Herd: Cache Stampede Protection

## Goal

1. Implement **stampede protection** cho cache: khi nhiều request cùng miss 1 key, hệ **chỉ fetch 1 lần**, các request khác **join** (single-flight).
2. Có **TTL + stale-while-revalidate (SWR)**: trả stale nhanh, refresh nền có kiểm soát.
3. Có **negative caching** (cache “not found”/error retryable) để giảm load khi key không tồn tại hoặc downstream lỗi lặp.
4. Có **backoff/jitter + circuit-ish** khi loader fail liên tục.
5. Có **metrics**: hits/misses/staleHits/joins/dedupedLoads/loadErrors, và test chứng minh “không herd”.

---

## Context

Cache stampede xảy ra khi:

* TTL hết cùng lúc (hoặc cache cold start)
* 1 hot key bị miss hàng loạt
* downstream chậm → mọi request cùng lao xuống DB → chết dây chuyền

Giải pháp production thường là: **single-flight + SWR + jitter + negative caching**.

---

## Constraints

* ❌ Không dùng library singleflight có sẵn.
* ✅ API phải generic (key → Promise<Value> loader).
* ✅ Không được để “promise rơi” (leak) trong map inflight.
* ✅ Có `AbortSignal` cho callers (tối thiểu: caller abort không làm hủy global load; request chỉ “detach”).
* ✅ Must support:

  * `ttlMs`
  * `staleTtlMs` (SWR window)
  * `maxConcurrentLoads` (cap) hoặc per-key only (core: per-key inflight)
  * `negativeTtlMs`
  * `jitterRatio` (0..1) để randomize TTL
* ✅ Tests phải chứng minh: 100 concurrent misses ⇒ loader chạy đúng 1 lần.

---

## Deliverables

1. `src/stampedeCache.ts` — core cache + single-flight + SWR
2. `src/errors.ts` — error types (optional)
3. `test/stampedeCache.test.ts` — concurrency tests
4. `README.md` — semantics + trade-offs + tuning

---

## API bạn sẽ build

```ts
export type CacheConfig = {
  ttlMs: number;            // fresh TTL
  staleTtlMs: number;       // additional SWR window after ttl (serve stale)
  negativeTtlMs: number;    // cache "not found"/loader errors (policy-based)
  jitterRatio: number;      // 0..1, randomize TTL to avoid synchronized expiry
};

export type CacheGetOptions = {
  signal?: AbortSignal;
  allowStale?: boolean;     // default true
};

export type CacheMetrics = {
  hitFresh: number;
  hitStale: number;
  miss: number;
  join: number;             // joined inflight instead of starting load
  loadStarted: number;
  loadSucceeded: number;
  loadFailed: number;
  negativeHit: number;
  refreshStarted: number;
};

export class StampedeCache<V> {
  constructor(cfg: CacheConfig, deps: { now: () => number; rand?: () => number });

  get(key: string, loader: () => Promise<V>, opts?: CacheGetOptions): Promise<V>;
  invalidate(key: string): void;

  snapshot(): CacheMetrics;
}
```

---

## Semantics bắt buộc

### Entry states

Mỗi key có thể ở các trạng thái:

* **Fresh**: `now < freshUntil` ⇒ trả ngay
* **Stale-but-servable**: `freshUntil <= now < staleUntil` ⇒

  * nếu `allowStale=true` ⇒ trả stale ngay **và** trigger refresh nền (không spam refresh)
  * nếu `allowStale=false` ⇒ phải await refresh (single-flight)
* **Expired**: `now >= staleUntil` ⇒ must load (single-flight)
* **Negative cached**: loader trước đó fail/not found, còn trong negative TTL ⇒ fail fast (hoặc return sentinel tuỳ policy)

### Single-flight

* Tại 1 thời điểm, **mỗi key chỉ có tối đa 1 loader inflight**
* Mọi request khác join promise đó
* Loader resolve/reject phải:

  * update cache entry + times
  * cleanup inflight map

### Jitter

* `effectiveTtl = ttlMs * (1 + randInRange(-jitterRatio, +jitterRatio))`
* Mục tiêu: tránh “đồng hồ TTL” của hot key hết cùng lúc trên nhiều instance.

### Caller Abort

* Nếu caller abort khi đang join: request đó reject `AbortError` (hoặc custom), nhưng **global load vẫn tiếp tục** cho request khác.

  * (Core simplest) implement “detach”: khi aborted, bạn reject caller bằng race, nhưng không cancel inflight.

---

## Starter skeleton (nhìn vào làm)

```ts
// src/stampedeCache.ts
type Entry<V> = {
  value?: V;
  error?: unknown;         // for negative cache
  freshUntil: number;
  staleUntil: number;
  negativeUntil: number;
  refreshing: boolean;
};

export type CacheConfig = {
  ttlMs: number;
  staleTtlMs: number;
  negativeTtlMs: number;
  jitterRatio: number;
};

export type CacheGetOptions = {
  signal?: AbortSignal;
  allowStale?: boolean;
};

export type CacheMetrics = {
  hitFresh: number;
  hitStale: number;
  miss: number;
  join: number;
  loadStarted: number;
  loadSucceeded: number;
  loadFailed: number;
  negativeHit: number;
  refreshStarted: number;
};

export class StampedeCache<V> {
  private store = new Map<string, Entry<V>>();
  private inflight = new Map<string, Promise<V>>();

  private m: CacheMetrics = {
    hitFresh: 0,
    hitStale: 0,
    miss: 0,
    join: 0,
    loadStarted: 0,
    loadSucceeded: 0,
    loadFailed: 0,
    negativeHit: 0,
    refreshStarted: 0,
  };

  constructor(private cfg: CacheConfig, private deps: { now: () => number; rand?: () => number }) {}

  snapshot(): CacheMetrics {
    return { ...this.m };
  }

  invalidate(key: string) {
    this.store.delete(key);
  }

  async get(key: string, loader: () => Promise<V>, opts?: CacheGetOptions): Promise<V> {
    const allowStale = opts?.allowStale ?? true;
    const now = this.deps.now();

    const e = this.store.get(key);
    if (e) {
      // negative cache hit
      if (e.negativeUntil > now) {
        this.m.negativeHit++;
        throw e.error ?? new Error("negative cached");
      }

      // fresh
      if (e.value !== undefined && e.freshUntil > now) {
        this.m.hitFresh++;
        return e.value;
      }

      // stale window
      if (e.value !== undefined && e.staleUntil > now) {
        if (allowStale) {
          this.m.hitStale++;
          this.triggerRefresh(key, loader, e);
          return e.value;
        }
        // else: must await refresh (single-flight)
      }
    }

    // expired or miss
    this.m.miss++;
    const p = this.singleFlight(key, loader);
    return this.raceWithAbort(p, opts?.signal);
  }

  private triggerRefresh(key: string, loader: () => Promise<V>, e: Entry<V>) {
    if (e.refreshing) return;
    e.refreshing = true;
    this.m.refreshStarted++;

    this.singleFlight(key, loader)
      .catch(() => {})
      .finally(() => {
        const cur = this.store.get(key);
        if (cur) cur.refreshing = false;
      });
  }

  private singleFlight(key: string, loader: () => Promise<V>): Promise<V> {
    const existing = this.inflight.get(key);
    if (existing) {
      this.m.join++;
      return existing;
    }

    this.m.loadStarted++;
    const p = (async () => {
      try {
        const v = await loader();
        this.m.loadSucceeded++;

        const now = this.deps.now();
        const { freshUntil, staleUntil } = this.computeTimes(now);

        const entry: Entry<V> = {
          value: v,
          error: undefined,
          freshUntil,
          staleUntil,
          negativeUntil: 0,
          refreshing: false,
        };
        this.store.set(key, entry);
        return v;
      } catch (err) {
        this.m.loadFailed++;

        const now = this.deps.now();
        const entry: Entry<V> = this.store.get(key) ?? {
          value: undefined,
          error: undefined,
          freshUntil: 0,
          staleUntil: 0,
          negativeUntil: 0,
          refreshing: false,
        };

        // negative cache for failures (policy: cache all errors; stretch: only retryable)
        entry.error = err;
        entry.negativeUntil = now + this.cfg.negativeTtlMs;
        entry.freshUntil = 0;
        entry.staleUntil = 0;

        this.store.set(key, entry);
        throw err;
      } finally {
        this.inflight.delete(key);
      }
    })();

    this.inflight.set(key, p);
    return p;
  }

  private computeTimes(now: number) {
    const r = this.deps.rand?.() ?? Math.random();
    const j = (r * 2 - 1) * this.cfg.jitterRatio; // [-j, +j]
    const ttl = Math.max(0, Math.round(this.cfg.ttlMs * (1 + j)));
    const freshUntil = now + ttl;
    const staleUntil = freshUntil + this.cfg.staleTtlMs;
    return { freshUntil, staleUntil };
  }

  private async raceWithAbort<T>(p: Promise<T>, signal?: AbortSignal): Promise<T> {
    if (!signal) return p;
    if (signal.aborted) throw Object.assign(new Error("aborted"), { name: "AbortError" });

    return await Promise.race([
      p,
      new Promise<never>((_, reject) => {
        const onAbort = () => reject(Object.assign(new Error("aborted"), { name: "AbortError" }));
        signal.addEventListener("abort", onAbort, { once: true });
        p.finally(() => signal.removeEventListener("abort", onAbort)).catch(() => {});
      }),
    ]);
  }
}
```

---

## Test plan bắt buộc (đây là “bằng chứng”)

### Test 1 — 100 concurrent misses ⇒ loader runs exactly once

* loader increments counter + delay 50ms
* `await Promise.all([...Array(100)].map(()=>cache.get("k", loader)))`
* expect counter == 1
* expect metrics: `loadStarted=1`, `join=99`, `miss=100`

### Test 2 — Fresh hit doesn’t call loader

* call get once to populate
* call get 10 lần trong TTL
* expect loader counter vẫn == 1, `hitFresh` tăng

### Test 3 — SWR: stale returns immediately + triggers refresh once

* set ttlMs=10, staleTtlMs=100
* populate, advance clock vượt freshUntil nhưng còn trong stale window
* call get 20 concurrent allowStale=true
* expect: trả value stale ngay, loader refresh started **1 lần** (not 20)

### Test 4 — allowStale=false forces await refresh (still single-flight)

* cùng tình huống stale, allowStale=false
* 20 concurrent calls → loader runs once, tất cả await

### Test 5 — Negative cache prevents hammering on failures

* loader luôn throw
* 50 concurrent calls => loader started 1, join 49
* call lại trong negativeTtlMs => **không gọi loader**, negativeHit++
* advance time > negativeTtlMs => loader gọi lại

### Test 6 — Caller abort detaches without canceling global load

* tạo 2 callers join cùng inflight
* abort 1 caller sớm => caller đó reject AbortError
* caller còn lại vẫn nhận success khi loader xong

---

## Checks (Definition of Done)

* Không stampede: concurrent miss/hot key chỉ 1 load.
* SWR hoạt động: stale hit nhanh, refresh không spam.
* Negative caching hoạt động: failure không gây bão.
* Inflight cleanup chắc chắn (không leak).
* Abort chỉ ảnh hưởng caller, không giết global load.

---

## README bắt buộc (10 dòng, đúng production)

1. Stampede là gì và tại sao giết DB
2. Single-flight: semantics join + cleanup
3. TTL vs staleTtl (SWR) và lợi ích tail latency
4. Jitter để tránh synchronized expiry
5. Negative caching: khi nào nên/không nên cache error
6. Abort semantics (detach)
7. Metrics bạn expose và dùng để alert
8. Failure modes: stuck inflight, cache poisoning, stale serving too long
9. Tuning playbook: ttl/stale/jitter/negativeTtl theo traffic & freshness needs
10. Integration: Bulkhead + Load Shedding + Circuit Breaker (khi loader là downstream)

---

## Stretch (Senior+ thật)

Chọn 1:

1. **Per-key mutex with waiters list** (thay promise map) để có insight wait time per joiner.
2. **Two-tier cache**: L1 in-process + L2 shared (Redis mock) và stampede protection ở L2.
3. **Soft errors policy**: chỉ negative-cache retryable errors, không cache programmer errors.
4. **Refresh budget**: giới hạn số refresh song song (global cap).
5. **Request coalescing across keys**: batch loads (giống dataloader) để giảm round trips.