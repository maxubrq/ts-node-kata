# 100 Typescript & NodeJS Kata

## Cách dùng bộ Kata này

**Mỗi kata** làm theo format cố định:

* **Goal**: năng lực chính cần rèn
* **Constraints**: ép bạn chọn trade-off đúng (như production)
* **Deliverable**: code + test + doc ngắn
* **Checks**: tiêu chí pass/fail rõ ràng
* **Stretch**: nâng độ khó (optional)

**Quy tắc vàng:** kata nào cũng có **test**, và đa số có **benchmark / failure-injection / logging**.

---

## Bản đồ 100 Kata (10 tầng × 10 kata)

### TẦNG 1 — TypeScript Core: typing để giảm bug thật (01–10)

1. **Type-level Guardrails**: tạo type buộc object có đủ field theo rule (không dùng `any`).
2. **Discriminated Union FSM**: mô hình hóa state machine + exhaustive check.
3. **Branded Types for IDs**: `UserId`, `OrderId` không thể lẫn nhau.
4. **Result/Either Pattern**: không throw, return `Result<T,E>` + helper.
5. **Parse Don’t Validate**: input unknown → parse thành domain type.
6. **Generic Repository Contract**: interface repo chuẩn, không leak persistence.
7. **Conditional Types for API**: type output thay đổi theo input.
8. **Readonly & Immutability**: enforce immutability ở boundary.
9. **Typed Event Map**: event name → payload type, emitter strongly typed.
10. **TS Error Budget**: refactor một module bỏ `as`/`any` xuống 0.

### TẦNG 2 — Runtime Truth: Node không giống TS (11–20)

11. **Error Boundaries**: phân loại lỗi (operational vs programmer).
12. **Async Stack Trace**: giữ context qua async chain (AsyncLocalStorage).
13. **Timeouts Everywhere**: wrapper promise có timeout + abort.
14. **AbortController End-to-End**: cancel request từ HTTP đến DB call.
15. **Backpressure Basics**: stream pipeline đúng (không OOM).
16. **Event Loop Lag Monitor**: đo lag + alert threshold.
17. **Graceful Shutdown**: drain in-flight + stop accept new.
18. **Unhandled Rejection Policy**: xử lý & quyết định crash hay not.
19. **Memory Leak Hunt**: tạo leak nhỏ, dùng heap snapshot tìm thủ phạm.
20. **Worker Threads Case**: offload CPU-bound, đo throughput.

### TẦNG 3 — HTTP API Production: đúng chuẩn, không “toy REST” (21–30)

21. **Idempotency Key**: POST thanh toán idempotent.
22. **Pagination Correctness**: cursor-based, không offset-based.
23. **ETag & Caching**: implement conditional GET.
24. **Rate Limiting**: token bucket (in-memory) + test.
25. **Request Validation**: schema + typed request body.
26. **Problem+JSON Errors**: chuẩn hóa error response.
27. **Structured Logging**: log JSON có request_id, user_id.
28. **OpenAPI from Types**: generate spec, tránh drift.
29. **API Versioning Strategy**: v1/v2 + deprecation plan.
30. **Multi-tenant Guards**: tenant isolation bằng middleware.

### TẦNG 4 — Data & Transactions: consistency là máu (31–40)

31. **Transaction Wrapper**: unit of work với rollback.
32. **Optimistic Locking**: version field + retry policy.
33. **Outbox Pattern**: write DB + event atomic.
34. **Exactly-once Illusion**: prove “at-least-once + idempotency”.
35. **Read Model Projection**: build projection từ event stream.
36. **N+1 Query Killer**: batch load + dataloader pattern.
37. **Schema Migration Safe**: expand/contract migration plan.
38. **Connection Pool Tuning**: simulate load + tune pool.
39. **Data Validation in DB**: constraints vs app validation trade-off.
40. **Hot Partition Fix**: thiết kế shard key / bucketing.

### TẦNG 5 — Concurrency & Backpressure: chịu tải thật (41–50)

41. **Max In-flight**: limiter per downstream.
42. **Queue Worker**: concurrency=N, retry, DLQ simulation.
43. **Circuit Breaker**: open/half-open/close + metrics.
44. **Bulkhead Isolation**: tách resource per route.
45. **Priority Queue**: SLA-aware processing.
46. **Load Shedding**: drop/deny chiến lược khi quá tải.
47. **Adaptive Concurrency**: tăng/giảm concurrency theo latency.
48. **Streaming Upload**: upload file không buffer toàn bộ.
49. **Fan-out Control**: parallel calls có cap + cancel.
50. **Thundering Herd**: cache stampede protection.

