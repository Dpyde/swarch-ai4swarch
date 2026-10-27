Agent: Claude Sonnet 5.5 (claude-sonnet-5-5) via Claude Code subagent

# ADR-0002: Publish an "OrderPlaced" event to a message broker for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-26
- **Deciders:** Not given (see Open questions)

## Context and problem

หลังลูกค้าสั่งซื้อสำเร็จ Online Boutique ต้องทำ 3 อย่างใหม่ตาม requirement:

1. ส่งอีเมลยืนยันคำสั่งซื้อผ่าน MailFast (external email provider ตัวใหม่) แทน logic mock ของ `emailservice`
2. ให้คะแนนสะสม (loyalty points) ผ่าน `loyaltyservice` ซึ่งยังไม่มี (ต้องสร้างใหม่)
3. บันทึก analytics event "order placed" ให้ทีม marketing

นอกจากนี้ marketing คาดว่าปีหน้าจะมี post-order action เพิ่ม (SMS, fraud scoring, การอัปเดต recommendations)

ปัญหาที่ต้องตัดสินใจ: ใครเป็นผู้เรียก/แจ้ง 3 action นี้ (และ action อนาคต) หลังสั่งซื้อ และด้วยรูปแบบการสื่อสารแบบใด (synchronous หรือ asynchronous) โดยที่ action ใหม่ไม่ควรทำให้ flow การสั่งซื้อช้าลงหรือล้มเหลว และการเพิ่ม action ถัดไปควรทำได้ง่าย

หมายเหตุ: ADR นี้เขียนจาก requirement เท่านั้น ไม่ได้อ่านโค้ดหรือ manifest ของระบบ ดังนั้นโครงสร้างปัจจุบันของ checkout flow (ใครเรียก `emailservice` อย่างไร) **Not evidenced in the provided files** และไม่ถูกสมมติเป็นข้อเท็จจริงในที่นี้ ตัวเลข SLA, traffic และงบประมาณก็ไม่ได้ให้มา (ดู Open questions)

## Decision drivers

เรียงตามความสำคัญ (ลำดับเป็นข้อเสนอ ต้องยืนยันกับผู้มีส่วนได้ส่วนเสีย):

- **Modifiability:** เพิ่ม post-order action ใหม่ (SMS, fraud scoring, recommendations update) โดยแก้ส่วนที่ผู้สั่งซื้อ (order flow) น้อยที่สุด
- **Availability / fault isolation:** MailFast ล่ม, `loyaltyservice` ล่ม หรือ analytics ช้า ต้องไม่ทำให้การสั่งซื้อล้มเหลว
- **Performance (latency ของการสั่งซื้อ):** ไม่ควรเพิ่ม latency ที่ผู้ใช้เห็นตามจำนวน action
- **Reliability ของ action (ไม่ทำหาย/ไม่ทำซ้ำ):** อีเมลและคะแนนต้องไม่หาย และไม่ถูกให้ซ้ำ
- **Cost / operational complexity:** ทีมต้องดูแลองค์ประกอบเพิ่ม (ขนาดทีมและทักษะ: ไม่ทราบ)
- **Constraint:** ต้องใช้ MailFast (external) และสร้าง `loyaltyservice` ใหม่

## Considered options

