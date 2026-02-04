# Kata 19 — Memory Leak Hunt

**Tạo leak nhỏ → đo RSS/heap → chụp heap snapshot → tìm thủ phạm → fix + verify**

Mục tiêu: bạn *thực sự* biết cách debug leak trong Node, không phải “đoán mò”.

---

## Goal

1. Cố ý tạo **1 memory leak nhỏ nhưng thật** (retained objects không được GC)
2. Dùng **heap snapshot** để tìm “retainer path” (ai đang giữ reference)
3. Fix leak và chứng minh:

* heapUsed ổn định / giảm
* retained object count giảm
* không còn growth theo thời gian

---

## Constraints

* ❌ Không dùng “global cache vô hạn” kiểu vô tình rồi bảo “đó là leak”
* ✅ Leak phải đến từ 1 trong các pattern production hay gặp:

  1. **EventEmitter listener leak**
  2. **Map cache không evict**
  3. **Timer/interval giữ closure**
  4. **AsyncLocalStorage store giữ object lớn**
  5. **Promise chain giữ reference**

Bạn chọn **1** (kata này đưa sẵn 2 scenario; bạn làm ít nhất 1).

---

## Deliverables

Folder `kata19/` gồm:

* `leak.ts` (leaky version)
* `fixed.ts` (fixed version)
* `README.md` (commands + “findings”: bạn đã thấy gì trong snapshot)
* 2 heap snapshots: `before.heapsnapshot`, `after.heapsnapshot` (hoặc `leak-*.heapsnapshot`)

---

# Part A — Leak Scenario 1: EventEmitter Listener Leak (classic)

### Setup

Bạn viết `leak.ts` mô phỏng service tạo request handlers liên tục, nhưng **không unsubscribe** listener.

#### Spec

* Có `bus` (EventEmitter hoặc typed emitter từ Kata 09)
* Mỗi “request” register listener vào `"tick"` hoặc `"user.created"`
* Listener closure capture object lớn (vd 100KB string) để leak rõ
* Không bao giờ `off()` / unsubscribe
* Loop 5–30s, log memory mỗi 1s

### Expected symptoms

* `process.memoryUsage().heapUsed` tăng đều
* Có thể thấy warning `MaxListenersExceededWarning` (nếu dùng Node EventEmitter)
* Heap snapshot: **Retainers** chỉ ra `EventEmitter._events` → listener functions → closure → big string/buffer

### Fix

* Unsubscribe on request end (finally)
* Hoặc dùng `.once`
* Hoặc đặt max listeners + use shared handler (không per request)

---

# Part B — Leak Scenario 2: Map Cache Leak (infinite keyspace)

### Setup

Bạn viết module cache `Map<string, BigObject>` nhưng không evict.

#### Spec

* Mỗi “request” tạo key mới (`req_${i}`)
* Cache store object lớn 50–200KB
* Không TTL / LRU
* Chạy 10–30s

### Expected symptoms

* HeapUsed tăng tuyến tính theo số requests
* Heap snapshot: retainer path từ `Map` (global) giữ toàn bộ values

### Fix

* Implement TTL eviction (periodic sweep)
* Hoặc LRU with size cap
* Hoặc avoid caching per-request keyspace

---

# Tooling — Heap Snapshot Workflow (DevTools)

## Option 1: Node `--inspect` + Chrome DevTools (recommended)

1. Run:

```bash
node --inspect leak.js
```

(hoặc `ts-node`/`tsx` tuỳ bạn, nhưng để đơn giản compile TS → JS)

2. Mở Chrome: `chrome://inspect` → “Open dedicated DevTools for Node”

3. Tab **Memory**:

* Take snapshot at start → `S1`
* Đợi heap tăng ~10–20MB → Take snapshot → `S2`
* Compare `S2` vs `S1`:

  * Sort by **Retained Size**
  * Mở object type bạn nghi ngờ (String, Array, (closure), Map, Listener)
  * Xem **Retainers** để thấy đường giữ reference

## Option 2: Programmatic heap snapshot (không cần Chrome UI)

Bạn có thể dùng Node built-in:

* `v8.writeHeapSnapshot()` (module `node:v8`)
  Chạy ở time points và lưu `.heapsnapshot`, sau đó mở bằng DevTools Memory.

---

# Template code bạn nên tự viết (không copy/paste y nguyên)

### Memory logger (dùng cho cả leak/fix)

* log mỗi 1s:

  * `rss`, `heapUsed`, `heapTotal`, `external`
* optional: `global.gc()` nếu bạn chạy với `--expose-gc` để “ép GC” trước khi đo (giúp phân biệt “pressure” vs “retained”)

**Rule quan trọng:** Leak thật sẽ vẫn tăng sau GC.

---

# What to write in README.md (bắt buộc)

1. Bạn chạy command gì để tạo leak
2. Sau bao lâu heap tăng bao nhiêu (kèm số)
3. Trong heap snapshot bạn thấy:

* Top 3 constructor/type theo retained size
* Retainer path của object bị leak (ghi bằng text)

4. Fix bạn làm là gì
5. Chạy lại fixed: heap ổn định thế nào (số)

---

## Definition of Done

Bạn pass kata nếu:

* Leak version: heapUsed tăng rõ ràng theo thời gian (sau khi GC vẫn tăng)
* Snapshot chỉ ra được **retainer path** cụ thể (ai giữ reference)
* Fixed version: heapUsed ổn định (hoặc sawtooth nhưng không trend lên)
* README mô tả “thủ phạm” và “vì sao” đúng

---

## Stretch (Senior+/Production)

1. Thêm **AsyncLocalStorage leak**: store giữ request body lớn, quên clear / nested run giữ reference
2. Dùng **Allocation instrumentation on timeline** để thấy chỗ allocate nhiều
3. “Leak via Promise”: store unresolved promises in Map, never settled
4. Thêm “alert”: nếu heapUsed slope > threshold trong 60s → emit warning (nối Kata 16)