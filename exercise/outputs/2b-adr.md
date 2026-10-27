Agent: Claude Sonnet 5.5 (claude-sonnet-5-5) via Claude Code subagent

# ADR-0002: Publish "order placed" event ผ่าน RabbitMQ เพื่อแยก post-order actions ออกจาก checkout

- **Status:** Proposed
- **Date:** 2026-10-26
- **Deciders:** ทีมพัฒนา (4 developers) และ marketing team [Open question: รายชื่อ deciders จริงไม่ได้ระบุใน context]

## Context and problem

Requirement R-12 (`docs/scenario.md`) ต้องการให้หลัง order ถูกสร้างแล้ว ระบบต้อง (1) ส่งอีเมลยืนยันผ่าน MailFast (external provider ใหม่ แทน logic ของ mock `emailservice`), (2) ให้ loyalty points ผ่าน `loyaltyservice` ใหม่ และ (3) บันทึก analytics event "order placed" ให้ marketing team นอกจากนี้ marketing คาดว่าจะมี post-order actions เพิ่มปีหน้า (SMS, fraud scoring, recommendations update)

สถานะปัจจุบัน: `checkoutservice.PlaceOrder` เรียก `paymentservice` (`chargeCard`), `shippingservice` (`shipOrder`) แล้วเรียก `emailservice` แบบ synchronous gRPC ตามลำดับ (`system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder` บรรทัด 252-277 และ `sendOrderConfirmation` บรรทัด 379-384) ถ้าส่งอีเมลล้มเหลว โค้ดแค่ `log.Warnf` และ order ยังสำเร็จ (บรรทัด 273-277) การเชื่อมต่อแต่ละ downstream ถูกกำหนดผ่าน env var และ `mustConnGRPC` ตอนเริ่มระบบ (เช่น `EMAIL_SERVICE_ADDR`, บรรทัด 115, 122) ดังนั้นการเพิ่ม downstream ใหม่แต่ละตัวแบบเดิมหมายถึงแก้โค้ด `checkoutservice` และ deploy ใหม่ทุกครั้ง นอกจากนี้ใน `main.go` ไม่พบ data store ของ `checkoutservice` เอง (Not evidenced in the provided files)

ข้อเท็จจริงจาก scenario ที่กระทบการตัดสินใจ:

| หัวข้อ | ข้อเท็จจริง |
|---|---|
| Checkout latency | p95 ของ `PlaceOrder` ต้อง <= 500 ms (ปัจจุบันประมาณ 300 ms) |
| Peak load | 200 orders/s ช่วง sale 11.11 |
| MailFast | p95 1.5 s, p99 8 s, ล้มเหลวประมาณ 0.5 % ของ calls, rate limit 50 emails/s |
| Loyalty points | ต้องได้ภายใน 1 นาทีหลังสั่งซื้อ และ exactly once (ห้ามได้แต้มซ้ำ) |
| Availability | order ต้องไม่ล้มเพราะ email, loyalty หรือ analytics ล่ม |
| Team | 4 developers เก่ง Go และ gRPC, 2 คนเคยใช้ RabbitMQ |
| Operations | ทุกอย่างรันบน Kubernetes อยู่แล้ว บริษัทนิยม open-source, self-hosted มากกว่า managed service ที่ใช้ได้เฉพาะ cloud |

ปัญหาเชิงตัวเลข: ถ้าเรียก MailFast แบบ synchronous ใน `PlaceOrder` ค่า p95 ของ MailFast (1.5 s) เพียงอย่างเดียวก็เกินงบ 500 ms ของ checkout ทั้งหมด และ p99 8 s จะทำให้ request ค้างนาน นอกจากนี้ที่ 200 orders/s ซึ่งสูงกว่า rate limit 50 emails/s ของ MailFast อยู่ 4 เท่า การส่งอีเมลทันทีทุก order จะถูก rate-limit แน่นอนที่ peak [คำนวณจาก scenario: 200 / 50 = 4 เท่า]

## Decision drivers

เรียงตามความสำคัญ:

1. **Availability:** order ต้องไม่ล้มเพราะ email, loyalty หรือ analytics ล่ม
2. **Performance (latency):** p95 `PlaceOrder` <= 500 ms ที่ 200 orders/s ไม่ว่า MailFast จะช้า (p95 1.5 s, p99 8 s)
3. **Correctness / reliability:** loyalty points ภายใน 1 นาที และ exactly once
4. **Modifiability:** เพิ่ม post-order actions ใหม่ (SMS, fraud scoring, recommendations) โดยไม่แก้ `checkoutservice` ทุกครั้ง
5. **Throughput / external limit:** รองรับ burst 200 orders/s กับ MailFast rate limit 50 emails/s
6. **Cost / operability / team fit:** open-source, self-hosted บน Kubernetes; ทีมเก่ง Go และ gRPC, 2 คนเคยใช้ RabbitMQ

## Considered options