1. **Synchronous direct calls:** ผู้ประมวลผลคำสั่งซื้อเรียก MailFast, `loyaltyservice` และระบบ analytics ตรง ๆ ทีละตัว (เพิ่มการเรียกใหม่ทุกครั้งที่มี action ใหม่)
2. **Event-driven ผ่าน message broker:** ผู้ประมวลผลคำสั่งซื้อ publish event `OrderPlaced` หนึ่งครั้ง แล้ว consumer แต่ละตัว (email adapter สำหรับ MailFast, `loyaltyservice`, analytics) subscribe และทำงานของตนเอง
3. **Post-order orchestrator service:** สร้าง service กลางตัวเดียวรับ `OrderPlaced` (เรียกแบบ synchronous หรือ async) แล้วเป็นผู้เรียก action ต่าง ๆ แบบ fan-out ตามลำดับ/เงื่อนไข

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous direct calls | เรียบง่าย เข้าใจง่าย ไม่ต้องมีโครงสร้างพื้นฐานใหม่; ได้ผลลัพธ์/error ทันที | latency ของการสั่งซื้อเพิ่มตามจำนวน action; ความล้มเหลวของ MailFast/loyalty/analytics อาจลาม (cascade) ไปถึงการสั่งซื้อ; ทุก action ใหม่ต้องแก้โค้ดฝั่ง order flow (modifiability ต่ำ) |
| 2. Event-driven ผ่าน message broker | แยก producer ออกจาก consumer (loose coupling); เพิ่ม consumer ใหม่ (SMS, fraud, recommendations) โดยไม่แก้ producer; fault isolation ดี เพราะ broker พักข้อความและ retry ได้; ไม่เพิ่ม latency ต่อ action | ต้องเพิ่มและดูแล broker (cost/ops); eventual consistency (คะแนนหรืออีเมลมาช้ากว่าผลสั่งซื้อ); ต้องออกแบบ idempotency/retry/dead-letter; debug flow ข้าม service ยากขึ้น; fraud scoring อาจต้องการผลก่อนยืนยันคำสั่งซื้อ ซึ่งแบบ async ไม่ตอบ |
| 3. Orchestrator service | จุดเดียวที่รู้ลำดับและเงื่อนไขของ post-order action; ควบคุม retry/ลำดับได้ชัด; producer เรียกครั้งเดียว | เพิ่ม service ใหม่ที่เป็นจุดรวมศูนย์ (อาจเป็น single point of failure/คอขวด); ยังต้องแก้ orchestrator ทุกครั้งที่เพิ่ม action (modifiability ปานกลาง); ถ้าเรียกแบบ synchronous ปัญหา latency/cascade ยังอยู่ |

## Decision

เสนอเลือก **Option 2: publish event `OrderPlaced` ผ่าน message broker** แล้วให้แต่ละ action เป็น consumer อิสระ:

- Email: consumer/adapter ที่เรียก MailFast (แทน logic mock ของ `emailservice`)
- Loyalty: `loyaltyservice` ใหม่ subscribe `OrderPlaced` และให้คะแนน
- Analytics: consumer บันทึก "order placed" event ให้ marketing

เหตุผล: ตอบ driver อันดับต้น ๆ ได้ดีที่สุด คือ modifiability (SMS, recommendations update เพิ่มเป็น consumer ใหม่โดยไม่แตะ producer) และ availability/fault isolation (ความล้มเหลวของ MailFast หรือ `loyaltyservice` ไม่ทำให้การสั่งซื้อล้ม) โดยไม่เพิ่ม latency ต่อ action; ข้อแลกเปลี่ยนคือ complexity และ eventual consistency ที่ต้องยอมรับ

ข้อยกเว้นที่ต้องพิจารณาแยก: **fraud scoring** หากต้องตัดสินก่อนยืนยันคำสั่งซื้อ จะไม่เหมาะกับ pattern นี้ และควรเป็น synchronous check ใน order flow (ตัดสินใจใน ADR แยกเมื่อมี requirement จริง)

## Consequences

