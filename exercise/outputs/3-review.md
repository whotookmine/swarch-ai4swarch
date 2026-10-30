> Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`)

# Review: ADR-0002 — Publish an "order placed" event to RabbitMQ for post-order actions

ผู้ตรวจ: architecture review board (แนว ATAM) · หลักฐานที่ใช้: ADR-0002 และ `docs/scenario.md` เท่านั้น (เลขบรรทัดใน `checkoutservice/main.go` ที่ ADR อ้างถึง ผู้ตรวจไม่ได้ตรวจสอบเอง)

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QA1 (Performance) | ลูกค้า | กด place order | ช่วง 11.11 ที่ **200 orders/s**, MailFast ตอบช้า (p99 **8 s**) | `PlaceOrder` publish `OrderPlaced` แล้วตอบกลับโดยไม่รอ MailFast | p95 `PlaceOrder` ≤ **500 ms** (ปัจจุบันประมาณ 300 ms ⇒ งบสำหรับ publish + confirm ประมาณ 200 ms) |
| QA2 (Availability/Reliability) | RabbitMQ node หรือ `loyaltyservice` | ล่ม / เข้าถึงไม่ได้ | peak load | order ยังสำเร็จ; event ไม่สูญหายและถูกประมวลผลเมื่อระบบกลับมา | order ล้มเหลว **0** รายการ; loyalty ได้แต้ม **ภายใน 1 นาที** และ **exactly once** (double award = 0, ขาดหาย = 0) |
| QA3 (Reliability) | `emailservice` | ส่งอีเมล 200/s เข้า MailFast ที่จำกัด **50/s** และล้มเหลวประมาณ **0.5 %** | peak load | buffer ในคิว, throttle, retry | อัตราส่งจริง ≤ 50/s เสมอ; อีเมลทุกฉบับถูกส่งหรือเข้า DLQ; ไม่ส่งซ้ำให้ลูกค้า |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| **Risk** | Dual-write: ตัดบัตรแล้วค่อย publish; ถ้า publish ล้มเหลว ADR เลือก "log แล้วตอบ success" | event หายถาวร ⇒ loyalty ไม่ได้แต้ม ซึ่งขัดกับ "exactly once" ใน QA2 ตรงๆ และไม่มีกลไกกู้คืน (ไม่มี outbox, `checkoutservice` ไม่มี database) |
| **Risk** | Retry ไป MailFast แบบ at-least-once แต่ไม่รู้ว่า MailFast รองรับ idempotency key หรือไม่ | กรณี timeout ที่จริงๆ ส่งสำเร็จแล้ว จะทำให้ลูกค้าได้อีเมลซ้ำ ที่ 200 orders/s × 0.5 % ≈ 1 ฉบับ/s ที่ต้อง retry |
| **Risk** | "Global throttle ≤ 50/s" ไม่ระบุวิธีทำเมื่อ `emailservice` มีหลาย replica | ถ้า throttle ต่อ pod รวมกันจะเกิน 50/s; ถ้าตั้งไว้ที่ 50 พอดี จะไม่มีที่เหลือให้ retry |
| **Sensitivity point** | Publish timeout + latency ของ publisher confirm บน quorum queue (replicate 3 node) | p95 `PlaceOrder` ไวต่อค่านี้โดยตรง; ADR ยอมรับเองว่ายังไม่ได้วัด **[Inferred]** |
| **Sensitivity point** | ระยะเวลา peak ของ 11.11 | backlog อีเมล = 150 × วินาทีของ peak (เช่น peak 1 ชม. ⇒ 540,000 ฉบับ, ใช้เวลา drain ประมาณ 3 ชม. ที่ 50/s) ซึ่งกำหนดขนาด disk/memory ของ broker และ email delay |
| **Trade-off point** | Publisher confirms + retry publish ภายใน latency budget | ยิ่ง retry/รอนาน reliability ของ event ยิ่งดี แต่ performance (QA1) ยิ่งแย่ |
| **Trade-off point** | "Log and continue" เมื่อ publish ล้มเหลว | availability ของ checkout ได้ (driver 1) แต่ความถูกต้องของ loyalty เสีย (driver 3) |
| **Non-risk** | แยก durable queue ต่อ consumer | backlog อีเมลไม่บล็อก loyalty จึงทำ 1-minute SLO ได้แม้ MailFast ช้า |
| **Non-risk** | Loyalty dedup ด้วย unique constraint บน order ID ในทรานแซกชันเดียวกับการเพิ่มแต้ม | at-least-once + idempotent consumer ให้ผล exactly-once ฝั่ง consumer ได้จริง |
| **Non-risk** | เลือก RabbitMQ แทน Kafka | 200 msg/s ต่ำมากสำหรับทั้งคู่; ทีมมีประสบการณ์ RabbitMQ 2/4 คน, self-hosted ตาม constraint |

## 3. Missing or weak

- **Claim ไม่ถูกต้อง:** "the only option that meets drivers 1–3 together" — option 3 ตามที่ตัดสินใจไว้ **ยังไม่ผ่าน** driver 3 (loyalty exactly once) เพราะ event หายได้ตอน publish ล้มเหลว ADR ยอมรับเรื่องนี้ใน Risks แต่ข้อสรุปใน *Why* ไม่สอดคล้องกัน
- **Option ที่ไม่ได้พิจารณา:** (a) RabbitMQ + **transactional outbox** (เพิ่ม store เล็กๆ ให้ `checkoutservice`) เป็น option แยก ไม่ใช่แค่ "separate decision"; (b) **RabbitMQ Streams** ซึ่งให้ replay ได้ ทำให้ข้อได้เปรียบหลักของ Kafka ที่ ADR ยกมาลดลง; (c) NATS JetStream (open-source, เบากว่า)
- **Consequence ที่ไม่ได้ระบุ:** loyalty message ที่เข้า DLQ จะเกิน 1 นาทีแน่นอน ต้องมีขั้นตอน redrive; การย้าย `sendOrderConfirmation` ออกทำให้ response ของ checkout ไม่บอกสถานะอีเมลอีกต่อไป; schema/versioning ของ `OrderPlaced` เมื่อมี consumer เพิ่มปีหน้า
- **Validation อ่อน:** load test ไม่ได้รวม **broker fault injection** (kill RabbitMQ node ระหว่าง publish) ซึ่งเป็นความเสี่ยงอันดับหนึ่ง และไม่มีเกณฑ์ "event หาย = 0"
- **Email delay:** ไม่มี target แต่ backlog อาจยาวหลายชั่วโมง; ควรเสนอ target ชั่วคราวให้ business ยืนยัน

## 4. Verdict: **Accept with changes**

1. **ปิดช่อง event หาย:** เพิ่ม option "RabbitMQ + outbox" แล้วตัดสินใจใน ADR นี้ หรือระบุชัดว่ายอมรับการสูญหาย (ด้วยตัวเลข) และแก้ข้อความ "meets drivers 1–3" ให้ตรงความจริง
2. **กำหนดงบ latency ของ publish ให้เป็นตัวเลข** (เช่น timeout ≤ 100 ms รวม retry) และเพิ่ม validation: kill broker node ที่ 200 orders/s ⇒ p95 ≤ 500 ms, failed orders = 0, lost events = 0
3. **ออกแบบ email consumer ให้ชัด:** global rate limiter ที่ตั้งต่ำกว่า 50/s (เผื่อ retry), idempotency key ต่อ order ID เพื่อกันอีเมลซ้ำ, และขนาด backlog/drain time จากสมมติฐาน peak duration ที่ระบุไว้
