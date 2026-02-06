# Kata 92 — Profiling CPU: Find Hot Path, Optimize with Evidence

## Goal (Senior+)

Bạn sẽ làm được 5 việc như một engineer production-grade:

1. **Tạo workload CPU-bound có thật** (không “for loop rỗng”).
2. **Profile CPU bằng công cụ built-in của Node/V8** để xác định **hot functions + call stacks**.
3. **Đề xuất 2–3 phương án tối ưu** (algorithm/data-structure), chọn 1 phương án hợp lý.
4. **Chứng minh cải thiện** bằng benchmark harness (Kata 91): ns/op, p95, variance.
5. **Bảo toàn correctness** (test + equivalence check).

---

## Context

Bạn có một service/module làm “search + scoring” cho một danh sách items (giả lập ranking). Production complaint:

* Latency tail cao (p95/p99 xấu)
* CPU tăng khi traffic tăng

Bạn cần chẩn đoán: CPU đang đốt vào đâu? tối ưu gì? mức cải thiện ra sao?

---

## Constraints

* ✅ Node 18+ (khuyên 20+), TypeScript.
* ✅ Profiler **không dùng lib ngoài bắt buộc**. Ưu tiên:

  * `node --prof` (V8 tick profiler)
  * `node --cpu-prof` (tạo `.cpuprofile`, mở bằng Chrome DevTools)
* ✅ Workload phải **deterministic** (seed), để bench/prof so sánh được.
* ✅ Có **correctness test**: before/after output must match.
* ❌ Không “tối ưu” bằng cách giảm workload (ăn gian input).
* ❌ Không tối ưu “mù” trước khi profile.

---

## Deliverables

Tạo package:

```
packages/kata-92/
  src/
    workload.ts
    score.ts
    score.optimized.ts
    generate.ts
  bench/
    suite.ts
    run.ts
  scripts/
    prof.sh
  test/
    correctness.spec.ts
  README.md
```

Bạn sẽ giao:

1. `score.ts` (baseline) + `score.optimized.ts` (optimized)
2. Script profile tạo output `.cpuprofile` hoặc `--prof` log + processed report
3. Benchmark suite so baseline vs optimized + report JSON
4. Tests chứng minh correctness

---

## What you must learn (đúng mindset)

* **Profile trước**: đo stack, không đo “cảm giác”.
* **Tối ưu đúng tầng**: algorithm > data-structure > micro-opts.
* **Kết quả phải có tail**: p95, không chỉ mean.
* **Regression-proof**: có benchmark + gate.

---

## Bài toán (CPU workload)

Bạn implement hàm `rank(items, query)` trả về top K items theo score.

### Domain

* `items`: 50k–200k records
* Mỗi record:

  * `id: string`
  * `title: string`
  * `tags: string[]`
  * `body: string` (độ dài vừa)
* `query`: string

### Score rule (baseline)

Score gồm:

1. Tokenize query + tokenize text (title/body/tags) theo whitespace + normalize lowercase.
2. Mỗi token match tăng điểm theo trọng số:

   * title match: +5
   * tag match: +3
   * body match: +1
3. Bonus: nếu query là substring của title (case-insensitive) +10
4. Trả về top K (K=20) theo score giảm dần.

### “Chỗ đau” cố tình

Baseline làm kiểu “ngây thơ”:

* tokenize mỗi item mỗi lần rank
* `toLowerCase()` lặp rất nhiều
* substring check tốn kém
* sort full array thay vì topK selection

Bạn sẽ profile để thấy hot path.

---

## Starter skeleton (điền TODO)

### `src/generate.ts` (deterministic dataset)

