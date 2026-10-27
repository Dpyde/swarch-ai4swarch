**ภาษา:** ภาษาไทย | [English](README.en.md)

# AI for Software Architecture: แบบฝึกหัดลงมือทำ

**2110521 Software Architecture · สัปดาห์ที่ 12 · Tutorial 5 / Assignment 5**

ในแบบฝึกหัดนี้ นิสิตจะใช้ **AI coding agent ใน VS Code** เป็นผู้ช่วยด้านสถาปัตยกรรมซอฟต์แวร์ กับระบบ microservices จริงชื่อ [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo) โดยใช้ agent ตัวใดก็ได้ที่นิสิตมี เช่น GitHub Copilot, Gemini, Claude, Codex, Cursor หรืออื่น ๆ แล้วประเมินว่างานที่ AI ทำออกมานั้นดีแค่ไหน เป้าหมายไม่ใช่ให้ AI ทำงานแทน แต่เพื่อค้นหาว่า **AI ช่วยสถาปนิกซอฟต์แวร์ได้ตรงไหน และตรงไหนที่ยังเชื่อถือไม่ได้** ซึ่งเป็นข้อสรุปหลักของการบรรยายสัปดาห์นี้

> ⚠️ **ยังไม่ต้องเปิด repository ต้นฉบับของ Online Boutique จนกว่าจะทำงานที่ 1 เสร็จ** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมของระบบอยู่ ซึ่งเป็นสิ่งที่นิสิตต้องกู้คืนและตรวจสอบด้วยตัวเอง ทุกอย่างที่ต้องใช้อยู่ในโฟลเดอร์ `system/online-boutique/` ของ repository นี้แล้ว นิสิตสามารถเปรียบเทียบกับแผนภาพต้นฉบับได้หลังจากส่งงานแล้ว

แบบฝึกหัดแบ่งเป็นสองส่วน

- **ส่วน A: ลงมือทำกับ AI agent (ประมาณ 40 นาที, ทำคนเดียว)**
  1. **กู้คืน (recover) สถาปัตยกรรม** จากซอร์สโค้ด (หัวข้อ T7: architecture recovery)
  2. **เขียน ADR** สำหรับความต้องการใหม่ร่วมกับ AI ทั้งแบบมีและไม่มีบริบท แล้วปรับให้เป็นฉบับของตัวเอง (T3: ADR generation)
  3. **ให้ AI รีวิว** ADR ของนิสิตแบบ ATAM (P5: AI ในบทบาท Chief Analyst)
- **ส่วน B: บทบาทสมมติ คณะกรรมการรีวิวสถาปัตยกรรม (ประมาณ 25 นาที, กลุ่มละ 4 คน)**
  - นำ ADR ของสมาชิกหนึ่งคนเข้าที่ประชุม **สถาปนิก** ปกป้อง ADR, **AI analyst** นำเสนอผลรีวิวของ AI, ส่วน **product owner** และ **SRE** นำข้อมูลที่ AI ไม่เคยเห็นมาด้วย คณะกรรมการตัดสินใจ และบันทึกว่าข้อกังวลใดที่ AI จับได้และข้อใดที่ AI พลาด

เวลาที่ใช้โดยประมาณ: ราว 75 นาทีในคาบเรียน

---

## ก่อนเข้าเรียน: เตรียมเครื่อง (ประมาณ 15 นาที ควรทำล่วงหน้า)

