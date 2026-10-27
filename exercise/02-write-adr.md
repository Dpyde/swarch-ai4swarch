**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ *ครบถ้วนน้อยกว่ามนุษย์* ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](../docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ได้ใช้ (ไม่ได้รับ scenario) ใส่คำถามเรื่อง latency, traffic, MailFast SLA ไว้ใน Open questions | ใช้ครบ: p95 ≤ 500 ms, 200 orders/s, MailFast p95 1.5 s / p99 8 s / fail 0.5 % / 50 emails/s, loyalty 1 นาที และคำนวณ 200/50 = 4 เท่า |
| ทางเลือกที่พิจารณา (ระบุ) | 3: sync direct calls, event-driven broker, orchestrator service | 4: sync ต่อไป, goroutine fire-and-forget, RabbitMQ, managed cloud pub/sub |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | แต้ม: consumer idempotent โดยใช้ order ID เป็น key; rate limit: ไม่ได้จัดการ เพราะไม่รู้ว่ามี (ถามใน Open questions) | แต้ม: idempotency key = `order_id`; rate limit: email consumer จำกัดไม่เกิน 50/s พร้อม retry และ DLQ แต่ยังมองข้าม backlog ที่ peak (เข้า 200/s ออก 50/s) ซึ่งผมเพิ่มเอง |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | ไม่แต่งตัวเลขหรือข้อเท็จจริงของระบบ ระบุชัดว่าไม่ได้อ่านโค้ด และเสนอ fraud scoring เป็นข้อยกเว้นแบบ sync ซึ่ง b ไม่พูดถึง | ไม่พบตัวเลขที่แต่งขึ้น เลขบรรทัดของ `main.go` ถูกเมื่อตรวจ (252-277, 379-384) มีรายละเอียดเทคนิคทั่วไป (publisher confirms, quorum queues) ที่ไม่มีใน context แต่ไม่ใช่ข้อเท็จจริงของระบบ |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | เพิ่มการคำนวณ backlog ของอีเมล: ที่ 200 orders/s เข้า แต่ MailFast ออกได้ 50/s จึงสะสมราว 150/s (ราว 90,000 ฉบับต่อ peak 10 นาที) และทำเป็น Negative consequence กับ Open question เรื่อง SLA | AI ระบุว่า "backlog เกิดขึ้นแน่" แต่ไม่ได้ประเมินขนาด ซึ่งเป็นตัวกำหนดว่า RabbitMQ พอหรือไม่ |
| 2 | ตั้ง rate limiter 45/s (เผื่อ headroom จาก 50/s), แยก queue ต่อ consumer ให้ชัดเพื่อไม่ให้ email backlog บล็อก loyalty ที่มีเวลา 1 นาที และให้ publisher confirm มี timeout/retry จำกัดเพื่อไม่ดัน p95 เกิน 500 ms | ค่าเริ่มต้นที่ตรงกับข้อกำหนด และกัน latency ที่เพิ่มจากการ publish |
| 3 | เพิ่ม Option Kafka แยกจาก managed pub/sub; ตัด fraud scoring ออกนอกขอบเขต (ถ้าต้องตัดสินก่อนชำระเงินให้เป็น sync); เพิ่ม reconciliation job สำหรับความเสี่ยง dual-write | ให้ options แตกต่างกันจริงและยอมรับข้อจำกัดของ pattern แบบ async |
| 4 | เปลี่ยน Deciders เป็นผู้เขียนและ review board แทนที่ AI เดา และเขียนที่มาของร่างไว้ใน ADR | ความโปร่งใสเรื่องที่มา |

ADR ฉบับสุดท้าย: [`docs/adr/0002-use-rabbitmq-for-post-order-events.md`](../docs/adr/0002-use-rabbitmq-for-post-order-events.md)

หมายเหตุตรงไปตรงมา: ร่างและการแก้ไขทำใน session เดียวกับที่ใช้ agent ตามที่ผมสั่งให้ "ทำให้หมด" ถ้าอาจารย์ต้องการให้ส่วนนี้เป็นผลงานของนิสิตล้วน ๆ ควรอ่านและปรับถ้อยคำเองก่อนส่ง
