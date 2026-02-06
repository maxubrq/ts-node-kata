# Kata 96 — BFF vs API: Same Problem, Two Designs, One Decision

## Goal (Senior+)

Bạn sẽ:

1. Thiết kế 2 phương án cho cùng use-case:

   * **Option A: Pure API** (client tổng hợp)
   * **Option B: BFF** (backend tổng hợp cho từng client)
2. Implement “mini working slice” cho cả 2 (mock downstream ok).
3. So sánh trade-offs theo tiêu chí production:

   * latency/tail, cacheability, coupling, release velocity
   * authz, data shaping, over/under-fetching
   * observability, failure handling
4. Chốt **Decision** (có điều kiện) + rollout plan.

---

## Context (bài toán cố định)

Bạn có 2 client:

* **Web** (dashboard, cần nhiều dữ liệu + linh hoạt)
* **Mobile** (bandwidth thấp, cần payload nhỏ, ít round-trips)

Bạn có 3 downstream services (giả lập):

1. `UserService` — profile + preferences
2. `OrderService` — recent orders + totals
3. `RecsService` — recommendations (hay timeout)

### Endpoint UX cần

Màn hình “Home” cần response:

```ts
type HomeDTO = {
  user: { id: string; name: string; tier: "free" | "pro" };
  stats: { orderCount30d: number; spend30d: number };
  recentOrders: { id: string; total: number; status: string }[];
  recs: { id: string; title: string }[];
  meta: { requestId: string; generatedAt: number };
};
```

Constraints UX:

* p95 target: **< 250ms**
* Nếu `RecsService` fail: vẫn trả Home nhưng `recs=[]` + warning flag
* Multi-tenant: `tenantId` bắt buộc (không leak)
* Auth: user token → scope

---

## Constraints

* ✅ Node + TS.
* ✅ Không dùng framework nặng; Fastify/Express đều được.
* ✅ Downstream có thể mock bằng in-process handlers nhưng phải có latency/failure injection.
* ✅ Có correlation id + structured logs.
* ✅ Có 1 load simulation nhẹ (hoặc benchmark request loop) để so p95.
* ✅ Viết “Decision Record” (ADR) trong README.
* ❌ Không chỉ viết lý thuyết; phải có code chạy được.

---

## Deliverables

```
packages/kata-96/
  src/
    common/
      types.ts
      http.ts
      observability.ts
      errors.ts
    downstream/
      user.ts
      orders.ts
      recs.ts
    optionA_pure_api/
      gateway.ts
      routes.ts
    optionB_bff/
      bff.ts
      routes.ts
      cache.ts
    load/
      run.ts
  test/
    contract.spec.ts
    failure.spec.ts
  README.md
```

Bạn giao:

1. Option A chạy: client call 3 endpoints rồi compose (simulate “client aggregation” ở gateway).
2. Option B chạy: 1 endpoint `/home` trả đúng `HomeDTO`.
3. Load script đo p50/p95 cho cả 2.
4. ADR: quyết định khi nào chọn BFF, khi nào không.

---

# Phần 1 — Setup downstream (bắt buộc)

## Downstream behavior (latency + failure injection)

* User: 30–60ms, hiếm fail
* Orders: 50–120ms, đôi lúc slow
* Recs: 80–200ms, **10% timeout**, **tail nặng**

Bạn phải implement downstream mocks có:

* deterministic randomness (seed)
* latency injection
* timeout simulation

---

# Phần 2 — Option A: Pure API (client aggregates)

## Design

Bạn expose 3 endpoint:

* `GET /user`
* `GET /orders/summary`
* `GET /recs`

Client (web/mobile) gọi 3 cái, tự compose.

### Bạn phải làm rõ:

* Mobile sẽ suffer do 3 round-trips
* Cache ở edge/CDN possible cho từng endpoint
* Client logic phình to + versioning phức tạp

## Code task (minimum)

* gateway “client simulator” thực hiện 3 calls song song, đo total time.
* handle partial failure: recs fail → degrade.

---

# Phần 3 — Option B: BFF (backend aggregates)

## Design

Expose:

* `GET /home` trả `HomeDTO`
* BFF gọi 3 downstream, aggregate, shape output cho:

  * `/home?client=web`
  * `/home?client=mobile` (payload nhỏ hơn)

