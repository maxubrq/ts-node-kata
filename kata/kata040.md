# Kata 40 — Hot Partition Fix: Shard key / Bucketing

## Goal

1. Mô phỏng một hệ thống partition theo `partitionKey`:

* có `P` partitions
* routing: `hash(partitionKey) % P`
* đo load per partition khi ingest events

2. Tạo dataset có **skew** (80/20 hoặc 95/5):

* một “super-key” nhận phần lớn traffic (vd: tenant lớn, celebrity user, product hot)
* chứng minh **hot partition**: 1 partition bị overload

3. Thiết kế fix:

* **bucketing/salting** partitionKey: `key#bucket`
* bucketCount = B
* routing distributed hơn
* vẫn query được theo key gốc (tạo “fan-out read” hoặc “index”)

4. Có tests/sim chứng minh:

* trước fix: max partition load quá cao
* sau fix: max load giảm rõ (đưa số liệu)
* trade-off: read fan-out tăng (đo số partitions touched)

---

## Constraints

* ✅ Có metric “imbalance”:

  * `maxLoad / avgLoad`
  * hoặc `p99PartitionLoad`
* ✅ Fix phải có **deterministic** bucket selection cho write (vd: random + stable seed hoặc round-robin).
* ✅ Có strategy cho read:

  * fan-out read across all buckets
  * hoặc maintain secondary index (stretch)
* ❌ Không được “tăng partitions lên” như giải pháp duy nhất (đó là né bài).

---

## Context (scenario demo)

Bạn có bảng/event stream `page_views` partition theo `userId`.

Nhưng có một user `user_celeb` tạo 50% traffic (bot / celebrity / bug).
Partition theo userId ⇒ 1 partition gánh hết.

Fix: bucket writes của `user_celeb` thành `user_celeb#0..B-1`.

---

## Deliverables

`kata-40/`

* `src/hash.ts` (stable hash)
* `src/partitioner.ts` (route + metrics)
* `src/workload.ts` (skew generator)
* `src/fix.ts` (bucketing write + fanout read)
* `src/run.ts` (compare before/after)
* `test/kata40.test.ts` (prove skew + fix)
* `notes/tradeoffs.md` (1 trang trade-offs)

---

## “Nhìn vào làm được” table

| Phần | Bạn làm gì       | Output                   | Checks          |
| ---- | ---------------- | ------------------------ | --------------- |
| 1    | Partition router | `route(key)->partition`  | deterministic   |
| 2    | Skew workload    | events with hotKey ratio | reproducible    |
| 3    | Baseline sim     | load per partition       | imbalance high  |
| 4    | Bucketing fix    | `key#bucket`             | imbalance lower |
| 5    | Read strategy    | fan-out read             | cost measured   |

---

## Starter Skeleton

### `src/hash.ts`

```ts
// Simple stable hash (FNV-1a 32-bit)
export function hash32(s: string): number {
  let h = 0x811c9dc5;
  for (let i = 0; i < s.length; i++) {
    h ^= s.charCodeAt(i);
    h = Math.imul(h, 0x01000193);
  }
  return h >>> 0;
}
```

### `src/partitioner.ts`

```ts
import { hash32 } from "./hash";

export class Partitioner {
  loads: number[];

  constructor(public readonly partitions: number) {
    this.loads = Array.from({ length: partitions }, () => 0);
  }

  route(partitionKey: string): number {
    return hash32(partitionKey) % this.partitions;
  }

  record(partitionKey: string, count = 1) {
    const p = this.route(partitionKey);
    this.loads[p] += count;
  }

  stats() {
    const total = this.loads.reduce((a, b) => a + b, 0);
    const avg = total / this.loads.length;
    const max = Math.max(...this.loads);
    const sorted = [...this.loads].sort((a, b) => a - b);
    const p95 = sorted[Math.floor(0.95 * (sorted.length - 1))];
    return {
      total,
      avg,
      max,
      p95,
      imbalance: avg === 0 ? 0 : max / avg,
      loads: [...this.loads],
    };
  }
}
```

### `src/workload.ts`

```ts
export type WorkloadSpec = {
  totalEvents: number;   // e.g. 100_000
  uniqueKeys: number;    // e.g. 10_000
  hotKey: string;        // e.g. "user_celeb"
  hotRatio: number;      // e.g. 0.5 (50% events)
  seed: number;          // deterministic randomness
};

function rng(seed: number) {
  let x = seed >>> 0;
  return () => {
    x = (x * 1664525 + 1013904223) >>> 0;
    return x / 0xffffffff;
  };
}

export function generateKeys(uniqueKeys: number): string[] {
  const keys: string[] = [];
  for (let i = 0; i < uniqueKeys; i++) keys.push(`user_${i}`);
  return keys;
}

export function* skewedEvents(spec: WorkloadSpec): Generator<string> {
  const r = rng(spec.seed);
  const keys = generateKeys(spec.uniqueKeys);

  for (let i = 0; i < spec.totalEvents; i++) {
    const pickHot = r() < spec.hotRatio;
    if (pickHot) yield spec.hotKey;
    else yield keys[Math.floor(r() * keys.length)];
  }
}
```

### `src/fix.ts`

