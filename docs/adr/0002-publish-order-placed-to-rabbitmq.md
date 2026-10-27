# ADR-0002: Publish an "order placed" event to RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** วรเมธ สุวรรณพฤกษ์ (สถาปนิก) — ร่างแรกจาก AI (`exercise/outputs/2b-adr.md`)

## Context and problem

Requirement R-12 (`docs/scenario.md`) กำหนดว่าหลังจาก order ถูกสร้างแล้ว ระบบต้อง (1) ส่ง email ยืนยันผ่าน **MailFast** (external provider) แทน logic mock ของ `emailservice` (2) ให้ loyalty points ผ่าน `loyaltyservice` ตัวใหม่ (3) บันทึก analytics event "order placed" และปีหน้าจะมี post-order action เพิ่ม (SMS, fraud scoring, recommendations update)

**ระบบปัจจุบัน (จากโค้ด):**

| ข้อเท็จจริง | หลักฐาน |
|---|---|
| `PlaceOrder` เรียก `paymentservice` → `shippingservice` → `cartservice` (EmptyCart) → `emailservice` แบบ synchronous gRPC ตามลำดับ | `system/online-boutique/src/checkoutservice/main.go` (`PlaceOrder`, บรรทัด 252–277) |
| ถ้าส่ง email ล้มเหลว จะแค่ log warning และ order ยังสำเร็จ | `main.go` บรรทัด 273–277 |
| การเรียก `SendOrderConfirmation` ใช้ `ctx` ของ request โดยไม่มี timeout แยกในไฟล์นี้ ดังนั้น latency ของ email ถูกบวกเข้ากับ `PlaceOrder` โดยตรง | `main.go` (`sendOrderConfirmation`, บรรทัด 379–384) |
| dependency ทุกตัวถูกผูกผ่าน env var + gRPC connection ใน struct `checkoutService`; การเพิ่ม action ใหม่ = แก้ struct, `main()` และ `PlaceOrder` ทุกครั้ง | `main.go` (`checkoutService` บรรทัด 66–86, `mustMapEnv`/`mustConnGRPC` บรรทัด 111–123) |
| `checkoutservice` ไม่มี data store ของตัวเอง (มีแต่ gRPC connection) | `main.go` (`checkoutService` struct) |

**ตัวเลขที่สำคัญ (`docs/scenario.md`):**

| หัวข้อ | ค่า | ผลต่อการตัดสินใจ |
|---|---|---|
| `PlaceOrder` p95 | ≤ 500 ms (ปัจจุบัน ~300 ms) | เหลือ budget ~200 ms เท่านั้น |
| MailFast | p95 1.5 s, p99 8 s, fail ~0.5 %, rate limit 50 emails/s | เรียกแบบ synchronous ใน `PlaceOrder` จะทำให้ p95 เกิน 500 ms แน่นอน |
| Peak load | 200 orders/s (11.11) | 200 > 50 → email ค้าง backlog สะสม **150 emails/s ≈ 9,000 emails ต่อนาทีของ peak** ต้องมี buffer |
| Loyalty points | ภายใน 1 นาที, **exactly once** | ต้องมี idempotency และ event ต้องไม่หายระหว่างทาง |
| Availability | order ต้อง **ไม่ fail** เพราะ email/loyalty/analytics ล่ม | post-order action ต้องแยกออกจาก critical path |
| Team / Ops | 4 devs ถนัด Go + gRPC; 2 คนเคยใช้ **RabbitMQ**; รันบน Kubernetes; ชอบ open-source self-hosted | เอื้อต่อ RabbitMQ in-cluster |

**ปัญหา:** จะเชื่อม `checkoutservice` กับ post-order action ทั้งหลายอย่างไร ให้ checkout เร็วและไม่ล้มตาม downstream, รองรับ rate limit ของ MailFast, ให้ points แบบ exactly once และเพิ่ม action ใหม่ได้ง่าย

## Decision drivers

1. **Availability:** order ต้องไม่ fail เมื่อ MailFast, `loyaltyservice` หรือ analytics ล่ม
2. **Performance:** `PlaceOrder` p95 ≤ 500 ms ที่ 200 orders/s
3. **Reliability / correctness:** loyalty points ภายใน 1 นาที และ exactly once; email ต้องไม่หายเมื่อ MailFast fail 0.5 % หรือติด rate limit 50/s
4. **Modifiability:** เพิ่ม SMS, fraud scoring, recommendations ปีหน้าได้โดยไม่ต้องแก้ `checkoutservice`
5. **Cost / operability:** open-source self-hosted บน Kubernetes, ทีมเล็ก 4 คน ใช้ทักษะที่มีอยู่

