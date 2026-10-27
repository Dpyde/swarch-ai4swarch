Agent: Claude Sonnet 5.5 (claude-sonnet-5-5) via Claude Code subagent

# Review ของ ADR-0002: ใช้ RabbitMQ ส่ง event "OrderPlaced"

ADR ที่รีวิว: `docs/adr/0002-use-rabbitmq-for-post-order-events.md` (Status: Proposed) หลักฐานที่ใช้: ADR และ `docs/scenario.md` เท่านั้น การอ้างบรรทัดของ `checkoutservice/main.go` ใน ADR ตรวจสอบไม่ได้ในรอบนี้ (ไม่ได้รับอนุญาตให้เปิดไฟล์นั้น)

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| S1 Performance | ลูกค้า | เรียก `PlaceOrder` | ช่วง 11.11 ที่ 200 orders/s, MailFast ช้า (p95 1.5 s, p99 8 s) | ตัดเงิน, ส่งของ, publish `OrderPlaced`, ตอบทันที | p95 ของ `PlaceOrder` ≤ 500 ms (ปัจจุบัน ~300 ms) |
| S2 Availability | ความล้มเหลวของระบบ | email, `loyaltyservice`, analytics หรือ RabbitMQ ล่ม | ช่วงปกติและ peak | order สำเร็จ, event รอใน queue หรือถูกบันทึกเพื่อกู้คืน | order ที่ล้มเพราะ post-order = 0 |
| S3 Reliability | ลูกค้าที่สั่งซื้อ | order สำเร็จ ขณะ message ถูกส่งซ้ำและ email backlog สูง | peak 200 orders/s | `loyaltyservice` ให้แต้ม | แต้มภายใน 1 นาที, exactly once (0 แต้มซ้ำ) |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | Publish แล้วตอบทันที ตัด MailFast ออกจาก critical path ของ `PlaceOrder` | MailFast p95 1.5 s เกินงบ 500 ms อยู่แล้ว การแยกทำให้ S1 เป็นไปได้ |
| Non-risk | Durable queue แยกต่อ consumer (email, loyalty, analytics) | email backlog ไม่บล็อก loyalty ที่มีเวลา 1 นาที (S3) และเพิ่ม consumer ได้โดยไม่แก้ producer |
| Non-risk | Loyalty ใช้ `order_id` เป็น idempotency key | แปลง at-least-once เป็น exactly-once effect ตรงกับข้อกำหนด "exactly once" |
| Risk | Dual-write: crash ระหว่าง `shipOrder` กับ publish, หรือ publish ล้มหลังเงินถูกตัด event หายถาวร publisher confirms + retry ไม่ครอบคลุมกรณี pod crash | ขัดกับ "loyalty ภายใน 1 นาที exactly once" (S3) ADR เสนอแค่ reconciliation job ที่ไม่มี SLA เวลา และเลื่อน outbox เป็น Open question |
| Risk | Email consumer ไม่มี idempotency (ระบุแค่ loyalty และ analytics) เมื่อ ack หายหรือ redeliver หลัง MailFast รับแล้ว | ลูกค้าได้อีเมลซ้ำ และ retry ใช้โควต้า 50/s ร่วมกัน |
| Risk | Rate limiter 45/s ต้องเป็น limit ระดับ global ถ้า email consumer มีหลาย pod แล้วแต่ละ pod จำกัดเอง | เกิน MailFast limit 50/s ได้ ADR ไม่ระบุวิธี |
| Risk | อีเมลช้าแบบรุนแรง: ที่ 45/s backlog ~155/s ถ้า peak 10 นาที ฉบับสุดท้ายรอราว 33 นาที ADR ใช้เลข "10 นาที" แต่ก็บอกว่ายังไม่ทราบระยะเวลา peak | ลูกค้าไม่ได้ confirmation นานมาก ถ้า peak ยาวกว่านี้ backlog และเวลาระบายโตเชิงเส้น ADR ยอมรับเป็น Negative แต่ไม่มี SLA/ทางออก |
| Sensitivity point | ระยะเวลา peak และอัตรา email limit: backlog = (200 − 45) × ระยะเวลา peak | เป็นตัวกำหนดความลึกของ queue ขนาด disk/memory และเวลาระบาย |
| Sensitivity point | Publisher-confirm timeout × จำนวน retry | ถ้า broker ช้าหรือล่ม ค่านี้บวกเข้า latency ของ `PlaceOrder` โดยตรง และอาจทำให้ p95 เกิน 500 ms (S1, S2) ADR ไม่ระบุค่า |
| Trade-off point | Eventual consistency: latency และ availability ดีขึ้น แต่ email/แต้มไม่ทันที และ debug ข้าม service ยากขึ้น | performance/availability ได้ ขณะที่ user-perceived timeliness และ testability เสีย |
| Trade-off point | RabbitMQ เพิ่ม stateful component ที่ 4 dev ต้อง operate เอง (HA, quorum queues) | modifiability ดีขึ้น แต่ operability และจุดล้มเหลวเพิ่ม ขณะที่ S2 ระบุว่า RabbitMQ ล่มห้ามกระทบ order |

