# Kata 79 — Refactor with Contract Tests: Change Implementation, Preserve Behavior

## Goal

1. Viết **contract tests** cho một component (port/interface) để **khóa behavior**.
2. Refactor implementation **A → B** (thay kiến trúc, tối ưu, đổi storage…) mà:

   * tất cả contract tests pass
   * consumer tests không phải sửa (hoặc sửa cực ít, chỉ wiring)
3. Có **failure model** & invariants rõ (edge cases, concurrency, idempotency).

---

## Context (bài toán)

Bạn có `IdempotencyStore` (từ Kata 71/72/73). Hiện implementation là `MemoryIdempotencyStore` kiểu Map đơn giản. Production yêu cầu:

* TTL (hết hạn tự expire)
* Atomic “get-or-set” để chống race:

  * N request cùng idempotencyKey → **chỉ 1** thực thi “compute”, còn lại nhận kết quả của nó
* Serialization safe (value là JSON-ish)
* Memory bounded (giới hạn số keys, eviction)

Bạn sẽ:

1. Define **contract** cho `IdempotencyStore`
2. Viết contract tests dùng chung chạy được cho nhiều implementation
3. Viết **Impl A** (naive)
4. Refactor sang **Impl B** (có TTL + singleflight + eviction) mà contract vẫn pass

---

## Deliverables (repo shape)

```
kata-79/
  src/
    application/
      ports/
        idempotencyStore.ts
    infra/
      idempotency/
        memoryV1.ts          # naive
        memoryV2.ts          # refactor target
        contractHarness.ts   # helpers for contract tests
  test/
    contracts.idempotency.test.ts
    consumer.behavior.test.ts
    refactor.proof.test.ts
  README.md
```

---

## Port spec (khóa contract)

### `src/application/ports/idempotencyStore.ts`

```ts
export type CacheHit<T> = { hit: true; value: T };
export type CacheMiss = { hit: false };

export interface IdempotencyStore<T> {
  /** Return cached value if present and not expired */
  get(key: string): Promise<CacheHit<T> | CacheMiss>;

  /** Set value with TTL (ms). TTL<=0 means "expire immediately" */
  set(key: string, value: T, ttlMs: number): Promise<void>;

  /**
   * Core contract: get-or-compute atomically.
   * - If key exists (not expired): return cached value, MUST NOT run compute.
   * - If missing/expired: run compute EXACTLY ONCE across concurrent callers.
   * - If compute throws: do NOT cache; all callers should see failure (or retries on next call).
   */
  getOrCompute(key: string, ttlMs: number, compute: () => Promise<T>): Promise<T>;

  /** For testing/ops visibility */
  size(): Promise<number>;
}
```

---

## Contract rules (bạn phải enforce)

1. **No double compute**: 50 concurrent callers → compute runs 1 lần.
2. **TTL**:

   * before expiry: get hits
   * after expiry: miss, compute re-runs
3. **Compute failure**:

   * compute throw → no caching
   * next call should attempt compute again
4. **Value identity**:

   * returned value equals cached value (deep equal)
5. **Eviction** (B only, but contract can include a “bounded” optional capability):

   * if maxKeys exceeded, evict something (policy must be documented)
     *Cách Senior+: contract cho “bounded store” là optional extension.*

---

## Contract test harness (quan trọng)

### `src/infra/idempotency/contractHarness.ts`

```ts
import type { IdempotencyStore } from "../../application/ports/idempotencyStore";

export type StoreFactory<T> = () => Promise<IdempotencyStore<T>>;

export function defineIdempotencyContracts<T>(name: string, make: StoreFactory<T>, sampleValue: () => T) {
  // In test file, you'll call this for V1 and V2.
  // This function should declare `describe(name, () => { it(...) ... })`
  // TODO: implement in test side (Vitest/Jest), or keep here as helper.
}
```

---

## Implementation A (naive) — `memoryV1.ts`

### Requirements

* Map-based
* TTL supported (basic)
* getOrCompute **NOT atomic** initially (để test fail), rồi bạn fix or you leave V1 failing and only V2 passes.

> Kata mục tiêu: V1 thường fail concurrency contract, V2 phải pass.

Skeleton:

```ts
import type { IdempotencyStore, CacheHit, CacheMiss } from "../../application/ports/idempotencyStore";

type Entry<T> = { value: T; expiresAt: number };

export class MemoryIdempotencyStoreV1<T> implements IdempotencyStore<T> {
  private map = new Map<string, Entry<T>>();

  async get(key: string): Promise<CacheHit<T> | CacheMiss> {
    const e = this.map.get(key);
    if (!e) return { hit: false };
    if (Date.now() >= e.expiresAt) { this.map.delete(key); return { hit: false }; }
    return { hit: true, value: e.value };
  }

  async set(key: string, value: T, ttlMs: number): Promise<void> {
    const expiresAt = Date.now() + Math.max(0, ttlMs);
    if (ttlMs <= 0) { this.map.delete(key); return; }
    this.map.set(key, { value, expiresAt });
  }

  async getOrCompute(key: string, ttlMs: number, compute: () => Promise<T>): Promise<T> {
    // TODO naive: check get then compute; this is racey
    const hit = await this.get(key);
    if (hit.hit) return hit.value;
    const v = await compute();
    await this.set(key, v, ttlMs);
    return v;
  }

  async size(): Promise<number> { return this.map.size; }
}
```

