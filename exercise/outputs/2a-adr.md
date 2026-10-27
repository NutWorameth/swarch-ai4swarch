Agent: Claude Code — Model: Claude Opus 5.5 (`claude-opus-5-5`)

# ADR-0002: Publish an "order placed" event to a message broker for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** ไม่ได้ระบุใน requirement (ดู Open questions)

## Context and problem

ADR นี้เขียนจาก requirement อย่างเดียว (`context: none`) ไม่ได้อ่านโค้ดหรือ manifest ของ Online Boutique ข้อความใดที่พูดถึงระบบปัจจุบันเกินจาก requirement จะติดป้าย **[Inferred]**

หลังลูกค้าสั่งซื้อสำเร็จ ระบบต้องทำ 3 อย่าง:

| # | Post-order action | ผู้รับผิดชอบ | สถานะ |
|---|---|---|---|
| 1 | ส่งอีเมลยืนยันคำสั่งซื้อผ่าน MailFast (external email provider) แทน logic จำลองใน `emailservice` | MailFast (ภายนอกระบบ) | ผู้ให้บริการรายใหม่ |
| 2 | ให้คะแนน loyalty | `loyaltyservice` | ยังไม่ได้สร้าง |
| 3 | บันทึก analytics event "order placed" ให้ทีม marketing | ยังไม่ระบุว่าใช้ระบบ analytics ใด | ใหม่ |

ทีม marketing คาดว่าปีหน้าจะมี post-order action เพิ่มอีก (SMS, fraud scoring, การอัปเดต recommendations)

**ปัญหาที่ต้องตัดสินใจ:** บริการที่ปิดคำสั่งซื้อควรเชื่อมกับ post-order action ที่มีอยู่และที่จะเพิ่มขึ้นเรื่อย ๆ อย่างไร โดยที่:
- ถ้า action ใด action หนึ่งล้มเหลวหรือช้า (เช่น MailFast ล่ม) ต้องไม่ทำให้การสั่งซื้อล้มเหลวหรือช้าลง
- การเพิ่ม action ใหม่ต้องไม่ต้องแก้บริการที่ปิดคำสั่งซื้อทุกครั้ง

ไม่มีหลักฐานในไฟล์ที่ได้รับว่าบริการใดเป็นผู้ปิดคำสั่งซื้อ และบริการนั้นเรียก `emailservice` อย่างไร **[Inferred]** น่าจะเป็น `checkoutservice` ที่เรียก `emailservice` แบบ synchronous

## Decision drivers

1. **Modifiability / extensibility**: เพิ่ม post-order action ใหม่ (SMS, fraud scoring, recommendations) ได้โดยไม่ต้องแก้หรือ redeploy บริการที่ปิดคำสั่งซื้อ
2. **Availability ของ checkout**: การสั่งซื้อต้องสำเร็จได้แม้ MailFast, `loyaltyservice` หรือระบบ analytics จะล่ม
3. **Reliability ของ post-order action**: event ต้องไม่หาย ลูกค้าต้องได้อีเมลและได้คะแนนในที่สุด (at-least-once) และการทำซ้ำต้องไม่ให้คะแนนซ้ำหรือส่งอีเมลซ้ำ
4. **Performance**: latency ของการสั่งซื้อไม่ควรเพิ่มขึ้นตามจำนวน action
5. **Cost / operability**: จำนวน infrastructure ใหม่ที่ต้องดูแล
6. **Integration กับ external provider**: แยก dependency ของ MailFast (API, rate limit, credential) ไว้ที่จุดเดียว

## Considered options

