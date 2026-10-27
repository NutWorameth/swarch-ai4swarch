Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`)

# Review: ADR-0002 Publish an "order placed" event to RabbitMQ (แนว ATAM)

ADR ที่รีวิว: `docs/adr/0002-publish-order-placed-to-rabbitmq.md` · ตัวเลขจาก `docs/scenario.md`

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QA1 (performance) | ลูกค้าช่วง 11.11 | เรียก `PlaceOrder` 200 orders/s | peak, ทุก component ปกติ | payment → shipping → เขียน order + outbox ใน transaction เดียว แล้วตอบกลับ โดยไม่รอ broker/MailFast | p95 ≤ 500 ms (เหลือ budget ~200 ms จาก ~300 ms ปัจจุบันให้การเขียน DB) |
| QA2 (availability) | MailFast หรือ `loyaltyservice` | ล่ม 10 นาที | peak 200 orders/s | order ยังสำเร็จ, event ค้างใน queue ของตัวเอง, ระบายต่อเมื่อกลับมา | order ที่ fail เพราะ downstream = 0 %; email ส่งไม่เกิน 50/s; queue `loyalty.order-placed` ไม่ติด backlog ของ email |
| QA3 (reliability) | RabbitMQ pod restart / consumer redelivery | broker ล่ม 2 นาที แล้ว message ถูกส่งซ้ำ | peak | outbox เก็บ event ไว้, relay ส่งใหม่, `loyaltyservice` dedupe ด้วย `order_id` | event หาย = 0, points ซ้ำ = 0, points ภายใน 60 s หลัง order |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | ไม่เลือก Option 1 (sync fan-out) | MailFast p95 1.5 s > budget 500 ms และ 200/s > rate limit 50/s ตัวเลขยืนยันชัด |
| Non-risk | แยก queue ต่อ consumer (`email.*`, `loyalty.*`, `analytics.*`) | backlog ของ email (150/s) ไม่ทำให้ loyalty เกิน 1 นาที ซึ่งเป็นจุดที่เสี่ยงที่สุดของ R-12 |
| Non-risk | RabbitMQ แทน Kafka | 200 events/s ต่ำมาก และ 2 ใน 4 คนในทีมเคยใช้ RabbitMQ ตรงกับข้อจำกัด open-source/self-hosted |
| Risk | ฐานข้อมูลใหม่ของ `checkoutservice` อยู่บน critical path **หลัง** การตัดเงิน | ถ้า DB ล่ม order จะ fail หลังตัดเงินไปแล้ว ADR ยอมรับเองว่าต้อง "ชดเชยด้วยมือ" ขัดกับ driver 1 ("order must never fail") และยังไม่ได้ระบุ HA ของ DB |
| Risk | relay ทำงานอยู่ใน `checkoutservice` แต่ไม่ได้บอกว่ากี่ replica | ถ้ามีหลาย replica relay จะอ่านแถวเดียวกันและ publish ซ้ำหรือสลับลำดับ ต้องมีการ lock แถวหรือเลือก leader ซึ่ง ADR ไม่ได้พูดถึง |
| Risk | "exactly once" ของ loyalty ขึ้นกับว่าการบันทึก `order_id` กับการเพิ่ม points อยู่ใน transaction เดียวกันหรือไม่ | ADR ระบุแค่ "unique constraint" ถ้าเพิ่มแต้มคนละ transaction กับ dedupe จะยังได้แต้มซ้ำหรือแต้มหายเมื่อ crash ระหว่างสองขั้น |
| Risk | ขัดกันในตัว: `loyaltyservice` ล่มเกิน 1 นาที = ผิดเงื่อนไข "within 1 minute" เสมอ แม้ queue จะ durable | ต้องมี availability target ของ `loyaltyservice` เอง (replica > 1) ไม่ใช่พึ่ง queue อย่างเดียว |
| Sensitivity point | rate limit ของ MailFast (50 vs ≥ 200/s) | 50/s: peak 1 ชม. ค้าง 540,000 ฉบับ ระบายนาน ≥ 3 ชม.; 200/s: ไม่มี backlog ค่านี้ค่าเดียวกำหนด email latency ทั้งหมด |
| Sensitivity point | ช่วง polling ของ relay และเกณฑ์ alert อายุแถวใน outbox | poll ถี่ = โหลด DB บน critical path เพิ่ม, poll ห่าง = ใช้ budget 1 นาทีของ loyalty ไปมาก |
| Trade-off point | transactional outbox | reliability ↑ (event ไม่หายเมื่อ broker ล่ม) แลกกับ availability ↓, latency ↑ และ operability ↓ (stateful component ชิ้นที่สอง) |
| Trade-off point | email consumer 1 replica | คุม rate limit ได้ง่าย แลกกับ availability ของ email ↓ (single point of failure) และ throughput ถูกจำกัดที่ pod เดียว (≈ 75–400 in-flight requests เมื่อเจอ p99 8 s) |
| Trade-off point | max length / TTL ของ queue email | ปกป้อง broker (operability) แต่ทิ้ง email ไป ขัดกับ driver 3 "email ต้องไม่หาย" |

## 3. Missing or weak

- **Options ที่ไม่ได้พิจารณา:** (a) outbox อย่างเดียวโดยไม่มีตาราง orders (R-12 ไม่ได้ขอให้เก็บ order ส่วนนี้เป็น scope creep); (b) publish ตรงพร้อม publisher confirms + retry/fallback buffer ในเครื่อง แล้วยอมรับความเสี่ยงที่เหลือ; (c) CDC แทน polling relay; (d) rate limiter ร่วมกันข้าม replica (หรือแบ่ง limit เป็น 50/N ต่อ replica) แทนการบังคับ 1 replica
- **Consequences ที่ไม่ได้ระบุ:** ประเภท queue ของ RabbitMQ (เช่น quorum queues) และจำนวน node สำหรับ HA; ขั้นตอน reprocess ของ DLQ (ถ้าไม่มี DLQ = email หาย); PII (`email`, `user_id`) ยังอยู่ใน broker และไปถึง analytics consumer ก่อนถูกตัด
- **Claims ที่ยังไม่มีหลักฐานหรือน่าจะผิด:**
  - "ลูกค้าจึงได้อีเมลและแต้มครบทุกคำสั่งซื้อ" เกินจริง เพราะยังมีกรณี DB ล่มหลังตัดเงิน, DLQ และ TTL
  - "ตอบ driver 1–4 ได้ครบ" ยังไม่จริงสำหรับ driver 1 เพราะ DB ใหม่ทำให้ order fail ได้
  - "เหลือแค่ต้นทุน publish ไป broker" (Consequences ข้อแรก) ขัดกับ Decision เพราะ critical path จริงคือการเขียน DB ไม่ใช่การ publish และยังไม่มีตัวเลข latency ของ DB ใน budget 200 ms
  - Fraud scoring อาจต้องเป็น synchronous (ADR ยอมรับเองใน Open questions) จึงอ้างไม่ได้ว่า "action ใหม่ไม่ต้องแก้ `checkoutservice`" ครอบคลุมทุก action

## 4. Verdict: **Accept with changes**

1. **แก้เรื่อง DB บน critical path:** ตัด scope ให้เหลือ outbox อย่างเดียว ระบุ HA ของ DB และงบ latency และกำหนดว่าจะทำอะไรอัตโนมัติเมื่อเขียน DB ไม่ได้หลังตัดเงิน (ไม่ใช่ "ชดเชยด้วยมือ") หรือแก้ driver 1 ให้ตรงกับความจริง
2. **ทำให้ exactly once พิสูจน์ได้:** ระบุว่า relay ทำงานพร้อมกันหลาย replica อย่างไร (lock แถว/leader), ให้ dedupe `order_id` กับการเพิ่ม points อยู่ใน transaction เดียว, ใช้ quorum queues และกำหนด replica/SLO ของ `loyaltyservice` ให้ทัน 1 นาที
3. **ทบทวนการออกแบบ email:** เปลี่ยนจาก 1 replica เป็น rate limiter ร่วมกัน, กำหนดขั้นตอน DLQ reprocess, เอา TTL/max length ที่ทิ้ง email ออก (หรือแก้ driver 3) และได้คำตอบเรื่อง rate limit 11.11 จาก MailFast ก่อนวันจริง
