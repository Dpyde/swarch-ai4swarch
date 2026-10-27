# ADR-0002: ใช้ RabbitMQ ส่ง event "OrderPlaced" เพื่อแยก post-order actions ออกจาก checkout

- **Status:** Proposed
- **Date:** 2026-10-26
- **Deciders:** Thukdanai Thaothawin (architect, ผู้เขียน); ทีมพัฒนา 4 คน และ review board (รอการตัดสินในส่วน B)

> ร่างตั้งต้นมาจาก `exercise/outputs/2b-adr.md` (AI) และถูกแก้ไขโดยผู้เขียนตามตารางในใบงาน `exercise/02-write-adr.md`

## Context and problem

Requirement R-12 (`docs/scenario.md`) กำหนดว่าหลังสั่งซื้อสำเร็จ ระบบต้อง (1) ส่งอีเมลยืนยันผ่าน MailFast ซึ่งเป็น external provider ใหม่และมาแทน logic mock ของ `emailservice` (2) ให้ loyalty points ผ่าน `loyaltyservice` ที่ต้องสร้างใหม่ และ (3) บันทึก analytics event "order placed" ให้ marketing นอกจากนี้ marketing คาดว่าปีหน้าจะมี post-order actions เพิ่ม (SMS, fraud scoring, recommendations update)

ปัจจุบัน `checkoutservice.PlaceOrder` เรียก `chargeCard`, `shipOrder` แล้วเรียก `sendOrderConfirmation` ไปยัง `emailservice` แบบ synchronous gRPC ต่อกันเป็นลำดับ ถ้าอีเมลล้มเหลวจะแค่ `log.Warnf` และ order ยังสำเร็จ (`system/online-boutique/src/checkoutservice/main.go` บรรทัด 252-277 และ 379-384; การเชื่อมต่อ downstream กำหนดผ่าน env เช่น `EMAIL_SERVICE_ADDR` ที่บรรทัด 115 และ `mustConnGRPC` ที่บรรทัด 122) การเพิ่ม downstream ใหม่แต่ละตัวแบบเดิมจึงต้องแก้และ deploy `checkoutservice` ใหม่ทุกครั้ง ส่วน data store ของ `checkoutservice` เอง Not evidenced in the provided files

| หัวข้อ | ข้อเท็จจริง (จาก `docs/scenario.md`) |
|---|---|
| Checkout latency | p95 ของ `PlaceOrder` ต้อง ≤ 500 ms (ปัจจุบันประมาณ 300 ms) |
| Peak load | 200 orders/s ช่วง 11.11 |
| MailFast | p95 1.5 s, p99 8 s, ล้มเหลวประมาณ 0.5 %, rate limit 50 emails/s |
| Loyalty points | ภายใน 1 นาที และ exactly once |
| Availability | order ต้องไม่ล้มเพราะ email, loyalty หรือ analytics ล่ม |
| Team | 4 developers เก่ง Go และ gRPC, 2 คนเคยใช้ RabbitMQ |
| Operations | รันบน Kubernetes อยู่แล้ว ต้องการ open-source, self-hosted |

ข้อสังเกตเชิงตัวเลข (คำนวณจากข้อมูลข้างต้น):

- เรียก MailFast แบบ sync ใน `PlaceOrder` ไม่ได้ เพราะ p95 ของ MailFast (1.5 s) เกินงบ 500 ms ของ checkout ทั้งหมดอยู่แล้ว
- ที่ 200 orders/s อีเมลเข้า 200/s แต่ MailFast รับได้ 50/s จึงเกิด backlog ประมาณ **150 emails/s** ตลอดช่วง peak (ทุก 10 นาทีของ peak สะสมราว 90,000 ฉบับ และต้องใช้เวลาระบายอีกราว 30 นาทีหลัง peak จบ) ดังนั้น broker อย่างเดียวไม่ทำให้อีเมลทันเวลา แต่ทำให้ checkout ไม่ได้รับผลกระทบ และต้องมีข้อตกลงเรื่องความล่าช้าของอีเมลที่ยอมรับได้

## Decision drivers

1. **Availability:** order ต้องไม่ล้มเพราะ email, loyalty หรือ analytics ล่ม
2. **Performance:** p95 `PlaceOrder` ≤ 500 ms ที่ 200 orders/s แม้ MailFast ช้า
3. **Reliability:** loyalty points ภายใน 1 นาทีและ exactly once; email ต้องไม่หายเงียบ ๆ
4. **Modifiability:** เพิ่ม action ใหม่ (SMS, fraud, recommendations) โดยไม่แก้ `checkoutservice`
5. **Throughput:** รับ burst 200 orders/s ภายใต้ MailFast rate limit 50 emails/s
6. **Cost / team fit:** open-source, self-hosted บน Kubernetes; ทักษะ Go และ gRPC; 2 คนเคยใช้ RabbitMQ

