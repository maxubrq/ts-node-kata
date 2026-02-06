# Kata 100 — Production Spec: System Intent + Failure Model + Ops Runbook

## Goal (Senior+)

Bạn sẽ viết 1 tài liệu spec cho **một service cụ thể**, đủ để:

1. Người mới vào team đọc 30 phút hiểu **service tồn tại để làm gì**.
2. Engineer khác có thể **implement/extend** mà không phá intent.
3. On-call có thể **triage incident** và **roll back** nhanh.
4. SRE/Infra có thể **deploy** và **monitor** với SLO rõ ràng.

> Đây là “artifact” level Staff: *thiết kế + vận hành + thay đổi*.

---

## Context (service để bạn viết spec)

Chọn **một service** (khuyến nghị lấy từ các kata trước để nối liền):

**Option khuyến nghị: `Home Aggregator BFF`** (từ Kata 96)

* Endpoint: `GET /home?client=web|mobile`
* Fan-out: UserService, OrderService, RecsService
* Degrade khi recs fail
* Có caching + singleflight (Kata 95)
* Có deadline/timeouts (Kata 84 mindset)
* Có tracing/logging (Kata 52–58)
* Release bằng feature flag/canary (Kata 99)

Nếu bạn không chọn BFF, bạn vẫn phải có:

* 1 endpoint chính
* 1 dependency quan trọng (DB hoặc downstream)
* 1 failure mode “thực tế” (timeout, rate limit, data drift…)

---

## Constraints

* ✅ Spec phải **đủ chi tiết để build & run**.
* ✅ Có **System Intent** (what/why/constraints).
* ✅ Có **Failure Model** (failure modes + detection + mitigation).
* ✅ Có **Ops Runbook** (alerts, dashboards, triage steps, rollback).
* ✅ Có **SLO** + golden signals.
* ✅ Có **change safety** (schema/versioning/deprecation plan).
* ❌ Không viết kiểu “văn mẫu chung chung”. Tất cả phải gắn với service bạn chọn.

---

# Deliverable

1 file duy nhất: `PRODUCTION_SPEC.md` (hoặc `docs/production-spec.md`) theo template dưới.

---

# TEMPLATE — PRODUCTION_SPEC.md (copy-paste rồi điền)

## 0) Metadata

* Service name:
* Owner team:
* On-call rotation:
* Repo path:
* Environments: dev / staging / prod
* Last updated:
* Related services:

---

## 1) System Intent (Commander’s Intent)

### 1.1 Why this service exists

* One sentence mission:
* Business capability enabled:
* What “success” means (measurable):

### 1.2 In-scope / Out-of-scope

**In-scope**

* …
  **Out-of-scope**
* …

### 1.3 Non-goals (explicit)

* …

### 1.4 Primary constraints

* Latency target (p95/p99):
* Availability target:
* Consistency model:
* Security/privacy constraints:
* Cost constraints:

---

## 2) External Contract

### 2.1 API surface

List endpoints:

* `GET /home?client=web|mobile`

  * Auth:
  * Required headers:
  * Request schema:
  * Response schema (DTO):
  * Error model (Problem+JSON recommended):
  * Idempotency (if applicable):
  * Cache headers (if any):

### 2.2 Compatibility & versioning

* Versioning strategy (URL/header):
* Backward compatibility rules:
* Deprecation policy:
* Rollout approach (feature flag, canary):

### 2.3 SLA/SLO

Define:

* SLO: availability, latency, error rate
* Error budget policy (simple):
* What is NOT covered by SLO (e.g., downstream outage):

---

## 3) Dependencies & Boundaries

### 3.1 Downstream dependencies

For each dependency, include:

| Dependency   | Purpose | Timeout | Retry policy   | Circuit breaker | Cache        | Notes        |
| ------------ | ------- | ------: | -------------- | --------------- | ------------ | ------------ |
| UserService  | profile |    80ms | 1 retry jitter | yes             | 30s          | critical     |
| OrderService | stats   |   120ms | 1 retry jitter | yes             | 5s           | critical-ish |
| RecsService  | recs    |   150ms | 0 retry        | yes             | 10s stale ok | non-critical |

### 3.2 Data stores (if any)

* DB type:
* Tables/collections:
* Migrations strategy (expand/contract):

### 3.3 Trust boundaries

* Tenant isolation:
* AuthZ model:
* Sensitive fields:
* Logging redaction rules:

---

## 4) Core Flow (Request Lifecycle)

### 4.1 Happy path (step-by-step)

Example for BFF:

1. Receive request → validate auth/tenant
2. Create `request_id`, set deadline (T_total = 250ms)
3. Fan-out to downstream with per-call budgets
4. Aggregate + shape response per client
5. Cache behavior (L1/L2 + SWR) decision
6. Return response + metrics/logs/traces

### 4.2 Time budget

Provide a hard budget table:

| Step                        | Budget | Notes            |
| --------------------------- | ------ | ---------------- |
| Auth + validation           | 10ms   | local            |
| UserService call            | 80ms   | must succeed     |
| OrderService call           | 120ms  | degrade allowed? |
| RecsService call            | 150ms  | optional         |
| Aggregation + serialization | 20ms   | local            |
| Total                       | 250ms  | p95 target       |

### 4.3 Degradation policy

Define what happens when:

* Recs timeout:
* Orders timeout:
* User fails:
* Cache unavailable:

---

## 5) Failure Model (đây là trái tim)

### 5.1 Failure modes list

You must include at least 12 failure modes, examples:

