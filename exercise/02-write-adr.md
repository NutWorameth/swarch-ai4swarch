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
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่มีตัวเลขเลย (ไม่ได้เห็น scenario) ย้ายเรื่อง volume, latency budget และ rate limit ของ MailFast ไปไว้ใน Open questions | ใช้ครบ: `PlaceOrder` p95 ≤ 500 ms (ปัจจุบัน ~300 ms), 200 orders/s, MailFast p95 1.5 s / p99 8 s / fail 0.5% / 50 emails/s, loyalty ภายใน 1 นาทีและครั้งเดียว แล้วคำนวณต่อเป็น backlog 150 emails/s (≈9,000 ฉบับ/นาที) และ in-flight ≈75 ด้วย Little's law |
| ทางเลือกที่พิจารณา (ระบุ) | (A) เรียกแต่ละ action ตรงแบบ synchronous (B) pub/sub ผ่าน message broker (C) post-order orchestrator service → เลือก B แต่ยังไม่เลือกเทคโนโลยี broker | (1) synchronous gRPC fan-out (2) goroutine ภายใน `checkoutservice` (3) RabbitMQ (4) Kafka → เลือก RabbitMQ; ตัด orchestrator ทิ้ง และข้อ 3 กับ 4 เป็นแนวคิดเดียวกัน ต่างกันแค่เครื่องมือ |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | ใช้หลักการทั่วไป: consumer idempotent ด้วย order ID และ "คุมอัตราการ consume" ที่ `emailservice` แต่ไม่มีตัวเลข | แต้ม: at-least-once + `order_id` เป็น unique key และแยก queue ของ loyalty ออกจาก email เพื่อให้ทัน 1 นาที; อีเมล: limiter ≤ 50/s, retry แบบ backoff, DLQ และใช้ queue เป็น buffer ของ backlog แต่ไม่ได้บอกว่าจะคุม 50/s รวมทุก replica อย่างไร |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | ไม่พบ แต่เสนอ "managed service" เป็นทางลดความเสี่ยง ซึ่งขัดกับนโยบายบริษัทที่ไม่ได้เห็น (ฟังดูดีแต่ไม่เหมาะสม) | มี 2 จุด: (1) "Kafka: ทีมไม่มีประสบการณ์" ทั้งที่ scenario ไม่พูดถึง Kafka (AI ยอมรับเองในวงเล็บ แต่ยังใช้เป็นเหตุผลเลือก RabbitMQ) (2) alert เมื่อ message age > 30 s ซึ่งไม่มีตัวเลขนี้ใน scenario; ส่วนการให้ `loyaltyservice` ใช้ Go มาจากทักษะทีมใน scenario จึงไม่นับ แต่ ADR ไม่ได้เขียนเหตุผลนี้ไว้; อ้างบรรทัดโค้ดใน `main.go` ถูกทุกจุด |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | เพิ่ม transactional outbox: `PlaceOrder` บันทึก order และ event ลงฐานข้อมูลใหม่ของ `checkoutservice` ใน transaction เดียว แล้วให้ relay publish ไป RabbitMQ แทนการ publish ตรง | ร่างของ AI อ้างว่าได้ "exactly once" แต่ถ้า RabbitMQ ล่มตอน publish event จะหายและลูกค้าไม่ได้แต้ม ซึ่งผิด requirement; ยอมแลกกับฐานข้อมูลชิ้นใหม่ที่ทีมต้องดูแล และเขียนราคานี้ไว้ใน Consequences |
| 2 | ขอให้ MailFast เพิ่ม limit เป็น ≥ 200 emails/s ช่วง 11.11 พร้อมแผนสำรอง และแทนคำว่า "ช้าเป็นนาที" ด้วยสูตร backlog = 150 × วินาทีของ peak | ร่างประเมินความช้าต่ำเกินไป: peak 1 ชั่วโมงจะค้าง 540,000 ฉบับ และอีเมลช้าราว 3 ชั่วโมง; ถ้าเข้า 200/s ออก 200/s จะไม่มีอะไรค้าง |
| 3 | ให้ `emailservice` รันเป็น consumer ตัวเดียว (1 replica) เพื่อคุม rate limit ของ MailFast | ร่างเขียนแค่ "limiter รวม ≤ 50/s" แต่ถ้ามีหลาย replica ที่จำกัดคนละ 50/s ยอดรวมจะเกิน limit; ตัวเดียวง่ายสุดสำหรับทีม 4 คน และอีเมลไม่หายเพราะรออยู่ใน queue |
| 4 | ลบข้อเท็จจริงที่ AI แต่งขึ้น: "ทีมไม่มีประสบการณ์ Kafka" และ alert > 30 s (ย้ายเกณฑ์ไปไว้ใน Open questions) และแก้ "ทีมมีประสบการณ์" เป็น "2 ใน 4 คนเคยใช้ RabbitMQ" | scenario ไม่พูดถึง Kafka และไม่มีตัวเลข 30 s ถ้ากรรมการถามจะอ้างที่มาไม่ได้ |
| 5 | analytics consumer ตัด `email` และ `user_id` ทิ้งก่อนบันทึก ส่งให้ marketing เฉพาะ `order_id`, items, total และ `occurred_at` | marketing ใช้นับยอดและวิเคราะห์คำสั่งซื้อ ไม่จำเป็นต้องรู้ตัวตนลูกค้า (ให้แต่ละฝ่ายเห็นเฉพาะข้อมูลที่จำเป็น) |
