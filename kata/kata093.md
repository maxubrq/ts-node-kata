# Kata 93 — Memory Optimization: Reduce Allocations, Measure Heap

## Goal (Senior+)

Bạn sẽ làm được 6 việc:

1. Tạo một workload có **allocation pressure rõ ràng** (GC chạy thường xuyên).
2. Đo memory bằng **heap snapshots** + **allocation sampling** (tối thiểu 1 trong 2).
3. Xác định **top allocation sources** (file/func/line hoặc call stack).
4. Refactor để **giảm alloc** (không chỉ giảm RSS “ảo”).
5. Chứng minh bằng số liệu:

   * `heapUsed` drift giảm
   * alloc rate giảm (nếu đo được)
   * p95 latency / ns/op cải thiện (dùng harness Kata 91)
6. Không phá correctness (tests + equivalence).

---

## Context

Một module xử lý “format + serialize + log event” đang gây:

* GC frequency cao
* latency tail xấu
* CPU tốn vào GC

Bạn nghi ngờ do:

* string concatenation
* tạo nhiều object trung gian
* JSON stringify “nặng”
* tạo arrays short-lived

---

## Constraints

* ✅ Node 20+ (khuyên) hoặc 18+.
* ✅ Không dùng lib benchmark/memory profiler ngoài bắt buộc.
* ✅ Phải có **workload runner** để tái hiện pressure.
* ✅ Phải dùng ít nhất:

  * **heap snapshot** (`--heapsnapshot-signal` hoặc inspector) *hoặc*
  * **GC trace + memory sampling** (`--trace-gc`, `process.memoryUsage()`)
* ✅ Tối ưu phải “hợp pháp”: không được bỏ bớt dữ liệu output.
* ❌ Không tối ưu bằng cách giảm số lần chạy workload.

---

## Deliverables

```
packages/kata-93/
  src/
    event.ts
    format.baseline.ts
    format.optimized.ts
    workload.ts
  bench/
    suite.ts
  scripts/
    run.ts
    gc-trace.sh
    heapshot.sh
  test/
    correctness.spec.ts
  README.md
```

---

## Bài toán (allocation-heavy)

Bạn có event:

```ts
export type Event = {
  ts: number;
  level: "info" | "warn" | "error";
  requestId: string;
  userId?: string;
  action: string;
  attrs: Record<string, string | number | boolean | null>;
};
```

Bạn cần tạo log line dạng JSON string (hoặc “logfmt-like”), baseline làm “ngây thơ” tạo nhiều object/string trung gian.

### Baseline behavior (cố tình tệ)

* clone object
* map/filter entries tạo arrays
* stringify nhiều lần
* concat string nhiều lần

### Optimized behavior (đích)

* giảm object creation
* giảm arrays tạm
* hạn chế stringify
* có thể dùng `JSON.stringify` 1 lần với replacer/ precomputed parts
* hoặc xây string bằng 1 pass (cẩn thận escaping)

---

## Starter skeleton (copy chạy được)

### `src/event.ts`

```ts
// src/event.ts
export type Event = {
  ts: number;
  level: "info" | "warn" | "error";
  requestId: string;
  userId?: string;
  action: string;
  attrs: Record<string, string | number | boolean | null>;
};
```

### `src/format.baseline.ts`

```ts
// src/format.baseline.ts
import type { Event } from "./event";

export function formatBaseline(e: Event): string {
  // Intentionally alloc-heavy:
  // - clones
  // - arrays
  // - string concat
  const base = {
    ts: e.ts,
    level: e.level,
    requestId: e.requestId,
    userId: e.userId ?? null,
    action: e.action,
  };

  const attrs = Object.entries(e.attrs)
    .filter(([, v]) => v !== null)
    .map(([k, v]) => [k, String(v)] as const);

  const payload = {
    ...base,
    attrs: Object.fromEntries(attrs),
  };

  // stringify twice (bad)
  const json = JSON.stringify(payload);
  return `[EVENT] ${e.level.toUpperCase()} ${json}`;
}
```

### `src/format.optimized.ts` (bạn điền TODO)

```ts
// src/format.optimized.ts
import type { Event } from "./event";

// Goal: fewer allocations, same output semantics (or documented contract)
export function formatOptimized(e: Event): string {
  // TODO:
  // - avoid Object.entries().map().filter() chain
  // - avoid cloning base objects
  // - stringify exactly once
  // - keep output stable/deterministic
  return "";
}
```

### `src/workload.ts`

```ts
// src/workload.ts
import type { Event } from "./event";

function xorshift32(seed: number) {
  let x = seed >>> 0;
  return () => {
    x ^= x << 13; x >>>= 0;
    x ^= x >>> 17; x >>>= 0;
    x ^= x << 5; x >>>= 0;
    return x;
  };
}

const KEYS = ["ip","ua","path","method","status","ms","cache","retry","region","feature"];

export function generateEvents(count: number, seed = 42): Event[] {
  const r = xorshift32(seed);
  const out: Event[] = [];
  for (let i = 0; i < count; i++) {
    const attrs: Record<string, string | number | boolean | null> = {};
    const n = 6 + (r() % 8);
    for (let j = 0; j < n; j++) {
      const k = KEYS[(r() % KEYS.length)];
      const t = r() % 4;
      attrs[k + "_" + j] =
        t === 0 ? (r() % 10_000) :
        t === 1 ? (r() % 2 === 0) :
        t === 2 ? `v_${r() % 10000}` :
        null;
    }

    out.push({
      ts: Date.now(),
      level: (["info","warn","error"] as const)[r() % 3],
      requestId: `req_${(r() % 1_000_000).toString(16)}`,
      userId: r() % 4 === 0 ? `usr_${(r() % 1_000_000).toString(16)}` : undefined,
      action: `act_${r() % 50}`,
      attrs,
    });
  }
  return out;
}
```