- **Positive:** เพิ่ม post-order action ใหม่ได้โดยไม่แก้ producer (quality attribute: modifiability)
- **Positive:** MailFast/`loyaltyservice`/analytics ล่มหรือช้า ไม่กระทบการสั่งซื้อ และ event ถูกพักไว้ประมวลผลภายหลังได้ (quality attribute: availability)
- **Positive:** latency ของการสั่งซื้อไม่เพิ่มตามจำนวน action (quality attribute: performance)
- **Negative:** ต้องมี message broker เพิ่ม ต้อง deploy/monitor/ดูแล (quality attribute: cost, operability)
- **Negative:** eventual consistency ผู้ใช้อาจเห็นคะแนนหรือรับอีเมลช้ากว่าผลสั่งซื้อ (quality attribute: consistency / user-perceived latency ของผลลัพธ์)
- **Negative:** การ trace และ debug ข้าม service ยากขึ้น (quality attribute: testability / observability)
- **Risks and mitigations:**
  - อีเมลหรือคะแนนซ้ำเมื่อ retry: ทำ consumer ให้ idempotent โดยใช้ order ID เป็น key (reliability)
  - ข้อความหายหรือ consumer ล้มต่อเนื่อง: ใช้ retry พร้อม dead-letter queue และ alert (availability)
  - Broker เองล่ม: ผลต่อการสั่งซื้อขึ้นกับว่า publish ล้มเหลวแล้วทำอย่างไร; ต้องกำหนดนโยบาย (เช่น ไม่ block คำสั่งซื้อ) และยืนยันกับผู้เกี่ยวข้อง (availability)
  - MailFast เป็น external dependency: จำกัดการเรียกด้วย timeout/retry ใน email consumer และเก็บ MailFast ไว้หลัง adapter เพื่อเปลี่ยนผู้ให้บริการได้ (modifiability)

## Validation

- ทดสอบแบบ fault injection: ปิด MailFast และ `loyaltyservice` แล้วตรวจว่าการสั่งซื้อยังสำเร็จและ event ถูกประมวลผลหลังระบบกลับมา
- วัด latency ของ flow การสั่งซื้อก่อน/หลังเพิ่ม publish (เกณฑ์ยอมรับ: ต้องกำหนด ดู Open questions)
- ตรวจ idempotency: ส่ง `OrderPlaced` ซ้ำ ต้องไม่เกิดอีเมลหรือคะแนนซ้ำ
- วัด end-to-end delay จาก publish ถึงอีเมล/คะแนน/analytics และอัตรา dead-letter
- พิสูจน์ความสามารถในการขยาย: เพิ่ม consumer ตัวอย่าง (เช่น SMS) โดยไม่แก้ producer
- ทบทวน ADR นี้เมื่อมี requirement ของ fraud scoring จริง หรือภายในวันที่ที่ทีมกำหนด (ยังไม่ระบุ)

## Open questions

- **Deciders:** ใครเป็นผู้ตัดสินใจ/อนุมัติ ADR นี้
- **ตัวเลขเชิงคุณภาพ:** latency เป้าหมายของการสั่งซื้อ, traffic/จำนวนคำสั่งซื้อต่อวินาที, ความล่าช้าสูงสุดที่ยอมรับได้ของอีเมล/คะแนน/analytics
- **ความเสียหายที่ยอมรับได้:** อีเมลหรือคะแนนหายหรือซ้ำได้ไหม; analytics event ยอมให้สูญหายบ้างหรือไม่
- **Broker:** ระบบมี message broker อยู่แล้วหรือไม่ และทีมมีทักษะ/งบประมาณสำหรับดูแลหรือไม่ (ผลิตภัณฑ์ใดเป็นทางเลือกไม่ได้ระบุ)
- **สถานะ checkout flow ปัจจุบัน:** ผู้เรียก `emailservice` ปัจจุบันคือใคร และเรียกอย่างไร (ไม่ได้อ่านโค้ดเพราะ context เป็น none)
- **MailFast:** สัญญา/rate limit/SLA/ค่าใช้จ่าย และข้อกำหนดด้านข้อมูลส่วนบุคคลของลูกค้า
- **Loyalty:** กฎการให้คะแนน, ที่เก็บข้อมูล (data store) ของ `loyaltyservice`, และ owner ของ service
- **Analytics:** ปลายทางของ event (ระบบใดที่ marketing ใช้) และรูปแบบ/schema ของ event
- **Fraud scoring:** ต้องตัดสินก่อนยืนยันคำสั่งซื้อ (synchronous) หรือหลังสั่งซื้อ (async)
- **นโยบายเมื่อ broker publish ล้มเหลว:** ให้ block คำสั่งซื้อหรือไม่
