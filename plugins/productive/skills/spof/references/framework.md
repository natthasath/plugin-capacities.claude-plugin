# หลักการวิเคราะห์ Single Point of Failure (SPOF)

SPOF คือจุดใดก็ตามในระบบที่ **ไม่มีการสำรอง (redundancy)** — ถ้าจุดนั้นล้มเหลว ระบบทั้งหมดหรือส่วนสำคัญของระบบ
จะหยุดทำงานทันที ไม่ใช่แค่ฮาร์ดแวร์เท่านั้น แต่รวมถึง network, data, third-party dependency, และแม้แต่ตัวคน

---

## หมวดหมู่ SPOF ทั้ง 6 ด้าน

### 1. Infrastructure (โครงสร้างพื้นฐาน)
- Server/VM เครื่องเดียวรัน service สำคัญ ไม่มี cluster หรือ failover
- Power supply, UPS, หรือ cooling system จุดเดียวใน data center
- Storage/disk เดียวไม่มี RAID หรือ replication
- Data center หรือ availability zone เดียว ไม่มี multi-AZ/multi-region

### 2. Network
- ISP หรือ internet uplink เส้นเดียว
- Router, Switch, หรือ Firewall ตัวเดียวที่ทุก traffic ต้องผ่าน
- Load Balancer ตัวเดียวไม่มี HA pair
- DNS provider รายเดียว ไม่มี secondary DNS

### 3. Database / Data
- Database instance เดียว ไม่มี replica หรือ standby
- ไม่มี automated backup หรือไม่เคยทดสอบ restore จริง
- Data ทั้งหมดอยู่ที่เดียว ไม่มี off-site/off-region copy
- Schema migration ที่ทำโดยคนเดียวไม่มี review หรือ rollback plan

### 4. Application / Software
- Service เดียวรับ load ทั้งหมด ไม่มี horizontal scaling
- Monolith ที่ shared state ทำให้ scale ไม่ได้อิสระ
- Session/state เก็บใน memory เครื่องเดียว (ไม่ externalize)
- Hardcoded config/credential ที่ผูกกับ environment เดียว

### 5. Third-party / Vendor
- Payment gateway, SMS/email provider, หรือ external API รายเดียวไม่มี fallback
- Cloud provider หรือ region เดียว (vendor lock-in)
- SSL/TLS Certificate หรือ Domain registration ที่ไม่มีระบบเตือนวันหมดอายุ
- License software ที่ต่ออายุโดยคนเดียวหรือ credit card ใบเดียว

### 6. Human / Process (Bus Factor)
- มีคนเดียวที่รู้วิธี deploy/restart/rollback ระบบ
- Credential, SSH key, หรือสิทธิ์ admin กระจุกอยู่ที่คนเดียว
- ไม่มี documentation หรือ runbook เมื่อคนสำคัญลาออก/ลาป่วย
- Approval process ที่ต้องผ่านคนคนเดียวเท่านั้น (single approver)

---

## เกณฑ์ให้คะแนนความเสี่ยง

ความเสี่ยง = **ผลกระทบ (Impact) × โอกาสเกิด (Likelihood)**

| Impact | ความหมาย |
|--------|----------|
| สูง | ระบบหยุดทั้งหมด กระทบผู้ใช้ทุกคน หรือข้อมูลสูญหายถาวร |
| กลาง | บาง feature ใช้ไม่ได้ หรือ performance ลดลงมาก แต่ยังใช้งานหลักได้ |
| ต่ำ | กระทบเฉพาะจุด มี workaround หรือกระทบผู้ใช้ส่วนน้อย |

| Likelihood | ความหมาย |
|------------|----------|
| สูง | เกิดขึ้นได้บ่อยหรือมีประวัติเคยเกิดมาแล้ว (เช่น hardware อายุมาก, dependency ที่ไม่เสถียร) |
| กลาง | มีความเป็นไปได้ตามวงจรปกติ (เช่น certificate หมดอายุปีละครั้ง, คนลาออก) |
| ต่ำ | โอกาสเกิดน้อย ต้องมีเหตุการณ์ผิดปกติร่วมด้วย |

**ระดับความเสี่ยงรวม (Impact × Likelihood matrix):**

| Impact \ Likelihood | ต่ำ | กลาง | สูง |
|----------------------|-----|------|-----|
| **สูง** | 🟡 ปานกลาง | 🔴 วิกฤต | 🔴 วิกฤต |
| **กลาง** | 🟢 ต่ำ | 🟡 ปานกลาง | 🔴 วิกฤต |
| **ต่ำ** | 🟢 ต่ำ | 🟢 ต่ำ | 🟡 ปานกลาง |

| ระดับ | การจัดการ |
|-------|-----------|
| 🔴 วิกฤต (Critical) | แก้ไขเร่งด่วน ควรอยู่ใน sprint ถัดไป |
| 🟡 ปานกลาง (Moderate) | วางแผนแก้ไขใน roadmap ระยะสั้น-กลาง |
| 🟢 ต่ำ (Low) | ติดตามไว้ ยังไม่ต้องเร่งแก้ |

---

## แนวทางแก้ไขมาตรฐาน (Mitigation Patterns)

| Pattern | ใช้แก้ SPOF ประเภทไหน | ตัวอย่าง |
|---------|------------------------|----------|
| **Redundancy / Clustering** | Infrastructure, Network | Server อย่างน้อย 2 เครื่อง, HA pair ของ Firewall/LB |
| **Replication / Standby** | Database | Read replica, Streaming replication, Multi-AZ RDS |
| **Load Balancing** | Application | กระจาย traffic หลาย instance พร้อม health check |
| **Multi-region / Multi-cloud** | Infrastructure, Vendor | กระจาย workload ข้าม region หรือมี DR site |
| **Automated Backup + Tested Restore** | Database, Data | Backup อัตโนมัติ + ซ้อม restore จริงเป็นระยะ |
| **Fallback Provider** | Third-party/Vendor | สำรอง SMS/Email/Payment gateway รายที่สอง |
| **Monitoring + Alerting** | ทุกหมวด | แจ้งเตือนก่อน certificate หมดอายุ, disk เต็ม, service down |
| **Documentation / Runbook** | Human/Process | เขียนขั้นตอน deploy/rollback ให้อย่างน้อย 2 คนทำเป็น |
| **Cross-training / Access Sharing** | Human/Process | กระจาย credential/สิทธิ์ให้มากกว่า 1 คน พร้อม access control ที่เหมาะสม |
| **Circuit Breaker / Graceful Degradation** | Application, Vendor | เมื่อ dependency ล่ม ระบบยังทำงานบางส่วนได้แทนที่จะ crash ทั้งหมด |

---

## Pitfall ที่พบบ่อยตอนวิเคราะห์ SPOF

- **มองแค่ hardware** — ลืมว่า vendor รายเดียว, คนคนเดียว, หรือ certificate ใบเดียวก็เป็น SPOF
- **มี redundancy แต่ไม่เคยทดสอบ failover จริง** — ระบบสำรองที่ไม่เคย test ก็เท่ากับไม่มี
- **แก้ SPOF แล้วสร้าง SPOF ใหม่** — เช่น เพิ่ม load balancer แต่มีตัวเดียว กลายเป็น SPOF ตัวใหม่แทน
- **ไม่ประเมิน cost-benefit** — บางระบบขนาดเล็กไม่คุ้มที่จะทำ full redundancy ทุกจุด ต้องดู business impact จริงประกอบ