1. **ติดตั้ง [VS Code](https://code.visualstudio.com/)** หากไม่มี VS Code ในเครื่อง ให้ข้ามขั้นตอนนี้และใช้ GitHub Codespaces แทน (ดูขั้นตอนที่ 3)
2. **เตรียม AI agent ที่อ่านและเขียนไฟล์ใน workspace ได้** ใช้ตัวใดตัวหนึ่งต่อไปนี้ก็ได้

   | Agent | ใช้งานที่ไหนใน VS Code | ตัวเลือกฟรีสำหรับนิสิต |
   |---|---|---|
   | **GitHub Copilot** ในโหมด **Agent** | หน้าต่าง Chat | Copilot Student ผ่าน [GitHub Education](https://docs.github.com/en/copilot/how-tos/copilot-on-github/set-up-copilot/enable-copilot/set-up-for-students) (การอนุมัติอาจใช้เวลาหลายวัน) หรือ Copilot Free |
   | **Gemini Code Assist** ในโหมด agent | แผง Gemini | รุ่นฟรีด้วยบัญชี Google ส่วนตัว |
   | **Gemini CLI** | terminal ใน VS Code | รุ่นฟรีด้วยบัญชี Google |
   | **Claude Code**, **Codex**, **Cursor**, **Cline** ฯลฯ | extension, terminal หรือตัว editor | ขึ้นอยู่กับเครื่องมือ |

   การแชตธรรมดาที่เปิดไฟล์ไม่ได้จะใช้ไม่ได้ เพราะ agent ต้องอ่านโค้ดและเขียนผลลัพธ์ลงไฟล์
3. **Fork repository นี้**
   - กดปุ่ม **Fork** (มุมขวาบนของหน้านี้) → **Create fork** fork ของนิสิตจะเป็นแบบ public ซึ่งจำเป็นสำหรับการส่งงาน
   - clone fork ของตัวเองลงเครื่อง แล้วเปิดโฟลเดอร์ใน VS Code
   - **ไม่มี VS Code ในเครื่อง?** ที่หน้า fork ของตัวเอง กด **Code** → **Codespaces** → **Create codespace on main** ระบบจะเปิด VS Code บนเบราว์เซอร์พร้อม fork ของนิสิต และใช้ GitHub Copilot Chat (โหมด Agent) ได้ทันที ให้ทำขั้นตอนที่เหลือใน codespace นี้ บัญชี GitHub ส่วนตัวมีชั่วโมงใช้งาน Codespaces ฟรีทุกเดือน เมื่อทำเสร็จให้หยุด (stop) codespace

     <img src="docs/img/codespaces.png" alt="repository ของแบบฝึกหัดที่เปิดใน GitHub Codespaces แสดง README, แผง Copilot Chat ในโหมด Agent และ terminal" width="560">
4. **ติดตั้ง extension สำหรับ Mermaid ที่แนะนำทั้งสองตัว** เมื่อ VS Code ถาม (หรือค้นหาในหน้าต่าง Extensions)
   - **Markdown Preview Mermaid Support** (`bierner.markdown-mermaid`): แสดงแผนภาพใน Markdown preview
   - **Mermaid** โดย Mermaid (`MermaidChart.vscode-mermaid-chart`): extension ทางการของ Mermaid ใช้ดูตัวอย่างและแก้ไขแผนภาพ และแจ้งข้อผิดพลาดของ syntax

     <img src="docs/img/mermaid-extensions.png" alt="หน้าต่าง Extensions ใน VS Code ที่ค้นหาคำว่า Mermaid แสดง extension Mermaid ทางการ และ Markdown Preview Mermaid Support" width="560">

5. **ตรวจว่า agent เห็นกติกาของ repo นี้** เปิดแชตหรือ session ใหม่ แล้วถามว่า *"What are the rules for AI agents in this repository?"* agent ควรสรุปเนื้อหาจาก [`AGENTS.md`](AGENTS.md) ได้ หากไม่ได้ ให้ขึ้นต้นข้อความของทุกงานด้วย `Read AGENTS.md first.`

## แบบฝึกหัดนี้ใช้กับ agent ใดก็ได้อย่างไร

- **กติกาอยู่ใน [`AGENTS.md`](AGENTS.md)** ซึ่ง agent ส่วนใหญ่อ่านอัตโนมัติ และมีไฟล์ชี้ทางขนาดเล็กให้เครื่องมือที่หาไฟล์ชื่ออื่นโหลดกติกาเดียวกัน ได้แก่ `GEMINI.md` สำหรับ Gemini, `CLAUDE.md` สำหรับ Claude Code และ `.github/copilot-instructions.md` สำหรับ Copilot
- **โจทย์แต่ละงานเป็นไฟล์ Markdown ธรรมดาในโฟลเดอร์ [`prompts/`](prompts/)** ใบงานจะบอกข้อความที่ต้องส่งให้ agent ไว้ครบ เช่น

  ```
  Follow prompts/1-recover-architecture.md
  run: A
  evidence: system/online-boutique/protos/demo.proto
  ```

- **agent จะเขียนผลลัพธ์ไว้ใน `exercise/outputs/`** นิสิตตรวจไฟล์นั้นแล้วจึงยอมรับการเปลี่ยนแปลง หาก agent พยายามแก้ไขไฟล์อื่น ให้ปฏิเสธหรือ undo
- **agent ต้องไม่** แก้ไข `system/`, `prompts/`, ตัวอย่าง ADR หรือใบงานของนิสิต และต้องไม่รันคำสั่งใน terminal หาก agent ขอรันคำสั่ง ให้ปฏิเสธ

เนื่องจากทุกคนใช้ไฟล์โจทย์และกติกาชุดเดียวกัน จึงเปรียบเทียบผลจาก agent ต่างตัวกันได้

**ห้ามให้ agent อ่านโฟลเดอร์ `docs/review-board/`** เพราะการ์ดบทบาทเหล่านี้ใช้ในส่วน B และประเด็นสำคัญคือ AI ไม่เคยเห็นข้อมูลในนั้น

## โควตาการใช้งาน

ส่วน A ใช้ประมาณ **5 คำขอ (agent requests)** ส่วน B ไม่ต้องใช้ AI

- คำขอแต่ละครั้งของ agent จะอ่านไฟล์และเขียนผลลัพธ์ จึงใช้โควตามากกว่าการแชตทั่วไป **ยิ่งอ่านไฟล์น้อยยิ่งประหยัด**
- **เริ่มแชตหรือ session ใหม่ทุกครั้งที่ทำ run ใหม่** เพราะบทสนทนาเก่าจะถูกส่งไปพร้อมทุกคำขอ
- หากโควตาฟรีหมดระหว่างทำ สามารถเปลี่ยนไปใช้ agent ตัวอื่นได้ โดยบันทึกไว้ใน prompt log

## กติกา

- **บันทึกทุกคำขอ** ไว้ใน [`exercise/00-prompt-log.md`](exercise/00-prompt-log.md) ได้แก่ ใช้ agent และโมเดลใด ถามอะไร agent อ่านไฟล์ใด และอยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่
- **ตรวจสอบคำกล่าวอ้างกับโค้ดจริง** คำกล่าวอ้างที่ไม่มี path ของไฟล์เป็นหลักฐาน ถือเป็นการเดาจนกว่านิสิตจะตรวจสอบเอง
- **อย่าวางคำตอบของ AI เป็นข้อสรุปของตัวเอง** โดยไม่ระบุที่มา ในใบงานแยกช่องสำหรับ *คำตอบของ AI* กับ *ความเห็นของนิสิต* ไว้แล้ว

## งานที่ต้องทำ

| ส่วน | # | ใบงาน | ไฟล์โจทย์ | จำนวนคำขอ | เวลา |
|---|---|---|---|---|---|
| A | 1 | [กู้คืนสถาปัตยกรรม](exercise/01-recover-architecture.md) | `prompts/1-recover-architecture.md` | 2 | 15 นาที |
| A | 2 | [เขียน ADR สำหรับความต้องการใหม่](exercise/02-write-adr.md) | `prompts/2-write-adr.md` | 2 | 20 นาที |
| A | 3 | [ให้ AI รีวิว ADR](exercise/03-ai-review.md) | `prompts/3-review-adr.md` | 1 | 5 นาที |
| B | 4 | [คณะกรรมการรีวิวสถาปัตยกรรม](exercise/04-review-board.md) | (การ์ดบทบาทใน `docs/review-board/`) | 0 | 25 นาที |
| | 5 | [สะท้อนคิดรายบุคคล](exercise/05-reflection.md) | | 0 | ทำหลังคาบได้ |
| | ★ | [โบนัส: กู้คืนการตัดสินใจที่ไม่ได้บันทึกไว้](exercise/bonus-recover-decision.md) | `prompts/bonus-recover-decision.md` | 1 | ไม่บังคับ ทำที่บ้าน |

> ใบงานทุกฉบับมีทั้งภาษาไทย (`*.md`) และภาษาอังกฤษ (`*.en.md`) **ให้กรอกเพียงภาษาเดียว** นิสิตตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้ ส่วนข้อความที่ส่งให้ agent ให้ใช้ภาษาอังกฤษตามที่กำหนด

## การส่งงาน (Assignment 5 ส่งภายในท้ายคาบ)

1. กรอกใบงาน 00–05 ในโฟลเดอร์ `exercise/` (ภาษาเดียวต่อใบงาน) รวมถึงบันทึกผลการประชุมคณะกรรมการของกลุ่ม เก็บไฟล์ที่ agent สร้างไว้ใน `exercise/outputs/` และเขียน ADR ฉบับสุดท้ายไว้ที่ `docs/adr/0002-*.md`
2. commit และ push ขึ้น repository ของตัวเอง
3. ส่ง URL ของ repository ใน MCV โดยเปิดให้ repository เป็นแบบ public

## โครงสร้างของ repository

```
AGENTS.md                   กติกาสำหรับ AI agent ทุกตัว (+ ไฟล์ชี้ทาง GEMINI.md, CLAUDE.md, .github/copilot-instructions.md)
prompts/                    ไฟล์โจทย์ที่ให้ agent ทำตาม
docs/
  adr/template.md           รูปแบบ ADR ที่ใช้ในวิชา
  adr/0001-…                ตัวอย่าง ADR (ใช้ในงานที่ 2)
  scenario.th.md            บริบททางธุรกิจสำหรับงานที่ 2 (ภาษาอังกฤษ: scenario.md)
  review-board/             การ์ดบทบาทสำหรับส่วน B (สำหรับมนุษย์เท่านั้น)
exercise/                   ใบงาน (นิสิตแก้ไขไฟล์เหล่านี้)
  outputs/                  ไฟล์ที่ AI agent เขียน
system/online-boutique/     ระบบที่ใช้วิเคราะห์ (ฉบับตัดทอน)
```

## ที่มาและลิขสิทธิ์

ระบบที่ใช้ศึกษาคือ [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo) ของ Google ภายใต้ Apache License 2.0 ดูสิ่งที่เปลี่ยนแปลงสำหรับแบบฝึกหัดนี้ได้ที่ [`system/README.md`](system/README.md)

การออกแบบแบบฝึกหัดอ้างอิงจาก Bucaioni et al., *AI Support for Software Architecture Practice: A Systematic Review and Future Directions*, ACM TOSEM 35(9), 2026