1. **A: Synchronous direct calls**: บริการที่ปิดคำสั่งซื้อเรียกแต่ละ action ตรง ๆ ตามลำดับ (email → loyalty → analytics) เหมือนการเรียก `emailservice` ในปัจจุบัน **[Inferred]**
2. **B: Event-driven publish/subscribe**: บริการที่ปิดคำสั่งซื้อ publish event `OrderPlaced` ไปยัง message broker แล้วแต่ละ consumer (`emailservice` → MailFast, `loyaltyservice`, analytics) subscribe แยกกันอย่างอิสระ
3. **C: Post-order orchestrator service**: บริการใหม่ (เช่น `postorderservice`) รับคำขอหนึ่งครั้งจากบริการที่ปิดคำสั่งซื้อ แล้วเรียกแต่ละ action เองพร้อม retry

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A: Synchronous calls | เรียบง่ายที่สุด ไม่มี infrastructure ใหม่ / รู้ผลทันที debug ง่าย / ลำดับการทำงานชัดเจน | checkout ผูกกับ availability ของทุก action (MailFast ล่ม = สั่งซื้อพลาดหรือช้า) / latency รวมเพิ่มตามจำนวน action / ทุก action ใหม่ต้องแก้และ redeploy บริการที่ปิดคำสั่งซื้อ / ต้องเขียน retry และ timeout เองในบริการนั้น |
| B: Pub/sub ผ่าน broker | เพิ่ม consumer ใหม่ได้โดยไม่แตะ publisher / checkout ไม่ขึ้นกับ consumer ที่ล่ม / broker buffer และ retry ให้ / แต่ละ consumer scale และ deploy แยกกันได้ | ต้องดูแล message broker เพิ่ม (cost, operability) / eventual consistency: อีเมลและคะแนนมาช้ากว่าการสั่งซื้อ / at-least-once ทำให้ consumer ต้อง idempotent / trace ข้ามบริการยากขึ้น / ต้องกำหนดและ version schema ของ event |
| C: Orchestrator | checkout เรียกแค่จุดเดียว / logic retry และลำดับรวมอยู่ที่เดียว / เห็นสถานะของแต่ละ action ได้ชัด | orchestrator ยังต้องแก้ทุกครั้งที่เพิ่ม action (แค่ย้ายจุดที่ต้องแก้) / ถ้าเรียกแบบ synchronous จะเป็น single point of failure ใหม่ / ถ้าจะไม่ให้ event หาย ต้องมี queue หรือ storage ของตัวเองอยู่ดี / เพิ่มบริการที่ต้องดูแล |

## Decision

เลือก **Option B: Event-driven publish/subscribe**

- บริการที่ปิดคำสั่งซื้อ publish event `OrderPlaced` **หนึ่งครั้ง** หลังคำสั่งซื้อสำเร็จ (ข้อมูลขั้นต่ำ เช่น order ID, user/customer ID, email, รายการสินค้า, ยอดรวม และ timestamp ส่วน schema จริงยังต้องตกลงกัน ดู Open questions)
- Consumer แต่ละตัวมี subscription หรือ queue ของตัวเอง:
  - `emailservice`: เปลี่ยนจาก logic จำลองเป็นการเรียก MailFast ทำให้ dependency ของ MailFast อยู่ที่บริการเดียว
  - `loyaltyservice` (สร้างใหม่): ให้คะแนนแบบ idempotent โดยใช้ order ID เป็น key
  - analytics consumer: ส่ง event "order placed" ไปยังระบบของทีม marketing
- Action ในอนาคต (SMS, fraud scoring, recommendations) เพิ่มเป็น consumer ใหม่ได้โดยไม่แก้ publisher
- ยังไม่เลือกเทคโนโลยี broker (เช่น RabbitMQ, Kafka, Google Cloud Pub/Sub) เพราะไม่มีข้อมูลเรื่อง volume, platform และทักษะของทีม ดู Open questions

**เหตุผล:** B ตอบ driver 1 (modifiability) และ 2 (availability ของ checkout) ได้ดีที่สุด ซึ่ง A ทำไม่ได้ ส่วน C แค่ย้ายจุด coupling ไปไว้ที่อื่น requirement ระบุชัดว่าจะมี action เพิ่ม จึงคุ้มกับต้นทุนของ broker และ eventual consistency

**ข้อยกเว้นที่ต้องพิจารณา:** fraud scoring ในอนาคตอาจต้อง *บล็อก* การสั่งซื้อ ถ้าเป็นอย่างนั้น ต้องเรียกแบบ synchronous *ก่อน* ปิดคำสั่งซื้อ ไม่ใช่เป็น consumer ของ `OrderPlaced` ควรตัดสินใจเรื่องนี้ใน ADR แยก

## Consequences

