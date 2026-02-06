# Kata 91 — Benchmark Harness (Node.js microbench đúng cách)

## Goal (Senior+)

Bạn sẽ tạo một **benchmark harness** chạy được từ CLI, hỗ trợ:

1. **Warmup + measured phase** (tách rõ).
2. **Time measurement chính xác** bằng `process.hrtime.bigint()`.
3. **Chống benchmark “ảo”** (dead-code elimination, constant folding, caching accidental).
4. **Kiểm soát GC** (tùy chọn) + phát hiện memory drift cơ bản.
5. **Thống kê tối thiểu**: mean/median/p95 + outlier trim đơn giản.
6. **So sánh 2 phiên bản** (baseline vs candidate) và xuất report JSON để CI gate.

---

## Context

Bạn có 2 hàm cần so kè:

* `baselineFn(input)` (cũ)
* `candidateFn(input)` (mới refactor)

Bạn cần câu trả lời “**nhanh hơn bao nhiêu**” và “**có ổn định không**”, không phải một con số đơn lẻ.

---

## Constraints

* ✅ Node 18+ (khuyến nghị 20+), TypeScript.
* ✅ Không dùng thư viện benchmark bên ngoài (không `benchmark.js`), tự viết.
* ✅ **Tách** warmup và measured.
* ✅ Có cơ chế **sink** để tránh tối ưu hóa bỏ computation.
* ✅ Có **input generator** để tránh đo trên 1 giá trị cố định.
* ✅ Có **report JSON** + console summary.
* ✅ Có ít nhất 1 **guard** chống kết quả sai (correctness check).
* ⚠️ Nếu dùng GC thủ công: chạy Node với `--expose-gc` (harness phải tự detect và degrade nếu không có).
* ❌ Không đo bằng `Date.now()`.

---

## Deliverables (repo/package kata-91)

Tạo structure:

```
packages/kata-91/
  src/
    bench.ts
    stats.ts
    suite.ts
    example/
      baseline.ts
      candidate.ts
  test/
    bench.spec.ts
  bench/
    run.ts
  README.md
```

Bạn sẽ giao:

1. `BenchHarness` chạy CLI:
   `pnpm bench --filter kata-91 -- --suite example --format pretty`
   hoặc `node dist/bench/run.js --suite example --json out.json`
2. Một suite mẫu benchmark so baseline vs candidate.
3. Unit tests cho `stats` + correctness guard.

---

## Definition: “Microbench đúng cách” (rules bạn phải encode)

### 1) Warmup

* Warmup để JIT ổn định (V8 tối ưu sau vài lần).
* Không tính vào kết quả.

### 2) Measured runs

* Chạy nhiều lần, mỗi lần nhiều iterations để giảm noise.

### 3) Sink chống “đo ảo”

* Không cho compiler/JIT tối ưu bỏ code:

  * Trả kết quả vào một biến global “volatile-like” (sink).
  * Hoặc accumulate hash.

### 4) Input variability

* Dùng generator tạo input khác nhau (nhưng deterministic), tránh constant folding.

### 5) Correctness guard

* Trước khi benchmark, verify `baseline` và `candidate` cho cùng input phải ra cùng output (hoặc cùng “shape” nếu output lớn).

### 6) Stats + outlier trim

* Tối thiểu: mean, median, p95, stddev (đơn giản).
* Trim 10% hai đầu (hoặc IQR filter) để giảm outlier.

### 7) Output + CI gate

* Xuất JSON chứa metrics.
* Có mode `--compare baseline.json candidate.json` (hoặc chạy 2 funcs cùng suite) và fail nếu regression vượt ngưỡng.

---

## Starter skeleton (copy chạy được, bạn điền TODO)

### `src/bench.ts`

