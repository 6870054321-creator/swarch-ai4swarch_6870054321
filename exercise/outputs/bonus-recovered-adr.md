# ADR-R001: Use synchronous gRPC with one shared `demo.proto` contract for all service-to-service calls

- **Status:** Reconstructed
- **Date:** 2026-10-28 (วันที่สร้าง ADR ย้อนหลัง; วันที่ตัดสินใจจริง Not evidenced in the provided files — header ลิขสิทธิ์ระบุปี 2018 ใน `main.go` และ 2020 ใน `demo.proto`)
- **Deciders:** Not evidenced in the provided files (header ลิขสิทธิ์ระบุ "Google LLC")

## Context and problem

- Online Boutique แบ่งเป็นหลาย service: `demo.proto` นิยาม 9 gRPC service คือ `CartService`, `RecommendationService`, `ProductCatalogService`, `ShippingService`, `CurrencyService`, `PaymentService`, `EmailService`, `CheckoutService`, `AdService` (`system/online-boutique/protos/demo.proto`)
- ทุก service อยู่ใน proto file เดียว `package hipstershop` และใช้ message ร่วมกัน เช่น `Money`, `Address`, `CartItem`, `Empty` ซึ่งถูกใช้ข้ามหลาย service (`demo.proto`: `Money` ใช้ใน `Product`, `GetQuoteResponse`, `ChargeRequest`, `OrderItem`)
- `frontend` รับ request จากผู้ใช้ทาง HTTP (`gorilla/mux`, `http.ListenAndServe`) แล้วเรียก backend 7 ตัวผ่าน gRPC (`system/online-boutique/src/frontend/main.go` (`mustConnGRPC` × 7))
- `checkoutservice` เป็นตัว orchestrate การสั่งซื้อ: `PlaceOrder` เรียก cart → product catalog → shipping quote → currency → payment → shipping → cart empty → email ตามลำดับแบบ blocking (`system/online-boutique/src/checkoutservice/main.go` (`PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`))
- ปัญหาที่ต้องตัดสินใจ: service เหล่านี้ควรสื่อสารกันแบบใด และ contract ระหว่าง service ควรจัดเก็บ/แชร์อย่างไร
- **Not evidenced in the provided files:** ภาษาที่ใช้ใน service อื่นนอกจาก Go, ขนาด traffic, latency target, ทีมที่เป็นเจ้าของแต่ละ service, เหตุผลที่ผู้ออกแบบเขียนไว้

## Decision drivers

ลำดับความสำคัญเป็น **[Inferred]** ทั้งหมด เพราะไม่มีเอกสาร rationale ในไฟล์ที่อนุญาต

- **Simplicity / understandability**: ระบบเป็น demo ให้คนอ่าน call flow ได้ง่าย [Inferred]
- **Interoperability**: contract ต้องสร้าง client/server stub ได้หลายภาษา [Inferred]
- **Modifiability ของ contract**: มีแหล่ง contract เดียว ชนิดข้อมูลร่วม (`Money`, `Address`) ไม่ซ้ำซ้อน [Inferred]
- **Performance**: binary protocol, connection ใช้ซ้ำ [Inferred]
- **Observability**: trace ข้าม service ได้ (`otelgrpc` ทั้งฝั่ง client และ server — Evidenced)
- **Consistency ของ checkout flow**: frontend ต้องได้ `OrderResult` (order id, tracking id) กลับมาทันที (`PlaceOrderResponse` — Evidenced)

## Considered options

