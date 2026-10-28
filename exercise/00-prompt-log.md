**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** ชัยชัย ปานปรีดา / 6870054321
**agent ที่ใช้:** Claude Code CLI, Cursor
**โมเดล:** Claude Opus 5.5
**แพ็กเกจที่ใช้:** Claude Pro

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code CLI + Claude Opus 5.5 | `prompts/1-recover-architecture.md` run A, evidence: `demo.proto` | `AGENTS.md`/`CLAUDE.md` (โหลดอัตโนมัติ), `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` | ✅ อยู่ในขอบเขต | 3 | ได้ 9 services แต่ไม่มี dependency และไม่เห็น frontend/Redis; ระบุ Gaps ตรงไปตรงมา; ตอนรันเปิด auto mode ไว้ |
| 2 | 1-B | Claude Code CLI + Claude Opus 5.5 | `prompts/1-recover-architecture.md` run B, evidence: `demo.proto` + `kubernetes-manifests/` | `prompts/1-…`, `demo.proto`, `kubernetes-manifests/` 13 ไฟล์ (README, kustomization + 11 yaml) และ list โฟลเดอร์ | ✅ อยู่ในขอบเขต | 4 | 17 dependencies ถูกเกือบทั้งหมด แต่ขาด frontend→shoppingassistantservice (HTTP); loadgenerator เป็น System_Ext น่าสงสัย |
| 3 | 2-a | Claude Code CLI + Claude Opus 5.5 | `prompts/2-write-adr.md` run a, context: none, requirement: R-12, language: Thai | `AGENTS.md`/`CLAUDE.md` (โหลดอัตโนมัติ), `prompts/2-write-adr.md`, `docs/adr/template.md` | ✅ อยู่ในขอบเขต (ไม่ได้เปิด scenario.md) | _ | Open questions ถามถึงปลายทางของ analytics, privacy ของข้อมูลใน event และ deciders; ตอนรันเปิด `00-prompt-log.md` ค้างไว้ใน editor (IDE ส่งชื่อไฟล์เป็น context) |
| 4 | 2-b | Claude Code CLI + Claude Opus 5.5 | `prompts/2-write-adr.md` run b, context: scenario.md + ADR-0001 + checkoutservice/main.go, language: Thai | `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go` | ✅ อยู่ในขอบเขต | _ | เลือก RabbitMQ (self-hosted บน K8s); ใช้ตัวเลข p95 500 ms, 200 orders/s, MailFast 50/s, backlog ≈150/s; dedupe แต้มด้วย order_id; Open questions ถามเรื่อง outbox, idempotency ของ MailFast, fraud scoring |
| 5 | 3 | Claude Code CLI + Claude Opus 5.5 | `prompts/3-review-adr.md` adr: `docs/adr/0002-publish-order-placed-events-via-rabbitmq.md`, language: Thai | `prompts/3-review-adr.md`, `docs/adr/0002-…`, `docs/scenario.md` | ✅ / ❌ | _ | Verdict: … ; แก้ ADR หรือไม่: ไม่ |
| 6 | | | | | | | |
| 7 | | | | | | | |
