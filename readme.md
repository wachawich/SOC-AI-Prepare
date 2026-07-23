# พรีเซนต์: SOC-AI-Prepare — จาก Raw FortiSOAR Data สู่ SOC-aware LLM Prompt

> ไฟล์นี้คือโครงเนื้อหาสำหรับพรีเซนต์ แต่ละ `##` = 1 หัวข้อ/สไลด์ ใต้หัวข้อคือ talking point ที่ใช้พูดได้เลย พร้อมตัวเลข/หลักฐานจริงจากการรันจริง (ไม่ใช่ mock)

---

## 1. เล่าปัญหาก่อน (Why)

พูดเปิดด้วยปัญหาที่มีอยู่จริงในทีม ก่อนเข้าเรื่องเทคนิค:

- ทีมมี pipeline ที่ส่ง alert เข้า LLM (Ollama `qwen2.5:7b` local) เพื่อช่วย SOC analyst วิเคราะห์เบื้องต้นอยู่แล้ว (`data/input.json` → `data/output.json`)
- แต่ prompt ที่ใช้อยู่ตอนนี้ถูกออกแบบมาสำหรับ **firewall log บรรทัดเดียว** เท่านั้น:

  ```
  "raw_log": "May 06 09:12:33 fw01 DENY TCP 192.168.1.50:54321 -> 45.33.32.156:4444 flags:SYN"
  "prompt": "You are a SOC analyst. Analyze this firewall log. Determine: 1) Is this C2 traffic?
             2) Severity (CRITICAL/HIGH/MEDIUM/LOW) 3) IOCs 4) Recommended actions."
  ```

- คำถามที่ตั้งไว้: **alert ที่มาจาก FortiSOAR จริง (Splunk ES notable, AWS CloudTrail ฯลฯ) จะใช้ prompt นี้ได้ไหม?** และถ้าไม่ได้ ต้องแก้อะไร
- งานที่ทำจึงแบ่งเป็น 2 เฟส ตรงกับ 2 notebook: **(1) เข้าใจข้อมูลจริงก่อน (2) ปรับ prompt ตามปัญหาที่เจอ แล้ว "พิสูจน์" ด้วยการรันจริงเทียบผล**

---

## 2. Phase 1 — ทำความเข้าใจข้อมูลจริง (`data-prepare.ipynb`)

พูดถึงงานสำรวจข้อมูลตัวอย่างจาก FortiSOAR (`data/mockup.json`, 20 records)

### สิ่งที่เจอ
- ข้อมูลมี **2 แหล่งปนกัน**: Splunk ES notable events (18 records, สังเกตจาก field `search_name`) และ AWS CloudTrail management events (2 records, field `eventVersion`)
- Splunk ES notable **ไม่ใช่ log บรรทัดเดียว** — เป็น JSON structured document ที่มี field ไม่เท่ากันในแต่ละ rule (9 ชนิด rule ต่างกัน เช่น DNS excessive queries, endpoint outbreak, DDoS, suspicious IP callback)
- มี field คุณค่าสูงที่ prompt เดิมไม่เคยใช้: **MITRE ATT&CK mapping** (`annotations.mitre_attack*`), **severity ที่ SIEM ประเมินไว้แล้ว**, **security domain** (endpoint/network/threat/change)
- **พบปัญหาคุณภาพข้อมูลที่เป็นรูปธรรม** (สำคัญมาก จะเชื่อมกับ Phase 2):
  - timestamp เป็น epoch string ต้อง cast เอง
  - `queue_id` เป็น string `"None"` ไม่ใช่ JSON null
  - **field `annotations` เพี้ยน**: บาง record มีค่าเป็น `"annotations": "{\\\\"` ซึ่งเป็น JSON ที่ export ไม่สมบูรณ์ (ตัดกลางคัน) — ตอนนั้นเราตั้งข้อสังเกตไว้ว่า "ต้องเผื่อ error handling ตอน parse หรือดร็อป field นี้ทิ้ง" **(สปอยล์: ข้อนี้กลับมาเป็นจริงตอนทดลองจริงใน Phase 2!)**
  - severity ใน mock มีแค่ 2 ระดับ (medium/high) ไม่ครอบคลุมทุกกรณี

