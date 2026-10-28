**ภาษา:** ภาษาไทย | [English](bonus-recover-decision.en.md)

# โบนัส (ไม่บังคับ ทำที่บ้าน): กู้คืนการตัดสินใจที่ไม่ได้บันทึกไว้ (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T5 (Dhar et al. สร้างการตัดสินใจจากโค้ดและแผนภาพเพื่อลด *การสูญหายของความรู้* หรือ knowledge vaporization), AICH4
> **คำถาม:** เมื่อ AI อธิบายว่า*ทำไม*ระบบจึงถูกสร้างแบบนี้ คำอธิบายส่วนใดมาจากหลักฐาน และส่วนใดเป็นการเดาอย่างมั่นใจ

## ลงมือทำ

เริ่มแชตหรือ session ใหม่ แล้วส่งข้อความนี้ให้ agent

```
Follow prompts/bonus-recover-decision.md
decision: Why do all services communicate synchronously over gRPC, using one shared demo.proto?
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/src/checkoutservice/main.go, system/online-boutique/src/frontend/main.go
language: Thai
```

บรรทัด `language: Thai` ทำให้ ADR ที่กู้คืนเป็นภาษาไทย หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้

agent จะเขียน ADR ที่กู้คืนมา พร้อมตาราง **Claim check** ไว้ที่ [`outputs/bonus-recovered-adr.md`](outputs/bonus-recovered-adr.md)

## ตรวจสอบตาราง claim check

เลือกข้อกล่าวอ้าง**ห้าข้อ**จากตารางของ agent โดยเป็นข้อที่ agent ระบุว่า *Evidenced* (มีหลักฐาน) อย่างน้อยสองข้อ และ *Inferred* (อนุมานเอา) อย่างน้อยสองข้อ แล้วตรวจสอบแต่ละข้อด้วยตัวเอง

| # | ข้อกล่าวอ้าง (ย่อ) | AI ระบุว่า | นิสิตระบุว่า | เหตุผล (path ของไฟล์หรือการให้เหตุผล) |
|---|---|---|---|---|
| 1 | (#12) จำนวน call ต่อ checkout ≈ 6 + 2N | Evidenced | Evidenced แต่**นับผิด** (จริง = 7 + 2N) | `checkoutservice/main.go:288-311` มี GetCart, GetProduct+Convert×N, GetQuote และ Convert ค่าส่ง; `:252-275` มี Charge, ShipOrder, EmptyCart, SendOrderConfirmation รวม 7 + 2N |
| 2 | (#10) email fail ไม่ทำให้ checkout fail แต่ service อื่น fail ทำให้ fail ทั้งหมด | Evidenced | ถูก**บางส่วน** | `main.go:263` `_ = cs.emptyUserCart(...)` ทิ้งผลของ EmptyCart ไว้ ถ้า cart fail ตอน EmptyCart checkout ยังสำเร็จ (ขัดกับ claim #11 ของ AI เอง) |
| 3 | (#14) `convertCurrency` ใช้ `context.TODO()` ทำให้ deadline/trace หลุด | Evidenced | Evidenced ✅ | `main.go:360` `.Convert(context.TODO(), …)` แทนที่จะส่ง `ctx` |
| 4 | (#21) เลือก gRPC เพื่อรองรับหลายภาษา | Inferred | Inferred ✅ (มีข้อมูลสนับสนุนนอกขอบเขต) | หลักฐาน 3 ไฟล์เป็น Go ทั้งหมด แต่ `src/` มี .cs, .js, .py, .java จริง จึงสมเหตุสมผล แต่ไม่มีเอกสารบอกว่าเป็นเหตุผลของการตัดสินใจ |
| 5 | (#22) ใช้ proto ไฟล์เดียวเพื่อแชร์ชนิดข้อมูลโดยไม่ซ้ำซ้อน | Inferred | Inferred และ**ข้อสมมติตั้งต้นผิดบางส่วน** | มีสำเนา `src/currencyservice/proto/demo.proto` และ `src/paymentservice/proto/demo.proto` ที่ต่างจาก `protos/demo.proto`; checkout import `…/checkoutservice/genproto` ที่ generate แยกต่อ service ข้อมูลจึงซ้ำซ้อนจริง |

**นับ:** จากข้อกล่าวอ้างทั้งหมด agent ระบุว่า *Evidenced* **18** ข้อ (#1–#18) ตามที่นิสิตตรวจ **16** ข้อ (#10 ถูกบางส่วน และ #12 นับผิด แม้มีหลักฐานจริงแต่ข้อสรุปคลาดเคลื่อน)

## สะท้อนคิด

1. agent ไม่ได้ติดป้ายการเดาว่าเป็น *Evidenced* ตรง ๆ เหตุผลของการตัดสินใจทุกข้อติดป้าย Inferred หรือ Assumption แต่มี claim ที่ติดป้าย Evidenced 2 ข้อที่ข้อสรุปคลาดเคลื่อนทั้งที่อ้างไฟล์ถูก ได้แก่ #12 นับ call เป็น 6 + 2N ทั้งที่จริงคือ 7 + 2N และ #10 ที่บอกว่า service อื่น fail แล้ว checkout fail ทั้งหมด ทั้งที่ผลของ EmptyCart ถูกทิ้ง ฉันตรวจโดยไล่ฟังก์ชันใน `checkoutservice/main.go:230-311` ทีละบรรทัด แปลว่าป้าย "Evidenced" รับประกันแค่ว่ามีไฟล์อ้างอิง ไม่ได้รับประกันว่าตีความถูก
2. นักพัฒนาใหม่จะเข้าใจว่าทุก service ใช้ contract `demo.proto` ร่วมกันไฟล์เดียว ทั้งที่ currencyservice และ paymentservice มีสำเนา proto ของตัวเองที่ไม่ตรงกับต้นฉบับ และแต่ละ service generate โค้ดแยกกัน การแก้ proto กลางจึงไม่ได้อัปเดตทุก service อัตโนมัติ นอกจากนี้ยังจะเข้าใจว่า frontend คุยกับ backend ด้วย gRPC ทั้งหมด ทั้งที่ shoppingassistantservice ใช้ HTTP
3. AI กู้คืน "อะไร" และ "อย่างไร" ได้ดี เช่น ลำดับ call แบบ blocking, การไม่ใช้ TLS, tracing ด้วย OpenTelemetry และการไม่มี message broker แต่ "ทำไม" มีเพียงผู้ออกแบบเดิมที่รู้ ตัวอย่างจาก ADR ของฉัน: เหตุผลที่เลือก RabbitMQ เพราะ "ทีมเคยใช้" ไม่มีทางกู้คืนได้จากโค้ด และในการประชุมยังพบว่าเคยใช้แค่บนโน้ตบุ๊ก รวมถึงการแบ่งเฟสเพราะงบ 2 สัปดาห์กับกฎ readiness ก็เป็นบริบทที่ไม่มีอยู่ในโค้ดเลย