1. ขยายแบบเดิม: `checkoutservice` เรียก MailFast, `loyaltyservice` และ analytics แบบ synchronous gRPC/HTTP (ทำต่อใน `PlaceOrder`)
2. Fire-and-forget ภายใน `checkoutservice`: เรียก downstream เหล่านั้นใน goroutine แยก (async ในโปรเซสเดียว ไม่มี broker)
3. Event-driven ด้วย message broker แบบ self-hosted (RabbitMQ): `checkoutservice` publish event `OrderPlaced` แล้ว consumer แต่ละตัว (email, loyalty, analytics) subscribe queue ของตัวเอง
4. Managed pub/sub ของ cloud (เช่น บริการ queue/topic ที่เป็น cloud-specific)

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous calls ต่อไป | เรียบง่ายที่สุด ตรงกับ pattern ที่โค้ดใช้อยู่แล้ว (`sendOrderConfirmation`); ทีมคุ้นเคย gRPC; ไม่มี component ใหม่ | MailFast p95 1.5 s ทำให้ p95 ของ `PlaceOrder` เกิน 500 ms (performance); p99 8 s ค้างนาน; ถูก rate limit 50/s ที่ 200 orders/s; ทุก downstream ใหม่ต้องแก้และ deploy `checkoutservice` (modifiability); latency และความเสี่ยงล่มสะสมต่อ downstream (availability) |
| 2. Goroutine fire-and-forget | latency ของ checkout ไม่ขึ้นกับ MailFast; ไม่มี component ใหม่; เปลี่ยนโค้ดน้อย | งานหายถ้า pod restart หรือ crash (ไม่มี durability) ซึ่งขัดกับ loyalty "exactly once ภายใน 1 นาที"; ไม่มี retry/backoff หรือ buffering ที่จัดการ rate limit 50/s; เพิ่ม action ใหม่ยังต้องแก้ `checkoutservice` |
| 3. RabbitMQ (self-hosted) publish/subscribe | checkout แค่ publish event แล้วตอบทันที จึงแยก latency/ความล่มของ MailFast, loyalty, analytics ออกจาก `PlaceOrder`; queue ของ email ช่วย buffer และ throttle ให้ไม่เกิน 50/s; message durable + ack + retry + dead-letter queue; เพิ่ม consumer ใหม่ (SMS, fraud, recommendations) โดยไม่แก้ producer; open-source รันบน Kubernetes ได้ ตรงกับ policy; 2 ใน 4 คนเคยใช้ | เพิ่ม component stateful ที่ต้อง operate (HA, monitoring); delivery เป็น at-least-once จึงต้องทำ idempotent consumer เอง; eventual consistency (อีเมลและแต้มไม่เกิดทันที); มี dual-write risk ระหว่างชำระเงิน/จัดส่งกับ publish (ดู Risks); ทีมอีก 2 คนต้องเรียนรู้ |
| 4. Managed cloud pub/sub | ไม่ต้อง operate broker เอง; มี durability และ scaling ในตัว | ขัดกับ policy ที่นิยม open-source, self-hosted และเลี่ยง cloud-only; ผูกกับ cloud vendor (portability); ต้นทุนต่อ message [ตัวเลขไม่มีใน context] |

## Decision

เลือก **Option 3: Event-driven ด้วย RabbitMQ (self-hosted บน Kubernetes)**

- `checkoutservice` publish event `OrderPlaced` (มี `order_id`, email, items ฯลฯ) หลังชำระเงินและจัดส่งสำเร็จ แล้วตอบ `PlaceOrder` ทันที โดยการ publish ล้มเหลวต้องไม่ทำให้ order ล้ม (ให้ log/metric เหมือนที่ปัจจุบันทำกับอีเมลที่ `log.Warnf`)
- ใช้ topic/fanout exchange โดยแต่ละ consumer มี durable queue ของตัวเอง: email (ปรับ `emailservice` ให้เรียก MailFast แทน mock logic), `loyaltyservice` (ใหม่), analytics
- **Email consumer** ควบคุมอัตราส่งไม่ให้เกิน 50 emails/s ของ MailFast และ retry พร้อม backoff เมื่อล้มเหลว (~0.5 %) แล้วส่งเข้า dead-letter queue เมื่อเกินจำนวนครั้ง
- **Loyalty consumer** ใช้ `order_id` เป็น idempotency key (เช่น unique constraint ใน data store ของ `loyaltyservice`) เพื่อให้ at-least-once delivery กลายเป็น exactly-once effect ตามที่ requirement ต้องการ
- Action ใหม่ในอนาคต (SMS, fraud scoring, recommendations update) เพิ่มโดยสร้าง consumer และ queue ใหม่ ไม่ต้องแก้ producer

เหตุผล: Option นี้เป็นตัวเลือกเดียวที่ตอบ driver อันดับต้นพร้อมกันได้ คือ checkout latency และ availability (แยกจาก MailFast ที่ช้า/ล่ม), durability สำหรับ loyalty ภายใน 1 นาที, การ buffer เมื่อ 200 orders/s ชนกับ rate limit 50/s, และ modifiability สำหรับ action ปีหน้า พร้อมทั้งสอดคล้องกับ policy self-hosted open-source และทักษะของทีม (2 คนเคยใช้ RabbitMQ)