## 3. Missing or weak

- **Options ที่ไม่ได้พิจารณา:** transactional outbox (ควรเป็นตัวเลือกหลักเทียบกับ publish ตรง ไม่ใช่แค่ Open question), broker เบา ๆ อื่นที่ self-hosted ได้ เช่น NATS/Redis Streams, และการเก็บ job ใน queue ภายใน `loyaltyservice`/`emailservice` เอง (ผ่าน gRPC แบบ async)
- **คำอ้างเกินจริง:** "เป็นตัวเลือกเดียวที่ตอบ driver อันดับต้นพร้อมกัน" ไม่ถูกต้อง Kafka ก็ตอบ availability, latency และ modifiability ได้ ถูกตัดออกด้วยเหตุผลเรื่องทักษะทีมและความซับซ้อน ซึ่งเป็นคนละเรื่อง ส่วน "Kafka หนักเกินความจำเป็น" ไม่มีตัวเลขสนับสนุน
- **ผลที่ไม่ได้ระบุ:** ค่า timeout และ retry ของการ publish, การวัดว่า publisher confirm บน quorum queue เพิ่ม latency เท่าไร (ไม่มีหลักฐาน [Inferred] ว่าน้อย), ผลของ fraud scoring แบบ sync ต่องบ 500 ms, schema versioning ของ event, นโยบาย MailFast ล่มถาวร (อยู่ใน Open question เท่านั้น)
- **การอ้างโค้ด:** ตัวเลขบรรทัดของ `checkoutservice/main.go` ยังไม่ได้ตรวจสอบกับไฟล์จริง ผู้อ่านควรยืนยัน ส่วน data store ของ `checkoutservice` ADR ระบุถูกต้องว่า Not evidenced

## 4. Verdict

**Accept with changes.** ทิศทางการแยกด้วย broker ถูกต้องและสอดคล้องกับ scenario แต่ยังรับประกัน S3 ไม่ได้ทุกกรณี

สามการเปลี่ยนแปลงสำคัญที่สุด:

1. **ปิดช่องว่าง dual-write:** ตัดสินใจเรื่อง transactional outbox (หรือ store ที่ใช้แทน) ก่อน Accepted และกำหนด SLA เวลาของ reconciliation มิฉะนั้นต้องแก้ข้อความ "exactly once ภายใน 1 นาที" ให้ตรงความจริง พร้อมกำหนดค่า timeout × retry ของการ publish ให้อยู่ในงบ 500 ms
2. **ทำให้ email path ปลอดภัยจริง:** ใช้ rate limiter แบบ global, ทำ email ให้ idempotent (กันส่งซ้ำ), และตกลง SLA ความล่าช้าของอีเมลกับ product owner บนตัวเลขระยะเวลา peak ที่ยืนยันแล้ว
3. **แก้เหตุผลการเลือกและเพิ่มตัวเลือก:** เพิ่ม outbox และ broker ทางเลือกอื่นใน Considered options และลบคำอ้างว่า RabbitMQ "เป็นตัวเลือกเดียว" โดยให้เหตุผลด้วยทักษะทีมและ operability แทน
