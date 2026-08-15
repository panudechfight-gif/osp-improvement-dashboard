# Adaptive Card for Power Automate · CM-TRS Job Tracking

แจ้งเตือน **Job ค้าง** เข้า Microsoft Teams ทุกวันเวลา **08:00 (Asia/Bangkok)**
ดึงข้อมูลจาก `JobMonitor_CMTRS_Job_Tracking.xlsx` (หรือ SharePoint List) → แสดงเป็นตาราง 8 คอลัมน์ตาม Focus ที่กำหนด

| # | Table Header | ที่มาในไฟล์ Excel |
|---|---|---|
| 1 | Sub System | คอลัมน์ B |
| 2 | Job ID | คอลัมน์ H |
| 3 | Priority | คอลัมน์ N |
| 4 | Create Time | คอลัมน์ O |
| 5 | Status | คอลัมน์ Q |
| 6 | Province Name | คอลัมน์ V |
| 7 | Site Name | คอลัมน์ AE |
| 8 | Assign to | คอลัมน์ AA |

---

## ไฟล์ในโฟลเดอร์นี้

| ไฟล์ | ใช้ทำอะไร | วางที่ไหน |
|---|---|---|
| `01-adaptive-card-power-automate.json` | **การ์ดหลัก** พร้อม expression `@{...}` | ช่อง *Adaptive Card* ของ action `Post adaptive card in a chat or channel` |
| `02-row-template.json` | แถวตาราง 8 คอลัมน์ (1 แถว = 1 Job) | ช่อง *Map* ของ action `Select` (สลับเป็น **code view** ก่อนวาง) |
| `03-adaptive-card-templating.json` | การ์ดเดียวกันแต่ใช้ `${...}` แบบ Adaptive Card Templating | วางที่ [Adaptive Cards Designer](https://adaptivecards.microsoft.com/designer) เพื่อดู/แก้ดีไซน์ |
| `04-sample-data.json` | ข้อมูลตัวอย่างจากไฟล์จริง (12 แถว) | ช่อง *Sample Data Editor* ของ Designer |
| `05-flow-expressions.md` | รวมสูตร expression ทุกตัว | อ้างอิงตอนสร้าง Flow |
| `06-adaptive-card-empty-state.json` | การ์ดกรณีไม่มีงานค้าง | สาขา `If no` ของ Condition |
| `07-row-template-mobile.json` | แถวแบบการ์ด (อ่านง่ายบนมือถือ) | ใช้แทน `02` ในช่อง *Map* |
| `preview.html` | เรนเดอร์การ์ดจริงในเบราว์เซอร์ + โหลด `.xlsx` มาทดสอบ + เช็กขนาดการ์ด | เปิดผ่าน http (`npx serve .`) |

> ⚠️ `01` และ `06` **ไม่ใช่ JSON ที่ valid ตามมาตรฐาน** เพราะมีบรรทัด `"items": @{body('Select_Job_Rows')}`
> (ค่าไม่มีเครื่องหมายคำพูดครอบ) — ตั้งใจให้เป็นแบบนี้ Power Automate จะแทนค่า expression **ก่อน** parse JSON
> จึงห้ามเอาไปวางใน Designer โดยตรง ให้ใช้ `03` แทน

---

## ขั้นตอนสร้าง Flow

### 0) เตรียมไฟล์ Excel (ข้ามได้ถ้าใช้ SharePoint List)

Excel connector ของ Power Automate อ่านได้เฉพาะข้อมูลที่เป็น **Table** เท่านั้น ไม่ใช่แค่ sheet เปล่า

1. เปิด `JobMonitor_CMTRS_Job_Tracking.xlsx` → เลือก `A1:AE118`
2. `Insert` → `Table` → ติ๊ก **My table has headers**
3. ตั้งชื่อ Table ว่า `JobTracking` (Table Design → Table Name)
4. บันทึกไฟล์ไว้บน **SharePoint / OneDrive for Business** (ห้ามเป็นเครื่อง local)

### 1) Trigger — `Recurrence`

| Field | Value |
|---|---|
| Frequency | Day |
| Interval | 1 |
| Time zone | (UTC+07:00) Bangkok, Hanoi, Jakarta |
| At these hours | 8 |
| At these minutes | 0 |

### 2) `Compose_ReportDate` และ `Compose_RunTime`

```
formatDateTime(convertTimeZone(utcNow(),'UTC','SE Asia Standard Time'),'dd MMM yyyy')
formatDateTime(convertTimeZone(utcNow(),'UTC','SE Asia Standard Time'),'dd-MMM-yy HH:mm')
```

### 3) ดึงข้อมูล — `List rows present in a table`

| Field | Value |
|---|---|
| Location / Document Library | ที่เก็บไฟล์ |
| File | `JobMonitor_CMTRS_Job_Tracking.xlsx` |
| Table | `JobTracking` |

**สำคัญ:** กด `Show advanced options` → ตั้ง **Top Count** = `5000` และเปิด **Pagination**
ไม่งั้นจะได้แค่ 256 แถวแรก

> ใช้ SharePoint List แทนได้ด้วย `Get items` — ดูตารางเทียบชื่อคอลัมน์ในหัวข้อ 9 ของ `05-flow-expressions.md`

### 4) `Filter Open Jobs`

- **From:** `value` (output ของขั้นที่ 3)
- **Condition** (สลับเป็น *Advanced mode*):

```
@not(contains(createArray('Closed','Completed','Cancelled','Resolved'), string(item()?['Status'])))
```

### 5) `Filter Critical` / `Filter Major` / `Filter Minor`

ทั้ง 3 อัน **From** = `body('Filter_Open_Jobs')`

```
@equals(item()?['Priority'], 'Critical')
@equals(item()?['Priority'], 'Major')
@not(contains(createArray('Critical','Major'), string(item()?['Priority'])))
```

### 6) `Compose_Top_Jobs` — เรียงลำดับ + ตัดยอด

```
take(union(body('Filter_Critical'), body('Filter_Major'), body('Filter_Minor')), 12)
```

ได้ Critical มาก่อน → Major → Minor แล้วเอาแค่ 12 แถว

> **ทำไมต้องตัดที่ 12?** Teams รับ Adaptive Card ได้ไม่เกิน **28 KB** ต่อใบ
> วัดจาก template ชุดนี้จริง: โครงการ์ดเปล่า 4.7 KB + แถวละ ~1.24 KB
> → 12 แถว = 19.5 KB (ปลอดภัย) · 18 แถว = 26.9 KB (เฉียด) · 20 แถว = 29.4 KB (การ์ดหาย)
>
> ไฟล์ตัวอย่างมี 117 job — ถ้าส่งหมด Flow จะขึ้น Succeeded แต่การ์ดจะไม่โผล่ใน Teams เลย
> เช็กขนาดจริงก่อนได้ที่ `preview.html`

### 7) `Select Job Rows` — สร้างแถวตาราง

- **From:** `outputs('Compose_Top_Jobs')`
- **Map:** กดปุ่มสลับเป็น **code view** (ไอคอน ⇄ มุมขวาบนของช่อง Map) แล้ววางเนื้อหา `02-row-template.json` ทั้งไฟล์

ผลลัพธ์คืออาร์เรย์ของ `ColumnSet` ที่พร้อมยัดลง body ของการ์ด

### 8) `Condition` — กันการ์ดเปล่า

```
length(body('Filter_Open_Jobs'))   is greater than   0
```

### 9) สาขา **If yes** — `Post adaptive card in a chat or channel`

| Field | Value |
|---|---|
| Post as | Flow bot |
| Post in | Channel |
| Team / Channel | ทีมและช่องที่ต้องการแจ้งเตือน |
| Adaptive Card | วางเนื้อหา `01-adaptive-card-power-automate.json` |

ก่อนวาง แก้ URL 2 จุดท้ายไฟล์ให้เป็นของจริง:

```json
"actions": [
  { "type": "Action.OpenUrl", "title": "📊 เปิด Dashboard", "url": "<URL Dashboard ของคุณ>" },
  { "type": "Action.OpenUrl", "title": "📁 เปิดไฟล์ต้นทาง (Excel / SharePoint)", "url": "<URL ไฟล์ Excel>" }
]
```

### 10) สาขา **If no** — วาง `06-adaptive-card-empty-state.json`

---

## จุดที่ผิดพลาดบ่อย

| อาการ | สาเหตุ / วิธีแก้ |
|---|---|
| Flow เขียว แต่การ์ดไม่ขึ้นใน Teams | การ์ดเกิน 28 KB → ลดจำนวนแถวใน `Compose_Top_Jobs` |
| การ์ดขึ้นแต่ตารางว่าง | ชื่อ action ในสูตรไม่ตรงกับชื่อจริง (เว้นวรรคต้องเป็น `_`) |
| ได้ข้อมูลแค่ 256 แถว | ยังไม่เปิด Pagination + Top Count ในขั้นที่ 3 |
| `Create Time` ออกมาเป็นตัวเลข เช่น `46249.26` | Excel ส่ง serial date มา → ใช้สูตรแปลงในหัวข้อ 8 ของ `05-flow-expressions.md` |
| `BadRequest: Invalid JSON` | เผลอครอบ `@{body('Select_Job_Rows')}` ด้วยเครื่องหมายคำพูด — ต้องไม่มี |
| ภาษาไทยกลายเป็น `???` | ตรวจว่าไฟล์ JSON บันทึกเป็น UTF-8 (ไม่มี BOM) |
| คอลัมน์ Assign to ว่าง (SharePoint) | เป็นฟิลด์ Person → ต้องใช้ `item()?['Assign_x0020_to']?['DisplayName']` |

---

## ทดสอบก่อนขึ้น Production

```bash
npx serve .
# เปิด http://localhost:3000/preview.html
```

หน้า preview จะ:
- เรนเดอร์การ์ดจริงด้วย `adaptivecards.js` + `adaptivecards-templating`
- ให้เลือกไฟล์ `.xlsx` ของคุณเอง (อ่านด้วย SheetJS) แล้วผูกข้อมูลจริงเข้าการ์ดทันที
- คำนวณ **ขนาดการ์ดเทียบลิมิต 28 KB** ให้เห็นก่อนส่งจริง
- สลับดูได้ทั้ง layout ตาราง 8 คอลัมน์ และแบบการ์ดสำหรับมือถือ

หรือดูดีไซน์อย่างเดียวที่ [adaptivecards.microsoft.com/designer](https://adaptivecards.microsoft.com/designer):
วาง `03-adaptive-card-templating.json` ในช่อง **Card Payload Editor** และ `04-sample-data.json` ในช่อง **Sample Data Editor**
(เลือก Host App = `Microsoft Teams`, Target version = `1.4`)

---

## ปรับแต่งเพิ่มเติม

**เปลี่ยนจำนวนแถว** — แก้เลข `12` ใน `Compose_Top_Jobs` (ตรวจขนาดที่ `preview.html` ทุกครั้ง)

**สลับเป็น layout มือถือ** — ใช้ `07-row-template-mobile.json` ในช่อง Map ของ `Select Job Rows`
แล้วลบ Container หัวตาราง (ก้อนที่มี `"style": "emphasis"` และมีคำว่า `Province Name`) ออกจาก `01`
layout นี้กินพื้นที่แค่ ~0.74 KB/แถว (12 แถว = 13.6 KB) จึงดันได้ถึง ~28 แถวโดยไม่ชนลิมิต

**แยกการ์ดตามผู้รับผิดชอบ** — เพิ่ม `Apply to each` วนตามรายชื่อใน `Assign to`
แล้วส่งด้วย `Post message in a chat` (Post as: Flow bot → Recipient: อีเมลผู้รับผิดชอบ)

**เพิ่มปุ่มกดรับงาน** — เพิ่ม `Action.Execute` ใน `actions` แล้วรับด้วย Flow อีกตัวที่ trigger จาก
`When someone responds to an adaptive card` (ต้องส่งการ์ดด้วย action
`Post adaptive card and wait for a response` แทน)