## Considered options

1. Synchronous calls ต่อไป: `checkoutservice` เรียก MailFast, `loyaltyservice` และ analytics ตรง ๆ ใน `PlaceOrder`
2. Fire-and-forget ด้วย goroutine ภายใน `checkoutservice` (ไม่มี broker)
3. RabbitMQ (self-hosted): `checkoutservice` publish `OrderPlaced` ให้ consumer แต่ละตัวมี durable queue ของตัวเอง
4. Apache Kafka (self-hosted): publish ลง topic แล้ว consumer group อ่านเอง
5. Managed pub/sub ของ cloud

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Sync ต่อไป | ง่ายที่สุด ตรงกับโค้ดปัจจุบัน ไม่มี component ใหม่ | MailFast p95 1.5 s ทำให้ p95 ของ `PlaceOrder` เกิน 500 ms; p99 8 s ค้างนาน; ชน rate limit 50/s ที่ 200 orders/s; ทุก downstream ใหม่ต้องแก้ `checkoutservice` (modifiability); ความเสี่ยงล่มสะสม (availability) |
| 2. Goroutine | latency ไม่ขึ้นกับ MailFast; ไม่มี component ใหม่ | งานหายเมื่อ pod restart จึงขัดกับ loyalty "exactly once ภายใน 1 นาที"; ไม่มี buffer จัดการ rate limit; action ใหม่ยังต้องแก้ producer |
| 3. RabbitMQ | checkout publish แล้วตอบทันที จึงตัด latency/ความล่มของ downstream ออก; queue ของ email ช่วย buffer และ throttle; durable + ack + retry + dead-letter queue; เพิ่ม consumer โดยไม่แก้ producer; open-source บน Kubernetes ตรง policy; 2 ใน 4 คนเคยใช้ | เป็น stateful component ที่ต้อง operate (HA, monitoring); delivery แบบ at-least-once ต้องทำ idempotent เอง; eventual consistency; dual-write risk ตอน publish |
| 4. Kafka | throughput สูง, replay event ได้ | หนักเกินความจำเป็นสำหรับ 200 events/s; ไม่มีใครในทีมมีประสบการณ์ (Not evidenced ใน scenario ว่ามี); operate ยากกว่า; per-message retry/DLQ ต้องสร้างเอง |
| 5. Managed pub/sub | ไม่ต้อง operate เอง | ขัด policy self-hosted/open-source; ผูก cloud vendor |

## Decision

เลือก **Option 3: RabbitMQ (self-hosted บน Kubernetes)** ดังนี้

- `checkoutservice` publish event `OrderPlaced` (`order_id`, email, items ที่จำเป็นต่อการทำอีเมลและคำนวณแต้ม) หลัง `shipOrder` สำเร็จ แล้วตอบ `PlaceOrder` ทันที ใช้ publisher confirms พร้อม timeout สั้นและ retry จำกัดจำนวนครั้ง เพื่อไม่ให้การ publish ทำให้ p95 เกิน 500 ms ถ้า publish ไม่สำเร็จให้ log เป็น error พร้อม metric/alert และ **ไม่ทำให้ order ล้ม**
- ใช้ fanout/topic exchange โดย consumer แต่ละตัวมี durable queue แยกกัน (email, loyalty, analytics) เพื่อให้ email backlog ไม่ไปบล็อก loyalty ที่มีเวลาจำกัด 1 นาที
- **Email consumer** (`emailservice` ที่เปลี่ยนไปเรียก MailFast) ใช้ rate limiter ตั้งต่ำกว่า 50/s เล็กน้อย (ค่าเริ่มต้นเสนอ 45/s เพื่อเผื่อ headroom ปรับตามผลทดสอบ) retry พร้อม exponential backoff สำหรับความล้มเหลวประมาณ 0.5 % และส่งเข้า dead-letter queue เมื่อเกินจำนวนครั้ง
- **Loyalty consumer** ใช้ `order_id` เป็น idempotency key (unique constraint ใน data store ของ `loyaltyservice`) เพื่อแปลง at-least-once เป็น exactly-once effect
- **Analytics consumer** ยอมรับ at-least-once และ dedupe ด้วย `order_id`
- Action อนาคต (SMS, recommendations update) เพิ่มเป็น consumer และ queue ใหม่ **ยกเว้น fraud scoring** ที่ถ้าต้องตัดสินก่อนยืนยัน/ชำระเงินจะเป็น synchronous call แยกต่างหากและอยู่นอกขอบเขตของ ADR นี้