### ทำไมส่วนนี้สำคัญกับพรีเซนต์
- นี่คือหลักฐานว่า **เราไม่ได้เดา** ว่า prompt เดิมมีปัญหาตรงไหน แต่ไปดูข้อมูลจริงก่อนแล้วค่อยสรุปเป็น requirement — เป็นวิธีการทำงานแบบ data-driven ไม่ใช่ assumption-driven

---

## 3. Phase 2 — เหตุผล 5 ข้อที่ต้องปรับปรุง prompt (`prompt-tuning.ipynb` ส่วนที่ 1)

แปลงข้อค้นพบจาก Phase 1 เป็นปัญหาของ prompt เดิมแบบเจาะจง — พูดทีละข้อ พร้อมยกตัวอย่างจากข้อมูลจริง:

| # | ปัญหาของ prompt เดิม | หลักฐานจากข้อมูลจริง |
|---|---|---|
| 1 | สมมติว่า alert เป็น raw log บรรทัดเดียวเสมอ | notable event ของ Splunk ES เป็น JSON หลาย field — ถ้ายัดเป็น string เดียวจะทิ้ง MITRE ATT&CK, security domain, severity เดิมไปหมด |
| 2 | ไม่บังคับ output format | คำตอบเดิม (`data/output.json`) เป็น markdown ยาวเป็นพารากราฟ — ระบบอัตโนมัติ (FortiSOAR playbook) parse ต่อไม่ได้แบบ deterministic |
| 3 | ไม่ anchor กับ severity ที่ SIEM ประเมินไว้แล้ว | ปล่อยโมเดลเดา severity เองอิสระ ตรวจสอบย้อนหลังไม่ได้ว่าเห็นด้วย/ไม่เห็นด้วยกับ SIEM ตรงไหน |
| 4 | ไม่ถามเรื่อง false positive / noise | หลาย rule ชื่อขึ้นต้นด้วย "TUNED" (เช่น Excessive DNS Queries ไป 8.8.8.8 ซ้ำๆ) ซึ่งมักเป็น noise ที่ SOC tune ทิ้งอยู่แล้ว — prompt เดิมไม่เปิดช่องให้บอกเลย |
| 5 | ไม่แยกมุมมองตาม security domain | endpoint (malware) / network (C2, DDoS) / threat (intel correlation) / change (insider) ต้องการคำถามวิเคราะห์ต่างกัน แต่ prompt เดิมใช้คำถามเดียวกับทุกแบบ |

---

## 4. Phase 2 — ออกแบบ prompt ใหม่ (SOC-aware, structured)

เล่าหลักการออกแบบ ไม่ใช่แค่โชว์ prompt แต่อธิบาย "ทำไมต้องเป็นแบบนี้"

### 4.1 รับ input เป็น "alert context" แบบยืดหยุ่น
- ฟังก์ชัน `build_alert_context()` แปลง record ได้ทั้ง 2 แบบ: firewall raw log (แบบเดิม) และ Splunk ES notable (แบบใหม่) ให้กลายเป็น JSON context เดียวกันก่อนส่งเข้าโมเดล
- แปลว่า prompt เดียวใช้ได้กับข้อมูลทุกแหล่ง ไม่ต้องเขียน prompt แยกทีละ integration

### 4.2 บังคับ output เป็น JSON schema ตายตัว
- ใช้ `response_format: {"type": "json_object"}` ของ OpenRouter
- Schema มีทั้ง `severity_assessment`, `severity_agrees_with_siem` (+เหตุผล), `false_positive_likelihood` (+เหตุผล), `mitre_attack[]`, `iocs{}` (แยก ip/domain/hash/user), `recommended_actions[]`, `summary`
- **จุดสำคัญ**: นี่คือสิ่งที่ทำให้เอาไปต่อกับ FortiSOAR playbook ได้จริง (set severity field, auto-tag false positive, auto-suggest action) โดยไม่ต้อง regex เดาจาก markdown

