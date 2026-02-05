# Kata 56 — Alert Design (SLO-ish)

*Thiết kế alert “ít nhưng đúng”: bắt sự cố thật, giảm noise, có runbook, có test bằng incident simulation.*

---

## 1) Goal (năng lực cần đạt)

Sau kata này bạn làm được một “alert pack” production-grade cho 1 service:

* Alert **ít** (khoảng 5–8 cái) nhưng cover đủ: **availability, latency, errors, saturation, readiness**.
* Alert theo tư duy **SLO-ish** (burn rate / multi-window) thay vì ngưỡng cứng bừa bãi.
* Mỗi alert có:

  * **Mục đích** (đánh vào user impact nào)
  * **Query** rõ ràng
  * **Severity** (page vs ticket)
  * **Runbook ngắn** (1–2 phút triage, link metrics/traces/logs)
* Có **incident simulation** + test: khi bạn inject lỗi/latency/saturation, alert phải “nổ đúng” và “không nổ sai”.

---

## 2) Context (bài toán)

Bạn có service đã làm các kata trước (tối thiểu phải có):

* Metrics:

  * `http_requests_total{route,method,status_class}`
  * `http_request_duration_seconds_bucket{route,...}` (latency histogram)
  * `node_event_loop_lag_seconds` (hoặc metric tương đương)
  * `http_in_flight_requests`
  * `readiness_status` (gauge 1/0 hoặc derived từ /readyz)
* Tracing/log correlation (tầng 6)

Bạn sẽ tạo:

* `alerts/alerts.yml` (Prometheus rules)
* `runbooks/` (markdown ngắn)
* `scripts/simulate.ts` (inject incident patterns)
* `test/alerts.spec.ts` (assert query logic bằng promtool-like harness hoặc “golden expectation”)

---

## 3) Constraints (Senior+ bắt buộc)

1. **Không** alert theo ngưỡng đơn lẻ kiểu “p95 > 500ms trong 1m” cho mọi thứ.
2. Mỗi alert phải:

   * Có **for:** để chống spike ngắn
   * Có **labels**: `severity`, `service`, `team`
   * Có **annotations**: `summary`, `description`, `runbook`
3. Phải phân biệt:

   * **Paging**: user-impact đáng tin
   * **Ticket**: sớm nhưng không đánh thức
4. **Giảm noise**:

   * Dùng **multi-window** (fast burn + slow burn)
   * Có **traffic guard** (đừng alert khi QPS gần 0)
5. **Runbook** phải trỏ thẳng:

   * metric nào nhìn trước
   * query nào
   * bước 1–2–3 để khoanh vùng

---

## 4) SLO model (đơn giản nhưng đúng)

Bạn chọn 1 SLO (ví dụ cho HTTP):

* **Availability SLO**: 99.9% “good” requests / 30 ngày
  Good = status_class != 5xx (hoặc status<500)

* **Latency SLO**: 99% requests < 500ms (tuỳ chọn)
  (Dùng histogram để đo “good latency”)

**Error budget** = 1 - SLO.

Bạn sẽ alert theo **burn rate** (đốt budget nhanh quá).

---

## 5) Alert pack tối thiểu (5–8 alerts)

### Alert 1 — Readiness down (Paging)

**Ý nghĩa**: service không nhận traffic được (kube sẽ rút khỏi LB) → user impact.

Query (gợi ý):

* Nếu bạn có gauge `service_ready` (1/0):

```promql
avg_over_time(service_ready[2m]) < 0.5
```

Traffic guard (tuỳ):

* hoặc chỉ cần readiness là đủ.

Config:

* `for: 1m`
* `severity: page`

Runbook: check `/readyz`, recent deploy, dependencies.

---

### Alert 2 — Availability burn fast (Paging)

**Ý nghĩa**: 5xx tăng nhanh → đốt budget nhanh.

Burn rate (fast window 5m):

```promql
(
  sum(rate(http_requests_total{status_class="5xx"}[5m]))
  /
  sum(rate(http_requests_total[5m]))
) > 0.02
```

Traffic guard:

```promql
sum(rate(http_requests_total[5m])) > 1
```

Config:

* `for: 2m`
* `severity: page`

> 0.02 ở đây chỉ là ví dụ. Bạn phải justify theo SLO/traffic.

---

### Alert 3 — Availability burn slow (Ticket)

**Ý nghĩa**: lỗi “rỉ rả” nhưng kéo dài.

