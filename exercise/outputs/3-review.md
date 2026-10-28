# Review: ADR-0002 Publish `OrderPlaced` events via RabbitMQ

อ่านเฉพาะ `docs/adr/0002-publish-order-placed-events-via-rabbitmq.md` และ `docs/scenario.md`

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| S1 Performance | ลูกค้า | กด place order | peak 11.11, **200 orders/s**, ทุกอย่างปกติ | `PlaceOrder` ทำ payment/shipping แล้ว publish พร้อมรอ publisher confirm | p95 `PlaceOrder` **≤ 500 ms** (budget เดิม ≈ 300 ms + publish ≤ 100 ms) |
| S2 Availability | infrastructure | RabbitMQ ทั้ง cluster ล่ม **และ** `checkoutservice` pod ถูก kill/OOM ระหว่างนั้น | peak 200 orders/s | order ยังสำเร็จ; event ที่ค้างถูกส่งเมื่อ broker กลับมา | order ล้ม = **0**; `OrderPlaced` หาย = 0 (หรือทุกตัวที่หายมีหลักฐานให้ replay) |
| S3 Correctness | broker | ส่ง `OrderPlaced` ซ้ำ / `loyaltyservice` restart กลางงาน | peak, คิว email backlog ≈ 150 msg/s | ให้ points ครั้งเดียวต่อ `order_id` | points ภายใน **1 นาที**, ซ้ำ = **0** |

## 2. Findings table

| Type | Finding | Why it matters |
|---|---|---|
| Risk | Retry buffer อยู่ **in-memory** ใน pod (Decision ข้อ 3) | pod ถูก SIGKILL/OOM/node ล่ม ⇒ ไม่มี SIGTERM handler ได้ทำงาน, event ใน buffer หายโดย**ไม่มี log**; ข้อความ "หายเฉพาะเมื่อ broker ล่มนาน **และ** pod ตาย" จึงกว้างกว่าที่เขียน และ loyalty "exactly once" กลายเป็น "zero times" ได้ |
| Risk | Crash ระหว่าง `paymentservice.Charge` กับ publish | ลูกค้าถูกตัดเงินแต่ไม่มี event ⇒ ไม่มีอีเมล ไม่มี points; ADR ไม่กล่าวถึงช่องนี้เลย |
| Risk | Log `event_lost` พร้อม **payload เต็ม** (มี email ลูกค้า) | PII ไปอยู่ใน log system; ADR ไม่ระบุ retention/การเข้าถึง |
| Risk | Publisher retry หลัง confirm timeout ⇒ event ซ้ำ ทั้งที่ broker เก็บไปแล้ว | loyalty กันได้ด้วย unique `order_id`; แต่ email กันซ้ำได้เฉพาะถ้า MailFast มี idempotency key (Open question) และ analytics ไม่มีแผน dedupe เลย |
| Sensitivity point | ระยะเวลา peak 11.11 × ส่วนต่าง 150 emails/s | backlog = 9,000/นาที; peak 1 ชม. ⇒ 540,000 ฉบับ ใช้เวลาระบาย > 3 ชม. ที่ 50/s; ขนาด disk ของ RabbitMQ และ email latency ขึ้นกับค่านี้ตรง ๆ แต่ยังไม่รู้ค่า |
| Sensitivity point | Publisher-confirm timeout **100 ms** | ถ้า broker ค้าง (ไม่ใช่ refuse) ทุก order จ่าย +100 ms ⇒ ≈ 400 ms ใกล้ 500 ms; ไม่มี circuit breaker ให้ข้ามการรอเมื่อรู้แล้วว่า broker ล่ม |
| Trade-off point | ไม่ใช้ transactional outbox | ได้ latency + ไม่มี DB ใหม่บน critical path (performance, availability ของ order) แลกกับ durability ของ event (reliability, correctness ของ loyalty) |
| Trade-off point | RabbitMQ 3-node quorum cluster | ได้ availability ของ post-order actions แต่เพิ่ม operability cost ให้ทีม 4 คน ที่ยังไม่มีคน on-call (Open questions) |
| Non-risk | Queue แยกต่อ consumer + token bucket 50/s + DLQ | backlog email ไม่ลาม loyalty; ไม่ชน rate limit; failure 0.5 % (≈ 1 msg/s ตอน peak) รับได้สบายด้วย retry 5 ครั้ง |
| Non-risk | Loyalty idempotent ด้วย unique `order_id` + ack หลัง commit | ได้ effectively-once จาก at-least-once ถูกหลัก; alert message age > 30 s ให้เวลาเหลือก่อน 1 นาที |
| Non-risk | Validation table มีเกณฑ์ตัวเลขผูกกับ scenario ครบ (200 orders/s, 500 ms, 60 s, 50 calls/s) | ตรวจได้จริง ไม่ใช่แค่ "ควรเร็ว" |