### 4.3 กฎป้องกัน hallucination (สำคัญมากสำหรับ security use case)
- สั่งชัดเจนว่าให้ดึง IOC จาก field ที่ให้มาเท่านั้น ห้ามสร้าง IP/hash/user ใหม่ที่ไม่มีในข้อมูล
- ถ้ามี MITRE ATT&CK มาให้แล้ว (`annotations.mitre_attack*`) ให้ **validate/normalize** ไม่ใช่เดาใหม่ — ป้องกันโมเดลมั่วเทคนิคที่ไม่มีหลักฐาน

---

## 5. Phase 2 — ทดลองจริง: เลือก 3 เคสที่ตอบโจทย์ต่างกัน

อธิบายว่าเลือก test case ยังไงไม่ใช่สุ่ม แต่เลือกให้ครอบคลุมสถานการณ์:

1. **`baseline_firewall_log`** (จาก `data/input.json` เดิม) — เช็คว่า prompt ใหม่ไม่ regression กับ input แบบเดิมที่ทีมใช้อยู่
2. **`suspicious_ip_callback`** (UC-CAT-012, มี src/dest IP, threat_score, MITRE T1041 ผูกมาแล้ว) — เช็คความสามารถดึง IOC + validate MITRE ATT&CK ที่มีอยู่แล้ว
3. **`excessive_dns_queries_tuned`** (ชื่อ rule มีคำว่า "TUNED", DNS query ไป 8.8.8.8 กว่า 4 แสนครั้ง) — เช็คว่าโมเดลจับสัญญาณ "นี่อาจเป็น noise" ได้ไหม (จุดที่ prompt เดิมทำไม่ได้เลย)

พูดเสริม: เพื่อให้ fair กับ prompt เดิม ยังจำลอง `to_old_style()` ที่พยายามฝืนยัด notable event เข้ากรอบ `raw_log` เดิม (เอาแค่ `orig_rule_description` มาใส่) — เพื่อโชว์ว่าต่อให้ฝืนใช้ integration เดิม ก็ยังเสียข้อมูลไปเยอะเมื่อเทียบกับที่ prompt ใหม่ส่งให้ครบ

---

## 6. ผลการทดลองจริง (หลักฐานจากการรัน OpenRouter จริง)

> โมเดลที่ใช้ทดสอบจริง: `qwen/qwen3.5-9b` ผ่าน OpenRouter — นี่คือผลจริง ไม่ใช่ mock

### เคส 1 — baseline_firewall_log

| | Old prompt | New prompt |
|---|---|---|
| Parse เป็น JSON ได้ไหม | ❌ ไม่ได้ (เป็น markdown report ยาว) | ✅ ได้ |
| Severity | "HIGH" (อยู่ในข้อความ ต้องอ่านเอง) | `HIGH`, confidence=`high` |
| False positive | ไม่ได้ถาม | `low` — ให้เหตุผลว่า port 4444 คือ default port ของ Metasploit reverse shell |
| MITRE ATT&CK | ไม่ได้ถาม | `T1571 / TA0011 — Remote Services` |
| IOC | อยู่ในตาราง markdown | `ips: [192.168.1.50, 45.33.32.156]`, ports: `4444` (structured, ดึงต่อได้ทันที) |
| Latency | 18.77s | 82.03s |

**พูดตรงนี้**: prompt ใหม่ให้ผลลัพธ์ที่ "แม่นเท่าเดิมหรือดีกว่า" (จับ Metasploit/port 4444 ได้เหมือนกัน) แต่อยู่ในรูปที่ **เอาไปเขียนโปรแกรมต่อได้ทันที** ต่างจาก markdown ที่ต้องให้คนอ่านเอง

### เคส 2 — suspicious_ip_callback