### TẦNG 6 — Observability: nhìn thấy hệ thống (51–60)

51. **Golden Signals**: instrument latency/traffic/errors/saturation.
52. **Tracing with Context**: trace id xuyên service boundary.
53. **Log Sampling Strategy**: sampling theo error + latency tail.
54. **Custom Metrics**: histogram p50/p95/p99.
55. **Health vs Readiness**: phân biệt đúng, test kịch bản.
56. **Alert Design**: alert ít nhưng đúng (SLO-ish).
57. **Debug Mode Safe**: bật debug không lộ secrets.
58. **Correlation IDs**: propagate qua message queue.
59. **Incident Repro Harness**: script tái hiện sự cố.
60. **Postmortem Template**: write ngắn nhưng sắc (cause, fix, follow-up).

### TẦNG 7 — Security & Safety: đừng để production “đốt” (61–70)

61. **Secrets Handling**: config load, rotate, no logging.
62. **AuthN/AuthZ**: RBAC/ABAC nhỏ nhưng chuẩn.
63. **JWT Pitfalls**: validate issuer/audience, clock skew.
64. **CSRF/CORS Correct**: implement rule set, test.
65. **Input Sanitization**: chống injection ở nơi cần.
66. **Dependency Risk Drill**: audit + lockfile strategy.
67. **DOS by JSON**: limit body, depth, arrays.
68. **SSRF Prevention**: allowlist host, block internal ranges.
69. **File Upload Security**: mime sniff, size limit, store.
70. **Security Logging**: audit trail minimal viable.

### TẦNG 8 — Architecture & Modularity: scale codebase (71–80)

71. **Hexagonal Boundary**: tách domain/application/infra.
72. **Dependency Inversion in TS**: composition root.
73. **Feature Flagging**: runtime flag + kill switch.
74. **Config System**: layered config + schema + reload.
75. **Monolith Modularization**: module boundaries & rules.
76. **Event-driven Module**: internal event bus typed.
77. **Anti-corruption Layer**: adapter cho external API.
78. **Library Design**: publish internal package với semantic version.
79. **Refactor with Contract Tests**: đổi impl không đổi behavior.
80. **Migration: Mono → Services**: carve-out plan cho 1 module.

### TẦNG 9 — Distributed Reality: consistency, retries, chaos (81–90)

81. **Retry Budget**: retry có giới hạn & jitter.
82. **Idempotent Consumer**: consume event at-least-once.
83. **Saga Orchestration**: workflow có compensate.
84. **Timeout & Deadline Propagation**: request deadline xuyên service.
85. **Partial Failure Handling**: degrade gracefully.
86. **Multi-region Thought Drill**: latency + consistency plan.
87. **Clock Skew Bugs**: reproduce & fix.
88. **Poison Message**: detect, quarantine, DLQ tools.
89. **Schema Evolution in Events**: backward/forward compatibility.
90. **Chaos Mini Drill**: inject latency/errors và chứng minh hệ sống.

### TẦNG 10 — Performance & Craft: nhanh, sạch, bền (91–100)

91. **Benchmark Harness**: microbench đúng cách.
92. **Profiling CPU**: tìm hot path, tối ưu có chứng cứ.
93. **Memory Optimization**: giảm alloc, đo heap.
94. **Cold Start**: tối ưu boot time.
95. **Caching Strategy**: what to cache & invalidation plan.
96. **BFF vs API**: trade-off drill với cùng bài toán.
97. **Testing Pyramid in Node**: unit/integration/contract/e2e.
98. **Flaky Test Killer**: tạo flaky, fix bằng deterministic.
99. **Release Playbook**: canary, rollback, feature flag.
100. **Production Spec Kata**: viết “System Intent + Failure Model + Ops Runbook” cho một service.

---

## Lộ trình luyện

* **Tuần 1–2:** Tầng 1–2 (20 kata) → nền TS + runtime.
* **Tuần 3–4:** Tầng 3–4 (20 kata) → API + data đúng chuẩn.
* **Tuần 5–6:** Tầng 5–6 (20 kata) → chịu tải + quan sát.
* **Tuần 7–8:** Tầng 7–8 (20 kata) → security + kiến trúc codebase.
* **Tuần 9–10:** Tầng 9–10 (20 kata) → distributed + performance + release.

Mỗi ngày 1 kata (60–120 phút). Nếu bận: 3 kata/tuần nhưng **không bỏ test**.

---

## “Bộ khung repo” để bạn làm 100 kata kiểu production

* `packages/kata-XX/` (mỗi kata là 1 package)
* Mỗi kata có:

  * `README.md` (Goal/Constraints/Checks)
  * `src/`
  * `test/`
  * `bench/` (nếu cần)
  * `docker-compose.yml` (nếu có DB/queue)
