# Kata 92.1 — Staff-grade CPU Profiling

**From Hot Path → Decision → Org-safe Rollout**

> Mục tiêu: biến kết quả profiling từ “tối ưu local” thành **quyết định kỹ thuật có trách nhiệm**, có thể sống lâu trong production.

---

## Deliverable tổng thể (bắt buộc)

Sau kata này, bạn phải có **1 tài liệu duy nhất** (markdown hoặc PDF):

> **Performance Investigation Report**

Nó phải đủ tốt để:

* Gửi cho Tech Lead / EM
* Gắn vào PR description
* Dùng làm post-incident learning
* Dùng làm reference cho dev khác sau này

---

## 1. Performance Investigation Report (chuẩn Staff)

### 1.1 Problem Statement (không nói code)

**Không được bắt đầu bằng: “hàm X chậm”**

Phải viết như này:

```md
## Problem

- Service: Search / Ranking
- Symptom:
  - p95 latency tăng từ ~120ms → ~260ms khi traffic > 2k RPS
  - CPU usage tăng tuyến tính theo traffic, không plateau
- Business impact:
  - Search response chậm → drop conversion ~3–5%
  - Autoscaling scale nhanh nhưng không cứu được tail latency
```

> Staff mindset: **luôn gắn performance với outcome**, không chỉ CPU%.

---

### 1.2 Hypotheses (trước khi profile)

Viết **ít nhất 3 giả thuyết**, dù biết có thể sai.

Ví dụ:

```md
## Initial Hypotheses

H1. CPU bị đốt bởi tokenization lặp lại cho mỗi item  
H2. Full array sort gây O(n log n) không cần thiết  
H3. Allocation pressure (string/array) gây GC pause, ảnh hưởng tail
```

> Luật: **Giả thuyết phải falsifiable** (profile có thể chứng minh đúng/sai).

---

### 1.3 Evidence — Profiling Artifacts

Đây là chỗ phân biệt Senior vs Staff.

Bạn phải có **bằng chứng thô**, không chỉ lời kể.

Bắt buộc ghi:

```md
## Profiling Evidence

Tool:
- Node v20.11
- --cpu-prof (30s workload)

Workload:
- 150k items
- 5 rotating queries
- ~20s steady-state
```

Và **bảng evidence**:

| Rank | Function / Stack   | Self Time | Total Time | Notes              |
| ---- | ------------------ | --------- | ---------- | ------------------ |
| 1    | tokenize()         | 31%       | 31%        | split + lowercase  |
| 2    | Array.sort         | 18%       | 18%        | full sort for topK |
| 3    | String.toLowerCase | 14%       | 22%        | repeated per item  |
| 4    | containsToken loop | 11%       | 19%        | O(n) scan          |

> Không có bảng này → **chưa phải Staff**.

---

### 1.4 Diagnosis (Why this dominates CPU)

Phần này **không được mô tả lại evidence**, mà phải **giải thích cơ chế**.

Ví dụ:

```md
## Diagnosis

The dominant cost is repeated normalization and tokenization:

- Each rank() call:
  - tokenizes query once (cheap)
  - tokenizes title/body/tags for every item (expensive)
- This creates:
  - high string allocation
  - high short-lived arrays
- Allocation pressure increases GC frequency,
  amplifying p95 latency under load.
```

> Staff signal: bạn **giải thích được causal chain**, không chỉ “cái này chiếm % cao”.

---

## 2. Decision Matrix (Staff bắt buộc)

Bạn **không được nhảy thẳng vào code**.

Phải có bảng trade-off:

| Option | Description                 | CPU | Memory | Complexity | Risk               |
| ------ | --------------------------- | --- | ------ | ---------- | ------------------ |
| A      | Pre-tokenize items          | ↓↓↓ | ↑↑     | Medium     | Cache invalidation |
| B      | Replace sort with topK heap | ↓   | =      | Medium     | Edge-case bug      |
| C      | Micro-opt tokenize          | ↓   | =      | Low        | Diminishing return |

Và **quyết định rõ ràng**:

```md
## Decision

Chosen: Option A + B

Reason:
- CPU reduction is structural (algorithmic)
- Memory increase (~+45MB per shard) is acceptable
- Aligns with steady-state workload (read-heavy)
```

