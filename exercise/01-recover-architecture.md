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
| `frontend` → `checkoutservice` | `kubernetes-manifests/frontend.yaml` (`CHECKOUT_SERVICE_ADDR=checkoutservice:5050`) | ✅ | พบใน `frontend.yaml:79` และโค้ดอ่านค่านี้จริงที่ `src/frontend/main.go:136` (`mustMapEnv`) |
| `checkoutservice` → `emailservice` | `kubernetes-manifests/checkoutservice.yaml` (`EMAIL_SERVICE_ADDR=emailservice:5000`) | ✅ | พบใน `checkoutservice.yaml:63` และโค้ดอ่านค่านี้จริงที่ `src/checkoutservice/main.go:115` |
| `recommendationservice` → `productcatalogservice` | `kubernetes-manifests/recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR`) | ✅ | พบใน `recommendationservice.yaml:60` และโค้ด Python อ่านค่านี้ที่ `src/recommendationservice/recommendation_server.py:131` (ถ้าไม่ได้ตั้งค่าจะ raise Exception) |
| `cartservice` → `redis-cart` | `kubernetes-manifests/cartservice.yaml` (`REDIS_ADDR=redis-cart:6379`; port `tcp-redis`) | ✅ | พบใน `cartservice.yaml:50` และโค้ดอ่าน `REDIS_ADDR` ที่ `src/cartservice/src/Startup.cs:29` |
| `loadgenerator` → `frontend` | `kubernetes-manifests/loadgenerator.yaml` (`FRONTEND_ADDR=frontend:80`) | ❓ | เส้นนี้มีจริงที่ `loadgenerator.yaml:69` แต่ `kustomization.yaml:24` คอมเมนต์ `loadgenerator.yaml` ออก จึงไม่ได้ deploy ในระบบจริง AI กำกับ "dev/test only" ไว้แค่ในช่อง Technology แต่ยังวาดกล่องไว้ในกรอบ `Online Boutique` ตำแหน่งจึงขัดกับป้ายของตัวเอง |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย):
  - **dependency ที่ไม่มีจริง:** ไม่พบ ทุกเส้นในตารางของ AI มีอยู่จริงใน manifest และเส้นที่ตรวจกับโค้ด (`frontend`, `checkoutservice`, `recommendationservice`, `cartservice`) โค้ดก็อ่านค่านั้นจริง เมื่อได้หลักฐานครบ AI ไม่ได้แต่งเส้นที่ไม่มีอยู่ขึ้นมาเอง
  - **dependency ที่ขาดไป:** `frontend` → `shoppingassistantservice` (HTTP): `frontend.yaml:83` ตั้งค่า `SHOPPING_ASSISTANT_SERVICE_ADDR` ไว้ และ `src/frontend/main.go:139` อ่านค่านี้ด้วย `mustMapEnv` ซึ่งจะ panic ถ้าไม่มีค่า ทำให้ frontend เปิดไม่ขึ้น ส่วน `src/frontend/handlers.go:464` เรียก service นี้ผ่าน `http://` AI ไม่ได้วาดเส้นนี้ เพราะในโฟลเดอร์ไม่มี manifest ของ `shoppingassistantservice` จึงย้ายไปไว้ใน Gaps
  - **container ที่น่าสงสัย:** `loadgenerator` (ดูแถว 5 ในตาราง)
  - **สาเหตุร่วมของสองข้อบน:** AI ใช้ "มีไฟล์ yaml ในโฟลเดอร์หรือไม่" เป็นเกณฑ์ว่าอะไรอยู่ในระบบ ทั้งที่ `loadgenerator` และ shopping assistant ไม่ได้ถูก deploy ตามปกติเหมือนกัน (`kustomization.yaml:24` และ `:35`) ผลคือแผนภาพวาดเกินหนึ่งกล่อง (`loadgenerator`) และขาดหนึ่งเส้น (assistant) ซึ่งตรงกับ AICH3 ที่ว่า AI เห็นโครงสร้างไฟล์ แต่ไม่เห็นว่าอะไรรันจริง
  - **ข้อสังเกต (ไม่นับเป็นความผิดพลาดของ AI):** โค้ดของ `frontend` และ `checkoutservice` มี dependency ไปยัง OpenTelemetry collector (`COLLECTOR_SERVICE_ADDR`, gRPC) แต่ใช้เฉพาะเมื่อ `ENABLE_TRACING=1` (`src/frontend/main.go:113`, `src/checkoutservice/main.go:90`) และ manifest ไม่ได้ตั้งค่านี้ไว้เลย ในการ deploy ตามปกติเส้นนี้จึงไม่ทำงาน แผนภาพที่ไม่มีเส้นนี้จึงถูกต้องสำหรับการ deploy ตามปกติ แต่แสดงว่าหลักฐานระดับ manifest ไม่ครอบคลุม dependency ที่โค้ดเปิดใช้ตามเงื่อนไข AI ระบุเรื่องนี้ไว้บางส่วนใน Gaps ("no collector container")

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

   **คำตอบ:** จาก `.proto` อย่างเดียว agent รู้ได้แค่ว่ามี service อะไรบ้างและแต่ละตัวให้บริการอะไร แต่ไม่รู้ว่าใครเรียกใคร ไม่รู้ว่ามีที่เก็บข้อมูล (`redis-cart`) และไม่รู้ว่าทางเข้าของระบบคือ `frontend` เพราะ proto เป็นเหมือนเมนูที่บอกว่าแต่ละ service ขายอะไร แต่ไม่ได้บอกว่าใครเป็นลูกค้า ผลคือ Run A ได้ 0 dependency ขณะที่ Run B ได้ 17 เส้นจากตัวแปร `*_ADDR` ใน manifest อย่างไรก็ตาม Run B ก็ยังไม่เห็น dependency ไปยัง collector ที่โค้ดเปิดใช้เฉพาะเมื่อ `ENABLE_TRACING=1` แสดงว่าหลักฐานแต่ละระดับเห็นได้แค่ในระดับของตัวเอง สิ่งที่ AI กู้คืนได้จึงมีเพดานอยู่ที่หลักฐานที่มันได้อ่าน

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

   **คำตอบ:** ข้อที่อันตรายที่สุดคือแผนภาพไม่มีเส้น `frontend` → `shoppingassistantservice` ทั้งที่ `src/frontend/main.go:139` บังคับว่าต้องมีค่า `SHOPPING_ASSISTANT_SERVICE_ADDR` ไม่เช่นนั้น frontend จะ panic ตั้งแต่ตอนเปิด สมาชิกใหม่ที่ดูแค่แผนภาพจะเข้าใจว่าค่านี้ไม่มีใครใช้และอาจลบทิ้ง ผลคือหน้าร้านล่มทั้งหมด ข้อนี้อันตรายกว่ากรณี `loadgenerator` ที่วาดเกินมา เพราะการวาดเกินทำให้เข้าใจผิดเรื่องขอบเขตของระบบ แต่การวาดขาดทำให้คนกล้าแก้สิ่งที่ไม่ควรแตะ
