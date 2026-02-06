# Kata 95 — Caching Strategy: What to Cache & Invalidation Plan

## Goal (Senior+)

Bạn sẽ thiết kế + implement một caching strategy “sống được”:

1. Chọn **cái nên cache** (và cái KHÔNG nên cache) dựa trên workload.
2. Implement **2 lớp cache**: in-memory (per instance) + shared (Redis giả lập hoặc interface).
3. Có **invalidation plan** rõ: TTL + event-driven + manual bust.
4. Chống 3 bệnh kinh điển:

   * cache stampede (thundering herd)
   * stale data phục vụ sai quá lâu
   * cache poisoning / tenant leak
5. Instrument metrics: hit rate, stale served, refresh latency.
6. Chứng minh bằng benchmark/load simulation (nhẹ) + tests.

---

## Context

Bạn có endpoint:

`GET /products/:id`

* DB call ~25–60ms
* 80% traffic là read
* 20% là update (admin, sync)
* Multi-tenant: `tenantId` bắt buộc (không được leak)
* Dữ liệu product thay đổi “thỉnh thoảng”, nhưng khi đổi giá thì cần phản ánh nhanh.

Bạn cần caching để giảm latency & DB load, nhưng không được sai dữ liệu quá mức.

---

## Constraints

* ✅ TypeScript + Node.
* ✅ Không dùng cache library có sẵn cho core logic (tự viết LRU/TTL đơn giản hoặc Map+TTL).
* ✅ Có tests cho:

  * hit/miss
  * stampede protection (singleflight)
  * invalidation correctness
  * tenant isolation
* ✅ Có failure model (Redis down, event lost, clock skew).
* ❌ Không “cache everything”.
* ❌ Không dùng “TTL là đủ” mà không nêu trade-off.

---

## Deliverables

```
packages/kata-95/
  src/
    domain.ts
    repo.ts
    cache/
      cache.interface.ts
      memoryCache.ts
      sharedCache.ts
      singleflight.ts
    productService.ts
    invalidation.ts
    metrics.ts
  test/
    cache.spec.ts
    service.spec.ts
  README.md
```

Bạn giao:

1. `ProductService.getProduct(tenantId, productId)` có caching chuẩn
2. `ProductService.updateProduct(...)` + invalidation
3. Spec doc trong README: what to cache + invalidation plan + failure model
4. Metrics counters/histograms (tối thiểu counters)

---

## Domain model

### `src/domain.ts`

```ts
export type TenantId = string & { __brand: "TenantId" };
export type ProductId = string & { __brand: "ProductId" };

export type Product = {
  tenantId: string;
  id: string;
  name: string;
  price: number;
  updatedAt: number; // epoch ms
};
```

---

## Workload assumptions (bắt buộc ghi trong README)

* Read-heavy (80/20)
* Hot keys: top 1% products chiếm 50% reads
* Update bursts: price sync theo batch
* Consistency requirement:

  * read có thể stale tối đa **5s** với price (hoặc bạn chọn), nhưng **không được leak tenant**

---

## Task A — Decide “What to cache” (Staff-ish thinking, nhưng bắt buộc)

Bạn phải viết trong README:

### Cache candidates (bắt buộc chọn ít nhất 2)

* ✅ **Product by id** (hot, expensive)
* ✅ **Derived view**: e.g. product JSON response (serialize cost)
* ✅ **Negative caching**: not-found (chặn DB hammering) nhưng TTL ngắn

### Explicitly NOT cache (chọn ít nhất 1)

* ❌ Anything tenant-unsafe (cache key thiếu tenantId)
* ❌ Highly personalized / auth-dependent responses (nếu có)
* ❌ Writes / commands (không cache)

---

## Task B — Cache key design (bắt buộc)

Cache key **phải** gồm:

* tenantId
* entity type
* version (optional but encouraged)

Ví dụ:
`product:v1:{tenantId}:{productId}`

Bạn phải có test chứng minh **tenant isolation**.

---

## Task C — Implement 2-layer cache

### L1: in-memory (per instance)

* TTL
* max size (LRU hoặc simple cap)
* fastest, nhưng không shared

### L2: shared cache (interface)

* giả lập Redis bằng Map (cho kata) nhưng interface phải “như Redis”
* TTL

> Lý do: mô phỏng production: L1 giảm latency, L2 giảm DB load cross instance.

---

## Task D — Stampede protection (singleflight)

Khi 200 request đồng thời miss cùng key:

* chỉ 1 request được fetch DB
* còn lại await cùng promise

Bạn phải implement `singleflight.ts`:

```ts
export class SingleFlight {
  private inFlight = new Map<string, Promise<any>>();

  do<T>(key: string, fn: () => Promise<T>): Promise<T> {
    const existing = this.inFlight.get(key);
    if (existing) return existing as Promise<T>;

    const p = fn().finally(() => this.inFlight.delete(key));
    this.inFlight.set(key, p);
    return p;
  }
}
```

