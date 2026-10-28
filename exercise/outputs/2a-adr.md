> Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`) · Run: a (zero-shot, context: none)

# ADR-0002: Publish an "order placed" event to a message broker for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-28
- **Deciders:** ยังไม่ระบุ (ดู Open questions)

## Context and problem

หลังจากลูกค้าสั่งซื้อสำเร็จ Online Boutique ต้องทำงานต่อเนื่อง (post-order actions) อย่างน้อย 3 อย่าง ตาม requirement:

| # | งานหลังสั่งซื้อ | ผู้รับผิดชอบ | สถานะ |
|---|---|---|---|
| 1 | ส่งอีเมลยืนยันคำสั่งซื้อ | MailFast (external email provider ใหม่) แทน logic mock ใน `emailservice` | ใหม่ / แทนของเดิม |
| 2 | ให้คะแนนสะสม (loyalty points) | `loyaltyservice` (ยังไม่ได้สร้าง) | ใหม่ |
| 3 | บันทึก analytics event "order placed" ให้ทีม marketing | ยังไม่ระบุปลายทาง | ใหม่ |

- ทีม marketing คาดว่าปีหน้าจะมี post-order actions เพิ่มอีก: SMS, fraud scoring, การอัปเดต recommendations
- ปัญหาที่ต้องตัดสินใจ: ส่วนที่ทำการสั่งซื้อควร **เรียกงานเหล่านี้อย่างไร** เพื่อให้ (ก) การสั่งซื้อไม่ล้มหรือช้าลงเพราะระบบปลายทาง เช่น MailFast และ (ข) เพิ่ม action ใหม่ได้โดยไม่ต้องแก้ service ที่สั่งซื้อทุกครั้ง
- service ใดเป็นผู้ "place order" และวิธีที่มันเรียก `emailservice` ในปัจจุบัน: Not evidenced in the provided files (run นี้ไม่มี context ให้อ่าน)
- ตัวเลขด้าน performance/availability (จำนวน order ต่อวินาที, latency target, SLA ของ MailFast): Not evidenced in the provided files

## Decision drivers

1. **Modifiability / extensibility** — เพิ่ม post-order action ใหม่ (SMS, fraud scoring, recommendations) ได้โดยไม่ต้องแก้ flow การสั่งซื้อ
2. **Availability ของการสั่งซื้อ** — MailFast, `loyaltyservice` หรือ analytics ล่ม ต้องไม่ทำให้การสั่งซื้อล้ม
3. **Performance (latency ของการสั่งซื้อ)** — ลูกค้าไม่ควรรอให้ทุก action ทำเสร็จก่อนได้ผลลัพธ์
4. **Reliability ของแต่ละ action** — อีเมลและคะแนนสะสมต้องไม่หาย และไม่ซ้ำ (เช่นไม่ให้คะแนนสองครั้ง)
5. **Cost / operational complexity** — จำนวน infrastructure ใหม่ที่ทีมต้องดูแล

## Considered options

1. **A: Synchronous orchestration** — service ที่สั่งซื้อเรียก `emailservice` (เปลี่ยนเป็น adapter ไป MailFast), `loyaltyservice` และ analytics ทีละตัวโดยตรง (เช่นผ่าน gRPC) ภายใน request เดียวกัน
2. **B: Event-driven publish/subscribe** — service ที่สั่งซื้อ publish event `OrderPlaced` ไปยัง message broker ครั้งเดียว แต่ละ consumer (`emailservice` → MailFast, `loyaltyservice`, analytics) subscribe และประมวลผลแบบ asynchronous ของตัวเอง
3. **C: Dedicated post-order orchestrator service** — service ที่สั่งซื้อเรียก `postorderservice` ตัวเดียว (sync หรือ async) แล้ว service นี้เรียก action ทั้งหมดต่อ พร้อม retry และเก็บสถานะ workflow

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A: Synchronous orchestration | เรียบง่าย ไม่ต้องมี infrastructure ใหม่; debug ง่ายเพราะ call chain ตรง; รู้ผลทันทีว่าแต่ละ action สำเร็จหรือไม่ | latency ของการสั่งซื้อ = ผลรวมของทุก action; ถ้า MailFast ล่มหรือช้า การสั่งซื้อก็ล้ม/ช้าตาม (availability ลดลง); ทุก action ใหม่ต้องแก้และ redeploy service ที่สั่งซื้อ (coupling สูง, modifiability ต่ำ); retry ต้องเขียนเองทุกจุด |
| B: Event-driven pub/sub | service ที่สั่งซื้อไม่ต้องรู้จัก consumer — เพิ่ม SMS/fraud/recommendations แค่เพิ่ม subscriber (modifiability สูง); consumer ล่มไม่กระทบการสั่งซื้อ (availability); latency การสั่งซื้อเพิ่มแค่การ publish หนึ่งครั้ง; broker buffer และ retry ให้เมื่อ MailFast ช้า | ต้องเพิ่มและดูแล message broker (cost, ops); eventual consistency — อีเมล/คะแนนมาถึงช้ากว่าการสั่งซื้อ; ต้องจัดการ duplicate delivery (idempotency) และ dead-letter; ต้องกัน "order สำเร็จแต่ publish ไม่สำเร็จ" (เช่น outbox pattern); trace ข้าม service ยากขึ้น |
| C: Post-order orchestrator | flow ทั้งหมดอยู่จุดเดียว เห็นสถานะชัด; รองรับลำดับ/เงื่อนไขระหว่าง action (เช่นต้องผ่าน fraud scoring ก่อน) ได้ดี; service ที่สั่งซื้อพึ่งแค่ service เดียว | orchestrator กลายเป็นจุดที่ต้องแก้ทุกครั้งที่มี action ใหม่ (modifiability ปานกลาง ย้ายปัญหามาอีกที่); ถ้าเรียกแบบ sync ยังเป็น single point of failure ของการสั่งซื้อ; ถ้าทำ async + เก็บ state ก็ต้องสร้าง service ใหม่ที่ซับซ้อนพอ ๆ กับ broker |

## Decision

เลือก **Option B: Event-driven publish/subscribe**

- service ที่สั่งซื้อ publish event `OrderPlaced` (มี order id, ผู้ใช้, อีเมล, รายการสินค้า, ยอดรวม) หลังบันทึก/ยืนยันการสั่งซื้อสำเร็จ
- consumer เริ่มต้น 3 ตัว:
  - `emailservice` — เปลี่ยน logic mock เป็น adapter ที่เรียก MailFast API (แยก MailFast ออกจาก flow การสั่งซื้อ)
  - `loyaltyservice` (ใหม่) — คำนวณและบันทึกคะแนนสะสม
  - analytics consumer — ส่ง event "order placed" ไปยังปลายทางของทีม marketing
- ทุก consumer ต้อง **idempotent** โดยใช้ order id เป็น key และมี dead-letter queue
- เทคโนโลยี broker (เช่น RabbitMQ, Kafka, Google Pub/Sub) ยังไม่ตัดสิน — ดู Open questions

เหตุผล: driver อันดับ 1–3 (modifiability, availability, latency ของการสั่งซื้อ) ชี้ไปที่ B ชัดเจน เพราะ marketing ระบุแล้วว่าจะมี action เพิ่มอีก 3 อย่าง ซึ่ง A และ C ต้องแก้ code กลางทุกครั้ง ส่วนต้นทุนเรื่อง broker และ eventual consistency ยอมรับได้ **[Inferred]** เพราะอีเมล, คะแนน และ analytics ไม่จำเป็นต้องเสร็จภายใน request ของลูกค้า — ข้อนี้ต้องยืนยันกับ product owner (ดู Open questions)

ข้อยกเว้นที่ต้องพิจารณาในอนาคต: fraud scoring อาจต้อง **block** การสั่งซื้อ ซึ่งไม่เหมาะกับ async consumer — ถ้าเป็นเช่นนั้นควรมี ADR แยก

## Consequences

- **Positive:** เพิ่ม SMS, fraud scoring (แบบไม่ block), recommendations update ได้โดยเพิ่ม subscriber ใหม่ ไม่ต้องแก้ service ที่สั่งซื้อ (quality attribute: modifiability)
- **Positive:** MailFast, `loyaltyservice` หรือ analytics ล่ม ไม่ทำให้การสั่งซื้อล้ม; message รอใน broker จนกว่า consumer จะกลับมา (quality attribute: availability)
- **Positive:** latency ของการสั่งซื้อเพิ่มแค่การ publish หนึ่งครั้ง ไม่ขึ้นกับความเร็วของ MailFast (quality attribute: performance)
- **Positive:** การเปลี่ยน email provider ในอนาคตกระทบแค่ `emailservice` (quality attribute: modifiability)
- **Negative:** ต้องติดตั้ง, monitor และดูแล message broker เพิ่ม (quality attribute: cost, operability)
- **Negative:** อีเมลและคะแนนสะสมมาถึงแบบ eventual — ลูกค้าอาจเห็นคะแนนช้ากว่าการสั่งซื้อ (quality attribute: consistency / usability)
- **Negative:** debug และ trace flow ข้าม service ยากขึ้น ต้องมี correlation id / distributed tracing (quality attribute: observability, testability)
- **Risks and mitigations:**
  - order สำเร็จแต่ publish event ไม่สำเร็จ → ใช้ transactional outbox หรือ retry ที่ publish side
  - broker ส่ง message ซ้ำ ทำให้ให้คะแนนหรือส่งอีเมลสองครั้ง → consumer idempotent โดยใช้ order id
  - MailFast rate limit หรือล่มนาน → retry แบบ exponential backoff + dead-letter queue + alert
  - schema ของ `OrderPlaced` เปลี่ยนแล้ว consumer พัง → version schema และเปลี่ยนแบบ backward-compatible เท่านั้น
  - broker เป็น single point of failure ของ post-order actions → deploy แบบ HA **[Inferred]** (ขึ้นกับเทคโนโลยีที่เลือก)

## Validation

- **Load test:** เทียบ latency ของการสั่งซื้อ (p95/p99) ก่อน-หลัง ต้องเพิ่มไม่เกินค่าเป้าหมาย (ค่าเป้าหมายยังไม่กำหนด — ดู Open questions)
- **Fault injection:** ปิด MailFast (หรือ stub), `loyaltyservice` และ analytics consumer แล้วสั่งซื้อ → การสั่งซื้อต้องสำเร็จ 100% และ message ถูกประมวลผลครบหลัง consumer กลับมา
- **Duplicate test:** ส่ง `OrderPlaced` เดิมซ้ำ → คะแนนสะสมและอีเมลต้องเกิดเพียงครั้งเดียว
- **Metrics:** consumer lag, จำนวน message ใน dead-letter queue, อัตราส่งอีเมลสำเร็จของ MailFast, เวลาจาก order ถึงอีเมลถึงลูกค้า
- **Modifiability check:** เมื่อเพิ่ม action แรกของปีหน้า (เช่น SMS) ต้องไม่มีการแก้ code ของ service ที่สั่งซื้อ
- **Review date:** ทบทวน ADR นี้ก่อนเริ่มทำ fraud scoring (ยังไม่ระบุวันที่)

## Open questions

- service ใดเป็นผู้ place order และปัจจุบันเรียก `emailservice` อย่างไร (protocol, sync/async)? Not evidenced in the provided files
- จำนวน order ต่อวินาที (ปัจจุบันและ peak) และเป้าหมาย latency ของการสั่งซื้อคือเท่าไร?
- ยอมรับได้หรือไม่ที่อีเมลยืนยันและคะแนนสะสมเกิดขึ้นช้ากว่าการสั่งซื้อ (eventual consistency)? delay สูงสุดที่ยอมรับได้คือเท่าไร?
- MailFast มี API แบบใด (REST/SMTP), rate limit, SLA, ราคา และรองรับ idempotency key หรือไม่?
- ระบบมี message broker อยู่แล้วหรือไม่? ทีมมีทักษะกับเทคโนโลยีใด (RabbitMQ, Kafka, Google Pub/Sub)? มีข้อจำกัดด้าน cloud/budget หรือไม่?
- ปลายทางของ analytics event สำหรับทีม marketing คืออะไร (data warehouse, analytics SaaS)?
- `loyaltyservice` ใช้ data store อะไร และกฎการให้คะแนนคืออะไร? ใครเป็นเจ้าของ service นี้?
- fraud scoring ต้อง block การสั่งซื้อหรือไม่ (ถ้าใช่ ต้องเป็น synchronous step และควรมี ADR แยก)?
- มีข้อกำหนดด้าน privacy (PII เช่นอีเมลลูกค้าใน event) หรือ data retention หรือไม่?
- ใครคือ deciders ของ ADR นี้?
