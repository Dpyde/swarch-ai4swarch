**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| frontend → cartservice | `frontend.yaml` (`CART_SERVICE_ADDR: cartservice:7070`) | ✅ | พบ env นี้ใน `frontend.yaml` (บรรทัด 73) และ `CartService` อยู่ใน `demo.proto`; `src/frontend/main.go` มี `mustConnGRPC(… cartSvcConn …)` |
| checkoutservice → emailservice | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR: emailservice:5000`) | ✅ | พบ env ที่บรรทัด 63; `src/checkoutservice/main.go` มี `mustMapEnv(&svc.emailSvcAddr, "EMAIL_SERVICE_ADDR")` และ `mustConnGRPC(… emailSvcConn …)`; `sendOrderConfirmation` เรียก `SendOrderConfirmation` จริง |
| checkoutservice → paymentservice | `checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR: paymentservice:50051`) | ✅ | พบ env ที่บรรทัด 61 และ `mustConnGRPC(… paymentSvcConn …)` ใน `main.go` |
| cartservice → redis-cart | `cartservice.yaml` (`REDIS_ADDR: redis-cart:6379`) | ✅ | พบ `value: "redis-cart:6379"` ที่บรรทัด 51; Deployment/Service `redis-cart` อยู่ในไฟล์เดียวกัน (บรรทัด 92, 146) และใช้ `emptyDir` (บรรทัด 141) |
| recommendationservice → productcatalogservice | `recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR`) | ✅ | พบ env ที่บรรทัด 60 (ผู้เรียกคือ recommendationservice ไม่ใช่ frontend) |
| frontend → shoppingassistantservice | (AI ไม่ได้วาด ระบุใน Gaps) | ❓ | `frontend.yaml` บรรทัด 83 มี `SHOPPING_ASSISTANT_SERVICE_ADDR` และ `src/frontend/main.go` บรรทัด 139 map env นี้ แต่ไม่มี `mustConnGRPC` สำหรับมัน (บรรทัด 141-147 มีเจ็ดตัวที่เหลือ) และไม่มีใน proto จึงไม่ควรวาดเป็น gRPC dependency AI ตัดสินใจถูก |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): ไม่พบ dependency ที่ผิดในรายการที่ตรวจ และจำนวน gRPC connection ของ `checkoutservice` (6) และ `frontend` (7) ใน `main.go` ตรงกับตาราง สิ่งที่ผมตั้งคำถามคือ (1) `loadgenerator` ถูกวาดใน System_Boundary เทียบเท่า container ธุรกิจ ทั้งที่ `kustomization.yaml` คอมเมนต์ออกและเป็นเครื่องมือทดสอบ (AI บอกในตารางแต่แผนภาพไม่แสดงความต่าง) (2) `redis-cart` ใช้ `emptyDir` จึงไม่ durable ซึ่งแผนภาพไม่บอก (3) `shoppingassistantservice` ไม่อยู่ในแผนภาพ แต่ AI แจ้งใน Gaps อย่างถูกต้อง (4) `checkoutservice` และ `frontend` ต่อ OpenTelemetry collector (`collectorConn` ใน `main.go`) ซึ่งไม่ปรากฏใน manifest ที่อ่าน

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

**คำตอบของ AI (สรุปจากผลลัพธ์):** Run A ได้ 9 container และ "None evidenced" สำหรับ dependency AI ระบุว่า proto ไม่แสดงผู้เรียก, data store, frontend หรือระบบภายนอก

**ความเห็นของนิสิต:** `.proto` บอกแค่ว่าแต่ละ service ให้อะไร (interface) ไม่ได้บอกว่าใครเรียกใคร เพราะการเรียกอยู่ในโค้ดและ env ตอน deploy ส่วนที่ A ไม่รู้เลยคือ redis-cart (ไม่มีใน proto), frontend ซึ่งเป็นทางเข้าของผู้ใช้, ผู้ใช้ และ loadgenerator แผนภาพ A จึงเป็นแค่รายการ service ส่วน B ที่เพิ่ม manifest ได้ dependency ครบ 17 เส้น แสดงว่าคุณภาพผลลัพธ์ขึ้นกับหลักฐานที่ให้มากกว่าตัวโมเดล
2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

**คำตอบของ AI:** AI ไม่ได้ตอบข้อนี้ (ให้นิสิตตัดสินเอง) ใน Gaps มันระบุแต่ว่าไม่พบ `shoppingassistantservice`

**ความเห็นของนิสิต:** ข้อผิดพลาดที่ทำให้เข้าใจผิดมากที่สุดคือ `loadgenerator` อยู่ใน System_Boundary ระดับเดียวกับ service จริง สมาชิกใหม่จะคิดว่าเป็นส่วนหนึ่งของระบบ production และอีกข้อคือไม่รู้ว่า `redis-cart` ใช้ `emptyDir` (ข้อมูลหายเมื่อ pod restart) ซึ่งสำคัญกับการตัดสินใจเรื่อง durability ส่วนเรื่อง shopping assistant ที่ไม่ได้วาด อันตรายน้อยกว่าเพราะ AI บอกไว้ใน Gaps แล้ว