---

## Implementation B (refactor target) — `memoryV2.ts`

### Requirements (Senior+)

* TTL supported
* **Singleflight** per key:

  * concurrent getOrCompute share the same in-flight promise
* Bounded memory:

  * `maxKeys` option
  * eviction policy: LRU hoặc FIFO (chọn 1, document)
* Deterministic time (inject clock for tests)

Skeleton:

```ts
import type { IdempotencyStore, CacheHit, CacheMiss } from "../../application/ports/idempotencyStore";

type Entry<T> = { value: T; expiresAt: number; lastAccessAt: number };

export type MemoryV2Options = {
  maxKeys: number;
  nowMs: () => number;
};

export class MemoryIdempotencyStoreV2<T> implements IdempotencyStore<T> {
  private map = new Map<string, Entry<T>>();
  private inflight = new Map<string, Promise<T>>();

  constructor(private opts: MemoryV2Options) {}

  private isExpired(e: Entry<T>) { return this.opts.nowMs() >= e.expiresAt; }

  async get(key: string): Promise<CacheHit<T> | CacheMiss> {
    const e = this.map.get(key);
    if (!e) return { hit: false };
    if (this.isExpired(e)) { this.map.delete(key); return { hit: false }; }
    e.lastAccessAt = this.opts.nowMs();
    return { hit: true, value: e.value };
  }

  async set(key: string, value: T, ttlMs: number): Promise<void> {
    if (ttlMs <= 0) { this.map.delete(key); return; }
    const now = this.opts.nowMs();
    this.map.set(key, { value, expiresAt: now + ttlMs, lastAccessAt: now });
    await this.evictIfNeeded();
  }

  private async evictIfNeeded() {
    // TODO: if map.size > maxKeys, evict LRU (min lastAccessAt)
  }

  async getOrCompute(key: string, ttlMs: number, compute: () => Promise<T>): Promise<T> {
    const hit = await this.get(key);
    if (hit.hit) return hit.value;

    const existing = this.inflight.get(key);
    if (existing) return existing;

    const p = (async () => {
      try {
        const v = await compute();
        await this.set(key, v, ttlMs);
        return v;
      } catch (e) {
        // IMPORTANT: do not cache failures
        throw e;
      } finally {
        this.inflight.delete(key);
      }
    })();

    this.inflight.set(key, p);
    return p;
  }

  async size(): Promise<number> { return this.map.size; }
}
```

---

## Contract Tests (bắt buộc) — `test/contracts.idempotency.test.ts`

Bạn viết theo kiểu “run same tests for both implementations”.

### Contract suite (minimum)

1. **Cache hit**

* set key ttl=1000
* get => hit

2. **TTL expiry**

* fake clock: now=0, set ttl=100
* now=150, get => miss

3. **No double compute**

* compute increments counter + awaits a barrier (delay)
* fire 50 concurrent `getOrCompute`
* expect counter === 1
* all results equal

4. **Compute failure not cached**

* compute throws first time
* assert first call rejects
* next call uses a compute that succeeds; counter should be 2 total

5. **Eviction (for V2)**

* maxKeys=2
* set 3 keys
* size <= 2
* (optional) assert LRU key evicted by access pattern

> Tip: V1 có thể skip eviction tests, hoặc bạn tạo “capabilities” flag trong factory để decide.

---

## Consumer behavior test — `test/consumer.behavior.test.ts`

Simulate real usage (idempotent payment):

* function `chargeOnce(key)` calls `store.getOrCompute(key, ttl, () => charge())`
* call it twice concurrently
* ensure `charge()` invoked once

This test là “living proof” consumer behavior is stable.

---

## Refactor proof — `test/refactor.proof.test.ts`

* Run contract suite on V1 (expect fail concurrency) để chứng minh test có lực
  *hoặc* mark V1 test as `it.fails` nếu dùng Vitest.
* Run contract suite on V2 (must pass)

---

## Checks (Definition of Done)

* ✅ Có contract tests chạy cho ít nhất 2 implementations.
* ✅ V2 pass toàn bộ contracts (incl. concurrency).
* ✅ Consumer test không đổi khi swap V1→V2 (chỉ đổi wiring).
* ✅ Failure model rõ và được test.

---

## Stretch (Senior+ → cực cao)

1. **Persistence adapter swap**

   * thêm `RedisIdempotencyStore` mock (không cần Redis thật), contract vẫn áp dụng
2. **Cancellation**

   * `getOrCompute` respects AbortSignal; in-flight cleanup đúng
3. **Metrics**

   * counters: hit/miss/compute_started/compute_deduped/evicted
4. **Stampede control**

   * nếu compute quá lâu: allow “stale-while-revalidate” (serve stale value, refresh background)
5. **Memory leak guard**

   * ensure inflight map always cleared even if compute hangs (timeout wrapper)