1. **Synchronous gRPC + proto file เดียว (`demo.proto`)** — ทางที่ระบบใช้จริง
2. **Synchronous gRPC + proto file แยกต่อ service** (แต่ละ service เป็นเจ้าของ contract ของตัวเอง)
3. **Synchronous REST/JSON over HTTP** (contract เป็น OpenAPI หรือไม่มี schema)
4. **Asynchronous messaging ผ่าน message broker** (เช่น RabbitMQ/Kafka) อย่างน้อยสำหรับขั้นที่ไม่ต้องรอผล เช่น email, shipping

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. gRPC + `demo.proto` เดียว | contract เดียว อ่านง่าย เห็นทั้งระบบในไฟล์เดียว; ชนิดข้อมูลร่วม (`Money`, `Address`) นิยามครั้งเดียว; strongly typed, gen stub ได้; trace ผ่าน `otelgrpc` ง่าย; ได้ผลลัพธ์ทันที (order id) | coupling สูง: แก้ message ร่วม กระทบทุก service; ทุก service ต้อง regenerate code; ไม่มี versioning แยก (`package hipstershop` ไม่มี `v1`); failure ลามแบบ chain |
| 2. gRPC + proto แยกต่อ service | ownership ชัด; versioning/deploy อิสระต่อ service; ลด blast radius ของการแก้ contract | ต้องจัดการ import/แชร์ `Money`/`Address` (common proto); มีไฟล์และ build pipeline มากขึ้น; ซับซ้อนเกินไปสำหรับ demo |
| 3. REST/JSON | debug ด้วย browser/curl ง่าย; เครื่องมือแพร่หลาย; ไม่ต้อง gen code | ไม่มี type contract บังคับ (ถ้าไม่ใช้ OpenAPI); payload ใหญ่กว่า, parse ช้ากว่า; ไม่มี health/stub มาตรฐานแบบ `grpc_health_v1` |
| 4. Async messaging | decouple เวลา: email/shipping ล่มไม่ทำให้ checkout fail; buffer ช่วง spike; availability ดีขึ้น | เพิ่ม infrastructure (broker) ต้อง operate; eventual consistency, ตอบ order id/tracking id ทันทียาก; trace และ debug ยากขึ้น; ต้องจัดการ idempotency/retry |

## Decision

เลือก **Option 1**: ทุก service สื่อสารแบบ synchronous request/response ผ่าน gRPC และใช้ contract เดียวคือ `protos/demo.proto`

- สิ่งที่ Evidenced: ทุก dependency ใน `frontend` และ `checkoutservice` สร้างด้วย `grpc.NewClient` (`mustConnGRPC`) และเรียก method ผ่าน client ที่ generate จาก `pb` (`genproto`) เช่น `pb.NewCartServiceClient(...).GetCart`; ไม่มี import ของ message broker ใน 2 ไฟล์นี้
- เหตุผล (**[Inferred]**): Online Boutique เป็นระบบ demo จึงให้ความสำคัญกับความเรียบง่ายและการเห็นทั้งระบบในที่เดียวมากกว่า ownership/versioning อิสระ; gRPC ให้ typed contract ข้ามภาษา; synchronous ทำให้ `PlaceOrder` คืน `OrderResult` ครบในครั้งเดียว และ trace ต่อเนื่องได้ด้วย `otelgrpc`

## Consequences

- **Positive:** contract เดียวทำให้เห็น API ทั้งระบบและชนิดข้อมูลร่วมในไฟล์เดียว (quality attribute: understandability / modifiability ของ contract ส่วนรวม)
- **Positive:** typed stub จาก proto ลด error การ serialize ระหว่าง service (quality attribute: reliability / interoperability)
- **Positive:** trace context ส่งต่อทุก hop ผ่าน `otelgrpc.NewClientHandler()`/`NewServerHandler()` และ `propagation.TraceContext{}` (quality attribute: observability)
- **Positive:** `checkoutservice` คืน `order_id` + `shipping_tracking_id` ให้ผู้ใช้ได้ทันที (quality attribute: usability / consistency)
- **Negative:** `PlaceOrder` เป็น chain ของ call แบบ blocking ประมาณ 6 + 2N call (N = จำนวนสินค้า: `GetProduct` + `Convert` ต่อชิ้นใน `prepOrderItems`) latency รวมเป็นผลบวกของทุก hop (quality attribute: performance)
- **Negative:** ถ้า cart, catalog, currency, shipping หรือ payment ล่ม checkout fail ทั้งหมด (`return nil, status.Errorf(...)`) มีเพียง email ที่ถูก tolerate (`log.Warnf`) (quality attribute: availability)
- **Negative:** แก้ message ร่วม เช่น `Money` ต้อง regenerate และ redeploy ทุก service ที่ใช้; ไม่มี version ใน package name (quality attribute: modifiability / deployability)
- **Negative:** ใช้ `insecure.NewCredentials()` ระหว่าง service — ไม่มี TLS ที่ระดับ application (quality attribute: security) — mTLS จาก service mesh ภายนอก Not evidenced in the provided files
- **Risks and mitigations:**
  - *Payment สำเร็จแต่ shipping fail* → ลูกค้าถูกตัดเงินแต่ไม่ได้ order (`chargeCard` ก่อน `shipOrder`, ไม่มี compensation) — mitigation: saga/refund หรือย้าย shipping เป็น async + retry [Inferred]
  - *ไม่มี timeout/retry ต่อ call*: `mustConnGRPC` สร้าง timeout 3 วินาทีแต่ไม่ได้ใช้ ctx นั้น (`_, cancel := ...`), `convertCurrency` ใช้ `context.TODO()` ทำให้ deadline และ trace ของ request หลุด — mitigation: ส่ง `ctx` จริง, ตั้ง deadline ต่อ call, ใช้ gRPC retry policy
  - *Breaking change ของ `demo.proto`* — mitigation: ใช้กฎ proto3 (ไม่ reuse field number; สังเกตว่า `PlaceOrderRequest` ข้าม field 4) และเพิ่ม `buf breaking` ใน CI [Inferred]

