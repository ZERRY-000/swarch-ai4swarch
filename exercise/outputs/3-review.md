Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`)

# Review: ADR-0002 Publish an "OrderPlaced" event to RabbitMQ for post-order actions

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QA-1 Performance | ลูกค้าช่วง 11.11 | เรียก `PlaceOrder` 200 orders/s | peak load, ทุก component ปกติ | บันทึก order + เขียน outbox แล้วคืนผลโดยไม่รอ MailFast/loyalty/analytics | p95 `PlaceOrder` ≤ **500 ms** (ปัจจุบันประมาณ 300 ms) |
| QA-2 Availability | MailFast หรือ RabbitMQ node ล่ม | ส่งอีเมล/publish ไม่สำเร็จ | peak 200 orders/s | order ยังสำเร็จ; event ค้างใน outbox หรือ queue แล้วส่งต่อเมื่อกลับมา | order fail เพราะ email/loyalty/analytics = **0 %**; event หาย = 0 |
| QA-3 Correctness | RabbitMQ redeliver หลัง `loyaltyservice` crash ก่อน ack | `OrderPlaced` ซ้ำ | peak, email queue มี backlog | dedup ด้วย `order_id` | points ให้ **ครั้งเดียว**, ภายใน **60 s** หลัง order |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | Decision ใช้ "outbox ใน DB ของ `checkoutservice`" แต่ Context ระบุว่า `checkoutservice` **ไม่มี data store** และ Open questions ยังถามว่า "ต้องเพิ่ม data store หรือไม่" — ADR ขัดแย้งกันเอง | DB ใหม่ + relay ไม่ปรากฏใน diagram, Pros/cons, หรือ Negative consequences → ต้นทุน ops จริงของ Option 3 ถูกประเมินต่ำ (cost, operability) |
| Risk | Outbox ไม่ "transactional" จริง: เขียน outbox **หลัง** `chargeCard`/`shipOrder` ซึ่งเป็น remote call; ถ้า pod ตายระหว่างตัดบัตรกับเขียน outbox → ตัดเงินแล้วแต่ไม่มี event | ข้อความ "ไม่มี event ตกหล่น" ใน Risks เกินจริง; loyalty "ห้ามหาย" ยังไม่ถูกรับประกัน (reliability) |
| Risk | Email backlog โต 150/s ตลอด peak และ drain ที่ 50/s → เวลา drain หลัง peak ≈ **3 × ระยะ peak** แม้ไม่มี order ใหม่ (เช่น peak 1 ชม. ≈ 540,000 messages, อีเมลช้าสุดรอประมาณ 3 ชม.) อยู่บน cluster เดียวกับ loyalty | quorum queue backlog ใหญ่กิน disk/memory; ถ้าเกิด resource alarm RabbitMQ จะ **block publisher ทั้งหมด** รวมถึง relay → loyalty เกิน 60 s (availability, timeliness) |
| Risk | Retry + at-least-once ทำให้ **อีเมลซ้ำ** ได้ แต่ไม่อยู่ใน Consequences; MailFast idempotency ยังไม่รู้ | ลูกค้าได้อีเมลยืนยันซ้ำ; retry 0.5 % ยังกินโควตา 50/s (user experience) |
| Sensitivity point | ช่วง poll ของ outbox relay + prefetch/จำนวน consumer ของ loyalty | ตรงกับงบ 60 s โดยตรง; relay ช้าหรือ batch ใหญ่กินงบนี้ก่อนถึง broker |
| Sensitivity point | Concurrency ของ email consumer (ADR ใช้ 75 จาก p95 1.5 s) | ที่ p99 8 s, 75 in-flight ให้ได้แค่ประมาณ 9 emails/s ใน tail → throughput จริงขึ้นกับ timeout ต่อ call มาก |
| Trade-off point | Async ผ่าน broker: ได้ performance + availability + modifiability แต่เสีย timeliness ของอีเมล, observability และเพิ่ม component stateful 2 ตัว (RabbitMQ + checkout DB) | ทีม 4 คนต้องดูแล; ควรระบุชัดว่ายอมแลกอะไร |
| Non-risk | Queue แยกต่อ consumer + loyalty dedup ด้วย unique constraint ใน transaction เดียวกับการเพิ่ม points | backlog อีเมลไม่บล็อก loyalty ในระดับ queue; at-least-once → effectively exactly once ได้จริง |
| Non-risk | เลือก RabbitMQ แทน Kafka | 200 events/s, มี DLQ/prefetch ในตัว, 2 คนเคยใช้, self-hosted ตาม policy — เหตุผลสอดคล้องกับ scenario |

## 3. Missing or weak

- **Option ที่ขาด:** เมื่อต้องเพิ่ม DB ให้ checkout อยู่แล้ว ควรพิจารณา **DB-backed job queue** (ใช้ outbox table เป็น queue ให้ worker อ่านเอง ไม่มี broker) — ลด component ไป 1 ตัว แลกกับ fan-out/modifiability ที่น้อยกว่า
- **Pros/cons ไม่ตรงกับ Decision:** Cons ของ Option 3 ยังเขียนว่า "broker ล่มตอน publish อาจเสีย event" ซึ่ง Decision อ้างว่าแก้แล้ว; Option 2 ก็ใช้ outbox ได้เช่นกัน การเปรียบเทียบจึงไม่ยุติธรรม
- **Consequence ที่ไม่ได้ระบุ:** latency ที่เพิ่มจากการเขียน DB ใน `PlaceOrder`; อีเมลซ้ำ; event มี email ของลูกค้า (PII) ส่งถึง analytics consumer ด้วย
- **Validation ขาด:** chaos test ที่ปิด **RabbitMQ เอง** และ checkout DB/relay — ซึ่งเป็นจุดที่ outbox ต้องพิสูจน์; ไม่มี test backlog อีเมลขนาดเท่า peak จริงบน cluster
- **Claim ที่อ่อน:** Little's law ควรใช้เวลาเฉลี่ย ไม่ใช่ p95 และต้องคิด p99 8 s ด้วย

## 4. Verdict: **Accept with changes**

1. **แก้ความขัดแย้งเรื่อง outbox:** ตัดสินใจ data store ของ `checkoutservice`, เพิ่ม DB + relay ใน diagram/cons/cost, ระบุว่าเขียน outbox ตรงไหนเทียบกับ `chargeCard` และปิด open question ที่ขัดกัน
2. **กำหนดขนาด broker สำหรับ email backlog:** หาระยะ peak และ email SLA, คำนวณ disk/memory ของ quorum queue, ตั้ง queue length limit/alarm เพื่อไม่ให้ backlog อีเมลบล็อก publisher และ loyalty
3. **เพิ่ม option DB-backed queue และกันอีเมลซ้ำ** (idempotency key ของ MailFast หรือ dedup table) พร้อม chaos test ที่ปิด RabbitMQ/relay ระหว่าง load test 200 orders/s