```ts
// src/generate.ts
export type Item = {
  id: string;
  title: string;
  tags: string[];
  body: string;
};

function xorshift32(seed: number) {
  let x = seed >>> 0;
  return () => {
    x ^= x << 13; x >>>= 0;
    x ^= x >>> 17; x >>>= 0;
    x ^= x << 5; x >>>= 0;
    return x;
  };
}

const WORDS = [
  "node", "typescript", "cache", "retry", "latency", "stream", "queue", "shard",
  "profile", "optimize", "token", "search", "rank", "memory", "perf", "trace",
  "service", "domain", "event", "idempotent", "circuit", "bulkhead", "saga",
];

function pickWord(r: () => number) {
  return WORDS[r() % WORDS.length];
}

export function generateItems(count: number, seed = 123): Item[] {
  const r = xorshift32(seed);
  const items: Item[] = [];
  for (let i = 0; i < count; i++) {
    const titleLen = 4 + (r() % 8);
    const bodyLen = 40 + (r() % 80);
    const tagLen = 2 + (r() % 6);

    const title = Array.from({ length: titleLen }, () => pickWord(r)).join(" ");
    const body = Array.from({ length: bodyLen }, () => pickWord(r)).join(" ");
    const tags = Array.from({ length: tagLen }, () => pickWord(r));

    items.push({
      id: `it_${i}`,
      title,
      body,
      tags,
    });
  }
  return items;
}
```

### `src/score.ts` (baseline intentionally slow)

```ts
// src/score.ts
import type { Item } from "./generate";

function tokenize(s: string): string[] {
  return s.toLowerCase().split(/\s+/g).filter(Boolean);
}

function containsToken(tokens: string[], token: string): boolean {
  // intentionally naive O(n)
  for (const t of tokens) if (t === token) return true;
  return false;
}

export function rankBaseline(items: Item[], query: string, k = 20): Item[] {
  const qTokens = tokenize(query);

  const scored = items.map((it) => {
    const titleTokens = tokenize(it.title);
    const bodyTokens = tokenize(it.body);
    const tagTokens = it.tags.map(t => t.toLowerCase());

    let score = 0;

    for (const qt of qTokens) {
      if (containsToken(titleTokens, qt)) score += 5;
      if (containsToken(tagTokens, qt)) score += 3;
      if (containsToken(bodyTokens, qt)) score += 1;
    }

    // substring bonus
    if (it.title.toLowerCase().includes(query.toLowerCase())) score += 10;

    return { it, score };
  });

  // full sort (slow)
  scored.sort((a, b) => b.score - a.score);

  return scored.slice(0, k).map(x => x.it);
}
```

### `src/score.optimized.ts` (bạn tối ưu sau khi profile)

```ts
// src/score.optimized.ts
import type { Item } from "./generate";

// TODO: implement optimized ranking
// allowed ideas: pre-normalize, pre-tokenize, Set, topK selection, avoid repeated lowercase, etc.

export function rankOptimized(items: Item[], query: string, k = 20): Item[] {
  // TODO
  return items.slice(0, k);
}
```

### `bench/suite.ts` (dùng harness Kata 91)

```ts
// bench/suite.ts
import { generateItems } from "../src/generate";
import { rankBaseline } from "../src/score";
import { rankOptimized } from "../src/score.optimized";

export async function createSuite() {
  const items = generateItems(80_000, 123);
  const queries = [
    "node latency retry",
    "typescript domain event",
    "cache stampede",
    "circuit bulkhead queue",
    "profile optimize perf",
  ];

  return {
    name: "kata-92",
    cases: [
      { name: "baseline", fn: (i: number) => rankBaseline(items, queries[i % queries.length], 20) },
      { name: "optimized", fn: (i: number) => rankOptimized(items, queries[i % queries.length], 20) },
    ],
    input: (i: number) => i,
    warmupIters: 50,
    itersPerSample: 20,
    samples: 25,
    correctnessChecks: 20,
    gcBetweenSamples: false,
    trimRatio: 0.1,
  };
}
```

---

## Correctness tests (bắt buộc)

### `test/correctness.spec.ts`

* generateItems fixed seed
* run cả 2 rank với vài queries
* assert:

  * top K length giống nhau
  * **IDs same order** (strict) *hoặc* same set (nếu bạn chọn stable tie-breaker)
* thêm tie-breaker để deterministic (gợi ý: nếu score bằng nhau, sort by id)

Bạn phải quyết định và document.

---

## Profiling steps (bắt buộc)

### Option A — `--cpu-prof` (dễ nhất)

1. Tạo file `scripts/prof.sh`:

   * Chạy workload 10–30s để profiler đủ mẫu.