## Considered options

1. **Synchronous gRPC fan-out** — ขยาย `PlaceOrder` ให้เรียก MailFast, `loyaltyservice`, analytics ต่อจาก `emailservice` เหมือนเดิม
2. **In-process async (goroutines)** — `PlaceOrder` ตอบกลับทันทีแล้วยิง action แต่ละอันใน goroutine ภายใน `checkoutservice`
3. **Event-driven ด้วย RabbitMQ (self-hosted)** — `checkoutservice` publish event `order.placed` ไปยัง exchange; consumer แต่ละตัว (`emailservice` → MailFast, `loyaltyservice`, analytics) มี queue ของตัวเอง
4. **Event-driven ด้วย Apache Kafka (self-hosted)** — แนวคิดเดียวกับข้อ 3 แต่ใช้ Kafka topic + consumer groups

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Sync gRPC fan-out | ง่ายสุด; ตรงกับ pattern เดิมและทักษะ Go/gRPC; ไม่มี component ใหม่ | MailFast p95 1.5 s ทำให้ p95 เกิน 500 ms; 200/s ชน rate limit 50/s; downstream ช้า/ล่ม = checkout ช้า; ทุก action ใหม่ต้องแก้ `PlaceOrder` (coupling สูง) |
| 2. In-process goroutines | latency ต่ำ; ไม่มี infra ใหม่ | งานค้างอยู่ใน memory หายเมื่อ pod restart; ไม่มี durable retry → ไม่รับประกัน points ภายใน 1 นาที/exactly once; rate limit 50/s ต้องคุมข้าม replica เอง; ยังต้องแก้ `checkoutservice` ทุก action ใหม่ |
| 3. RabbitMQ | ตัด downstream ออกจาก critical path; queue durable เป็น buffer ให้ backlog email ระหว่าง peak; consumer คุม rate 50/s ได้เอง; retry + dead-letter queue; เพิ่ม action ใหม่ = bind queue ใหม่ ไม่ต้องแก้ checkout; open-source, รันบน Kubernetes; ทีม 2 คนเคยใช้ | component stateful ใหม่ต้องดูแล (HA, monitoring); delivery เป็น at-least-once → `loyaltyservice` ต้อง dedupe เอง; publish ยังเป็น dual-write (ดู Risks); ไม่มี replay ย้อนหลังแบบ log |
| 4. Kafka | throughput สูง; เก็บ event เป็น log และ replay ได้ (ดีกับ analytics/recommendations); open-source | operate ยากกว่าสำหรับทีม 4 คน; ยังต้อง dedupe ฝั่ง consumer; 200 events/s ไม่ต้องการ throughput ระดับ Kafka |

## Decision

เลือก **Option 3: RabbitMQ (self-hosted บน Kubernetes)** ร่วมกับ **transactional outbox** ใน `checkoutservice` เพราะตอบ driver 1–4 ได้ครบ และตรงกับ driver 5 (open-source และ 2 ใน 4 คนในทีมเคยใช้ RabbitMQ) อีกทั้ง 200 events/s ไม่ต้องการ throughput สูงระดับ Kafka

รายละเอียด:

