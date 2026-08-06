# Fair-per-user render scheduler — Hybrid Redis counter + DB status

**Ngày:** 2026-08-06
**Category:** architecture
**Tags:** queue, fairness, redis, bullmq, scheduling, backpressure, race-condition
**Stack:** NestJS, BullMQ, Redis, Postgres (Prisma), Node.js worker replicas

---

## Dự án

api-mkt-post-video — service render + đăng video hàng loạt (news + video kiến thức) cho nhiều user. Worker BullMQ chạy nhiều replica, mỗi replica pull job từ Redis queue.

## Context

Sau khi thêm `POST /posts/batch` (tối đa 20 video/lần), user A submit batch 20 video **ngay trước** user B submit 20 video → B phải đợi hết 20 video của A (~40 phút render) mới thấy video đầu tiên của mình được xử lý.

Vấn đề gốc: **BullMQ FIFO queue** — job đẩy vào theo thứ tự, worker pop theo thứ tự. Không phân biệt user. Batch chỉ khuếch đại bug đã có sẵn (user cũ dồn 20 click liên tiếp cũng gây tình trạng tương tự).

## Vấn đề: Queue monopoly

Khi 1 user chiếm hết N slot trong queue, mọi user khác bị starve tới khi user đầu render xong hết. Với queue 20K job và 5 render slot → user chờ có thể tính bằng **giờ**.

Business impact:
- User mới paste 1 video → phải chờ user cũ đang batch 20 → churn ngay lượt dùng đầu tiên.
- Không có priority/tier logic → free user và paid user chung 1 queue.

## Yêu cầu ràng buộc

- User A batch KHÔNG chặn hoàn toàn user B (fairness per user).
- Không thêm worker instance (không tăng tiền hạ tầng ngay).
- Không đổi worker code (rủi ro cao).
- Không phụ thuộc BullMQ Pro ($395/năm — có feature `group` cho fair scheduling native nhưng phải trả phí).
- Rollback dễ (bật/tắt bằng env flag).
- Scale được — queue phình 20K+ job vẫn nhanh.
- Fit codebase: Postgres source of truth, Redis đã có, scheduler app sẵn.

## Giải pháp: Hybrid

**Nguyên tắc**: **Redis** dùng cho **cap check O(1)** (atomic INCR/DECR qua Lua script). **DB** dùng cho **hàng chờ durable có thứ tự** (`ORDER BY created_at`).

### Kiến trúc 4 thành phần

```
1. Cap check per user     → Redis counter (Lua atomic)
2. Buffer durable         → DB status PENDING_DISPATCH
3. Trigger dispatch       → Event render.succeeded/failed
4. Safety net drift       → Reconciler cron 5 phút
```

### Flow tổng

```
User bấm tạo video / batch
        │
        ▼
   API CreatePostHandler
        │
        ▼
   INSERT VideoPost (Postgres)
        │
        ▼
   Lua atomic: INCR user:render:count:{userId}
        │
     ┌──┴──────────────────┐
     │                     │
   ≤ K                   > K
     │                     │
     ▼                     ▼
  Enqueue BullMQ       DECR rollback
  status=RENDERING     status=PENDING_DISPATCH
                       (KHÔNG enqueue)
        ↓                     ↓
        └──────┬──────────────┘
               ▼
       BullMQ worker render
               │
               ▼
       Emit event posts.render.succeeded/failed
               │
               ▼
       Dispatcher listener (scheduler process):
         1. DECR counter
         2. findOldestPendingDispatchForUser (DB)
         3. Nếu có + slot còn:
              tryTransitionStatus(PENDING_DISPATCH → RENDERING)
              INCR counter
              queue.add(payload)
```

**Mô tả từng bước:**

1. **User submit** — bấm tạo 1 video hoặc batch (tối đa 20). Request đi vào `API CreatePostHandler`.

2. **Insert DB** — mỗi post được lưu 1 row `VideoPost` trong Postgres với status ban đầu. Đây là **source of truth durable** — nếu Redis rớt sau đó thì DB vẫn giữ được data.

3. **Cap check qua Lua atomic** — với mỗi post, gọi Lua script `INCR user:render:count:{userId}` trong Redis. Lua đảm bảo `INCR + check + rollback` chạy trong 1 lệnh atomic, không có race giữa nhiều API replica.