| | Old prompt | New prompt |
|---|---|---|
| Parse เป็น JSON ได้ไหม | ❌ ไม่ได้ | ✅ ได้ |
| Severity vs SIEM | พูดถึงในข้อความ แต่ไม่มี field เทียบชัดเจน | `HIGH`, **`severity_agrees_with_siem: true`** — โมเดลบอกชัดว่าเห็นด้วยกับ SIEM (severity เดิมคือ high) พร้อมอ้างอิง `threat_score` และ threat-intel source ที่แนบมา |
| False positive | พูดถึง "อาจเป็น GitHub Pages ที่ถูกต้อง" ในพารากราฟ | `medium` — ระบุชัดว่า IP `185.199.109.153` อยู่ใน range ของ GitHub Pages/CDN ซึ่งอาจเป็น traffic ปกติ **นี่คือ nuance ที่มีคุณค่ามาก เพราะไม่ fix severity ไปทางเดียว แต่ให้เหตุผลทั้งสองด้าน** |
| MITRE ATT&CK | ไม่ได้ validate กับของเดิม | `T1041 — Exfiltration Over C2 Channel` ตรงกับ `annotations.mitre_attack` ที่ Splunk ผูกมาให้อยู่แล้ว (โมเดล validate ไม่ได้เดาใหม่) |
| IOC | อยู่ในตาราง markdown | `ips: [185.199.109.153, 8.8.8.8, 9.9.9.9]` — ตรงกับ field `dest`/`src` ในข้อมูลจริงเป๊ะ ไม่มี IP ที่ไม่มีอยู่จริงหลุดเข้ามา (ไม่ hallucinate) |
| Latency | 26.04s | 100.24s |

**พูดตรงนี้**: นี่คือเคสที่โชว์คุณค่าเชิงคุณภาพชัดที่สุด — โมเดล "เห็นด้วยกับ SIEM แบบมีเหตุผลอ้างอิงได้" และ "ให้ความน่าจะเป็น false positive แบบ nuanced" ซึ่ง prompt เดิมไม่มีโครงสร้างให้ตอบแบบนี้เลย

### เคส 3 — excessive_dns_queries_tuned (เคสที่ "ล้มเหลว" แต่มีค่ามาก)

| | Old prompt | New prompt |
|---|---|---|
| ผลลัพธ์ | `None` (ไม่มี content กลับมา) | `None` (ไม่ใช่ JSON ที่ parse ได้) |
| Latency | **1715.11s (~28.5 นาที)** | **1233.25s (~20.5 นาที)** |

**นี่คือจุดที่พรีเซนต์ได้ทรงพลังที่สุด**: record นี้คือตัวเดียวกับที่ Phase 1 (`data-prepare.ipynb`) เคยตั้งข้อสังเกตไว้ล่วงหน้าแล้วว่ามี **field `annotations` ที่เพี้ยน** (`"annotations": "{\\\\"` — JSON export ไม่สมบูรณ์) ตอนวิเคราะห์ข้อมูลตอนนั้นเราเขียนไว้ว่า *"ต้องเผื่อ error handling ตอน parse หรือดร็อป field นี้ทิ้ง"* — พอเอาไปทดลองยิง LLM จริง ก็เจอปัญหาจริงตามที่คาดไว้ (latency พุ่งผิดปกติ, content ว่างเปล่า)

**สรุป**: นี่คือหลักฐานว่าขั้นตอน "ทำความเข้าใจข้อมูลก่อน" ใน Phase 1 ไม่ใช่แค่ทฤษฎี แต่ **ทำนายปัญหาที่เกิดขึ้นจริงในโปรดักชันได้ล่วงหน้า**

---

## 7. สรุปคุณค่าที่ส่งมอบ (สำหรับสไลด์สรุป)

พูดเป็นข้อสรุปสั้นๆ กระชับ ปิดท้าย:

1. **Parseability**: จาก 0% → 100% (ในเคสที่ได้ผลลัพธ์กลับมา) ทำให้ต่อกับ FortiSOAR playbook ได้จริงโดยไม่ต้อง regex เดา
2. **Calibration กับ SIEM**: prompt ใหม่บอกได้ว่า "เห็นด้วยหรือไม่เห็นด้วยกับ severity ที่ SIEM ให้มา พร้อมเหตุผล" — เพิ่มความน่าเชื่อถือและ auditability
3. **ลด alert fatigue**: เพิ่มความสามารถประเมิน false positive/noise ซึ่ง prompt เดิมไม่เคยทำได้ — ตรงกับ pain point จริงของ SOC ที่เจอ rule ประเภท "TUNED" ซ้ำๆ จำนวนมาก
4. **ป้องกัน hallucination**: IOC และ MITRE ATT&CK ที่ได้มาตรงกับข้อมูลต้นทางเป๊ะ ไม่มีการสร้างข้อมูลใหม่ที่ไม่มีหลักฐาน — สำคัญมากสำหรับงาน security ที่ผลลัพธ์ผิดอาจนำไปสู่การ block/investigate ผิดเป้า
5. **ค้นพบ (ไม่ใช่แค่ทำนาย) ปัญหาข้อมูลจริงที่ต้องแก้ก่อนขึ้นโปรดักชัน**: malformed `annotations` field ทำให้ latency พุ่งและ LLM ไม่ตอบ — เป็น action item ที่ชัดเจนและวัดผลได้

---

## 8. Known limitations / ความเสี่ยงที่ต้องพูดตรงๆ (ทำให้พรีเซนต์น่าเชื่อถือขึ้น)

อย่าพูดแต่ข้อดี ให้พูดข้อจำกัดด้วยเพื่อความน่าเชื่อถือ:

- **Latency ของ prompt ใหม่สูงกว่า prompt เดิม 3-4 เท่า** ในเคสที่สำเร็จ (82s vs 18.77s, 100s vs 26s) เพราะ context ยาวกว่าและต้อง reasoning หลายมิติ — ต้อง trade-off กับ SLA ของ SOC ที่ต้องการความไว
- **โมเดล free-tier ไม่เสถียร**: เคส DNS queries ใช้เวลากว่า 20-28 นาทีแล้วไม่ได้ผลลัพธ์เลยทั้งสอง prompt — ถ้าจะขึ้นโปรดักชันจริงต้องมี timeout ที่สั้นกว่านี้มาก + fallback (เช่น retry ด้วย local Ollama หรือ escalate ให้ analyst ตรวจเอง) + sanitize input ก่อนส่ง (แก้ที่ต้นตอ: clean field `annotations` ที่เพี้ยนก่อน)
- **ทดสอบแค่ 3 เคสจาก mock data 20 records** — ยังไม่ครอบคลุมทุก rule/domain (เช่น ยังไม่ทดสอบ change domain, AWS CloudTrail record) ต้องขยายชุดทดสอบก่อนสรุปเป็น production-ready

---

## 9. Next steps (ปิดท้ายพรีเซนต์ด้วย roadmap)

- แก้ sanitize `annotations` field (และ field เพี้ยนอื่นๆ ที่เจอใน `data-prepare.ipynb`) ก่อนส่งเข้า LLM ทุกครั้ง
- เพิ่ม timeout ที่สมเหตุสมผล (เช่น 30-60s) + fallback strategy เมื่อ LLM ไม่ตอบ/ตอบช้าเกินไป
- ขยายชุดทดสอบให้ครบทั้ง 9 ชนิด rule + 4 security domain + AWS CloudTrail
- นำ `build_alert_context()` ไปเป็น normalizer กลางของ pipeline จริง เชื่อมกับ FortiSOAR playbook
- ตั้ง metric ติดตาม `severity_agrees_with_siem` และ `false_positive_likelihood` ระยะยาว เพื่อ calibrate prompt เพิ่มเติมตามพฤติกรรมจริง
- พิจารณาเทียบ cost/latency ระหว่าง OpenRouter free-tier กับ Ollama local (`qwen2.5:7b`) ที่ทีมใช้อยู่เดิม ว่าแบบไหนเหมาะกับ production มากกว่า