- **Publish (transactional outbox):** ใน `PlaceOrder` แทนที่การเรียก `sendOrderConfirmation` (`main.go` บรรทัด 273–277) ด้วยการบันทึก order และ event `order.placed` ลงฐานข้อมูลใหม่ของ `checkoutservice` (ตาราง orders และ outbox) **ใน transaction เดียวกัน** จากนั้น relay ภายใน `checkoutservice` อ่านแถวที่ยังไม่ส่งจาก outbox แล้ว publish ไปยัง topic exchange `orders` แบบ persistent พร้อม publisher confirms และ mark ว่าส่งแล้ว ถ้า RabbitMQ ล่ม event จะรออยู่ใน outbox แทนที่จะหาย และ order ยังสำเร็จได้ ฐานข้อมูลต้องเป็น open-source ที่รันเองบน Kubernetes (ยังไม่เลือก ดู Open questions) payload มี `order_id`, `user_id`, `email`, items, total และ `occurred_at` (ข้อมูลเหล่านี้มีอยู่แล้วใน `PlaceOrder`: `orderID`, `req.UserId`, `req.Email`, `orderResult`, `total`)
- **Email:** `emailservice` กลายเป็น consumer ของ queue `email.order-placed` และรันเป็น **consumer ตัวเดียว (1 replica)** เพื่อคุม rate limit ของ MailFast ไว้ที่จุดเดียว เพราะถ้ามีหลาย replica ที่ต่างคนต่างจำกัด 50/s ยอดรวมจะเกิน limit ถ้า consumer ตัวนี้ล่ม อีเมลจะรออยู่ใน queue ไม่หาย เรียก MailFast ด้วย rate limiter ≤ limit ที่ตกลงกับ MailFast (ปัจจุบัน 50 emails/s), retry ด้วย exponential backoff สำหรับ 0.5 % ที่ fail และส่งเข้า dead-letter queue เมื่อเกินจำนวน retry ที่กำหนด ด้วย Little's law consumer ตัวเดียวต้องรองรับ request ที่ค้างพร้อมกันราว limit × 1.5 s คือ ≈ 75 ที่ 50/s หรือ ≈ 300 ที่ 200/s (มากกว่านั้นเมื่อเจอ p99 8 s)
- **ขอเพิ่ม rate limit ของ MailFast:** ขอให้ MailFast เพิ่ม limit เป็น **≥ 200 emails/s** ช่วง 11.11 เพราะเมื่ออีเมลเข้า 200/s และส่งออกได้ 200/s จะไม่มี backlog เลย ขณะที่ limit 50/s ทำให้ค้างเพิ่ม 150 ฉบับทุกวินาที **แผนสำรอง** ถ้า MailFast ไม่ยอม: ระบบยังทำงานได้ตามการออกแบบนี้ แต่อีเมลจะช้าอย่างน้อย 3 เท่าของช่วง peak (ดู Consequences)
- **Loyalty:** `loyaltyservice` (Go เพราะทีมถนัด Go ตาม scenario) consume queue `loyalty.order-placed`, ใช้ `order_id` เป็น idempotency key (unique constraint ใน store ของตัวเอง) แล้ว ack หลังบันทึกสำเร็จ ได้ผลลัพธ์ exactly once จากสองส่วนรวมกัน: outbox ทำให้ event ถูกส่งออกไป**อย่างน้อยหนึ่งครั้งแน่นอน**แม้ RabbitMQ ล่มชั่วคราว และ `order_id` กันไม่ให้ได้แต้ม**ซ้ำ**เมื่อ event ถูกส่งซ้ำ
- **Analytics:** consumer ของ queue `analytics.order-placed` ตัดข้อมูลส่วนตัวของลูกค้า (`email` และ `user_id`) ทิ้งก่อนบันทึก และส่งให้ marketing เฉพาะ `order_id`, items, total และ `occurred_at` เพราะการนับยอดและวิเคราะห์คำสั่งซื้อไม่ต้องใช้ข้อมูลที่ระบุตัวลูกค้า
- **อนาคต:** SMS / fraud scoring / recommendations = queue ใหม่ bind กับ exchange เดิม ไม่ต้องแก้ `checkoutservice`

