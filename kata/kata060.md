# Kata 60 — Postmortem Template

*Viết ngắn nhưng sắc: nhìn 5 phút hiểu chuyện, 30 phút biết phải làm gì tiếp.*

---

## 1) Goal (năng lực cần đạt)

Sau kata này bạn:

* Viết **postmortem 1–2 trang** nhưng **đủ lực** cho engineering + ops.
* Tách bạch **impact / cause / fix / follow-ups** (không trộn).
* Chứng minh **fix là thật** bằng dữ liệu (artifact từ Kata 59).
* Không đổ lỗi cá nhân; tập trung **systemic causes**.
* Có **owner + deadline** cho follow-ups (không để trôi).

---

## 2) Context (đầu vào bắt buộc)

Bạn **phải dùng dữ liệu thật** từ Incident Repro Harness (Kata 59):

* Artifact JSON (latency p50/p95/p99, error rate, saturation)
* Logs/traces mẫu (trace_id)
* Alert đã bắn (Kata 56)
* Readiness/health behavior (Kata 55)

> Không có dữ liệu → postmortem **không đạt**.

---

## 3) Template chuẩn (copy–paste dùng ngay)

> **Quy ước:** mỗi section ≤ 6–8 bullet. Không viết văn. Không kể lể.

---

### 0. Metadata

* **Incident ID:** INC-YYYYMMDD-XX
* **Service:** `<service-name>`
* **Environment:** prod / staging
* **Start:** `<timestamp>`
* **End:** `<timestamp>`
* **Duration:** `<minutes>`
* **Severity:** SEV-1 / SEV-2 / SEV-3
* **Owner:** `<name>`
* **Status:** Draft / Final

---

### 1. Executive Summary (5–7 dòng, bắt buộc)

* **What happened:** (1–2 câu, plain English)
* **User impact:** (ai bị ảnh hưởng, bao nhiêu %, trong bao lâu)
* **Root cause:** (1 câu, *system-level*)
* **Resolution:** (1 câu, fix chính)
* **Prevention:** (1 câu, sẽ ngăn lặp lại thế nào)

> Nếu người không thuộc team đọc phần này mà **chưa hiểu**, bạn viết lại.

---

### 2. Impact (định lượng, không cảm tính)

* **Availability:** X% requests fail (5xx)
* **Latency:** p95 từ A → B ms; p99 từ C → D ms
* **Throughput:** QPS giảm/tăng bao nhiêu
* **Blast radius:** routes / tenants / regions
* **User-visible symptoms:** timeout / error message / degraded feature

*Attach:* link artifact + dashboard query.

---

### 3. Detection (vì sao biết, biết sớm hay muộn)

* **Primary alert:** `<alert-name>` (time)
* **Secondary signals:** logs/traces/readyz
* **Detection delay:** `<minutes>`
* **What worked:** alert đúng / signal rõ
* **What didn’t:** noise / missing alert / false negative

---

### 4. Timeline (facts only)

> Mỗi dòng = **time + fact**. Không suy diễn.

* T0 — Deploy version X
* T+5m — Latency p95 vượt ngưỡng
* T+7m — Alert A fired
* T+10m — Traffic shifted / mitigation
* T+18m — Rollback complete
* T+25m — Metrics back to baseline

---

### 5. Root Cause (5 Whys gọn, hệ thống)

* **Symptom:** p95 tăng mạnh
* **Why #1:** downstream payment latency spike
* **Why #2:** retry + timeout mismatch gây cascade
* **Why #3:** no bulkhead between routes
* **Why #4:** missing load test for tail latency
* **Root cause:** *Retry policy + timeout misaligned under tail latency; lack of isolation.*

> Không nêu tên người. Không dùng “human error”.

---

### 6. Contributing Factors (không phải root)

* Alert threshold hơi cao → detection chậm
* Log sampling chưa ưu tiên tail đủ
* Readiness chưa flip nhanh khi dep timeout

---

### 7. Resolution (đã làm gì để dập lửa)

* Rollback to version X-1
* Disable feature flag `<name>`
* Reduce concurrency to N
* Increase timeout temporarily

*Evidence:* timestamps + metric snapshot.

---

### 8. Fix Verification (chứng minh fix thật)

* **Repro harness:** `latency-tail --seed 1337`
* **Before:** p95=1200ms, error rate=3.2%
* **After:** p95=320ms, error rate=0.3%
* **Result:** PASS (see artifact `<path>`)

> **Bắt buộc** có so sánh before/after.

---

### 9. Follow-ups (actionable, có owner)

| Action                     | Type    | Owner | Due        |
| -------------------------- | ------- | ----- | ---------- |
| Align retry/timeout policy | Prevent | A     | YYYY-MM-DD |
| Add bulkhead for /orders   | Prevent | B     | YYYY-MM-DD |
| Add SLO burn alert (slow)  | Detect  | C     | YYYY-MM-DD |
| Extend harness scenario    | Test    | D     | YYYY-MM-DD |

> Không có owner/due date → coi như **chưa làm**.

---

### 10. Lessons Learned (3–5 bullets)

* Tail latency > average; test phải cover tail.
* Retry không miễn phí; cần budget.
* Isolation rẻ hơn scale khi có cascade.

---

### 11. Appendix (tuỳ chọn)

* Links dashboards
* Trace IDs tiêu biểu
* Config diffs
* Harness artifacts

---

## 4) Constraints (Senior+ bắt buộc)

1. **≤ 2 trang A4** (markdown ~ 800–1200 từ).
2. Mọi claim kỹ thuật **phải có số liệu**.
3. Không đề xuất giải pháp mơ hồ (“improve monitoring”).
4. Follow-up phải **phòng ngừa**, không chỉ “phát hiện”.
5. Viết để **người mới vào team** vẫn hiểu.

---

## 5) Definition of Done (pass/fail)

PASS khi:

* Executive Summary rõ trong 60 giây.
* Root cause là **systemic**, không cá nhân.
* Có **repro + verification** bằng dữ liệu.
* Follow-ups rõ owner + deadline.
* Đọc xong biết **làm gì tiếp**.

---

## 6) Stretch (Senior++)

1. **One-pager for exec:** rút gọn còn 1 trang, tập trung impact + prevention.
2. **Error budget math:** tính budget burned (%/day).
3. **Counterfactual:** “Nếu có X từ trước, incident có xảy ra không?”
4. **Pattern extraction:** ghi lại thành “Incident Pattern #N” cho knowledge base.