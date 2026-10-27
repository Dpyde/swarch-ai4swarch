Agent: Claude Sonnet 5.5 (claude-sonnet-5-5) via Claude Code subagent

# ADR-0002: Use synchronous gRPC with one shared demo.proto for all service-to-service communication

- **Status:** Reconstructed
- **Date:** Not evidenced in the provided files (ย้อนรอยเมื่อ 2026-10-26)
- **Deciders:** Not evidenced in the provided files (ไฟล์ลิขสิทธิ์ระบุเพียง Google LLC)

## Context and problem

Online Boutique แยกเป็นหลาย service (cart, recommendation, product catalog, shipping, currency, payment, email, checkout, ad) ที่ต้องเรียกกัน คำถามคือ service เหล่านี้ควรสื่อสารกันแบบใดและนิยาม contract ไว้ที่ใด

สิ่งที่หลักฐานแสดง:

- `system/online-boutique/protos/demo.proto` ไฟล์เดียว (`package hipstershop`) นิยามทั้ง 9 `service` และ message ทั้งหมด (เช่น `CartService`, `CheckoutService`, `PaymentService`) และ message ร่วมอย่าง `Money`, `Address`, `CartItem` ถูกใช้ข้าม service
- `system/online-boutique/src/checkoutservice/main.go` (`PlaceOrder`) เรียก cart, product catalog, currency, shipping, payment และ email ผ่าน gRPC client แบบ blocking ตามลำดับ และรอผลก่อนไปขั้นถัดไป
- `system/online-boutique/src/frontend/main.go` เปิด gRPC connection (`mustConnGRPC`) ไปยัง currency, product catalog, cart, recommendation, shipping, checkout และ ad โดยตัวมันเองรับ HTTP จากผู้ใช้ (`http.ListenAndServe`, `mux.NewRouter`)
- ทั้งสอง service ใช้ `insecure.NewCredentials()` และที่อยู่ปลายทางมาจาก environment variable (เช่น `CART_SERVICE_ADDR`)

## Decision drivers

[Inferred] เรียงตามความสำคัญ ไม่มีไฟล์ใดระบุ driver ตรง ๆ

- Modifiability / interoperability: contract ชัดเจนและใช้ร่วมกันได้
- Simplicity ของ flow ที่ต้องได้ผลทันที (คำนวณราคา, ตัดบัตร)
- Performance (latency) ของ call ภายใน
- Observability: trace ต่อเนื่องข้าม service
- ความง่ายในการ demo/เรียนรู้ระบบ

## Considered options

1. Synchronous gRPC ระหว่าง service ทั้งหมด โดยใช้ `demo.proto` ไฟล์เดียว (ตัวเลือกที่ถูกเลือกตามหลักฐาน)
2. Synchronous REST/JSON over HTTP พร้อม contract แยกต่อ service (เช่น OpenAPI)
3. Asynchronous messaging ผ่าน message broker (event-driven) สำหรับบาง flow เช่น ส่งอีเมล/สั่งจัดส่ง
4. gRPC แต่แยกไฟล์ `.proto` ต่อ service

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. gRPC sync + `demo.proto` เดียว | contract แบบ typed สร้าง client/stub จากไฟล์เดียว; message ร่วม (`Money`, `Address`) นิยามที่เดียว; flow เขียนตามลำดับอ่านง่าย (`PlaceOrder`); มี hook สำหรับ tracing (`otelgrpc`) | service ผูกกันตอน runtime (checkout เรียก 6 service); ถ้าปลายทางล่ม request ล้มทันที; ไฟล์ proto เดียวเป็นจุดเปลี่ยนร่วมของทุก service |
| 2. REST/JSON sync | เครื่องมือแพร่หลาย debug ง่าย | ไม่มี schema ร่วมโดยปริยาย; overhead ของ JSON; ต้องดูแล contract หลายชุด |
| 3. Async broker | ลดการผูกกันตอน runtime; ทน downstream ล่มชั่วคราว | เพิ่ม infrastructure และความซับซ้อน; ผลลัพธ์ไม่ทันที (ไม่เหมาะกับการคิดราคา/ตัดบัตรที่ต้องรู้ผลเลย); ไม่พบ broker ในหลักฐาน |
| 4. gRPC + proto แยก | ขอบเขตการเปลี่ยนแปลงเล็กลงต่อ service; ลด coupling ของ contract | message ร่วมซ้ำหรือต้อง import ข้ามไฟล์; จัดการ version หลายไฟล์ |

## Decision

เลือก synchronous gRPC ระหว่าง service โดยนิยาม contract ทั้งหมดใน `demo.proto` ไฟล์เดียว

เหตุผล (ส่วนใหญ่ **[Inferred]**): ขั้นตอน checkout ต้องการผลลัพธ์ก่อนไปต่อ (ราคา → ตัดบัตร → จัดส่ง) ซึ่งสอดคล้องกับการเรียกแบบ blocking ใน `PlaceOrder`; การใช้ proto เดียวทำให้ทุก service ใช้ type เดียวกันสำหรับ `Money`/`Address`/`CartItem` และเป็นระบบตัวอย่างที่ต้องเข้าใจง่าย

## Consequences