```mermaid
C4Container
  title Containers: Online Boutique post-order actions (ADR-0002)
  Person(customer, "Customer", "Places orders")
  System_Boundary(ob, "Online Boutique") {
    Container(checkout, "checkoutservice", "Go", "Places orders, writes order + event to outbox, relays order.placed")
    ContainerDb(checkoutdb, "Checkout DB (orders + outbox)", "Not decided", "Orders and unsent order.placed events")
    ContainerQueue(mq, "RabbitMQ", "RabbitMQ", "orders exchange, one queue per consumer")
    Container(email, "emailservice", "Consumer (1 replica)", "Sends confirmation via MailFast within the agreed rate limit")
    Container(loyalty, "loyaltyservice", "Go", "Awards points, idempotent by order_id")
    ContainerDb(loyaltydb, "Loyalty store", "Not decided", "Points and processed order_ids")
    Container(analytics, "analytics consumer", "Not decided", "Records order events without customer PII")
  }
  System_Ext(mailfast, "MailFast", "External email provider")
  Rel(customer, checkout, "Places order", "via frontend")
  Rel(checkout, checkoutdb, "Writes order + outbox event in one transaction", "Not decided")
  Rel(checkout, mq, "Relays order.placed from outbox", "AMQP")
  Rel(mq, email, "Delivers", "AMQP")
  Rel(mq, loyalty, "Delivers", "AMQP")
  Rel(mq, analytics, "Delivers", "AMQP")
  Rel(loyalty, loyaltydb, "Reads/writes", "Not decided")
  Rel(email, mailfast, "Sends email", "Not evidenced in the provided files")
  UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## Consequences

- **Positive:** `PlaceOrder` ไม่ต้องรอ MailFast (p95 1.5 s / p99 8 s) อีกต่อไป เหลือแค่ต้นทุน publish ไป broker in-cluster (quality attribute: performance)
- **Positive:** MailFast, `loyaltyservice` หรือ analytics ล่ม ไม่ทำให้ order fail; message รออยู่ใน queue จนกว่า consumer กลับมา (quality attribute: availability)
- **Positive:** queue รองรับ backlog ระหว่าง peak (150 emails/s ถ้า MailFast ยังจำกัดที่ 50/s) และระบายตาม rate limit; retry + DLQ ลดการสูญหายของ email (quality attribute: reliability)
- **Positive:** outbox ทำให้ event ไม่หายแม้ RabbitMQ ล่มตอนสั่งซื้อ ลูกค้าจึงได้อีเมลและแต้มครบทุกคำสั่งซื้อ (quality attribute: reliability)
- **Positive:** action ใหม่ปีหน้าเพิ่มได้ด้วยการเพิ่ม consumer ไม่ต้องแตะ checkout (quality attribute: modifiability)
- **Positive:** อีเมลและ `user_id` ของลูกค้าไม่หลุดเข้าไปใน analytics ของทีม marketing (quality attribute: security / privacy)
- **Negative:** มี component stateful ใหม่ (RabbitMQ cluster) ต้อง deploy, monitor, upgrade ด้วยทีม 4 คน (quality attribute: operability / cost)
- **Negative:** `checkoutservice` ต้องมีฐานข้อมูลของตัวเอง (orders + outbox) ซึ่งเป็น component stateful ชิ้นที่สองที่ทีมต้องดูแล การเขียนฐานข้อมูลใช้เวลาจาก budget ~200 ms ที่เหลือของ `PlaceOrder` และถ้าฐานข้อมูลนี้ล่ม จะสั่งซื้อไม่ได้ (quality attribute: operability, performance, availability)
- **Negative:** email ไม่ได้ส่งทันทีอีกต่อไป ถ้า MailFast ยังจำกัดที่ 50/s ช่วง peak อีเมลจะค้างเพิ่ม 150 ฉบับทุกวินาที (ค้าง = 150 × จำนวนวินาทีของ peak) และหลังจบ peak ต้องใช้เวลาระบายอย่างน้อย 3 เท่าของช่วง peak เช่น peak 1 ชั่วโมงจะค้าง 540,000 ฉบับ ลูกค้าที่สั่งช่วงท้าย peak จะได้อีเมลช้าราว 3 ชั่วโมงขึ้นไป ถ้า MailFast เพิ่ม limit เป็น ≥ 200/s ปัญหานี้จะหายไป (quality attribute: performance จากมุมผู้ใช้ / usability)
- **Negative:** ระบบเป็น eventually consistent และ debug ยากขึ้นข้าม async boundary; ต้อง propagate trace context ผ่าน message header (โค้ดปัจจุบันใช้ OpenTelemetry กับ gRPC อยู่แล้ว: `main.go` บรรทัด 134–140) (quality attribute: observability)
- **Risks and mitigations:**
  - **Dual-write / event หาย:** แก้ด้วย transactional outbox: order และ event ถูกบันทึกพร้อมกัน แล้ว relay ส่งต่อเมื่อ RabbitMQ พร้อม จึงไม่มี event หายเมื่อ broker ล่ม ความเสี่ยงที่เหลือ: (1) relay ส่งช้า → ตั้ง alert ตามอายุแถวที่ยังไม่ส่งใน outbox ให้ส่งทันภายใน 1 นาทีของเงื่อนไข loyalty (2) ตาราง outbox โตขึ้นเรื่อย ๆ → ลบแถวที่ส่งแล้วเป็นระยะ (3) ฐานข้อมูลล่มหลังตัดเงินแล้ว → retry สั้น ๆ ถ้ายังไม่ได้ให้ log + alert เพื่อชดเชยด้วยมือ
  - **Points ซ้ำ:** redelivery ของ RabbitMQ → idempotency key `order_id` + unique constraint ใน `loyaltyservice`
  - **Points เกิน 1 นาที:** queue ของ loyalty แยกจาก email จึงไม่ติด backlog ของ MailFast; ตั้ง alert เมื่อ message age ใน `loyalty.order-placed` เกินเกณฑ์ (ดู Open questions)
  - **Backlog email ไม่หมด:** ถ้า MailFast ไม่เพิ่ม limit และ peak นาน backlog จะโตตามสูตรใน Consequences → monitor queue depth, กำหนด max length / TTL ตาม policy ที่ตกลงกับ marketing
  - **Poison message:** ใช้ dead-letter queue + alert
  - **Consumer อีเมลตัวเดียวล่ม:** อีเมลหยุดส่งชั่วคราวแต่ไม่หาย เพราะรออยู่ใน queue → ให้ Kubernetes เปิด pod ใหม่อัตโนมัติ และตั้ง alert เมื่อจำนวนงานค้างใน `email.order-placed` สูงผิดปกติ

## Validation

- **Load test** ด้วย `loadgenerator` (Locust) ที่ 200 orders/s: ตรวจว่า `PlaceOrder` p95 ≤ 500 ms
- **Chaos test:** ปิด `emailservice`, `loyaltyservice`, analytics consumer และจำลอง MailFast fail/ช้า (p99 8 s) → อัตรา order สำเร็จต้องไม่ลดลง; **ปิด RabbitMQ ระหว่าง load test** แล้วเปิดกลับมา → จำนวน order ต้องเท่ากับจำนวนครั้งที่ให้แต้ม (ไม่มี event หาย)
- **Loyalty correctness:** ส่ง message ซ้ำ (redelivery/consumer restart) แล้วตรวจว่า points ถูกให้ครั้งเดียวต่อ `order_id`; วัด lag ระหว่าง `occurred_at` กับเวลาที่บันทึก points ต้อง ≤ 1 นาทีที่ peak
- **Metrics ใน production:** queue depth และ message age ต่อ queue, อายุแถวที่ยังไม่ส่งใน outbox, จำนวนใน DLQ, MailFast send rate (≤ limit ที่ตกลงกับ MailFast), publish failure rate ของ relay
- **Review date:** ทบทวน ADR นี้หลัง 11.11 sale และก่อนเพิ่ม action ใหม่ (SMS, fraud scoring) [วันที่ต้องยืนยันกับทีม]

## Open questions

- **ระยะเวลา peak 11.11:** ไม่ระบุ จำเป็นต่อการประเมินขนาด backlog email (ค้าง = 150 × จำนวนวินาทีของ peak ถ้า limit ยังเป็น 50/s) และเวลาที่ใช้ระบายหมด
- **SLA ของ email:** ลูกค้าต้องได้ email ภายในกี่นาที? ยอมรับ delay ระหว่าง peak ได้แค่ไหน? MailFast เพิ่ม rate limit เป็น ≥ 200/s ช่วง 11.11 ได้หรือไม่ และมีค่าใช้จ่ายเท่าไหร่?
- **Fraud scoring:** ถ้าต้องตัดสินก่อนยืนยัน order จะเป็น synchronous ไม่ใช่ post-order event — ต้องยืนยันกับ marketing
- **Protocol/API ของ MailFast** และรองรับ idempotency key หรือไม่ (เพื่อกัน email ซ้ำเมื่อ retry): Not evidenced in the provided files
- **Data store ของ `loyaltyservice`** และของ analytics (marketing ใช้เครื่องมืออะไร): ไม่ระบุ
- **ฐานข้อมูลของ `checkoutservice`:** จะใช้ฐานข้อมูล open-source ตัวใด (รันเองบน Kubernetes) และใครดูแล backup
- **Latency ของการ publish ไป RabbitMQ in-cluster** พร้อม publisher confirms: ไม่มีตัวเลข ต้องวัดจาก load test
- **ใครดูแล RabbitMQ** (on-call, backup, upgrade) และใช้ operator ตัวใด: ไม่ระบุ
- **Review date ที่แน่นอน:** ไม่ระบุ
- **เกณฑ์ alert ของ `loyalty.order-placed` และ outbox:** message age กี่วินาทีจึงแจ้งเตือน ต้องน้อยกว่า 1 นาทีพอให้แก้ทันก่อนผิดเงื่อนไข loyalty