### BFF requirements (Senior+)

* **Timeout budget** + deadline propagation (Kata 84 mindset)
* **Bulkhead**: recs không được kéo sập toàn request
* **Cache**:

  * cache user profile 30s
  * cache stats/orders 5s
  * recs cache 10s (stale ok)
* **SWR** optional (bonus)
* Observability: traces/logs per downstream

---

# Phần 4 — Cross-cutting requirements (bắt buộc cả 2)

## 1) Latency budget table

Trong README, bạn phải có bảng:

| Component            | p50 | p95 | Budget |
| -------------------- | --- | --- | ------ |
| User                 | …   | …   | 70ms   |
| Orders               | …   | …   | 120ms  |
| Recs                 | …   | …   | 150ms  |
| Aggregation overhead | …   | …   | 20ms   |
| Total                | …   | …   | 250ms  |

## 2) Failure model

* Recs timeout → degrade gracefully
* Orders slow → still return user + empty recentOrders? (bạn quyết định)
* User fail → request fail (bạn quyết định) + explain

## 3) Contract tests

* `HomeDTO` must match schema for both clients.
* Tenant isolation enforced in keys + routes.

---

# Phần 5 — Measurement (bắt buộc)

Bạn phải chạy load simulation 500–2000 requests (sequential ok nhưng phải có concurrency option) và report:

* mean / p50 / p95
* error rate
* % degraded responses (recs empty due to timeout)
* downstream call counts (để thấy benefit cache/BFF)

---

# Kata task list (copy như checklist)

### A. Implement downstream mocks

* [ ] user/orders/recs handlers với latency + failure injection
* [ ] deterministic seed + ability to configure (env)

### B. Option A: Pure API

* [ ] 3 endpoints
* [ ] “client aggregator” đo end-to-end time (simulate web/mobile)
* [ ] degrade on recs fail

### C. Option B: BFF

* [ ] `/home?client=web|mobile`
* [ ] parallel fanout có cap + timeout per downstream
* [ ] cache policy + stampede protection (singleflight)
* [ ] degrade strategy documented

### D. Observability

* [ ] request_id propagate
* [ ] structured logs with downstream timing
* [ ] metrics counters (hits, misses, degraded)

### E. Measurement

* [ ] load script + report p95
* [ ] compare Option A vs B

### F. ADR (Decision Record)

* [ ] decision + when to revisit

---

# ADR template (bắt buộc điền)

Tạo `README.md` có section:

```md
## ADR: BFF vs Pure API

### Decision
Chọn: (A) / (B) / Hybrid

### Context
- Clients: web + mobile
- Constraints: p95 < 250ms, recs flaky
- Team constraints: release cadence, ownership, observability maturity

### Options considered
A. Pure API (client aggregation)
B. BFF aggregation

### Trade-offs
- Latency & tail:
- Cacheability:
- Coupling & versioning:
- Security surface:
- Ops complexity:

### Decision drivers (top 3)
1) …
2) …
3) …

### Consequences
Good:
Bad:
Mitigations:

### Rollout plan
- Feature flag / canary
- Metrics to watch
- Rollback criteria
```

---

# Checks (Definition of Done)

Bạn pass kata nếu:

1. Cả Option A và Option B chạy được, trả đúng output.
2. Có degrade đúng khi recs timeout.
3. Có load report p50/p95 cho cả 2.
4. Có ADR chốt quyết định dựa trên số liệu + trade-offs.
5. Có tenant isolation test.
6. Có observability tối thiểu (request_id + downstream timing logs).

---

# Stretch (Senior+ → Staff-grade)

Chọn ≥ 5:

1. **Hybrid**: giữ core APIs, thêm BFF chỉ cho mobile.
2. **Schema evolution**: version field cho `HomeDTO`, deprecate plan.
3. **Edge caching vs BFF caching**: phân tích cụ thể cache layer placement.
4. **Cost model**: requests count, CPU time, egress size; so sánh $$.
5. **Resilience patterns**: circuit breaker cho recs + fallback cache.
6. **Multi-region**: BFF đặt gần client hay gần downstream? latency trade-off.
7. **Contract tests**: Pact-ish (tự viết) giữa BFF và downstream.