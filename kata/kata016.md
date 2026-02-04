# Kata 16 — Event Loop Lag Monitor

**Đo event loop lag + alert theo threshold** (production-grade, dùng để phát hiện backpressure, sync CPU spikes, GC pauses, “đang OOM dần”, v.v.)

---

## Goal

Bạn xây 1 module `eventLoopLag.ts` có thể:

1. Đo **event loop lag** định kỳ (ms)
2. Tính các thống kê rolling: `p50/p95/p99`, `max`, `avg` (trong window N giây)
3. Alert khi vượt threshold theo rule rõ ràng (ví dụ `p99 > 200ms` trong 30s)
4. Có hook để:

* log structured
* emit metric (Prometheus/StatsD giả lập)
* trigger “circuit breaker” (optional)

---

### Constraints

* ❌ Không `any`
* ✅ Không tạo overhead lớn (monitor phải “nhẹ”)
* ✅ Cleanup được (`start/stop`)
* ✅ Không spam alert (cooldown / debounce)
* ✅ Có demo script tạo lag giả (CPU block) để thấy alert hoạt động

---

## Spec

### Thông số bạn phải support

* `sampleIntervalMs` (default 100ms)
* `windowMs` (default 30_000ms)
* Threshold rules (ít nhất 2 rule):

  * `p99Ms > X`
  * `maxMs > Y`
* `alertCooldownMs` (default 60_000ms)

### Output metric snapshot

```ts
type LagSnapshot = {
  at: number;
  samples: number;
  avgMs: number;
  p50Ms: number;
  p95Ms: number;
  p99Ms: number;
  maxMs: number;
};
```

### Alert event

```ts
type LagAlert = {
  at: number;
  snapshot: LagSnapshot;
  triggeredBy: "p99" | "max";
  message: string;
};
```

---

## Implementation options (choose one)

* **Option A (recommended):** `perf_hooks.monitorEventLoopDelay()` (Node supports, low overhead)
* **Option B:** Manual drift measurement bằng `setInterval` + `performance.now()` (portable)

Kata yêu cầu bạn implement **Option B bắt buộc**, Option A là stretch.

---

## Starter skeleton (điền TODO)