4. **Rẽ nhánh theo cap:**
   - **`≤ K`** (còn slot): giữ counter đã INCR, **enqueue vào BullMQ** ngay, set status = `RENDERING`. Job này sẽ được worker pick up bình thường.
   - **`> K`** (đã hết slot): Lua **DECR rollback** counter về giá trị trước đó, **KHÔNG enqueue**, set status = `PENDING_DISPATCH`. Post nằm chờ trong DB.

5. **Worker render** — BullMQ worker pull job từ queue, render video, upload output. Khi xong (hoặc fail) → phát event `posts.render.succeeded` / `posts.render.failed` **qua outbox pattern** (insert row `outbox_events` cùng transaction với update status → OutboxRelay đọc và phát cross-process).

6. **Dispatcher listener nhận event** (chạy trong scheduler process):
   - **DECR counter** — trả slot cho user vừa render xong.
   - **Query DB** tìm post `PENDING_DISPATCH` cũ nhất của user đó (`ORDER BY created_at`).
   - **Nếu có + slot còn:** atomic transition `PENDING_DISPATCH → RENDERING` (Postgres `updateMany WHERE status='PENDING_DISPATCH'` — chỉ 1 replica thắng nếu multi-replica), INCR counter, `queue.add()` để worker pick tiếp.

7. **Cascade** — cứ mỗi job xong lại kéo 1 job PENDING_DISPATCH của cùng user vào BullMQ. Vì cap K áp per user, nhiều user cùng có job xen kẽ trong queue — worker pull FIFO tự nhiên sẽ **fair round-robin** giữa các user.

**Điểm chốt:** BullMQ vẫn FIFO như cũ (không đổi worker code). Fairness đến từ việc **giới hạn số job/user tại 1 thời điểm** trong queue — user A không thể chiếm hết slot khi user B đang chờ.

## Nguyên lý cốt lõi

**Cap per user + buffer durable + counter atomic + trigger event-driven.**

- Cap `K` (đề xuất **K=2**) = số job tối đa/user tại 1 thời điểm trong BullMQ.
- Redis counter `user:render:count:{userId}` = số hiện tại. Check O(1) bằng `INCR`/`DECR` atomic (Lua script).
- DB status `PENDING_DISPATCH` = buffer durable, có thứ tự theo `created_at`.
- Trigger = domain event có sẵn (`posts.render.succeeded/failed`) — outbox pattern để cross-process (worker → scheduler).

Vì mỗi user chỉ có ≤ K job trong BullMQ tại 1 thời điểm, FIFO tự nhiên xen kẽ giữa các user.

## Vì sao Hybrid không phải chỉ DB hay chỉ Redis

| | DB only | Redis only | **Hybrid** |
|---|---|---|---|
| Cap check O(1) | ❌ O(N) getJobs | ✅ INCR | ✅ INCR |
| Hàng chờ có thứ tự | ✅ ORDER BY created_at | ❌ chỉ reject/delay | ✅ DB PENDING_DISPATCH |
| Durable khi Redis rớt | ✅ | ❌ mất counter | ✅ DB survives |
| Đơn giản | ✅ | ✅ | Trung bình (2 nguồn state) |
| Scale > 5K job | ❌ chậm | ✅ | ✅ |

Đánh đổi Hybrid: có 2 nguồn state → cần reconciler đồng bộ khi lệch.

## Giải pháp thay thế: BullMQ Pro Groups

**Cách làm:** dùng feature `group` của BullMQ Pro, gắn `groupId = userId` khi enqueue. Worker tự round-robin xen kẽ giữa các group → user không phải đợi user khác.

```typescript
await queue.add(payload, { group: { id: userId } });
```

**Ưu điểm:**
- **Cực đơn giản** — chỉ thêm 1 field khi enqueue. Không cần counter, không cần dispatcher, không cần reconciler, không cần outbox event cho slot management.
- **Fair scheduling tự động** ở tầng queue engine, không phải logic app.
- **Migration path rõ ràng** khi sau này chuyển sang: xoá dispatcher + cap logic, thay `queue.add(payload)` bằng `queue.add(payload, { group: { id: userId } })`.

**Nhược điểm:**
- **Mất phí $395/năm** cho BullMQ Pro license. Không phù hợp giai đoạn đầu / dự án nhỏ.

**Khi nào nên chọn:** khi ngân sách cho phép, muốn giảm complexity operational (bớt 1 nguồn state, bớt reconciler, bớt outbox liên quan slot). Không phải giai đoạn hiện tại.

## Chi tiết công nghệ