เหตุผล: เป็นตัวเลือกเดียวที่ตอบ driver อันดับต้นพร้อมกัน คือ availability, latency, durability ของ loyalty และ modifiability โดยไม่ขัด policy self-hosted และใช้ประสบการณ์ RabbitMQ ที่ทีมมี Kafka ถูกตัดออกเพราะเกินความต้องการและทีมไม่มีประสบการณ์ ขณะที่ Option 2 ไม่ทน crash

## Consequences

- **Positive:** latency ของ `PlaceOrder` ไม่ขึ้นกับ MailFast จึงรักษา p95 ≤ 500 ms ได้ (performance)
- **Positive:** email, loyalty หรือ analytics ล่มไม่กระทบ order เพราะ message รอใน queue (availability)
- **Positive:** เพิ่ม consumer ใหม่โดยไม่แก้ `checkoutservice` (modifiability)
- **Positive:** ใช้ open-source บน Kubernetes เดิม (cost, portability)
- **Negative:** อีเมลช้ากว่าเดิมอย่างมีนัยสำคัญช่วง peak เพราะรับเข้า 200/s ส่งได้ไม่เกิน 50/s (backlog ราว 150/s) (performance ที่ผู้ใช้รับรู้)
- **Negative:** RabbitMQ เป็น stateful component เพิ่ม ต้องทำ HA และ monitoring (availability, operability)
- **Negative:** ต้องเขียน idempotency, retry และ DLQ handling เอง debug ข้าม service ยากขึ้น (modifiability, testability)
- **Risks and mitigations:**
  - *Dual-write:* ถ้า publish ล้มหลังตัดเงินแล้ว event หาย คะแนนและอีเมลจะหาย ลดด้วย publisher confirms + retry; ทางเลือกที่แข็งกว่าคือ transactional outbox แต่ต้องมี data store ฝั่ง checkout (Not evidenced) จึงเลื่อนเป็น Open question และเสนอ reconciliation job เทียบ order กับแต้มที่ได้รับ
  - *Broker ล่ม:* ใช้ quorum queues หลาย replica และ alert; checkout ต้องไม่ล้มเมื่อ publish ไม่ได้
  - *Backlog อีเมล:* alert บน queue depth และตกลง SLA ของอีเมลกับ product owner; ประเมินใช้ MailFast bulk API หากมี (ไม่ทราบ)
  - *PII:* event มีอีเมลลูกค้า ต้องเข้ารหัส/ยืนยันตัวตนการเชื่อมต่อ broker และจำกัดสิทธิ์ queue

## Validation

- Load test ที่ 200 orders/s (Locust `loadgenerator`) โดยจำลอง MailFast p95 1.5 s / p99 8 s / fail 0.5 % / limit 50/s และตรวจว่า p95 `PlaceOrder` ≤ 500 ms
- Fault injection: ปิด MailFast, `loyaltyservice`, analytics และ RabbitMQ ทีละตัว ตรวจว่า order ยังสำเร็จ
- ส่ง message ซ้ำเพื่อตรวจว่าไม่มีแต้มซ้ำ และแต้มปรากฏภายใน 1 นาทีที่ peak
- วัด email send rate (ต้องไม่เกิน 50/s), queue depth และเวลาระบาย backlog เทียบกับ SLA ที่ตกลง
- ทบทวน ADR นี้หลังเพิ่ม consumer ตัวที่ 4 หรือหลัง sale 11.11 ครั้งแรก

## Open questions

- peak 200 orders/s คงอยู่นานเท่าไร และ SLA ความล่าช้าของอีเมลที่ยอมรับได้ (กำหนดว่า backlog รับได้หรือต้องหา bulk API/ผู้ให้บริการเสริม)
- `checkoutservice` มี data store ให้ทำ transactional outbox ได้หรือไม่ (Not evidenced); ถ้าไม่มี ยอมรับความเสี่ยง event หายหรือเพิ่ม store
- data store และกติกาคิดแต้มของ `loyaltyservice`
- SLA ของ analytics (ยอมให้สูญหายได้ไหม) และ schema ของ event
- fraud scoring ต้องทำก่อนหรือหลังชำระเงิน
- ใครรับผิดชอบ HA/monitoring ของ RabbitMQ และข้อกำหนด PII
- นโยบายเมื่อ MailFast ล้มถาวรหลัง retry