- **Positive:** เพิ่ม post-order action ใหม่ได้ด้วยการเพิ่ม consumer โดยไม่แก้บริการที่ปิดคำสั่งซื้อ (quality attribute: modifiability)
- **Positive:** การสั่งซื้อสำเร็จได้แม้ MailFast, `loyaltyservice` หรือ analytics ล่ม เพราะ broker เก็บ event ไว้รอ (quality attribute: availability)
- **Positive:** latency ของ checkout เพิ่มแค่เวลา publish หนึ่งครั้ง ไม่เพิ่มตามจำนวน action (quality attribute: performance)
- **Positive:** แต่ละ consumer scale, deploy และล้มเหลวแยกกันได้ ส่วน rate limit ของ MailFast จัดการได้ที่ `emailservice` ด้วยการคุมอัตราการ consume (quality attribute: scalability, fault isolation)
- **Negative:** ต้องติดตั้ง ดูแล และ monitor message broker เพิ่ม (quality attribute: cost, operability)
- **Negative:** อีเมลยืนยันและคะแนน loyalty มาถึงแบบ eventual ไม่ใช่ทันที (quality attribute: consistency, user experience)
- **Negative:** การ trace คำสั่งซื้อหนึ่งรายการข้ามหลายบริการแบบ asynchronous ทำได้ยากขึ้น (quality attribute: observability, testability)
- **Risks and mitigations:**
  - *Event หายถ้าคำสั่งซื้อสำเร็จแต่ publish ล้มเหลว* → ใช้ transactional outbox หรือ retry ที่ publisher และ alert เมื่อ publish ล้มเหลว **[Inferred]** การทำ outbox ต้องมี data store ที่ publisher เขียนได้ ซึ่งไม่มีหลักฐานว่ามีอยู่แล้ว
  - *Duplicate delivery (at-least-once)* → consumer ทุกตัวต้อง idempotent ด้วย order ID (ห้ามให้คะแนนซ้ำหรือส่งอีเมลซ้ำ)
  - *Poison message / consumer ล้มเหลวซ้ำ* → ใช้ dead-letter queue, retry แบบ backoff และ alert
  - *Schema ของ event เปลี่ยนแล้วทำให้ consumer พัง* → กำหนด schema และ versioning (เช่น เพิ่ม field ได้แต่ห้ามลบ)
  - *MailFast ล่มหรือ rate limit* → retry ใน `emailservice` พร้อม backoff และ dead-letter
  - *Broker เป็น single point of failure ใหม่* → ใช้ broker แบบ HA หรือ managed service และให้ publisher จัดการกรณี broker ไม่พร้อม (ผ่าน outbox)
  - *ข้อมูลส่วนบุคคล (email) อยู่ใน event ที่ทีม marketing เข้าถึง* → ส่งเฉพาะ field ที่จำเป็นให้ analytics consumer หรือ mask PII **[Inferred]** ยังไม่มีข้อกำหนดด้าน privacy ใน requirement

## Validation

- **Fault-injection test:** ปิด MailFast (หรือ mock), `loyaltyservice` และ analytics consumer แล้วยืนยันว่าการสั่งซื้อยังสำเร็จ และเมื่อเปิดกลับมา event ที่ค้างอยู่ถูกประมวลผลครบ
- **Duplicate test:** ส่ง `OrderPlaced` เดิมซ้ำแล้วยืนยันว่าได้อีเมลหนึ่งฉบับและคะแนนหนึ่งครั้ง
- **Load test:** วัด latency ของ checkout ก่อนและหลังการเปลี่ยนแปลง (ยังไม่มีเป้าหมายตัวเลข ดู Open questions)
- **Metrics ที่ต้องติดตาม:** consumer lag, จำนวนใน dead-letter queue, publish failure rate, เวลาจากสั่งซื้อจนอีเมลส่งออก
- **Modifiability check:** เมื่อเพิ่ม consumer ตัวแรกของปีหน้า (เช่น SMS) ต้องไม่มีการแก้โค้ดของบริการที่ปิดคำสั่งซื้อ
- **Review date:** ไม่ได้ระบุ ควรทบทวนก่อนเพิ่ม action ใหม่ชุดแรกของปีหน้า

## Open questions

- **Deciders:** ใครเป็นผู้ตัดสินใจหรือเป็นเจ้าของ ADR นี้
- **ระบบปัจจุบัน:** บริการใดปิดคำสั่งซื้อและเรียก `emailservice` อย่างไร (Not evidenced in the provided files)
- **Volume:** จำนวนคำสั่งซื้อต่อวินาทีทั้ง peak และปกติ ใช้ประกอบการเลือก broker และการ sizing
- **Latency budget:** เป้าหมาย latency ของ checkout และเวลาสูงสุดที่ยอมรับได้จากการสั่งซื้อจนอีเมลถึงลูกค้าและคะแนนเข้าบัญชี
- **Platform และทักษะทีม:** deploy บนอะไร มี managed broker ให้ใช้หรือไม่ และทีมคุ้นเคยกับ broker ตัวใด
- **MailFast:** รูปแบบ API, SLA, rate limit, การรองรับ idempotency key และวิธีจัดการ credential
- **Analytics:** ปลายทางของ event สำหรับทีม marketing คืออะไร และต้องการ field ใดบ้าง
- **`loyaltyservice`:** กฎการคำนวณคะแนน และมี data store ของตัวเองหรือไม่
- **Fraud scoring:** ต้องบล็อกการสั่งซื้อ (synchronous ก่อนปิดคำสั่งซื้อ) หรือแค่ให้คะแนนย้อนหลัง
- **Privacy/compliance:** มีข้อกำหนดเรื่อง PII ใน event และการเก็บข้อมูลหรือไม่
- **Delivery guarantee:** ยอมรับ at-least-once + idempotency ได้หรือไม่ และกรณีที่ event หายต้องการ outbox หรือไม่
