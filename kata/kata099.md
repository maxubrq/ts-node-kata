# Kata 99 — Release Playbook: Canary, Rollback, Feature Flag

## Goal (Senior+)

Bạn sẽ xây một “release system” tối thiểu cho 1 service Node:

1. **Feature flag** (kill switch) + config rollout.
2. **Canary rollout plan** (10% → 25% → 50% → 100%).
3. **Automated gates** dựa trên metrics (error rate, p95 latency).
4. **Rollback plan** rõ ràng (code rollback + flag rollback).
5. **Runbook khi incident**: triage, mitigate, verify, postmortem hooks.
6. **Evidence artifacts**: checklist + decision log.

---

## Context (bài toán cố định)

Bạn vừa làm xong một thay đổi “rủi ro vừa”:

* bật **new ranking algorithm** (hoặc `search_rank_v2` từ Kata 96)
* có thể tăng CPU/latency và có thể bug logic

Yêu cầu:

* không được deploy “big bang”
* phải có rollback nhanh **không cần redeploy** (flag off)

---

## Constraints

* ✅ Không phụ thuộc platform cụ thể (K8s, ECS…) nhưng playbook phải map được.
* ✅ Có flag service / config loader (file/env) + runtime refresh (poll).
* ✅ Có “simulated metrics” để test gates (không cần Prometheus thật).
* ✅ Có “release checklist” + “rollback checklist”.
* ❌ Không viết playbook kiểu chung chung; phải có **threshold số** và **timeline**.

---

## Deliverables

```
packages/kata-99/
  src/
    flags/
      flags.ts
      provider.ts
    metrics/
      metrics.ts
      gates.ts
    release/
      playbook.md
      checklist.md
      decision-log.md
      rollback.md
    app/
      server.ts
      ranking.ts
      ranking_v2.ts
  scripts/
    simulate-release.ts
  test/
    flags.spec.ts
    gates.spec.ts
```

Bạn giao:

1. `playbook.md` (core)
2. `checklist.md` (pre + during + post)
3. `rollback.md` (flag rollback + code rollback)
4. `simulate-release.ts` chạy mô phỏng canary + gate pass/fail
5. Tests cho flags + gates

---

# Part 1 — Feature Flag system (bắt buộc)

## Requirements

* Flag có:

  * `enabled: boolean`
  * `rolloutPercent: number` (0–100)
  * `allowlistUserIds?: string[]` (optional)
  * `killSwitch: boolean` (override off)
* Evaluation rule:

  1. nếu `killSwitch=true` → always OFF
  2. nếu userId trong allowlist → ON
  3. else hash(userId) < rolloutPercent → ON

### `src/flags/flags.ts` skeleton

```ts
export type Flag = {
  key: string;
  enabled: boolean;
  rolloutPercent: number; // 0..100
  killSwitch?: boolean;
  allowlistUserIds?: string[];
};

export type FlagsSnapshot = Record<string, Flag>;

function hashTo0_99(input: string): number {
  // deterministic stable hash
  let h = 2166136261;
  for (let i = 0; i < input.length; i++) {
    h ^= input.charCodeAt(i);
    h = Math.imul(h, 16777619);
  }
  return (h >>> 0) % 100;
}

export function isFlagOn(snapshot: FlagsSnapshot, key: string, userId: string): boolean {
  const f = snapshot[key];
  if (!f) return false;
  if (f.killSwitch) return false;
  if (!f.enabled) return false;
  if (f.allowlistUserIds?.includes(userId)) return true;
  return hashTo0_99(userId) < f.rolloutPercent;
}
```

### Provider (runtime refresh)

* Poll file `flags.json` mỗi 2s (hoặc env reload simulation)
* Expose current snapshot
* Log when flags change (audit trail)

---

# Part 2 — Canary Plan (bắt buộc có timeline)

Bạn phải viết trong `playbook.md` đúng format:

## Canary steps

* Step 0: **Dark launch** (enabled=true, rolloutPercent=0, allowlist internal QA)
* Step 1: 10% for 15 minutes
* Step 2: 25% for 30 minutes
* Step 3: 50% for 60 minutes
* Step 4: 100%