- **Positive:** contract เป็น typed และสม่ำเสมอ (modifiability/interoperability); flow ตรงไปตรงมา วิเคราะห์ง่าย (simplicity); trace context ส่งต่อข้าม call ได้ผ่าน `otelgrpc` (observability)
- **Negative:** `checkoutservice` พึ่งพา 6 service แบบ synchronous จึงเป็นจุดที่ availability ของ checkout ขึ้นกับ service เหล่านั้น; latency ของ `PlaceOrder` เป็นผลรวมของ call ตามลำดับ (performance); การแก้ `demo.proto` กระทบทุก service ที่ generate code จากมัน (modifiability)
- **Risks and mitigations:** ความล้มเหลวลามข้าม service — หลักฐานแสดงการลดผลกระทบบางส่วนเท่านั้น: ความล้มเหลวของ email ถูกแค่ log (`log.Warnf`) และผล `emptyUserCart` ถูกเมินด้วย `_ =` ส่วนการตัดบัตร/จัดส่งที่ล้มจะคืน error; ไม่พบ retry, timeout ของ call ธุรกิจ หรือ circuit breaker ในไฟล์ที่อ่าน (`convertCurrency` ใช้ `context.TODO()`) ข้อเสนอ: เพิ่ม deadline/retry ต่อ call และพิจารณา async สำหรับ email

## Validation

ไม่พบวิธีตรวจสอบในไฟล์หลักฐาน ข้อเสนอ **[Inferred]**: load test ของ `PlaceOrder` วัด latency/ error rate เมื่อหนึ่ง service ปลายทางช้าหรือล่ม และทบทวน ADR เมื่อต้องเพิ่ม flow ที่ไม่ต้องการผลทันที

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | `demo.proto` ไฟล์เดียวนิยาม service ทั้ง 9 ตัว | Evidenced | `system/online-boutique/protos/demo.proto` (`service CartService` ... `service AdService`) |
| 2 | message ร่วม (`Money`, `Address`, `CartItem`) ถูกใช้ข้าม service | Evidenced | `demo.proto` (`Money` ใน `Product`, `ChargeRequest`; `Address` ใน `GetQuoteRequest`, `PlaceOrderRequest`) |
| 3 | checkout เรียก shipping, product catalog, cart, currency, email, payment ผ่าน gRPC | Evidenced | `checkoutservice/main.go` (`mustConnGRPC` ทั้ง 6, `pb.New*Client`) |
| 4 | การเรียกใน `PlaceOrder` เป็น blocking ตามลำดับ | Evidenced | `checkoutservice/main.go` (`PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`) |
| 5 | frontend เรียก backend ผ่าน gRPC และรับ HTTP จากผู้ใช้ | Evidenced | `frontend/main.go` (`mustConnGRPC`, `http.ListenAndServe`, `mux.NewRouter`) |
| 6 | frontend มี `SHOPPING_ASSISTANT_SERVICE_ADDR` แต่ไม่เปิด gRPC conn และไม่อยู่ใน `demo.proto` จึงไม่ใช่ "ทุก service ใช้ gRPC/demo.proto" ตามหลักฐานนี้ | Evidenced | `frontend/main.go` (`shoppingAssistantSvcAddr`, ไม่มี conn); ไม่มีใน `demo.proto` (โปรโตคอลของมัน: Not evidenced in the provided files) |
| 7 | ใช้ `insecure.NewCredentials()` ไม่เข้ารหัส/ยืนยันตัวตนในการเรียกภายใน | Evidenced | `checkoutservice/main.go`, `frontend/main.go` (`mustConnGRPC`) |
| 8 | ที่อยู่ปลายทางกำหนดผ่าน environment variable | Evidenced | `mustMapEnv(... "CART_SERVICE_ADDR")` ฯลฯ |
| 9 | trace context ส่งต่อข้าม gRPC | Evidenced | `otelgrpc.NewClientHandler/NewServerHandler`, `propagation.TraceContext{}` |
| 10 | ความล้มเหลวของ email ไม่ทำให้ order ล้ม และผล `emptyUserCart` ถูกเมิน | Evidenced | `checkoutservice/main.go` (`log.Warnf`, `_ = cs.emptyUserCart`) |
| 11 | ไม่พบ retry/circuit breaker/ broker ในไฟล์ที่อ่าน; `convertCurrency` ใช้ `context.TODO()` | Evidenced | `checkoutservice/main.go` (ข้อจำกัด: ขอบเขตแค่ไฟล์ที่อ่าน) |
| 12 | เหตุผลที่เลือก sync เพราะ checkout ต้องการผลลัพธ์ทันทีก่อนขั้นถัดไป | Inferred | สอดคล้องกับลำดับใน `PlaceOrder` แต่ไม่มีไฟล์ระบุเจตนา |
| 13 | เหตุผลที่ใช้ proto เดียว: type ร่วม, ง่ายต่อการ demo | Inferred | ไม่มีเอกสารเจตนาในไฟล์ที่อ่าน |
| 14 | ตัวเลือก REST/JSON, async broker, proto แยก เป็นทางเลือกที่ทีมพิจารณา | Inferred | ไม่มีหลักฐานว่าถูกพิจารณาจริง |
| 15 | Decision drivers (modifiability, performance, simplicity ฯลฯ) | Inferred | ไม่มีไฟล์ระบุ driver |
| 16 | gRPC เป็น typed/binary จึงเร็วกว่า JSON; async ลด coupling แต่ซับซ้อนกว่า | Assumption | ความรู้ทั่วไป ไม่เฉพาะระบบนี้ |
| 17 | availability ของ checkout ขึ้นกับ 6 service ที่เรียกแบบ sync และ latency เป็นผลรวมของ call | Inferred | ข้อสรุปจากโครงสร้าง call ใน `PlaceOrder`; ไม่มีการวัดจริง |
| 18 | วันที่และผู้ตัดสินใจ | Inferred | Not evidenced in the provided files |