Test: spawn 50 promises, assert DB called 1 lần.

---

## Task E — Invalidation plan (phần quan trọng nhất)

Bạn phải implement **3 cơ chế invalidation** đồng thời:

### 1) TTL (safety net)

* L1 TTL = 2s
* L2 TTL = 5s (ví dụ)
* Negative cache TTL = 1s

### 2) Write-through invalidation (synchronous)

Khi `updateProduct` thành công:

* delete L1 + L2 key ngay
* hoặc set new value (write-through) nếu bạn đảm bảo atomic

### 3) Event-driven invalidation (asynchronous)

Có event: `ProductUpdated(tenantId, productId, updatedAt)`

* consumer sẽ bust cache keys cho entity

> Yêu cầu: event có thể trễ / mất → TTL vẫn cứu.

---

## Task F — Stale-while-revalidate (SWR) (Senior+)

Nếu cache entry hết TTL nhưng vẫn “fresh-enough” trong stale window:

* serve stale ngay (low latency)
* trigger background refresh (singleflight)
* record metric `stale_served++`

Bạn define:

* `ttl = 2s`
* `staleWindow = 10s`

---

## Starter skeleton (đủ để code ngay)

### `src/cache/cache.interface.ts`

```ts
export type CacheEntry<T> = { value: T; expiresAt: number; staleAt: number };

export interface Cache {
  get<T>(key: string): CacheEntry<T> | undefined;
  set<T>(key: string, entry: CacheEntry<T>): void;
  del(key: string): void;
}
```

### `src/cache/memoryCache.ts`

* Map + TTL + cap
* TODO: LRU optional

### `src/repo.ts` (fake DB)

```ts
import type { Product } from "./domain";

export interface ProductRepo {
  getById(tenantId: string, id: string): Promise<Product | null>;
  update(p: Product): Promise<void>;
}

export class FakeProductRepo implements ProductRepo {
  public calls = 0;
  private store = new Map<string, Product>();

  key(t: string, id: string) { return `${t}:${id}`; }

  async getById(t: string, id: string) {
    this.calls++;
    await new Promise(r => setTimeout(r, 30)); // simulate DB
    return this.store.get(this.key(t, id)) ?? null;
  }

  async update(p: Product) {
    await new Promise(r => setTimeout(r, 10));
    this.store.set(this.key(p.tenantId, p.id), p);
  }
}
```

### `src/productService.ts` (bạn implement)

Yêu cầu function:

* `getProduct(tenantId, productId): Promise<Product | null>`
* `updateProduct(product): Promise<void>`

Phải:

* check L1 → L2 → DB
* singleflight on misses
* SWR behavior
* invalidation on update
* metrics

---

## Tests (bắt buộc)

### `test/service.spec.ts`

1. **L1 hit**: call 2 lần, DB gọi 1
2. **L2 hit**: clear L1, call, DB không gọi
3. **Stampede**: 50 concurrent reads, DB gọi 1
4. **Invalidation on update**: update xong, next read fetch DB (hoặc returns new value nếu write-through)
5. **Tenant isolation**: same productId diff tenant must not share cache
6. **Negative caching**: missing product cached 1s, DB not hammered

---

## Metrics (bắt buộc)

Tối thiểu counters:

* `cache_l1_hit`
* `cache_l2_hit`
* `cache_miss`
* `cache_stale_served`
* `cache_refresh_started`
* `cache_bust`

Gợi ý: implement `metrics.ts` kiểu simple in-memory.

---

## Failure model (bắt buộc trong README)

Bạn phải trả lời:

1. **Shared cache down** → fallback: L1 only + DB (degrade). No crash.
2. **Event lost** → TTL là safety net.
3. **Clock skew** (nếu multi-node) → prefer monotonic? (ở kata: use `Date.now()`, doc risk)
4. **Poisoning** → validate tenant in cache key + never store auth-dependent response
5. **Hot key** → singleflight + SWR giảm stampede

---

## Checks (Definition of Done)

Bạn pass kata nếu:

* Có README quyết định “what to cache” + “what not”
* 2-layer cache + singleflight + SWR
* Invalidation 3 cơ chế
* Tests pass đầy đủ
* Metrics xuất được và hợp lý

---

## Stretch (Senior+ → Staff-grade)

Chọn ≥ 5:

1. **Versioned cache keys**: `product:v2:...` để migrate dễ
2. **Probabilistic early refresh**: refresh trước khi TTL hết (jitter) giảm herd
3. **Write-through atomicity**: update DB + set cache consistent (cẩn thận race)
4. **Cache admission policy**: chỉ cache hot keys (LFU-ish) để giảm memory
5. **Latency-aware TTL**: TTL dài hơn cho keys có DB latency cao
6. **SLO-based caching**: define target p95, tune TTL/stale window theo metric
7. **Chaos test**: inject shared cache failures 30% và chứng minh hệ vẫn đúng