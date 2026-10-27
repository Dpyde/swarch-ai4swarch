**ภาษา:** ภาษาไทย | [English](scenario.md)

# สถานการณ์: สิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อ

> สถานการณ์ทางธุรกิจ**สมมติ**สำหรับงานที่ 2 ตัวเลขเป็นตัวเลขสมมติแต่สอดคล้องกันภายใน ให้ถือว่าเป็นข้อเท็จจริง
>
> หมายเหตุ: AI agent อ่านไฟล์ภาษาอังกฤษ [`scenario.md`](scenario.md) ไฟล์นี้มีไว้ให้นิสิตอ่านเท่านั้น

## ความต้องการใหม่ (R-12)

ข้อความภาษาอังกฤษสำหรับวางในข้อความถึง agent (`<paste R-12 here>`):

> After an order is placed, Online Boutique must:
> 1. send the order confirmation email through **MailFast**, a new external email provider, which replaces the mock `emailservice` logic;
> 2. award **loyalty points** through a new `loyaltyservice` (to be built);
> 3. record an **"order placed" analytics event** for the marketing team.
>
> Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).

ความหมายภาษาไทย: หลังจากลูกค้าสั่งซื้อ Online Boutique ต้อง
1. ส่งอีเมลยืนยันคำสั่งซื้อผ่าน **MailFast** ซึ่งเป็นผู้ให้บริการอีเมลภายนอกรายใหม่ แทนที่ logic จำลองใน `emailservice`
2. มอบ**แต้มสะสม (loyalty points)** ผ่าน `loyaltyservice` ใหม่ (ยังต้องสร้าง)
3. บันทึก**เหตุการณ์ analytics "มีการสั่งซื้อ"** ให้ทีมการตลาด

ทีมการตลาดคาดว่า**ปีหน้าจะมีงานหลังการสั่งซื้อเพิ่มขึ้นอีก** (SMS, การให้คะแนนความเสี่ยงการทุจริต, การอัปเดตระบบแนะนำสินค้า)

## ระบบทำงานอย่างไรในปัจจุบัน

`checkoutservice.PlaceOrder` เรียก `paymentservice`, `shippingservice` แล้วจึงเรียก `emailservice` **แบบ synchronous ผ่าน gRPC** ตามลำดับ หากการเรียกอีเมลล้มเหลว checkout จะบันทึกแค่คำเตือน (warning) และคำสั่งซื้อยังสำเร็จตามปกติ

## เป้าหมายด้านคุณภาพและข้อจำกัด

| หัวข้อ | ข้อเท็จจริง |
|---|---|
| Checkout latency | p95 ของ `PlaceOrder` ต้องไม่เกิน **500 ms** (ปัจจุบันประมาณ 300 ms) |
| ปริมาณงานช่วงพีก | **200 คำสั่งซื้อ/วินาที** ช่วงเทศกาล 11.11 |
| MailFast | p95 **1.5 วินาที**, p99 **8 วินาที**; ล้มเหลวประมาณ **0.5%** ของการเรียก; จำกัดที่ **50 อีเมล/วินาที** |
| แต้มสะสม | ต้องให้แต้ม**ภายใน 1 นาที**หลังสั่งซื้อ และ**ครั้งเดียวเท่านั้น** (ห้ามได้แต้มซ้ำ) |
| ความพร้อมใช้งาน | คำสั่งซื้อต้อง**ไม่ล้มเหลว**เพราะระบบอีเมล แต้มสะสม หรือ analytics ล่ม |
| ทีม | นักพัฒนา 4 คน ถนัด Go และ gRPC; สองคนเคยใช้ **RabbitMQ** |
| การปฏิบัติการ | ทุกอย่างรันบน Kubernetes อยู่แล้ว บริษัทเลือกใช้ component แบบ **open-source ที่ติดตั้งเอง** มากกว่าบริการ managed ที่มีเฉพาะบนคลาวด์ |