## Validation

- Load test checkout flow วัด p95/p99 latency ของ `PlaceOrder` เทียบจำนวนสินค้าในตะกร้า (ยืนยันผลกระทบ 2N call) [Inferred]
- Fault injection: ปิด `currencyservice` หรือ `shippingservice` แล้วดูอัตรา checkout fail และกรณี payment สำเร็จแต่ shipping fail
- ตรวจ distributed trace ว่า span ของ `CurrencyService/Convert` ต่อกับ trace ของ `PlaceOrder` หรือไม่ (เพราะใช้ `context.TODO()`)
- ตั้ง breaking-change check ของ `demo.proto` ใน CI และ review ADR นี้ใหม่เมื่อมีทีม/ภาษาที่ต้อง version contract แยก
- Not evidenced in the provided files: มี load test, SLO หรือ CI check เหล่านี้อยู่แล้วหรือไม่

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | `demo.proto` นิยาม 9 gRPC service ในไฟล์เดียว `package hipstershop` | Evidenced | `system/online-boutique/protos/demo.proto` (`package hipstershop`, `service CartService` … `service AdService`) |
| 2 | message ร่วม (`Money`, `Address`, `CartItem`, `Empty`) ถูกใช้ข้ามหลาย service | Evidenced | `demo.proto` (`Product.price_usd`, `GetQuoteRequest.address`, `ChargeRequest.amount`, `ListProducts(Empty)`) |
| 3 | `frontend` รับผู้ใช้ทาง HTTP และเรียก backend 7 ตัวทาง gRPC | Evidenced | `system/online-boutique/src/frontend/main.go` (`mux.NewRouter`, `http.ListenAndServe`, `mustConnGRPC` × 7) |
| 4 | `frontend` อ่าน `SHOPPING_ASSISTANT_SERVICE_ADDR` แต่ไม่สร้าง gRPC connection ไปยัง service นี้ จึงไม่ทราบ protocol | Evidenced | `frontend/main.go` (`shoppingAssistantSvcAddr`, ไม่มี `mustConnGRPC` คู่กัน) |
| 5 | `checkoutservice` เรียก 6 service แบบ synchronous ตามลำดับใน `PlaceOrder` | Evidenced | `system/online-boutique/src/checkoutservice/main.go` (`PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`, `chargeCard`, `shipOrder`) |
| 6 | ไม่มีการใช้ message broker ในไฟล์ evidence | Evidenced | import list ของ `checkoutservice/main.go` และ `frontend/main.go` (มีแค่ `grpc`, `otel`, `mux`, `logrus` ฯลฯ) |
| 7 | ใช้ client stub ที่ generate จาก proto (`genproto`) | Evidenced | `checkoutservice/main.go` (`pb ".../checkoutservice/genproto"`, `pb.NewCartServiceClient`) |
| 8 | trace ส่งต่อข้าม service ผ่าน `otelgrpc` และ W3C TraceContext | Evidenced | `checkoutservice/main.go` (`otelgrpc.NewServerHandler`, `mustConnGRPC` → `otelgrpc.NewClientHandler`, `propagation.TraceContext{}`); `frontend/main.go` (`otelhttp.NewHandler`) |
| 9 | ใช้ `insecure.NewCredentials()` ระหว่าง service | Evidenced | `mustConnGRPC` ในทั้ง 2 ไฟล์ |
| 10 | การส่ง email fail ไม่ทำให้ checkout fail แต่ service อื่น fail ทำให้ fail ทั้งหมด | Evidenced | `checkoutservice/main.go` (`sendOrderConfirmation` → `log.Warnf`; ขั้นอื่น `return nil, status.Errorf`) |
| 11 | ผลของ `EmptyCart` ถูกทิ้ง | Evidenced | `checkoutservice/main.go` (`_ = cs.emptyUserCart(...)`) |
| 12 | จำนวน call ต่อ checkout ≈ 6 + 2N | Evidenced | `checkoutservice/main.go` (`prepOrderItems` loop: `GetProduct` + `convertCurrency` ต่อชิ้น) |
| 13 | payment ถูก charge ก่อน shipping โดยไม่มี compensation | Evidenced | `checkoutservice/main.go` (`chargeCard` ก่อน `shipOrder`; error path คืน `codes.Unavailable` อย่างเดียว) |
| 14 | `convertCurrency` ใช้ `context.TODO()` ทำให้ deadline/trace ของ request หลุด | Evidenced | `checkoutservice/main.go` (`convertCurrency`) |
| 15 | timeout 3 วินาทีใน `mustConnGRPC` ไม่ได้ถูกใช้ | Evidenced | `mustConnGRPC` (`_, cancel := context.WithTimeout(...)`) |
| 16 | `PlaceOrderResponse` คืน order id + tracking id แบบ synchronous | Evidenced | `demo.proto` (`PlaceOrderResponse`, `OrderResult`) |
| 17 | proto ไม่มี version ใน package name | Evidenced | `demo.proto` (`package hipstershop`) |
| 18 | `PlaceOrderRequest` ข้าม field number 4 | Evidenced | `demo.proto` (`email = 5` หลัง `address = 3`) — สาเหตุ Not evidenced |
| 19 | เวลาที่ตัดสินใจราว 2018–2020 โดย Google LLC | Inferred | copyright header ใน 3 ไฟล์ (ปีลิขสิทธิ์ไม่ใช่วันที่ตัดสินใจ) |
| 20 | เหตุผลหลักคือความเรียบง่ายสำหรับระบบ demo | Inferred | ไม่มี rationale ในไฟล์; `go_package = ".../microservices-demo/..."` บอกแค่ว่าเป็น demo |
| 21 | เลือก gRPC เพื่อรองรับหลายภาษา | Inferred | service ใน evidence เป็น Go ทั้งหมด; ภาษาอื่น Not evidenced in the provided files |
| 22 | ใช้ proto file เดียวเพื่อแชร์ชนิดข้อมูลร่วมโดยไม่ซ้ำซ้อน | Inferred | โครงสร้าง `demo.proto` สอดคล้อง แต่ไม่มีเหตุผลเขียนไว้ |
| 23 | Option 2, 3, 4 ถูกพิจารณาตอนตัดสินใจ | Inferred | ไม่มีบันทึก; ผู้ reconstruct เสนอเป็นทางเลือกเปรียบเทียบ |
| 24 | gRPC/protobuf มี payload เล็กและเร็วกว่า REST/JSON | Assumption | ความรู้ทั่วไปในอุตสาหกรรม |
| 25 | async messaging ช่วย availability แต่เพิ่ม eventual consistency และภาระ operate broker | Assumption | ความรู้ทั่วไปในอุตสาหกรรม |
| 26 | proto แยกต่อ service ช่วย ownership และ versioning อิสระ | Assumption | ความรู้ทั่วไปในอุตสาหกรรม |
| 27 | synchronous chain ทำให้ latency สะสมและ failure ลาม | Assumption | ความรู้ทั่วไป; ตัว chain เองเป็น Evidenced ใน claim #5 |
| 28 | mitigation: saga/refund, deadline ต่อ call, retry policy, `buf breaking` ใน CI | Assumption | แนวปฏิบัติทั่วไป; ไม่มีในระบบตาม evidence (การมี service mesh/mTLS ภายนอก: Not evidenced in the provided files) |