### Lua script atomic (Redis)

Để tránh race giữa 2 API replica cùng lúc INCR-check-rollback:

```lua
-- tryReserveUserSlot(userId, k):
--   INCR counter, nếu ≤ K return 1 (OK), nếu > K rollback DECR return 0
local n = redis.call('INCR', KEYS[1])
redis.call('EXPIRE', KEYS[1], 86400)
if n <= tonumber(ARGV[1]) then
  return 1
end
redis.call('DECR', KEYS[1])
return 0
```

```lua
-- releaseUserSlot(userId): DECR (clamp về 0 nếu underflow — bug indicator)
local n = redis.call('DECR', KEYS[1])
if n < 0 then
  redis.call('SET', KEYS[1], 0)
  return 0
end
return n
```

**Vì sao Lua**: Redis chạy Lua script atomic — INCR-check-rollback nằm trong 1 lệnh, không thể có 2 request cùng đọc `n=2`, cùng nghĩ "ok slot còn", cùng push → BullMQ 3 job nhưng counter chỉ ghi 2.

### DB status thêm 1 giá trị

`POST_STATUS` const object (không phải Prisma enum) → thêm `PENDING_DISPATCH` không cần migration. Prisma chấp nhận vì cột `status` là `String`.

### Event outbox

Worker render xong → insert `outbox_events` row trong CÙNG transaction với update `videoPost.status='POSTED'` → OutboxRelay task đọc từ outbox, phát event qua EventEmitter cross-process → scheduler listener nhận.

Vì sao outbox thay vì phát trực tiếp: worker crash sau khi UPDATE DB nhưng trước khi phát event → nếu phát trực tiếp thì event mất, counter kẹt vĩnh viễn. Outbox pattern đảm bảo event durable cùng data change.

### Reconciler safety net

Cron 5 phút chạy trong scheduler app:

```typescript
const actualPerUser = await repo.countActiveRenderPerUser();  // DB truth
for (const [userId, actual] of actualPerUser) {
  const stored = await slotService.getCount(userId);           // Redis
  if (stored !== actual) {
    await slotService.setCount(userId, actual);                // sync về đúng
  }
}
for (const userId of pendingUsers) {
  if (getCount(userId) < cap) {
    await dispatcher.dispatchForUser(userId);                  // force push
  }
}
```

Cứu 3 tình huống chính:
1. Event mất (outbox delay, worker crash) → counter cao hơn thực tế → PENDING_DISPATCH kẹt.
2. Enqueue fail sau reserve → counter dư.
3. DB tampering / cancel/delete không release slot → drift.

## Ví dụ trace cụ thể

**Setup:** K=2, `RENDER_MAX_CONCURRENCY=1`.

**t=0** — user A batch 20:
- A#1: `INCR count:A = 1 ≤ 2` → push BullMQ. Status = RENDERING.
- A#2: `INCR count:A = 2 ≤ 2` → push BullMQ.
- A#3: `INCR count:A = 3 > 2` → `DECR = 2` → status = PENDING_DISPATCH.
- A#4..A#20: cùng vậy → PENDING_DISPATCH.

BullMQ hiện có: `[A#1, A#2]`.

**t=5s** — user B batch 20: tương tự → BullMQ có `[A#1, A#2, B#1, B#2]`.

**t=2m** — worker render A#1 xong → outbox → event → scheduler listener:
- `DECR count:A = 1`.
- `findOldestPendingDispatch(A)` → A#3.
- `tryTransitionStatus(A#3, PENDING_DISPATCH → RENDERING)` — atomic Postgres updateMany với `WHERE status='PENDING_DISPATCH'`.
- `INCR count:A = 2`, `queue.add(A#3)`.

BullMQ: `[A#2, B#1, B#2, A#3]`. Cứ thế cascade.

**Thứ tự render cuối cùng:** `A#1 → A#2 → B#1 → B#2 → A#3 → A#4 → B#3 → B#4 → ...` — xen kẽ 2-2 fair per user.

## Q&A

### Q: Vì sao chọn K=2 chứ không K=1?

- K=1: xen kẽ 1-1 hoàn hảo nhưng có **gap latency** giữa "render xong" → outbox → event → dispatcher → BullMQ (100-500ms). Trên 100 job = 10-50s idle throughput.
- K=2: luôn có 1 job/user sẵn trong BullMQ → không gap. Xen kẽ 2-2 vẫn fair.

### Q: Vì sao Redis counter O(1) mà BullMQ `getJobs` O(N)?