```promql
(
  sum(rate(http_requests_total{status_class="5xx"}[1h]))
  /
  sum(rate(http_requests_total[1h]))
) > 0.005
and
sum(rate(http_requests_total[1h])) > 0.2
```

* `for: 30m`
* `severity: ticket`

---

### Alert 4 — Latency tail burn fast (Paging)

Dùng p99 hoặc p95 tùy SLO.

p99 theo route (rollup):

```promql
histogram_quantile(
  0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
) > 1.5
and
sum(rate(http_requests_total[5m])) > 1
```

* `for: 5m`
* `severity: page`

> Senior+ note: Nếu hệ ít traffic, p99 rất rung. Có thể dùng p95 cho paging, p99 cho ticket.

---

### Alert 5 — Latency tail burn slow (Ticket)

```promql
histogram_quantile(
  0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket[1h]))
) > 0.8
and
sum(rate(http_requests_total[1h])) > 0.2
```

* `for: 30m`
* `severity: ticket`

---

### Alert 6 — Saturation: event loop lag (Paging hoặc Ticket tùy)

Nếu bạn có `node_event_loop_lag_seconds` gauge/hist:

```promql
avg_over_time(node_event_loop_lag_seconds[5m]) > 0.1
and
sum(rate(http_requests_total[5m])) > 1
```

* `for: 5m`
* `severity: page` (nếu service CPU-bound critical) hoặc `ticket`

---

### Alert 7 — Saturation: in-flight stuck / backlog (Ticket)

```promql
avg_over_time(http_in_flight_requests[10m]) > 200
and
histogram_quantile(0.95, sum by (le)(rate(http_request_duration_seconds_bucket[10m]))) > 0.8
```

* `for: 10m`
* `severity: ticket`

> Ý tưởng: in-flight cao *kèm* latency cao → saturation thật, tránh false positive.

---

## 6) Deliverables chi tiết

### 6.1 `alerts/alerts.yml`

* 1 group: `kata56-service-alerts`
* mỗi alert có labels: `severity`, `service="kata-service"`, `team="core"`
* annotations: `summary`, `description`, `runbook`

### 6.2 `runbooks/*.md` (mỗi alert 1 file, max 1 trang)

Template runbook bắt buộc:

* **What it means** (user impact)
* **Immediate checks (2 minutes)**

  * QPS? error rate? p95/p99?
  * readiness?
  * deploy vừa xảy ra?
* **Likely causes**
* **Mitigations**

  * rollback / feature flag / shed load / scale up / disable expensive path
* **Deep dive**

  * link trace query (by trace_id exemplar hoặc log filter)

### 6.3 `scripts/simulate.ts`

Tạo 4 chế độ:

* `--mode=errors` (tăng 5xx)
* `--mode=latency-tail` (10% requests 2s)
* `--mode=cpu-block` (block event loop 200ms mỗi tick)
* `--mode=dep-down` (readiness fail)

### 6.4 Tests

Bạn có 2 lựa chọn (cái nào làm được):

1. **Logic tests**: verify promQL expressions “đúng cấu trúc” + thresholds (lint-like)
2. **Replay tests**: feed time-series synthetic (đơn giản) và assert alert should fire/not fire.

---

## 7) Checklist “ít nhưng đúng” (bạn tự audit)

* [ ] Có ít nhất 1 alert cho **availability**
* [ ] Có ít nhất 1 alert cho **latency tail**
* [ ] Có ít nhất 1 alert cho **saturation**
* [ ] Có readiness alert (hạ LB)
* [ ] Có fast + slow window cho availability/latency (SLO-ish)
* [ ] Có traffic guard
* [ ] Runbook mỗi alert ≤ 1 trang, triage 2 phút
* [ ] Simulation khiến alert nổ đúng, mode bình thường không nổ

---

## 8) Definition of Done (pass/fail)

PASS khi:

1. Có 5–8 alerts như pack ở trên (hoặc tương đương) + justify thresholds.
2. Mỗi alert có severity + for + runbook link.
3. Có simulation script tạo ra ít nhất 3 incident patterns.
4. Có test/harness chứng minh “nổ đúng / không nổ sai” ở ít nhất 2 pattern.

---

## 9) Stretch (Senior++)

1. **True burn rate multi-window** kiểu Google SRE:

   * page nếu burn rate 14.4x trong 5m và 6x trong 1h (ví dụ)
2. **Route-level paging** chỉ cho “critical routes”, route khác ticket.
3. **Error budget dashboard**: remaining budget theo 30d.
4. **Alert dedupe**: latency page suppressed nếu readiness down (root cause).