---

## Workload runner (để tạo pressure thật)

### `scripts/run.ts`

Chạy format hàng triệu lần, giữ sink để tránh DCE:

```ts
// scripts/run.ts
import { generateEvents } from "../src/workload";
import { formatBaseline } from "../src/format.baseline";
import { formatOptimized } from "../src/format.optimized";

const mode = process.argv.includes("--optimized") ? "optimized" : "baseline";
const events = generateEvents(200_000, 42);

let sink = 0;
function eat(s: string) { sink = (sink + s.length) | 0; }

const fn = mode === "optimized" ? formatOptimized : formatBaseline;

// Warmup
for (let i = 0; i < 20_000; i++) eat(fn(events[i % events.length]));

// Measured pressure loop (~ few seconds)
const loops = 15;
for (let l = 0; l < loops; l++) {
  for (let i = 0; i < events.length; i++) {
    eat(fn(events[i]));
  }
}

console.log({ mode, sink });
```

---

## Đo memory (bắt buộc có chứng cứ)

### Option A — GC trace + sampling (dễ nhất)

Tạo `scripts/gc-trace.sh`:

* Baseline:

  * `node --trace-gc dist/scripts/run.js`
* Optimized:

  * `node --trace-gc dist/scripts/run.js --optimized`

Bạn phải ghi trong README:

* tần suất GC
* dấu hiệu pause tăng/giảm (đọc log)
* heapUsed trước/sau (sampling)

Bạn cũng phải thêm sampling trong run.ts (optional but recommended):

* in mỗi N iterations: `process.memoryUsage().heapUsed`

### Option B — Heap snapshot (chuẩn hơn)

Tạo `scripts/heapshot.sh` dùng signal snapshot:

* Chạy Node với:

  * `node --heapsnapshot-signal=SIGUSR2 dist/scripts/run.js`
* Gửi signal giữa lúc chạy:

  * `kill -USR2 <pid>`
* Snapshot `.heapsnapshot` sinh ra, mở bằng Chrome DevTools (Memory tab).

README phải ghi:

* top retained objects
* top allocation suspects (dựa trên dominators / retaining paths)

> Nếu bạn muốn “đúng Staff”, dùng cả A + B.

---

## Benchmark (dùng Kata 91)

Tạo `bench/suite.ts` so baseline vs optimized:

* đo `ns/op` và `p95`
* thêm `memHeapUsedBytes` delta nếu muốn (không hoàn hảo nhưng hữu ích)

---

## What “optimization” thường đúng trong bài này

Bạn phải chọn ít nhất 2 chiến thuật:

1. **Loại bỏ chain `entries().filter().map()`**

   * dùng for-in + if
2. **Tránh clone object**

   * build payload trực tiếp
3. **Stringify một lần**

   * không stringify nested rồi stringify outer
4. **Pre-size array / reuse**

   * nếu vẫn cần entries, dùng 1 array và push
5. **Stable output**

   * nếu output phụ thuộc iteration order, document (JS object order ổn định theo insertion nhưng vẫn nên deterministic)

---

## Correctness tests (bắt buộc)

`test/correctness.spec.ts`:

* với seed fixed, generate 200 events
* assert `formatOptimized(e)` **bằng** `formatBaseline(e)` (strict string match)
  *hoặc* nếu bạn đổi format để tối ưu hơn, phải:

  * define “contract output” (JSON parse equivalence)
  * compare semantic equality

Staff-grade thì giữ output identical là đẹp nhất.

---

## Checks (Definition of Done)

Bạn pass nếu:

1. Có workload tái hiện pressure (GC trace cho thấy GC diễn ra).
2. Có **evidence**:

   * GC trace log hoặc heap snapshot
   * số liệu heapUsed/GC frequency trước và sau
3. Optimized giảm:

   * **heapUsed drift** (ít tăng dần)
   * **GC frequency** hoặc “allocation symptoms”
4. Benchmark chứng minh:

   * ns/op tốt hơn (tối thiểu 15–25% thường achievable)
   * p95 không tệ hơn
5. Correctness tests pass.

---

## Stretch (Senior+ → rất cao)

Chọn ≥ 4:

1. **Allocation flame**: dùng `--cpu-prof` để chứng minh CPU giảm vì alloc/GC giảm (liên kết Kata 92).
2. **Memory budget contract**: ghi rõ “tăng/giảm bao nhiêu MB tại 200k events”.
3. **Streaming logger**: thay vì build string full, write chunks (cẩn thận API).
4. **Object shape stability**: tối ưu để payload object shape ổn định (giảm hidden class churn).
5. **Regression gate**: fail CI nếu heapUsed drift > X hoặc ns/op regression > 5%.
6. **Retained memory check**: sau workload, force GC (nếu `--expose-gc`), assert heapUsed quay về gần baseline (phát hiện leak vs alloc).

---

## README Template (bạn phải điền)

* Problem
* Setup (node version, commands)
* Evidence baseline (GC trace / heap snapshot findings)
* Changes (what you changed, why it reduces alloc)
* Results table (heapUsed drift, GC count, ns/op, p95)
* Risks/trade-offs
* Regression gate rule