- `getJobs` = LRANGE/ZRANGE + HMGET từng job hash. N job = N Redis operations (pipeline vẫn tuyến tính).
- Redis counter = 1 INCR/DECR = 1 op, không phụ thuộc N.

Với 20K job: `getJobs` ~1-2s, counter ~50µs. **Nhanh gấp 20,000-40,000 lần.**

### Q: Vì sao vẫn cần DB (không chỉ Redis)?

- Cần "hàng chờ có thứ tự" (`ORDER BY created_at`) để dispatcher biết pop cái nào tiếp.
- Redis List có thứ tự nhưng không durable như Postgres.
- Post đã có DB row (đã insert VideoPost) — reuse cột `status` rẻ hơn tạo Redis List riêng.
- User có thể query `GET /posts?status=PENDING_DISPATCH` qua API.

### Q: Hành vi khi 1 user active (không có ai khác)?

Counter chỉ tăng cho user A → luôn ≤ K → không có PENDING_DISPATCH. Render tuần tự A#1 → A#2 → ... y hệt hiện tại. **Cap chỉ có ý nghĩa khi ≥ 2 user active.**

### Q: Multi-replica scheduler có race không?

Dispatcher listener trên nhiều replica có thể fire cho cùng 1 event (outbox at-least-once delivery). Nhưng `tryTransitionStatus` là **atomic Postgres updateMany với `WHERE status='PENDING_DISPATCH'`** → chỉ 1 replica thắng (rows updated = 1), các replica khác thấy 0 rows → return sớm. Không cần distributed lock riêng.

### Q: Tự động scale với WORKER_REPLICAS thế nào?

Cap K tự tính:
```
K = max(1, ceil(WORKER_REPLICAS × RENDER_CONCURRENCY × USER_SLOT_RATIO))
```

- `USER_SLOT_RATIO=0.5` → 1 user chiếm tối đa nửa capacity.
- Hạ ratio 0.3 khi có nhiều concurrent user, muốn fair chặt hơn.
- `USER_MAX_QUEUED_JOBS` override cứng khi cần.

## Kiến thức nền tảng

**Pattern name trong ngành:** "fair-share scheduling with per-tenant in-flight cap" hoặc "fair queue with outer scheduler + event-driven dispatch".

**Common trong industry:**
- **AWS SQS FIFO** — MessageGroupId: message cùng group xử lý tuần tự, khác group song song.
- **Sidekiq Enterprise Rate Limiting** — global concurrent limit per key.
- **Google Cloud Tasks** — rate limit per queue/handler.
- **BullMQ Pro Groups** — feature trả phí, `job.opts.group.id`. Migration sau này đơn giản: xoá dispatcher + cap logic, thay `queue.add(payload)` bằng `queue.add(payload, {group: {id: userId}})`.

**Ý tưởng chung:** tách "hàng chờ toàn cục có thứ tự" khỏi "hàng thực thi có giới hạn per-tenant". Hàng chờ durable (DB/Redis persistent), hàng thực thi ephemeral (BullMQ active queue).

## Rủi ro + phòng thủ

### Event loss / delay

**Nguyên nhân có thể:**
- OutboxRelay crash/OOM → event chờ trong bảng outbox.
- Bug relay skip event.
- Publisher throw không retry đúng.

**Hậu quả:** counter Redis không được DECR → cap giả tưởng đầy → không enqueue tiếp → PENDING_DISPATCH kẹt.

**Phòng thủ:**
- **Reconciler task** (safety net chính) — 5 phút chạy 1 lần.
- **Metric alert** `pending_dispatch_age_seconds` p99 > threshold.
- **TTL 24h** trên Redis counter — user không hoạt động → counter tự expire.

### Counter drift trong hybrid

Redis counter và BullMQ thực tế có thể lệch do:
- **Race**: KHÔNG xảy ra vì Lua atomic.
- **Event mất**: DECR không chạy → cap giả đầy.
- **Enqueue fail sau reserve**: Lua đã INCR, `queue.add` throw (Redis rớt) → phải catch + `releaseSlot`. Quên catch → counter dư.
- **Cancel/delete không release**: dev extend cancel handler quên gọi `releaseSlot` → drift.

### Cancel/Delete flow

- Cancel handler BẮT BUỘC gọi `releaseSlot(userId)` khi cancel post đang RENDERING/GENERATING/PENDING_DISPATCH.
- BullMQ job bị `queue.remove()` trực tiếp → KHÔNG có event render.failed → counter không DECR → drift.
- Mitigation: reconciler cứu trong 5 phút.