2. Lệnh mẫu:

   * `node --cpu-prof --cpu-prof-dir=./profiles dist/scripts/runWorkload.js`
3. Mở file `.cpuprofile` bằng Chrome DevTools:

   * DevTools → Performance → Load profile

**Bạn phải ghi lại trong README:**

* top 5 functions by self time
* top stacks causing CPU
* hypothesis: nguyên nhân + fix

### Option B — `--prof` (tick profiler)

* `node --prof dist/scripts/runWorkload.js`
* `node --prof-process isolate-*.log > processed.txt`
* đọc report để tìm hot functions

---

## Workload runner (để profile đúng)

Tạo `scripts/runWorkload.ts`:

* generateItems(150k)
* loop rank 200–500 lần với queries
* print checksum của output để tránh DCE (“sink” ở level app)
* run ~15–25s

---

## What optimization should look like (không được làm trước khi profile)

Sau khi profile baseline, bạn **chọn 1 chiến lược chính** + 1–2 micro-fixes.

Gợi ý những tối ưu thường “ăn tiền” (nhưng bạn phải chứng minh bằng profile):

1. **Pre-normalize + pre-tokenize items** (một lần):

   * store `titleLower`, `bodyLower`
   * store token sets: `Set<string>` cho title/body/tags
   * trade-off: memory tăng, CPU giảm
2. **Top-K selection thay vì sort full**

   * dùng min-heap size K hoặc partial selection
3. **Giảm allocations**

   * tránh `split`/`filter` liên tục
   * tokenize query 1 lần
4. **Substring check**

   * nếu query ngắn hoặc tokens check đã đủ, có thể short-circuit
   * tránh `.toLowerCase()` lặp: normalize queryLower 1 lần

Bạn phải document trade-off: CPU vs memory, cold-start vs steady-state.

---

## Checks (Definition of Done)

Bạn pass kata nếu đạt tất cả:

1. Có **profile artifact** (`.cpuprofile` hoặc `processed.txt`) + README ghi nhận:

   * top hotspots (function names)
   * call stacks chính
   * nguyên nhân (allocations? algorithm? repeated normalization?)
2. Có **optimized implementation** và **giải thích**:

   * thay đổi gì
   * trade-off gì
3. Benchmark (Kata 91 harness) chứng minh:

   * optimized **nhanh hơn baseline** ít nhất **30% mean** *hoặc* **p95 tốt hơn rõ rệt** (bạn chọn KPI, nhưng phải cụ thể)
   * variance không tệ hơn đáng kể
4. Correctness tests pass và deterministic.

---

## Stretch (Senior+ → “rất cao”)

Chọn ít nhất 4:

1. **Flamegraph reasoning**: chụp screenshot (hoặc ghi textual summary) của hottest stack và giải thích “why this stack dominates”.
2. **Two-phase optimization**:

   * Phase 1: algorithmic improvement (ví dụ pre-tokenize + Set)
   * Phase 2: allocation reduction (tokenization rewrite)
   * chứng minh impact riêng từng phase bằng benchmark.
3. **Perf regression gate**:

   * fail CI nếu optimized chậm hơn baseline > 5% (hoặc baseline commit-to-commit regression).
4. **Memory budget**:

   * đo `heapUsed` trước/sau, document overhead của cache/token sets.
   * đặt budget: “CPU giảm X%, memory tăng YMB”.
5. **Node inspector profiling**:

   * chạy `node --inspect` + record CPU profile live (document steps).
6. **Realistic tail**:

   * inject “noisy neighbor”: chạy thêm một CPU task song song để xem optimized có giữ lợi thế ở p95 không.
7. **Counterfactual**:

   * thử 1 tối ưu “nghe hợp lý” nhưng thực tế không tốt, và chứng minh bằng profile/bench vì sao (rất production-grade).

---

## Tip quan trọng (để không tự lừa mình)

* Khi profile: chạy đủ lâu (>= 10s) để sample ổn.
* Khi bench: giữ input deterministic, tăng samples/iters nếu noise.
* Đừng tối ưu “toàn bộ”: tập trung 1–2 hotspots có self-time cao.