| Failure mode        | Symptom           | Detection                 | Immediate mitigation   | Long-term fix                |
| ------------------- | ----------------- | ------------------------- | ---------------------- | ---------------------------- |
| RecsService timeout | recs empty, p95 ↑ | downstream timeout metric | serve stale/empty recs | reduce payload, tune timeout |
| Cache stampede      | DB QPS spike      | cache_miss↑ + inFlight↑   | enable singleflight    | admission policy             |
| Tenant leak bug     | data exposure     | audit logs / tests        | kill switch + incident | keying + contract tests      |
| Clock skew          | TTL weirdness     | cache anomalies           | rely on monotonic?     | NTP + TTL guard              |

### 5.2 Retry & timeout rules

* Global deadline propagation:
* Per dependency timeouts:
* Retry budget:
* Jitter policy:
* What is NEVER retried:

### 5.3 Backpressure & load shedding

* Max in-flight per route:
* When to return 429/503:
* Priority rules:

### 5.4 Data correctness under failure

* Stale data tolerance window:
* What fields can be stale / must be fresh:
* How to signal partial results to clients:

---

## 6) Observability (Logs, Metrics, Traces)

### 6.1 Logging

* Log format: JSON
* Required fields: request_id, tenant_id, user_id (if allowed), route, latency_ms, status, error_code
* Redaction: list fields
* Sampling rules: tail/error-based

### 6.2 Metrics (Golden signals + custom)

Must include:

* traffic: rps
* errors: 5xx rate, timeouts
* latency: p50/p95/p99
* saturation: event loop lag / cpu proxy / queue depth
  Custom:
* cache hit rates (L1/L2)
* degraded response rate
* downstream per-service latency & error

### 6.3 Tracing

* trace propagation rules
* span naming conventions
* what to attach as attributes

### 6.4 Dashboards

List 3 dashboards:

1. Service overview
2. Dependency health
3. Cache & degradation

---

## 7) Ops Runbook (triage like a pro)

### 7.1 Common alerts (with thresholds)

Provide at least 8 alerts:

* High 5xx rate > 0.5% for 5m
* p95 latency > 250ms for 10m
* Downstream timeouts spike
* Cache miss > X
* Degraded responses > Y
* Event loop lag > Z
* Memory RSS > budget
* Restart loop / crash rate

Each alert must include:

* meaning
* likely causes
* first 3 checks

### 7.2 Incident playbooks (step-by-step)

Provide at least 3:

**Playbook A: p95 latency spike**

1. Check RPS change
2. Check downstream latency breakdown
3. Check cache hit rate
4. If Recs is culprit → enable `recs_kill_switch`
5. If DB QPS spike → enable stricter load shedding
6. Verify recovery within 10m

**Playbook B: 5xx spike**
**Playbook C: Tenant isolation suspected**

### 7.3 Rollback / mitigation levers

List every lever:

* Feature flags (keys + default)
* Circuit breaker overrides
* Rate limit knobs
* Cache TTL knobs
* Canary step rollback procedure

### 7.4 Operational tasks

* How to rotate secrets
* How to reindex/rebuild caches
* How to run migrations safely
* How to do safe deploy (link to Kata 99 playbook)

---

## 8) Change Safety (migrations, schema, rollouts)

### 8.1 Schema evolution

* DB expand/contract steps (if DB)
* Event schema versioning rules (if events)

### 8.2 Backward/forward compatibility

* What must remain compatible
* Contract tests enforced where

### 8.3 Release strategy

* Canary steps + gates
* Rollback criteria

---

## 9) Security & Privacy

* AuthN method:
* AuthZ model:
* Tenant boundary enforcement
* SSRF/DoS controls (body limits, timeouts)
* PII handling + redaction
* Audit logging minimal set

---

## 10) Appendices

### A) Glossary

### B) Error codes

### C) Example logs

### D) Example incident timeline (mini postmortem)

---

# Kata Tasks (Checklist)

Bạn làm theo checklist này:

1. [ ] Chọn service (khuyến nghị BFF Home)
2. [ ] Điền sections 0–4 (Intent + Contract + Core Flow)
3. [ ] Viết Failure Model >= 12 failure modes (table)
4. [ ] Viết Observability: metrics/logs/traces + dashboards
5. [ ] Viết Ops Runbook: alerts + 3 playbooks + rollback levers
6. [ ] Viết Change Safety: schema/versioning/release
7. [ ] Security & privacy: tenant + PII + redaction
8. [ ] Review: một người khác đọc thử (tưởng tượng) và bạn tự hỏi: “Nếu tôi ngủ, họ có vận hành được không?”

---

# Definition of Done (pass kata)

Bạn pass nếu:

* Tài liệu **đủ cụ thể**: có số (timeouts, thresholds, SLO), có bảng, có steps.
* Failure model ≥ 12 và mỗi cái có detection + mitigation.
* Ops runbook có ít nhất 8 alerts + 3 playbooks.
* Có rollback levers (flag keys) và criteria rõ.
* Không có section nào chỉ toàn buzzword.

---

# Stretch (Senior+ → Staff/Principal)

Chọn ≥ 6:

1. **Threat model mini** (STRIDE-lite): 6 threats + mitigations.
2. **Capacity model**: rough sizing (RPS, CPU per request, cache size).
3. **Cost guardrails**: egress, downstream call count, cache hit target.
4. **GameDay plan**: 3 chaos experiments + expected outcomes.
5. **Kill-switch matrix**: mapping alert → lever.
6. **Runbook automation hooks**: scripts/commands placeholders.
7. **Spec-to-tests mapping**: mỗi failure mode có test/monitor tương ứng.