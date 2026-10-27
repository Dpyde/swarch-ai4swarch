**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** Thukdanai Thaothawin / 6631319121 (GitHub: Dpyde)
**agent ที่ใช้:** Claude Code (desktop app) โดยแต่ละ run ถูกส่งให้ subagent ที่เริ่มด้วยบริบทใหม่ (fresh context) ทำงานแยกกัน
**โมเดล:** Claude Sonnet 5.5 (`claude-sonnet-5-5`)
**แพ็กเกจที่ใช้:** Claude (แผนของบัญชีนิสิต) ไม่ใช่ Copilot/Gemini ตามตัวอย่าง

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code subagent, Sonnet 5.5 | `prompts/1-…` run A, evidence: `protos/demo.proto` | `AGENTS.md`, `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` | ใช่ | 3 | ได้เพียง 9 gRPC service ไม่มี dependency, data store หรือผู้ใช้ ตรงกับที่คาดว่า proto อย่างเดียวไม่พอ |
| 2 | 1-B | Claude Code subagent, Sonnet 5.5 | `prompts/1-…` run B, evidence: `demo.proto` + `kubernetes-manifests/` | `AGENTS.md`, `prompts/…`, `demo.proto`, ไฟล์ทั้งหมดใน `kubernetes-manifests/` | ใช่ (agent รายงานว่าสั่ง list โฟลเดอร์ manifests หนึ่งครั้ง ซึ่งอยู่ในโฟลเดอร์ที่อนุญาต) | 5 | ได้ 12 container และ 17 dependency พร้อมหลักฐาน ตรวจแล้วถูก |
| 3 | 2-a | Claude Code subagent, Sonnet 5.5 | `prompts/2-…` run a, context: none, language: Thai | `AGENTS.md`, `prompts/2-write-adr.md`, `docs/adr/template.md` | ใช่ (ไม่เปิด scenario) | 3 | ครบทุกหัวข้อ แต่ไม่มีตัวเลข จึงไปอยู่ใน Open questions แทน |
| 4 | 2-b | Claude Code subagent, Sonnet 5.5 | `prompts/2-…` run b, context: scenario + ADR-0001 + `checkoutservice/main.go`, language: Thai | `AGENTS.md`, `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-…`, `checkoutservice/main.go` | ใช่ | 4 | ใช้ตัวเลขครบ อ้างบรรทัดโค้ดถูก (ตรวจแล้ว) แต่ยังมีช่องโหว่เรื่อง backlog อีเมลและ dual-write |
| 5 | 3 | Claude Code subagent, Sonnet 5.5 | `prompts/3-…` adr: `docs/adr/0002-use-rabbitmq-for-post-order-events.md`, language: Thai | `AGENTS.md`, `prompts/3-review-adr.md`, ADR-0002, `docs/scenario.md` | ใช่ | 4 | หาข้อบกพร่องของ ADR ได้ตรงจุด (ดูใบงาน 03) |
| 6 | โบนัส | Claude Code subagent, Sonnet 5.5 | `prompts/bonus-…` language: Thai | `AGENTS.md`, `prompts/bonus-recover-decision.md`, `docs/adr/template.md`, `demo.proto`, `checkoutservice/main.go`, `frontend/main.go` | ใช่ | 4 | ตาราง Claim check 18 แถว |
| 7 | | | | | | | |

หมายเหตุ: ระบบแนบ `CLAUDE.md` เข้าบริบทของ agent โดยอัตโนมัติ (ไฟล์นี้แค่ชี้ไปที่ `AGENTS.md`) agent ทุก run รายงานว่าไม่ได้เปิดเอง ส่วน session ผู้ประสานงานของผมอ่านใบงาน `prompts/` `docs/scenario.md` และ ADR-0001 เพื่อเตรียมงาน แต่ไม่ได้ส่งบริบทนั้นต่อให้ subagent run 1-A, 1-B และ 2-a ทุก run จึงเริ่มด้วยบริบทว่าง ไม่มี agent ตัวใดอ่าน `docs/review-board/`