```ts
// src/bench.ts
export type BenchFn<TInput> = (input: TInput) => unknown | Promise<unknown>;

export type BenchCase<TInput> = {
  name: string;
  fn: BenchFn<TInput>;
};

export type BenchConfig<TInput> = {
  name: string;
  cases: BenchCase<TInput>[];
  // generate deterministic inputs; must be cheap
  input: (i: number) => TInput;

  // warmup iterations per case
  warmupIters: number;

  // measured: total iterations per sample
  itersPerSample: number;

  // measured: number of samples
  samples: number;

  // optional correctness check on N inputs
  correctnessChecks?: number;

  // if true, try to call global.gc() between samples (if available)
  gcBetweenSamples?: boolean;

  // trim ratio for outlier removal: 0.1 trims 10% low + 10% high
  trimRatio?: number;
};

export type BenchSample = {
  ns: number;               // total time in ns for itersPerSample
  memRssBytes?: number;     // optional
  memHeapUsedBytes?: number;// optional
};

export type BenchResult = {
  suite: string;
  caseName: string;
  config: Omit<BenchConfig<any>, "cases" | "input"> & { trimRatio: number };
  samples: BenchSample[];
  summary: {
    nsPerOp: {
      mean: number;
      median: number;
      p95: number;
      stddev: number;
    };
    opsPerSec: {
      mean: number;
      median: number;
      p95: number;
    };
  };
};

let SINK = 0; // cheap sink to prevent elimination
function sink(x: unknown) {
  // mix into SINK deterministically
  const t = typeof x;
  if (t === "number") SINK = (SINK + (x as number)) | 0;
  else if (t === "string") SINK = (SINK + (x as string).length) | 0;
  else if (x && t === "object") SINK = (SINK + 1) | 0;
  else SINK = (SINK + 0) | 0;
  return SINK;
}

function nowNs(): bigint {
  return process.hrtime.bigint();
}

function getMem() {
  const m = process.memoryUsage();
  return { rss: m.rss, heapUsed: m.heapUsed };
}

export async function runSuite<TInput>(
  cfg: BenchConfig<TInput>,
  suiteName: string,
  // optional: supply baseline fn name for correctness
): Promise<BenchResult[]> {
  const trimRatio = cfg.trimRatio ?? 0.1;

  // ---------- Correctness guard ----------
  if (cfg.correctnessChecks && cfg.cases.length >= 2) {
    const n = cfg.correctnessChecks;
    for (let i = 0; i < n; i++) {
      const input = cfg.input(i);
      const out0 = await cfg.cases[0].fn(input);
      for (let c = 1; c < cfg.cases.length; c++) {
        const outC = await cfg.cases[c].fn(input);
        // TODO: decide strict equality vs stable stringify
        // Keep it strict for kata; allow customizing later.
        if (!Object.is(out0, outC)) {
          throw new Error(
            `Correctness check failed: ${cfg.cases[0].name} !== ${cfg.cases[c].name} at i=${i}`
          );
        }
      }
    }
  }

  const results: BenchResult[] = [];

  for (const benchCase of cfg.cases) {
    // ---------- Warmup ----------
    for (let i = 0; i < cfg.warmupIters; i++) {
      const input = cfg.input(i);
      const out = await benchCase.fn(input);
      sink(out);
    }

    const samples: BenchSample[] = [];

    for (let s = 0; s < cfg.samples; s++) {
      if (cfg.gcBetweenSamples && typeof (global as any).gc === "function") {
        (global as any).gc();
      }

      const memBefore = getMem();
      const t0 = nowNs();

      for (let i = 0; i < cfg.itersPerSample; i++) {
        const input = cfg.input(s * cfg.itersPerSample + i);
        const out = await benchCase.fn(input);
        sink(out);
      }

      const t1 = nowNs();
      const memAfter = getMem();

      const ns = Number(t1 - t0);
      samples.push({
        ns,
        memRssBytes: memAfter.rss - memBefore.rss,
        memHeapUsedBytes: memAfter.heapUsed - memBefore.heapUsed,
      });
    }

    // TODO: import stats helpers
    const { summarize } = await import("./stats");
    const summary = summarize(samples.map(x => x.ns), cfg.itersPerSample, trimRatio);

    results.push({
      suite: suiteName,
      caseName: benchCase.name,
      config: {
        warmupIters: cfg.warmupIters,
        itersPerSample: cfg.itersPerSample,
        samples: cfg.samples,
        gcBetweenSamples: cfg.gcBetweenSamples ?? false,
        correctnessChecks: cfg.correctnessChecks,
        trimRatio,
      },
      samples,
      summary,
    });
  }

  return results;
}
```

### `src/stats.ts`