```ts
import { hash32 } from "./hash";

// Write-side bucketing: key -> key#bucket
export function bucketedKey(key: string, buckets: number, eventId?: string): string {
  // Choose bucket deterministically to avoid “all to one bucket”.
  // Option A: hash(eventId) % buckets (best if you have eventId/requestId)
  // Option B: hash(key + random) (not deterministic)
  const basis = eventId ?? `${key}:${Math.random()}`; // TODO: remove Math.random in final
  const b = hash32(basis) % buckets;
  return `${key}#${b}`;
}

// Read-side fanout: original key -> all bucketed keys
export function fanoutKeys(key: string, buckets: number): string[] {
  const out: string[] = [];
  for (let b = 0; b < buckets; b++) out.push(`${key}#${b}`);
  return out;
}
```

> TODO quan trọng: bỏ `Math.random()` và dùng eventId ổn định.

### `src/run.ts`

```ts
import { Partitioner } from "./partitioner";
import { skewedEvents } from "./workload";
import { bucketedKey } from "./fix";

export async function compare() {
  const P = 32;
  const total = 100_000;

  const spec = {
    totalEvents: total,
    uniqueKeys: 10_000,
    hotKey: "user_celeb",
    hotRatio: 0.5,
    seed: 42,
  };

  // Baseline: partition by userId
  const base = new Partitioner(P);
  for (const key of skewedEvents(spec)) base.record(key);

  // Fix: bucket hot keys (and optionally all keys)
  const buckets = 16;
  const fixed = new Partitioner(P);
  let i = 0;
  for (const key of skewedEvents(spec)) {
    // only bucket hotKey for realism
    const pk = key === spec.hotKey ? bucketedKey(key, buckets, `evt_${i}`) : key;
    fixed.record(pk);
    i++;
  }

  console.log("BASE", base.stats());
  console.log("FIX ", fixed.stats());
}

compare();
```

---

## Tests (vitest) — prove hot partition + improvement

### `test/kata40.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { Partitioner } from "../src/partitioner";
import { skewedEvents } from "../src/workload";
import { bucketedKey, fanoutKeys } from "../src/fix";

describe("kata40 - hot partition fix", () => {
  it("baseline shows hot partition under skew", async () => {
    const P = 32;
    const spec = {
      totalEvents: 100_000,
      uniqueKeys: 10_000,
      hotKey: "user_celeb",
      hotRatio: 0.5,
      seed: 42,
    };

    const base = new Partitioner(P);
    for (const key of skewedEvents(spec)) base.record(key);

    const st = base.stats();
    // Expect high imbalance (tune threshold based on your sim)
    expect(st.imbalance).toBeGreaterThan(3); // usually much larger under 50% hot
  });

  it("bucketing hot key reduces max/avg imbalance", async () => {
    const P = 32;
    const spec = {
      totalEvents: 100_000,
      uniqueKeys: 10_000,
      hotKey: "user_celeb",
      hotRatio: 0.5,
      seed: 42,
    };

    const base = new Partitioner(P);
    for (const key of skewedEvents(spec)) base.record(key);
    const baseImb = base.stats().imbalance;

    const buckets = 16;
    const fixed = new Partitioner(P);
    let i = 0;
    for (const key of skewedEvents(spec)) {
      const pk = key === spec.hotKey ? bucketedKey(key, buckets, `evt_${i}`) : key;
      fixed.record(pk);
      i++;
    }

    const fixImb = fixed.stats().imbalance;

    expect(fixImb).toBeLessThan(baseImb);
    // also ensure meaningful improvement
    expect(fixImb).toBeLessThan(baseImb * 0.6);
  });

  it("fanout read touches B buckets (trade-off)", async () => {
    const buckets = 16;
    const ks = fanoutKeys("user_celeb", buckets);
    expect(ks.length).toBe(16);
    expect(ks[0]).toBe("user_celeb#0");
    expect(ks[15]).toBe("user_celeb#15");
  });
});
```

---

## TODO bắt buộc (để kata “chuẩn production”)

1. **Deterministic bucket selection**
   Trong `bucketedKey`:

* bỏ `Math.random()`
* bắt buộc truyền `eventId` (hoặc requestId) vào để hash
  => viết lại signature:

```ts
export function bucketedKey(key: string, buckets: number, eventId: string): string
```

2. **Chốt strategy: bucket hotKey only vs bucket all**

* hot-only: minimal read fanout impact
* bucket-all: distribution đều hơn nhưng mọi read-by-key đều fanout

3. **Viết `notes/tradeoffs.md`**

* Khi nào dùng bucketing
* Khi nào dùng “better shard key” (tenant_id + user_id)
* Khi nào dùng secondary index/materialized view

---

## Definition of Done (Checks)

Bạn pass kata này khi:

* Baseline sim cho imbalance rõ rệt (`max/avg` cao).
* Fix bằng bucketing làm imbalance giảm đáng kể (có số).
* Bạn đo và ghi trade-off read fanout (B buckets).
* Code bucket selection deterministic.

---

## Stretch (Senior+)

1. **Adaptive bucketing**

* chỉ bucket các keys vượt threshold (hot key detection)

2. **Two-level partition key**

* `tenantId + userId` để tránh “tenant hot” hoặc “global hot”

3. **Secondary index to avoid fanout reads**

* maintain table `key -> list of buckets used recently` hoặc store aggregated counters

4. **Backfill / migration**

* chuyển từ non-bucketed sang bucketed mà không downtime (nối Kata 37 mindset)