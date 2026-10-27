**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** วรเมธ สุวรรณพฤกษ์ 6431340021
**agent ที่ใช้:** Claude Code (CLI)
**โมเดล:** Opus 5.5
**แพ็กเกจที่ใช้:** Claude Pro (สมัครไว้ทำงานนอกอยู่แล้วครับ)

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code + Opus 5.5 | `prompts/1-…` run A | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` | ใช่ | 2 | เปิด session ด้วย `--tools "Read,Write,Glob"` ปิด MCP และ auto memory; ไม่มี dependency ในตารางและแผนภาพเลย ย้ายสิ่งที่เดาไปไว้ใน Gaps พร้อมป้าย [Inferred]; ช่อง Technology ใส่ "gRPC" ซึ่งเป็น protocol ไม่ใช่ภาษา; `demo.proto` บรรทัด 19 (`go_package`) มีชื่อ repo ต้นฉบับ ข้อ [Inferred] ของ checkout จึงอาจมาจากความจำของโมเดล |
| 2 | 1-B | Claude Code + Opus 5.5 | `prompts/1-…` run B | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto`, ทุกไฟล์ใน `kubernetes-manifests/` (14 ไฟล์ รวม `README.md` และ `kustomization.yaml`) | ใช่ | 4 | `/clear` ก่อนรัน เงื่อนไขเครื่องมือเดียวกับ 1-A; ได้ 12 containers และ 17 dependencies ตรงกับแผนภาพ; ทุกเส้นอ้าง env `*_ADDR` แต่ AI ยอมรับเองใน Gaps ว่า env ไม่ได้พิสูจน์ว่าเรียกจริง; ไม่วาด `shoppingassistantservice` แต่ไปใส่ไว้ใน Gaps |
| 3 | 2-a | Claude Code + Opus 5.5 | `prompts/2-…` run a | `prompts/2-write-adr.md`, `docs/adr/template.md` | ใช่ | | `context: none`, `language: Thai`, session ใหม่ด้วยเงื่อนไขเครื่องมือเดียวกับงานที่ 1; ไม่ได้เปิด scenario; เลือก pub/sub แต่ไม่เลือกเทคโนโลยี broker; ไม่มีตัวเลขเลย ย้ายไป Open questions |
| 4 | 2-b | Claude Code + Opus 5.5 | `prompts/2-…` run b | `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go` | ใช่ | | `language: Thai`, `/clear` ก่อนรัน; ไม่ได้เปิด `outputs/2a-adr.md`; เลือก RabbitMQ; ใช้ตัวเลขจาก scenario ครบและอ้างบรรทัดโค้ดถูกทุกจุด; แต่งเพิ่ม 2 จุด (ทีมไม่มีประสบการณ์ Kafka, alert > 30 s) |
| 5 | 3 | | `prompts/3-…` | | | | |

