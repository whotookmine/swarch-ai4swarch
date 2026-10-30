**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:**
**agent ที่ใช้:** เช่น GitHub Copilot (Agent mode), Gemini Code Assist, Gemini CLI, Claude Code, Codex, Cursor, Cline: …
**โมเดล:** …
**แพ็กเกจที่ใช้:** เช่น Copilot Student, Copilot Free, Gemini รุ่นฟรี: …

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | claude OPUS 5.5 | `prompts/1-…` run A | prompts/1-recover-architecture.md, system/online-boutique/protos/demo.proto | อยู่ | 2 | |
| 2 | 1-B | claude OPUS 5.5 | `prompts/1-…` run B | prompts/1-recover-architecture.md, system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/ | อยู่ | 5 | |
| 3 | 2-a | claude OPUS 5.5 | `prompts/2-…` run a | prompts/2-write-adr.md | อยู่ | 2 | |
| 4 | 2-b | claude OPUS 5.5 | `prompts/2-…` run b | prompts/2-write-adr.md, docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go | อยู่ | 5 | |
| 5 | 3 | claude OPUS 5.5 | `prompts/3-…` | prompts/3-review-adr.md, docs/adr/0002-use-rabbit-mq-for-post-order-actions.md, docs/scenario.md | อยู่ | 5 | สามารถหา Weak ได้ดีมาก |
| 6 | | | | | | | |
| 7 | | | | | | | |