## Consequences

- **Positive:** latency ของ `PlaceOrder` ไม่ขึ้นกับ MailFast (p95 1.5 s, p99 8 s) จึงรักษาเป้า p95 <= 500 ms ได้ (quality attribute: performance)
- **Positive:** email, loyalty หรือ analytics ล่มไม่กระทบ order เพราะ message รอใน queue (quality attribute: availability)
- **Positive:** queue ดูดซับ burst 200 orders/s และ consumer ส่งตาม rate limit 50/s (quality attribute: performance / scalability)
- **Positive:** เพิ่ม consumer ใหม่ได้โดยไม่แก้ `checkoutservice` (quality attribute: modifiability)
- **Positive:** ใช้ open-source บน Kubernetes เดิม ไม่ผูก cloud vendor (quality attribute: cost / portability)
- **Negative:** มี component stateful เพิ่มให้ operate และเป็น single point of failure ถ้าไม่ทำ HA (quality attribute: availability / operability)
- **Negative:** eventual consistency: อีเมลและแต้มมาช้ากว่าเดิม; ที่ peak backlog ของอีเมลเกิดขึ้นแน่ เพราะรับเข้า 200/s ส่งออกได้ 50/s ผู้ใช้อาจได้อีเมลช้า (quality attribute: performance ฝั่ง user-perceived)
- **Negative:** consumer ต้องเขียน idempotency และ retry เอง ความซับซ้อนของโค้ดและการ debug เพิ่ม (quality attribute: modifiability / testability)
- **Risks and mitigations:**
  - *Dual-write:* ถ้า publish ล้มหลังชำระเงินแล้ว event หาย จะไม่มีแต้ม/อีเมล ลดความเสี่ยงด้วยการ retry publish พร้อม publisher confirms หรือใช้ transactional outbox [ต้องมี data store ฝั่ง checkout ซึ่ง Not evidenced ใน `main.go` ดู Open questions]
  - *Broker ล่ม:* ทำ RabbitMQ cluster/quorum queues และ monitoring
  - *Backlog อีเมลที่ peak:* ตั้ง alert บน queue depth และประเมินระยะเวลา sale เทียบกับ backlog (ดู Open questions)
  - *PII:* event มีอีเมลลูกค้า ต้องกำหนดการเข้าถึงและการเข้ารหัสของ broker [ข้อกำหนดไม่มีใน context]

## Validation

- Load test ที่ 200 orders/s (เช่นด้วย Locust `loadgenerator`) โดยจำลอง MailFast ที่ p95 1.5 s / p99 8 s / fail 0.5 % และตรวจว่า p95 ของ `PlaceOrder` <= 500 ms
- Fault injection: ปิด MailFast, `loyaltyservice`, analytics และ RabbitMQ ทีละตัว ตรวจว่า order ยังสำเร็จ (ยกเว้นกรณี broker ล่ม ที่ต้องไม่ทำให้ order ล้ม)
- ตรวจว่าแต้ม loyalty ปรากฏภายใน 1 นาทีหลังสั่งซื้อ และไม่มีแต้มซ้ำเมื่อส่ง message ซ้ำ (ทดสอบ redelivery)
- วัด email send rate ไม่เกิน 50/s และ queue depth/เวลาหน่วงของอีเมลที่ peak
- กำหนดวันทบทวน ADR หลังเพิ่ม consumer ตัวที่ 4 (เช่น SMS)

## Open questions

- ระยะเวลาที่ peak 200 orders/s คงอยู่ (ช่วง 11.11 นานเท่าไร) และ SLA ของความล่าช้าอีเมลที่ยอมรับได้ เพื่อประเมิน backlog จาก 200/s เข้า กับ 50/s ออก
- Deciders จริง และผู้ถือ requirement ฝั่ง marketing
- SLA ของ analytics event (ยอมสูญหายได้หรือไม่ ใช้ at-least-once พอหรือไม่) และ schema ของ event
- `checkoutservice` มี data store ที่ใช้ทำ transactional outbox ได้หรือไม่ (Not evidenced in the provided files) ถ้าไม่มี จะยอมรับความเสี่ยง event หาย หรือเพิ่ม store
- `loyaltyservice` จะใช้ data store อะไรเก็บแต้มและ idempotency key และกติกาการคิดแต้ม
- ขนาดทีม operate: ใครรับผิดชอบ HA/monitoring ของ RabbitMQ (ทีมมี 4 คน, 2 คนเคยใช้ RabbitMQ)
- ข้อกำหนดด้านความปลอดภัย/PII สำหรับอีเมลลูกค้าใน message และ credentials/สัญญา (ราคา, quota) ของ MailFast
- เมื่อ MailFast ล้มเหลวถาวรหลัง retry จะให้ทำอย่างไร (แจ้งทีม, ส่งซ้ำด้วยมือจาก dead-letter queue หรือ fallback)