## Bug thực tế đã gặp

Sau khi implement, gặp case: **counter Redis stuck ở 2** (cap max) nhưng **không có job nào đang render trong worker**.

**Trace ra**:
1. Rebuild containers (worker) khi 2 job đang chạy → BullMQ job bị kill giữa chừng.
2. Worker không kịp phát event `posts.render.succeeded/failed`.
3. Dispatcher không nhận event → không DECR counter.
4. Post `PENDING_DISPATCH` mới đến bị chặn vì counter đầy.

**Fix tức thời:** `DEL user:render:count:{userId}` trong Redis.
**Fix dài hạn:** reconciler task đã cover — nhưng lúc đó reconciler chỉ handle scope `'news'`, không handle scope `'storyboard'` (video kiến thức mới thêm). → Extend reconciler cover cả 2 scope.

**Bài học meta**: mỗi khi thêm scope slot mới (news → storyboard → comic), phải extend đồng bộ 3 chỗ:
1. `UserRenderSlotService` — thêm scope enum.
2. Dispatcher listener — tạo instance mới cho scope đó (guard event theo `source`).
3. Reconciler — loop qua cả 2 scope, đừng default 'news'.

## Hạn chế của thiết kế

- **KHÔNG tăng throughput** — fair scheduling chỉ giải công bằng, không giải tốc độ. Muốn nhanh hơn: tăng `WORKER_REPLICAS`.
- **CHỈ cap 1 queue** (news/storyboard hiện tại) — các queue khác (comic, video-talk) không có cap. Nếu 1 user spam comic, vẫn chặn queue đó.
- **KHÔNG có priority theo tier user** — free và paid cùng K. Muốn phân tier phải extend `hasSlotForUser(userId, tier)`.
- **KHÔNG có aging trên PENDING_DISPATCH** — post kẹt vô hạn cho tới khi cancel/dispatch. Rare-case bug + reconciler down → chờ dev intervention.
- **Complexity 2 nguồn state** — debug khó: check cả DB + Redis + BullMQ actual khi user complain "video kẹt".
- **Phụ thuộc Redis operational** — Redis rớt → `tryReserveUserSlot` throw → chọn fail-open (bypass cap, mất fairness tạm) hoặc fail-closed (chặn tạo video, UX tệ) — cần chốt trước implement.

## Lesson

- **Fair scheduling không cần rewrite queue engine.** Chỉ cần 1 lớp "outer scheduler" (cap + dispatcher event-driven) bên trên FIFO có sẵn. BullMQ vẫn FIFO, worker code không đổi.
- **Redis + DB kết hợp bù nhau**: Redis O(1) cho check nóng, DB durable cho state cần recover. Đừng ép 1 nguồn làm cả 2 việc.
- **Lua script là cách atomic thực sự trong Redis** — INCR/DECR + condition + rollback trong 1 lệnh. Không thay được bằng WATCH/MULTI/EXEC vì WATCH có retry loop.
- **Outbox pattern là bắt buộc** khi có multi-process depend vào domain event. Publisher trực tiếp trong tx → mất event khi crash sau commit DB.
- **Reconciler là safety net phải có**, không phải nice-to-have. Mọi hệ có 2 nguồn state đều drift theo thời gian.
- **Metric drift là bug indicator** — `counter_drift_corrected_total > 0` liên tục → có bug event/race chưa fix.
- **Dark launch bằng env flag** (`DISPATCHER_ENABLED=false`) — deploy code, chưa bật hành vi. Bật dần theo tier user hoặc theo tuần.
- **Multi-replica dispatcher không cần lock** khi có atomic compare-and-set ở DB (`updateMany WHERE status=X`) — Postgres tự serialize, chỉ 1 replica thắng.

## Reference

- BullMQ docs: https://docs.bullmq.io/
- BullMQ Pro Groups: https://docs.bullmq.io/bullmq-pro/groups
- Weighted Fair Queueing: https://en.wikipedia.org/wiki/Weighted_fair_queueing
- Outbox pattern: https://microservices.io/patterns/data/transactional-outbox.html
- Redis Lua scripting: https://redis.io/docs/latest/commands/eval/
- Postgres SKIP LOCKED: https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE
- AWS SQS FIFO MessageGroupId: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/using-messagegroupid-property.html
