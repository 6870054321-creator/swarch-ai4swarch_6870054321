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
| frontend → cartservice | `frontend.yaml` (CART_SERVICE_ADDR) | ✅ | env ที่ `frontend.yaml:73-74` และโค้ดเรียกจริงที่ `src/frontend/rpc.go:58,63,68` (GetCart/EmptyCart/AddItem) ผ่าน gRPC |
| checkoutservice → emailservice | `checkoutservice.yaml` (EMAIL_SERVICE_ADDR) | ✅ | `checkoutservice.yaml:63-64` และ `src/checkoutservice/main.go:380` เรียก SendOrderConfirmation แบบ synchronous gRPC |
| checkoutservice → paymentservice | `checkoutservice.yaml` (PAYMENT_SERVICE_ADDR) | ✅ | `checkoutservice.yaml:61-62` และ `main.go:370` เรียก Charge |
| recommendationservice → productcatalogservice | `recommendationservice.yaml` (PRODUCT_CATALOG_SERVICE_ADDR) | ✅ | `recommendation_server.py:131-136` สร้าง ProductCatalogServiceStub และเรียก ListProducts ที่บรรทัด 73 |
| cartservice → redis-cart | `cartservice.yaml` (REDIS_ADDR) | ✅ | `cartservice.yaml:50-51` REDIS_ADDR=redis-cart:6379 และ `:118` image redis:alpine |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B
- **ขาด dependency** frontend → shoppingassistantservice: `frontend.yaml:83-84` มี SHOPPING_ASSISTANT_SERVICE_ADDR และ `src/frontend/handlers.go:464` เรียกผ่าน **HTTP** (ไม่ใช่ gRPC) แต่ service นี้ไม่มี manifest (`kustomization.yaml:35` ถูก comment) แผนภาพจึงทำให้ดูเหมือนทุก service คุยกันด้วย gRPC ทั้งหมด
  - **container ที่น่าสงสัย:** loadgenerator ถูกวาดเป็น System_Ext แต่จริง ๆ เป็นเครื่องมือทดสอบโหลดภายใน cluster ที่ถูกปิดไว้ (`kustomization.yaml:24`) ไม่ใช่ระบบภายนอก
  - ป้ายบนลูกศรเป็นการเดาจากชื่อ RPC [Inferred] ไม่ได้ยืนยันจากโค้ด

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. Run A มีแค่ `demo.proto` ซึ่งบอกว่าแต่ละ service ให้บริการ RPC อะไร แต่ไม่บอกว่าใครเรียกใคร agent จึงวาด dependency ไม่ได้เลย และไม่เห็น frontend, Redis หรือ loadgenerator เพราะสิ่งเหล่านี้ไม่ได้ประกาศเป็น gRPC service ใน proto ข้อมูลการเชื่อมต่อจริงอยู่ใน env `*_ADDR` ของ manifest ซึ่ง Run B ได้อ่าน
2. การที่ shoppingassistantservice หายไปจะทำให้สมาชิกใหม่เข้าใจผิดมากที่สุด เพราะแผนภาพบอกเป็นนัยว่าระบบสื่อสารด้วย gRPC ล้วน ทั้งที่ frontend มีการเรียก HTTP ไปยัง service ที่ไม่มี manifest ถ้ามีคนเปิด component นี้ภายหลัง ก็จะไม่มีใครรู้ว่ามี dependency นี้อยู่