### Success criteria gates (must be numeric)

Trong mỗi step, evaluate metrics window (rolling 5m):

* Error rate (5xx): **<= 0.3%**
* p95 latency: **<= baseline_p95 * 1.10** (<= +10%)
* Saturation (event loop lag / CPU proxy): **<= threshold** (bạn chọn số, ví dụ lag p95 < 50ms)

Nếu fail → rollback.

---

# Part 3 — Metrics + Gates (bắt buộc)

Bạn implement “gates” độc lập platform:

### `src/metrics/metrics.ts`

* thu thập:

  * requestCount
  * errorCount
  * latency samples (ms)
  * optionally eventLoopLag samples

### `src/metrics/gates.ts`

* function `evaluateGates(current, baseline, thresholds)` trả:

  * PASS/FAIL
  * reasons list
  * recommended action

Bạn cần tests:

* fail khi error rate vượt
* fail khi p95 > baseline*1.1
* pass nếu trong ngưỡng

---

# Part 4 — Rollback plan (bắt buộc 2 loại)

## Rollback Type A — Flag rollback (fastest)

* Set `killSwitch=true` hoặc `rolloutPercent=0`
* Verify:

  * canary traffic stops using v2 (log marker)
  * metrics recover within 5–10 minutes

## Rollback Type B — Code rollback

* Revert deploy (image tag N-1)
* Verify:

  * no schema mismatch
  * outbox/event schema backward compatible
* Note: Code rollback chậm hơn, dùng khi flag rollback không đủ (bug affects both paths)

Bạn phải viết `rollback.md` gồm:

* Trigger conditions
* Steps
* Verification checklist
* Communication template ngắn (status update)

---

# Part 5 — “What to watch” (đúng production)

Bạn phải thêm vào playbook:

### Golden signals

* Traffic: RPS
* Errors: 5xx, timeouts
* Latency: p50/p95/p99
* Saturation: CPU proxy / event loop lag / queue depth

### Debug hooks (safe)

* Request log sampling theo:

  * error
  * slow requests (p95+)
* Correlation id

---

# Part 6 — Simulated release (bắt buộc)

`scripts/simulate-release.ts`:

* giả lập baseline metrics
* giả lập canary step metrics thay đổi theo rolloutPercent
* đôi lúc inject regression (latency+error) để thấy gate fail → rollback flag

Output:

* từng step: PASS/FAIL + reasons
* final: “rolled out” hoặc “rolled back”

---

# Checklists (bắt buộc)

## Pre-release checklist (10–15 dòng)

* migrations backward compatible
* feature flag default OFF
* dashboards ready
* rollback tested (flag + code)
* SLO thresholds confirmed
* on-call aware

## During release checklist

* set step percent
* monitor 5m
* evaluate gates
* decision log entry (who/when/why)

## Post-release checklist

* remove temporary debug logs
* set flag to fully ON + remove allowlist
* write short post-release note: metrics delta, lessons

---

# Definition of Done (pass kata)

Bạn pass nếu:

1. Có **flag evaluation** deterministic + runtime refresh.
2. Có **playbook** với:

   * canary steps + timeline
   * numeric gates
   * rollback procedures
3. Có **simulate-release** chứng minh gates & rollback hoạt động.
4. Có tests cho flags + gates.
5. Có decision log template.

---

# Stretch (Senior+ → Staff-grade)

Chọn ≥ 6:

1. **Blast radius control**: canary theo tenant tier / region.
2. **Progressive delivery**: auto-advance step nếu PASS liên tục 2 windows.
3. **Error budget**: gate dựa trên burn rate (SLO-ish) thay vì % cứng.
4. **Schema + event compatibility section**: expand/contract & event versioning notes.
5. **Human factors**: comms template cho incident channel + stakeholder update cadence.
6. **Rollback drill**: script giả lập “flip killSwitch” và verify metrics recover.
7. **Post-release cleanup plan**: remove flag, remove old code path, timeline.