```ts
/* kata16.ts */
import { performance } from "node:perf_hooks";

type LagSnapshot = {
  at: number;
  samples: number;
  avgMs: number;
  p50Ms: number;
  p95Ms: number;
  p99Ms: number;
  maxMs: number;
};

type LagAlert = {
  at: number;
  snapshot: LagSnapshot;
  triggeredBy: "p99" | "max";
  message: string;
};

type LagMonitorOptions = {
  sampleIntervalMs?: number;   // default 100
  windowMs?: number;           // default 30_000
  alertCooldownMs?: number;    // default 60_000
  threshold?: {
    p99Ms?: number;            // default 200
    maxMs?: number;            // default 500
  };
  onSnapshot?: (s: LagSnapshot) => void;
  onAlert?: (a: LagAlert) => void;
};

function clampInt(n: number, min: number, max: number): number {
  if (!Number.isFinite(n)) return min;
  return Math.max(min, Math.min(max, Math.trunc(n)));
}

function percentile(sorted: number[], p: number): number {
  // p in [0,1]. Uses nearest-rank-ish interpolation.
  if (sorted.length === 0) return 0;
  const idx = (sorted.length - 1) * p;
  const lo = Math.floor(idx);
  const hi = Math.ceil(idx);
  if (lo === hi) return sorted[lo];
  const w = idx - lo;
  return sorted[lo] * (1 - w) + sorted[hi] * w;
}

type Sample = { at: number; lagMs: number };

export class EventLoopLagMonitor {
  private readonly sampleIntervalMs: number;
  private readonly windowMs: number;
  private readonly alertCooldownMs: number;
  private readonly thresholdP99Ms: number;
  private readonly thresholdMaxMs: number;
  private readonly onSnapshot?: (s: LagSnapshot) => void;
  private readonly onAlert?: (a: LagAlert) => void;

  private timer: NodeJS.Timeout | null = null;
  private samples: Sample[] = [];
  private lastTickAt = 0;
  private lastAlertAt = 0;

  constructor(opts: LagMonitorOptions = {}) {
    this.sampleIntervalMs = clampInt(opts.sampleIntervalMs ?? 100, 10, 5_000);
    this.windowMs = clampInt(opts.windowMs ?? 30_000, 1_000, 10 * 60_000);
    this.alertCooldownMs = clampInt(opts.alertCooldownMs ?? 60_000, 0, 60 * 60_000);

    this.thresholdP99Ms = clampInt(opts.threshold?.p99Ms ?? 200, 1, 60_000);
    this.thresholdMaxMs = clampInt(opts.threshold?.maxMs ?? 500, 1, 60_000);

    this.onSnapshot = opts.onSnapshot;
    this.onAlert = opts.onAlert;
  }

  start(): void {
    if (this.timer) return;

    this.lastTickAt = performance.now();

    this.timer = setInterval(() => {
      const now = performance.now();
      const expected = this.lastTickAt + this.sampleIntervalMs;
      const lag = Math.max(0, now - expected);
      this.lastTickAt = now;

      const at = Date.now();
      this.samples.push({ at, lagMs: lag });

      this.evictOld(at);

      const snap = this.computeSnapshot(at);
      if (this.onSnapshot) this.onSnapshot(snap);

      this.maybeAlert(at, snap);
    }, this.sampleIntervalMs);

    // Optional: avoid keeping process alive
    this.timer.unref?.();
  }

  stop(): void {
    if (!this.timer) return;
    clearInterval(this.timer);
    this.timer = null;
    this.samples = [];
  }

  private evictOld(nowAt: number): void {
    const cutoff = nowAt - this.windowMs;
    // TODO: evict efficiently (shift in while loop is ok for kata, optimize later if needed)
    while (this.samples.length > 0 && this.samples[0].at < cutoff) {
      this.samples.shift();
    }
  }

  private computeSnapshot(nowAt: number): LagSnapshot {
    const lags = this.samples.map((s) => s.lagMs).sort((a, b) => a - b);
    const n = lags.length;

    let sum = 0;
    let max = 0;
    for (const v of lags) {
      sum += v;
      if (v > max) max = v;
    }

    return {
      at: nowAt,
      samples: n,
      avgMs: n === 0 ? 0 : sum / n,
      p50Ms: percentile(lags, 0.5),
      p95Ms: percentile(lags, 0.95),
      p99Ms: percentile(lags, 0.99),
      maxMs: max,
    };
  }

  private maybeAlert(nowAt: number, snap: LagSnapshot): void {
    if (snap.samples < 10) return; // avoid noisy early alert
    if (this.alertCooldownMs > 0 && nowAt - this.lastAlertAt < this.alertCooldownMs) return;

    if (snap.p99Ms >= this.thresholdP99Ms) {
      this.lastAlertAt = nowAt;
      this.emitAlert(nowAt, snap, "p99", `event loop lag p99=${snap.p99Ms.toFixed(1)}ms >= ${this.thresholdP99Ms}ms`);
      return;
    }

    if (snap.maxMs >= this.thresholdMaxMs) {
      this.lastAlertAt = nowAt;
      this.emitAlert(nowAt, snap, "max", `event loop lag max=${snap.maxMs.toFixed(1)}ms >= ${this.thresholdMaxMs}ms`);
      return;
    }
  }

  private emitAlert(at: number, snapshot: LagSnapshot, triggeredBy: "p99" | "max", message: string): void {
    const alert: LagAlert = { at, snapshot, triggeredBy, message };
    if (this.onAlert) this.onAlert(alert);
  }
}

// ---------- Demo: generate lag ----------
function busyWait(ms: number): void {
  const start = performance.now();
  while (performance.now() - start < ms) {
    // burn CPU
  }
}

async function demo() {
  const mon = new EventLoopLagMonitor({
    sampleIntervalMs: 100,
    windowMs: 10_000,
    alertCooldownMs: 3_000,
    threshold: { p99Ms: 120, maxMs: 250 },
    onSnapshot: (s) => {
      // log every ~1s (throttle in real systems)
      if (s.at % 1000 < 120) {
        console.log(JSON.stringify({ kind: "snapshot", ...s }));
      }
    },
    onAlert: (a) => {
      console.log(JSON.stringify({ kind: "ALERT", ...a }));
    },
  });

  mon.start();

  // pattern: every 2s, block CPU for 300ms to trigger alerts
  const t = setInterval(() => busyWait(300), 2_000);
  t.unref?.();

  // run for a bit
  await new Promise<void>((r) => setTimeout(r, 12_000));
  clearInterval(t);
  mon.stop();
}

if (process.argv[1]?.endsWith("kata16.ts") || process.argv[1]?.endsWith("kata16.js")) {
  demo().catch((e) => {
    console.error("DEMO FAILED:", e);
    process.exitCode = 1;
  });
}
```

---

## Tasks bạn phải làm (cụ thể)

1. **Eviction đúng window**: `evictOld()` giữ samples trong `windowMs`
2. **Percentiles đúng**: `percentile()` chạy đúng cho p50/p95/p99
3. **Debounce alert**: `alertCooldownMs` hoạt động
4. **Demo**: `busyWait(300)` phải tạo alert (p99 hoặc max)
5. **Overhead nhỏ**: sampleInterval 100ms, window 30s vẫn ổn

---

## Required behaviors

* Khi CPU block, snapshot cho thấy `maxMs` tăng mạnh, `p99Ms` tăng theo
* Alert chỉ bắn tối đa 1 lần mỗi `alertCooldownMs`
* Stop thì interval dừng, không leak

---

## Stretch (Senior+/Production)

1. **Option A**: dùng `monitorEventLoopDelay()` (perf_hooks) để lấy histogram và percentiles “xịn” hơn
2. **Health endpoint**: expose `GET /healthz` trả `LagSnapshot` gần nhất
3. **Auto mitigation**:

   * nếu lag > threshold: giảm concurrency limiter (nối Kata 15)
4. **SLO-driven**: alert khi `p99 > X` liên tục 3 windows liên tiếp (anti-flap)
5. **GC correlation**: log `heapUsed` cùng snapshot để phân biệt CPU vs memory pressure