```ts
// src/stats.ts
function mean(xs: number[]): number {
  let s = 0;
  for (const x of xs) s += x;
  return s / xs.length;
}

function median(xs: number[]): number {
  const a = [...xs].sort((x, y) => x - y);
  const mid = Math.floor(a.length / 2);
  return a.length % 2 === 0 ? (a[mid - 1] + a[mid]) / 2 : a[mid];
}

function percentile(xs: number[], p: number): number {
  const a = [...xs].sort((x, y) => x - y);
  const idx = Math.min(a.length - 1, Math.max(0, Math.floor((p / 100) * (a.length - 1))));
  return a[idx];
}

function stddev(xs: number[]): number {
  const m = mean(xs);
  let s2 = 0;
  for (const x of xs) {
    const d = x - m;
    s2 += d * d;
  }
  return Math.sqrt(s2 / xs.length);
}

function trim(xs: number[], ratio: number): number[] {
  const a = [...xs].sort((x, y) => x - y);
  const k = Math.floor(a.length * ratio);
  return a.slice(k, a.length - k);
}

export function summarize(sampleNsTotal: number[], itersPerSample: number, trimRatio: number) {
  const trimmed = trim(sampleNsTotal, trimRatio);

  // Convert total ns per sample -> ns/op
  const nsPerOp = trimmed.map(ns => ns / itersPerSample);
  const opsPerSec = nsPerOp.map(nsOp => 1e9 / nsOp);

  const nsOpMean = mean(nsPerOp);
  const nsOpMedian = median(nsPerOp);
  const nsOpP95 = percentile(nsPerOp, 95);
  const nsOpStd = stddev(nsPerOp);

  const opsMean = mean(opsPerSec);
  const opsMedian = median(opsPerSec);
  const opsP95 = percentile(opsPerSec, 95);

  return {
    nsPerOp: { mean: nsOpMean, median: nsOpMedian, p95: nsOpP95, stddev: nsOpStd },
    opsPerSec: { mean: opsMean, median: opsMedian, p95: opsP95 },
  };
}
```

### `src/suite.ts`

```ts
// src/suite.ts
import type { BenchConfig } from "./bench";

export type SuiteMap = Record<string, () => Promise<BenchConfig<any>>>;

export const suites: SuiteMap = {
  example: async () => {
    const { baselineFn } = await import("./example/baseline");
    const { candidateFn } = await import("./example/candidate");

    // deterministic input: xorshift
    let seed = 123456789;
    function rand() {
      seed ^= seed << 13;
      seed ^= seed >>> 17;
      seed ^= seed << 5;
      return seed >>> 0;
    }

    return {
      name: "example",
      cases: [
        { name: "baseline", fn: baselineFn },
        { name: "candidate", fn: candidateFn },
      ],
      input: (i: number) => {
        // avoid constant folding: variable length strings
        const n = (rand() % 64) + 16;
        const v = (rand() ^ i) >>> 0;
        return { n, v };
      },
      warmupIters: 20_000,
      itersPerSample: 50_000,
      samples: 25,
      correctnessChecks: 200,
      gcBetweenSamples: false,
      trimRatio: 0.1,
    } satisfies BenchConfig<any>;
  },
};
```

### `src/example/baseline.ts` và `candidate.ts` (mẫu)

```ts
// src/example/baseline.ts
export type Input = { n: number; v: number };

export function baselineFn(input: Input): number {
  // baseline: build string then compute checksum
  let s = "";
  for (let i = 0; i < input.n; i++) s += String.fromCharCode(97 + ((input.v + i) % 26));
  let h = 0;
  for (let i = 0; i < s.length; i++) h = (h * 31 + s.charCodeAt(i)) | 0;
  return h;
}
```

```ts
// src/example/candidate.ts
import type { Input } from "./baseline";

export function candidateFn(input: Input): number {
  // candidate: avoid building big string; hash on the fly
  let h = 0;
  for (let i = 0; i < input.n; i++) {
    const code = 97 + ((input.v + i) % 26);
    h = (h * 31 + code) | 0;
  }
  return h;
}
```

### `bench/run.ts` (CLI tối thiểu)