> Staff không tối ưu “vì thích”, mà vì **phù hợp với workload & constraints**.

---

## 3. Optimization Phases (rất quan trọng)

### Phase 1 — Algorithmic

* Pre-normalize & pre-tokenize items
* Use Set for membership
* Replace full sort with top-K heap

### Phase 2 — Allocation cleanup

* Reuse arrays where possible
* Avoid intermediate strings
* Short-circuit scoring when max possible score < current minTopK

Bạn phải benchmark **mỗi phase riêng**.

Ví dụ bảng:

| Phase    | Mean ns/op | p95 ns/op | Δ vs baseline |
| -------- | ---------- | --------- | ------------- |
| Baseline | 18,200     | 24,900    | —             |
| Phase 1  | 9,800      | 13,100    | −46%          |
| Phase 2  | 8,700      | 11,900    | −52%          |

> Đây là **killer Staff signal**: bạn chứng minh **từng quyết định tạo ra bao nhiêu giá trị**.

---

## 4. Correctness & Safety Section (Staff-grade)

### 4.1 Correctness Contract

Viết rõ:

```md
## Correctness Contract

- For identical input set + query:
  - Output item IDs must match baseline
  - Ordering must be deterministic
- Tie-breaker:
  - score desc, then id asc
```

Có test dẫn chiếu.

---

### 4.2 Risk Analysis

Bắt buộc có:

```md
## Risks

- Memory overhead increases linearly with item count
- Cache invalidation required on item update
- Cold start slower due to preprocessing
```

Và **mitigation**:

```md
Mitigation:
- Lazy pre-tokenization with LRU
- Background rebuild on deploy
```

---

## 5. Benchmark & Regression Gate (org-level)

### 5.1 KPI bạn chọn

Bạn phải **chọn 1 KPI chính**, ví dụ:

* p95 ns/op
* or ops/sec under fixed CPU

Không được “mean + p95 + p99 lẫn lộn”.

---

### 5.2 CI Gate Spec

Viết rõ rule:

```md
## Regression Gate

- Optimized must be >= 1.3x faster than baseline (p95)
- Variance (stddev) must not increase > 10%
- If violated → CI fail
```

> Staff mindset: **performance là contract**, không phải cảm nhận.

---

## 6. Rollout & Monitoring Plan (rất Staff)

Bạn phải viết **sau khi merge thì sao**:

```md
## Rollout Plan

- Deploy behind feature flag: search_rank_v2
- Canary 10% traffic for 24h
- Monitor:
  - CPU per request
  - p95 latency
  - GC pause time
```

```md
## Rollback Criteria

- p95 latency increases > 10%
- Memory RSS exceeds budget
```

---

## 7. Counterfactual (cực mạnh)

Bắt buộc có **1 tối ưu thất bại**:

```md
## Counterfactual

Attempted:
- Micro-optimize tokenize() using regex reuse

Result:
- CPU improvement < 3%
- Profile still dominated by repeated calls

Conclusion:
- Algorithmic change dominates micro-optimization
```

> Đây là thứ **rất hiếm nhưng cực kỳ Staff**: chứng minh bạn **không cherry-pick kết quả**.

---

## 8. What This Unlocks (meta insight)

Kết report bằng phần này:

```md
## Learnings

- Profiling shifted focus from "fast code" to "right algorithm"
- Benchmarks + gate convert optimization into maintainable asset
- Memory vs CPU trade-off must be explicit, not accidental
```

---

## Definition of Done — 92.1 (Staff)

Bạn **chỉ pass** nếu:

1. Có **Performance Investigation Report hoàn chỉnh** (tất cả sections trên)
2. Có **profile artifact** + benchmark numbers
3. Có **decision matrix + trade-off**
4. Có **regression gate rule**
5. Có **rollout & rollback plan**
6. Có **counterfactual optimization**

Nếu thiếu **1 mục**, kata này **chưa đạt Staff**.

---

## Vì sao 92.1 quan trọng?

* Senior: *“Tôi làm code này nhanh hơn.”*
* Staff: *“Tôi đảm bảo hệ thống này sẽ không chậm trở lại.”*

---

Nếu bạn muốn, bước tiếp theo rất tự nhiên là:

* **Kata 93.1 — Memory Budget & Allocation Contract (Staff)**
* **Kata 94.1 — Cold Start as a First-Class Metric**