## 3. Missing or weak

- **Option ที่ไม่ได้พิจารณา:** (a) **durable local spool** บน PersistentVolume แทน in-memory buffer (ทางกลางระหว่าง buffer กับ outbox, ไม่ต้องมี DB ใหม่); (b) broker อื่นที่เบากว่า Kafka เช่น NATS JetStream / Redis Streams; (c) hybrid: loyalty ผ่าน sync gRPC แบบ idempotent + timeout สั้น, ใช้ queue เฉพาะ email; (d) เพิ่ม rate limit / provider อีเมลตัวที่สอง เป็น option ระดับสถาปัตยกรรม ไม่ใช่แค่ mitigation
- **Consequence ที่ไม่ได้เขียน:** อีเมลยืนยันอาจถึงช้าหลายชั่วโมงตอน 11.11 (ยอมรับโดยยังไม่มี email SLA); schema/versioning ของ `OrderPlaced` เมื่อมี consumer เพิ่มปีหน้า (modifiability); PII ใน log
- **Claim ที่น่าจะผิด/ไม่มีหลักฐาน:** "RabbitMQ ล่มไม่กระทบ order" จริงเฉพาะ correctness ไม่ใช่ latency (ดู sensitivity 100 ms); "event ไม่หายในกรณีปกติ" ไม่รวม pod crash; "Kafka หนักเกินไป" สมเหตุสมผลแต่ไม่มีตัวเลข operate cost เทียบ
- **Validation ขาด:** kill `checkoutservice` pod (SIGKILL) ระหว่าง broker outage; วัด p95 ขณะ broker **ค้าง** (packet drop) ไม่ใช่แค่ปิด; ทดสอบ duplicate event กับ email และ analytics

## 4. Verdict: **Accept with changes**

ทิศทาง (event + RabbitMQ) ถูกต้องและตอบ driver ได้ดีกว่า option อื่น แต่ต้องแก้ก่อน `Accepted`:

1. **ปิดช่อง event หาย:** เปลี่ยน in-memory buffer เป็น durable spool (PV) หรือให้เหตุผลเชิงตัวเลขว่าทำไมไม่ใช้ outbox, เขียน failure mode "crash หลัง Charge" ให้ชัด, และเพิ่ม test SIGKILL ระหว่าง broker outage
2. **ได้ email SLA + ระยะเวลา peak จาก business ก่อนตัดสิน:** คำนวณ backlog, disk ของ RabbitMQ, เวลาระบาย; ถ้า SLA รับไม่ได้ ให้ยกระดับ "ขอ rate limit เพิ่ม / provider ที่สอง" เป็นส่วนหนึ่งของ decision
3. **จัดการ duplicate และ PII:** ระบุ dedupe ของ email (MailFast idempotency key หรือ processed-`order_id` store) และ analytics; เพิ่ม circuit breaker รอบ publish; ห้าม log email ลูกค้าเต็มใน `event_lost` (log เฉพาะ `order_id` + เก็บ payload ใน store ที่ควบคุมสิทธิ์)