```ts
// bench/run.ts
import { runSuite } from "../src/bench";
import { suites } from "../src/suite";
import * as fs from "node:fs";

function parseArgs(argv: string[]) {
  const m = new Map<string, string | boolean>();
  for (let i = 0; i < argv.length; i++) {
    const a = argv[i];
    if (a.startsWith("--")) {
      const k = a.slice(2);
      const v = argv[i + 1];
      if (!v || v.startsWith("--")) m.set(k, true);
      else { m.set(k, v); i++; }
    }
  }
  return m;
}

function pretty(results: any[]) {
  for (const r of results) {
    const ns = r.summary.nsPerOp;
    const ops = r.summary.opsPerSec;
    console.log(`\n[${r.suite}] ${r.caseName}`);
    console.log(`  ns/op   mean=${ns.mean.toFixed(1)}  median=${ns.median.toFixed(1)}  p95=${ns.p95.toFixed(1)}  sd=${ns.stddev.toFixed(1)}`);
    console.log(`  ops/s   mean=${ops.mean.toFixed(0)} median=${ops.median.toFixed(0)} p95=${ops.p95.toFixed(0)}`);
  }
}

async function main() {
  const args = parseArgs(process.argv.slice(2));
  const suiteName = String(args.get("suite") ?? "example");
  const format = String(args.get("format") ?? "pretty");
  const out = args.get("out");

  const suiteFactory = suites[suiteName];
  if (!suiteFactory) throw new Error(`Unknown suite: ${suiteName}`);

  const cfg = await suiteFactory();
  const results = await runSuite(cfg, suiteName);

  if (format === "pretty") pretty(results);

  if (out && typeof out === "string") {
    fs.writeFileSync(out, JSON.stringify({ suite: suiteName, results }, null, 2), "utf8");
    console.log(`\nWrote ${out}`);
  }
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

---

## Tests bạn phải có

### 1) Stats correctness

`test/bench.spec.ts`:

* `summarize()` trả median/p95 đúng với dataset nhỏ.
* `trim()` loại đúng số phần tử.

### 2) Correctness guard hoạt động

* Cố tình tạo `candidateFn` sai -> harness phải throw trước khi đo.

---

## Checks (Definition of Done)

Bạn pass kata nếu:

1. **Warmup không tính vào measured** (code structure tách rõ).
2. Có **sink** và input generator (không đo trên constant).
3. Có **correctness check** chạy trước benchmark.
4. Output có **mean/median/p95** và trim outlier.
5. CLI chạy được và **xuất JSON report**.
6. Kết quả hợp lý: candidate mẫu ở trên **nhanh hơn baseline** (thường rõ rệt).

---

## Stretch (Senior+ → Staff-grade)

Chọn ít nhất 3, càng nhiều càng tốt:

1. **Regression gate**:

   * Chạy `baseline` và `candidate` trong cùng suite, tính ratio `candidate / baseline`.
   * Fail nếu chậm hơn baseline > X% (ví dụ 5%).
   * Xuất verdict vào JSON: `pass/fail`, `regressionPercent`.
2. **Async benchmark mode**:

   * Hỗ trợ fn async (đã có `await`), nhưng thêm option `mode: "sync" | "async"` để tránh await overhead khi không cần.
3. **CPU affinity / noise control doc**:

   * README hướng dẫn chạy: tắt app khác, chạy nhiều lần, pin CPU (doc), set `NODE_OPTIONS`, v.v.
4. **GC diagnostics**:

   * Ghi `gcBetweenSamples` + delta heapUsed/rss mỗi sample.
   * Flag nếu heap drift tăng đều (suspect allocation).
5. **Overhead calibration**:

   * Có “empty case” đo overhead loop + input generation.
   * Subtract overhead (cẩn thận: chỉ cho insight, không dùng làm “truth”).
6. **Compare across commits**:

   * Load `baseline.json` từ file, compare với run hiện tại.
   * Xuất bảng diff + reason.
7. **Confidence metric nhẹ**:

   * Bootstrap đơn giản hoặc compute “% samples candidate faster than baseline”.

---

## Ghi chú Senior+ (bắt buộc đọc, nhưng không cần code hết)

* Microbench không đại diện production end-to-end. Nó trả lời: “hàm A có nhanh hơn hàm B trong điều kiện lý tưởng không”.
* Đừng benchmark khi có I/O thật; tách I/O ra hoặc mock.
* Nếu kết quả nhảy loạn: tăng `itersPerSample`, tăng `samples`, bật trim, giảm input cost.