* Tooling đề xuất: `vitest`, `tsx`, `eslint`, `tsup`, `pino`, `zod` (tuỳ kata).

---

## Bonus: Cách biến kata thành điểm mạnh

Sau mỗi tầng, bạn viết 1 trang:

* **What I used to do**
* **What I do now**
* **Trade-offs I accept**
* **Production failure I can now prevent**

Đó là thứ biến 100 kata thành “bằng chứng năng lực”, không phải bài tập.

## Kata chi tiết

1. [#001 - Type-level Guardrails](./kata/kata001.md)
2. [#002 - Discriminated Union FSM](./kata/kata002.md)
3. [#003 - Branded Types for IDs](./kata/kata003.md)
4. [#004 - Result/Either Pattern](./kata/kata004.md)
    - [#004.1 - Bonus - ResultAsync (Promise<Result<…>>) + Timeout](./kata/kata004.1.md)
    - [#004.2 - Bonus - ValidationResult](./kata/kata004.2.md)
5. [#005 - Parse Don’t Validate](./kata/kata005.md)
    - [#005.1 - Bonus - Problem+JSON Adapter](./kata/kata005.1.md)
6. [#006 - Generic Repository Contract](./kata/kata006.md)
    - [#006.1 - Bonus - Versioned Update API](./kata/kata006.1.md)
7. [#007 - Conditional Types for API](./kata/kata007.md)
8. [#008 - Readonly & Immutability](./kata/kata008.md)
    - [#008.1 - Bonus - Immutable Context (AsyncLocalStorage)](./kata/kata008.1.md)
9. [#009 - Typed Event Map](./kata/kata009.md)
10. [#010 - TS Error Budget](./kata/kata010.md)
11. [#011 - Error Boundaries (Operational vs Programmer)](./kata/kata011.md)
12. [#012 - Async Context Trace (AsyncLocalStorage)](./kata/kata012.md)
13. [#013 - Timeouts Everywhere (timeout + abort + deadline)](./kata/kata013.md)
14. [#014 - AbortController End-to-End](./kata/kata014.md)
15. [#015 - Backpressure Basics (Stream pipeline đúng, không OOM)](./kata/kata015.md)
16. [#016 - Kata 16 — Event Loop Lag Monitor](./kata/kata016.md)
17. [#017 - Graceful Shutdown](./kata/kata017.md)
18. [#018 - Unhandled Rejection Policy](./kata/kata018.md)
19. [#019 - Memory Leak Hunt](./kata/kata019.md)
20. [#020 - Worker Threads Case](./kata/kata020.md)
21. [#021 - Idempotency Key](./kata/kata021.md)
22. [#022 - Pagination Correctness](./kata/kata022.md)
23. [#023 - ETag & Caching](./kata/kata023.md)
24. [#024 - Rate Limiting (Token Bucket, in-memory) + Tests](./kata/kata024.md)
25. [#025 - Request Validation: schema → typed request body](./kata/kata025.md)
26. [#026 - Problem+JSON: chuẩn hoá error response](./kata/kata026.md)
27. [#027 - Structured Logging: JSON logs với request_id + user_id](./kata/kata027.md)
28. [#028 - OpenAPI from Types: generate spec, avoid drift](./kata/kata028.md)
29. [#029 - API Versioning Strategy: v1/v2 + deprecation plan](./kata/kata029.md)
30. [#030 - Multi-tenant Guards: Tenant isolation bằng middleware](./kata/kata030.md)
31. [#031 - Transaction Wrapper: Unit of Work với rollback](./kata/kata031.md)
32. [#032 - Optimistic Locking: version field + retry policy](./kata/kata032.md)
33. [#033 - Outbox Pattern: DB write + event atomic](./kata/kata033.md)
34. [#034 - Exactly-once Illusion: at-least-once + idempotency](./kata/kata034.md)
35. [#035 - Read Model Projection: build projection từ event stream](./kata/kata035.md)
36. [#036 - N+1 Query Killer: batch load + DataLoader pattern](./kata/kata036.md)
37. [#037 - Schema Migration Safe: Expand/Contract migration plan](./kata/kata037.md)
38. [#038 - Connection Pool Tuning: simulate load + tune pool](./kata/kata038.md)
39. [#039 - Data Validation in DB: Constraints vs App validation](./kata/kata039.md)
40. [#40 - Hot Partition Fix: Shard key / Bucketing](./kata/kata040.md)
41. [#041 - Max In-flight per Downstream](./kata/kata041.md)