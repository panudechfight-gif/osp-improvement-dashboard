# CM-TRS Job Tracking → Adaptive Card for Power Automate

Template Adaptive Card + คู่มือทำ Flow ใน Power Automate แบบ Step by Step
สำหรับส่งสรุปงาน CM-TRS เข้า Microsoft Teams อัตโนมัติทุกวัน

---

## 📁 ไฟล์ในโฟลเดอร์นี้

| ไฟล์ | ใช้ทำอะไร | ใช้ที่ไหน |
|---|---|---|
| `01-card-template.json` | การ์ดแบบ **Templating syntax** (`${...}`, `$data`) | ใช้ดูตัวอย่าง/ออกแบบใน [Adaptive Cards Designer](https://adaptivecards.io/designer/) |
| `02-sample-data.json` | ข้อมูลตัวอย่าง (ดึงจากไฟล์ Excel จริง 29 job) | วางใน **Sample Data Editor** ของ Designer |
| `03-select-map.json` | JSON สำหรับ action **Select** (สร้างแถวตาราง) | วางในช่อง **Map** ของ Select |
| `04-card-powerautomate.json` | การ์ดฉบับ **ใช้งานจริงใน Flow** (ฝัง expression `@{...}`) | วางใน **Compose** ชื่อ `Card_JSON` |

---

## ⚠️ อ่านก่อน 1 ข้อ (สำคัญมาก มือใหม่พลาดตรงนี้บ่อยที่สุด)

**Power Automate ไม่มี Adaptive Card Templating Engine**

แปลว่า ถ้าคุณเอา `01-card-template.json` ไปวางใน action ของ Teams ตรง ๆ
การ์ดจะแสดงตัวอักษร `${jobs}` `${$data['Job ID']}` ออกมาดิบ ๆ ไม่มีข้อมูล

👉 วิธีที่ถูกต้องคือ **ให้ Flow สร้าง JSON ของแถวตารางเอง** ด้วย action `Select`
แล้วยัดเข้าไปในการ์ด — นี่คือสิ่งที่ไฟล์ `03` + `04` ทำ

| ไฟล์ | Syntax | เอาไปใช้กับ |
|---|---|---|
| `01-card-template.json` | `${jobs}` | Designer, Bot Framework, Power Apps |
| `04-card-powerautomate.json` | `@{body('Select_Rows')}` | **Power Automate ← ใช้อันนี้** |

---

## 🎯 คอลัมน์ที่แสดงบนการ์ด (ตาม Focus ที่กำหนด)

| # | Header | คอลัมน์ใน Excel | น้ำหนักความกว้าง |
|---|---|---|---|
| 1 | Sub System | `Sub System` | 12 |
| 2 | Job ID | `Job ID` | 14 |
| 3 | Priority | `Priority` | 10 (มีสี + ไอคอน) |
| 4 | Create Time | `Create Time` | 12 |
| 5 | Status | `Status` | 10 (มีสี) |
| 6 | Province Name | `Province Name` | 9 |
| 7 | Site Name | `Site Name` | 20 |
| 8 | Assign to | `Assign to` | 13 |

> คอลัมน์อื่นในไฟล์ Excel (31 คอลัมน์) จะถูกดึงมาด้วยแต่ **ไม่แสดงบนการ์ด**
> ถ้าอยากเพิ่ม/ลดคอลัมน์ ให้แก้ทั้ง `03-select-map.json` (แถวข้อมูล) และ `04-card-powerautomate.json` (แถวหัวตาราง) ให้ตรงกัน

### กติกาสีที่ใช้

**Priority**

| ค่า | ไอคอน | สี |
|---|---|---|
| Critical | 🔴 | `attention` (แดง) |
| Major | 🟠 | `warning` (ส้ม) |
| อื่น ๆ / Minor | 🟡 | `default` |

**Status** (ตามค่าที่พบจริงในไฟล์)

| ค่า | สี | ความหมาย |
|---|---|---|
| `Held` | `attention` แดง | งานค้าง — ต้องตาม |
| `Transfered` | `warning` ส้ม | โอนงานออก |
| `On-Site` / `Departed` | `good` เขียว | ทีมเคลื่อนแล้ว |
| `Accepted` | `accent` น้ำเงิน | รับงานแล้ว |
| `Assigned` (default) | `default` | เพิ่งมอบหมาย |

---

## 🧪 ขั้นที่ 0 — ลองดูหน้าตาการ์ดก่อน (5 นาที ไม่ต้องมี Flow)

1. เปิด <https://adaptivecards.io/designer/>
2. มุมขวาบน เลือก Host App = **Microsoft Teams**
3. กดปุ่ม **Open** → เลือกไฟล์ `01-card-template.json` (หรือ copy เนื้อไฟล์ไปวางในช่อง `Card Payload Editor`)
4. ที่ช่องล่าง **Sample Data Editor** → วางเนื้อไฟล์ `02-sample-data.json`
5. กด **Preview mode** → จะเห็นการ์ดพร้อมข้อมูลจริง 15 แถว

ถ้าหน้าตาโอเคแล้ว ค่อยไปทำ Flow

---

## 🔧 ขั้นเตรียมไฟล์ Excel (ทำครั้งเดียว)

1. เปิด `JobMonitor_CMTRS_Job_Tracking.xlsx`
2. คลุมข้อมูลทั้งหมดรวมหัวตาราง (A1 ถึงคอลัมน์สุดท้าย) → กด **Ctrl + T** → ติ๊ก *My table has headers* → OK
3. ไปแท็บ **Table Design** → ตั้งชื่อ Table ว่า **`JobTable`**
4. อัปโหลดไฟล์ขึ้น **SharePoint** หรือ **OneDrive for Business**
   (ห้ามเก็บไว้บนเครื่อง — Power Automate มองไม่เห็น)

> **⚠️ เรื่อง Create Time:** connector Excel จะคืนค่าวันที่เป็น "ตัวเลข serial" (เช่น `46265.69`) ถ้าเซลล์นั้นเป็นชนิด Date
> ไฟล์ที่แนบมาเก็บเป็น **ข้อความ** อยู่แล้ว (`15-Aug-26 16:43`) จึงใช้ได้ทันที
> ถ้าไฟล์จริงของคุณเป็น Date ให้เพิ่มคอลัมน์ช่วยใน Excel:
> `=TEXT([@[Create Time]],"dd-mmm-yy hh:mm")` แล้วชี้ Flow มาที่คอลัมน์ใหม่นี้แทน

---

## 🚀 คู่มือทำ Flow — Step by Step

> **กฎเหล็ก:** ต้อง **เปลี่ยนชื่อ action ให้ตรงเป๊ะ** ตามที่เขียนไว้ทุกขั้น
> เพราะ expression ในไฟล์ `04` อ้างอิงชื่อพวกนี้ เช่น `body('Filter_Open')`
> วิธีเปลี่ยนชื่อ: คลิกจุดสามจุด `…` บน action → **Rename**
> (เว้นวรรคในชื่อจะกลายเป็น `_` อัตโนมัติ — พิมพ์ `Filter Open` ก็ได้ ระบบจะอ่านเป็น `Filter_Open`)

### Step 1 — สร้าง Flow

1. เข้า <https://make.powerautomate.com>
2. **Create** → **Scheduled cloud flow**
3. Flow name: `CM-TRS Daily Job Monitor`
4. Starting: วันนี้ / เวลา `08:00` — Repeat every `1` `Day` → **Create**

### Step 2 — ตั้ง Time zone ให้ trigger

คลิก action **Recurrence** → **Show advanced options**

| ช่อง | ค่า |
|---|---|
| Time zone | `(UTC+07:00) Bangkok, Hanoi, Jakarta` |
| At these hours | `8` |
| At these minutes | `0` |

### Step 3 — ดึงข้อมูลจาก Excel

**+ New step** → ค้นหา `Excel Online (Business)` → เลือก **List rows present in a table**

| ช่อง | ค่า |
|---|---|
| Location | SharePoint Site / OneDrive for Business |
| Document Library | Documents |
| File | เลือกไฟล์ `JobMonitor_CMTRS_Job_Tracking.xlsx` |
| Table | `JobTable` |

จากนั้นคลิก `…` บน action นี้ → **Settings** → เปิด **Pagination** = On → Threshold = `5000` → Done
(ถ้าไม่เปิด จะดึงได้แค่ 256 แถว)

✅ ชื่อ action: ปล่อยไว้ตามเดิม `List rows present in a table`

### Step 4 — กรองเอาเฉพาะงานที่ยังไม่ปิด

**+ New step** → `Data Operation` → **Filter array**
→ Rename เป็น **`Filter Open`**

| ช่อง | ค่า |
|---|---|
| From | เลือก dynamic content **value** (จาก Step 3) |

จากนั้นกดปุ่ม **Edit in advanced mode** ใต้เงื่อนไข แล้ววาง:

```
@and(not(equals(item()?['Status'],'Closed')),not(equals(item()?['Status'],'Cancelled')))
```

### Step 5 — นับจำนวนแยกตาม Priority (3 actions)

ทำ **Filter array** เพิ่มอีก 3 อัน แบบเดียวกัน ทุกอันตั้ง **From** = expression:

```
body('Filter_Open')
```

| Rename เป็น | Edit in advanced mode ใส่ |
|---|---|
| `Filter Critical` | `@equals(item()?['Priority'],'Critical')` |
| `Filter Major` | `@equals(item()?['Priority'],'Major')` |
| `Filter Minor` | `@equals(item()?['Priority'],'Minor')` |

> 💡 วิธีเร็ว: ทำอันแรกเสร็จ แล้วกด `…` → **Copy to my clipboard** → **+ New step** → **My clipboard** → วาง → แก้เงื่อนไข + Rename

### Step 6 — เรียงลำดับ Critical มาก่อน Major

Power Automate ไม่มี action "Sort" — เราใช้วิธี **ต่อ array ตามลำดับที่ต้องการ** แทน

**+ New step** → `Data Operation` → **Compose** → Rename เป็น **`Sorted Jobs`**

ช่อง Inputs ใส่ expression:

```
union(union(body('Filter_Critical'),body('Filter_Major')),body('Filter_Minor'))
```

### Step 7 — ตัดเอาเฉพาะ N รายการแรก

**+ New step** → **Compose** → Rename เป็น **`Top Jobs`**

Inputs ใส่ expression (เลข `15` = จำนวนแถวที่จะโชว์ ปรับได้):

```
take(outputs('Sorted_Jobs'),15)
```

> **อย่าใส่เกิน 20 แถว** — Teams จำกัดขนาดการ์ดที่ ~28 KB การ์ดจะไม่ส่งถ้าใหญ่เกิน

### Step 8 — สร้างข้อความสรุป Status

**+ New step** → **Compose** → Rename เป็น **`Status Breakdown`**

Inputs ใส่ expression นี้ (วางทั้งก้อน):

```
concat('Assigned ',string(sub(length(split(string(body('Filter_Open')),'"Status":"Assigned"')),1)),' · On-Site ',string(sub(length(split(string(body('Filter_Open')),'"Status":"On-Site"')),1)),' · Accepted ',string(sub(length(split(string(body('Filter_Open')),'"Status":"Accepted"')),1)),' · Departed ',string(sub(length(split(string(body('Filter_Open')),'"Status":"Departed"')),1)),' · Held ',string(sub(length(split(string(body('Filter_Open')),'"Status":"Held"')),1)),' · Transfered ',string(sub(length(split(string(body('Filter_Open')),'"Status":"Transfered"')),1)))
```

<details>
<summary>มันทำงานยังไง? (คลิกดู)</summary>

`string(body('Filter_Open'))` แปลง array ทั้งก้อนเป็นข้อความ JSON
แล้ว `split(...,'"Status":"Held"')` ตัดข้อความด้วยคำนั้น
ถ้าเจอ 2 ครั้ง จะได้ 3 ชิ้น → `sub(3,1)` = **2** คือจำนวนงานสถานะ Held

เป็นทริกนับจำนวนแบบไม่ต้องวน loop ทำให้ Flow เร็วและประหยัด action

</details>

### Step 9 — สร้าง JSON ของแถวตาราง ⭐ หัวใจของงาน

**+ New step** → `Data Operation` → **Select** → Rename เป็น **`Select Rows`**

| ช่อง | ค่า |
|---|---|
| From | expression: `outputs('Top_Jobs')` |
| Map | 👇 ดูข้างล่าง |

ที่ช่อง **Map** ให้กดปุ่มสลับโหมดรูป **`T`** ทางขวา (สลับจากโหมดตาราง key/value → โหมดข้อความ)
แล้ววางเนื้อหาทั้งหมดของไฟล์ **`03-select-map.json`** ลงไป

> ผลลัพธ์ที่ได้ = array ของ `ColumnSet` 15 ก้อน = 15 แถวในตาราง
> ถ้าลืมกดปุ่ม `T` แล้ววาง JSON ลงในโหมดตาราง จะ error ทันที

### Step 10 — ประกอบการ์ดทั้งใบ

**+ New step** → **Compose** → Rename เป็น **`Card JSON`**

ช่อง Inputs → วางเนื้อหาทั้งหมดของไฟล์ **`04-card-powerautomate.json`**

> Power Automate จะแปลง `@{...}` ทุกจุดให้เป็นค่าจริงตอนรัน
> จุดสำคัญคือบรรทัด `"items": @{body('Select_Rows')}` — **ห้ามใส่ `"` ครอบ** เพราะต้องยัด array ไม่ใช่ข้อความ

### Step 11 — ส่งเข้า Teams

**+ New step** → ค้นหา `Microsoft Teams` → เลือก **Post adaptive card in a chat or channel**

| ช่อง | ค่า |
|---|---|
| Post as | `Flow bot` |
| Post in | `Channel` |
| Team | เลือกทีมของคุณ |
| Channel | เช่น `NOC-Daily-Report` |
| Adaptive Card | expression: `outputs('Card_JSON')` |

**Save** → **Test** → **Manually** → **Run flow** 🎉

---

## 📋 สรุปโครงสร้าง Flow ทั้งหมด

```
⏰ Recurrence                    ทุกวัน 08:00 (Bangkok)
│
├─ 📊 List rows present in a table    ← Excel: JobTable
│
├─ 🔍 Filter Open                     ตัดงานที่ปิดแล้วออก
├─ 🔍 Filter Critical                 นับ Critical
├─ 🔍 Filter Major                    นับ Major
├─ 🔍 Filter Minor                    นับ Minor
│
├─ 🔀 Sorted Jobs      (Compose)      union() → Critical มาก่อน
├─ ✂️  Top Jobs         (Compose)      take(...,15)
├─ 🔢 Status Breakdown (Compose)      สรุปจำนวนแยก Status
│
├─ 🧱 Select Rows      (Select)       สร้าง ColumnSet 15 แถว  ← ไฟล์ 03
├─ 🎴 Card JSON        (Compose)      ประกอบการ์ดทั้งใบ       ← ไฟล์ 04
│
└─ 💬 Post adaptive card in a chat or channel
```

---

## 🧾 ตารางรวม Expression ทุกตัว (คัดลอกไปใช้ได้เลย)

| ใช้ที่ | Expression |
|---|---|
| Filter Open — From | `body('List_rows_present_in_a_table')?['value']` |
| Filter Open — เงื่อนไข | `@and(not(equals(item()?['Status'],'Closed')),not(equals(item()?['Status'],'Cancelled')))` |
| Filter Critical/Major/Minor — From | `body('Filter_Open')` |
| Sorted Jobs | `union(union(body('Filter_Critical'),body('Filter_Major')),body('Filter_Minor'))` |
| Top Jobs | `take(outputs('Sorted_Jobs'),15)` |
| Select Rows — From | `outputs('Top_Jobs')` |
| Teams — Adaptive Card | `outputs('Card_JSON')` |
| จำนวนทั้งหมด | `length(body('Filter_Open'))` |
| จำนวนที่แสดง | `length(outputs('Top_Jobs'))` |
| จำนวนที่เหลือ | `sub(length(body('Filter_Open')),length(outputs('Top_Jobs')))` |
| วันที่ไทย | `formatDateTime(convertTimeZone(utcNow(),'UTC','SE Asia Standard Time'),'dd/MM/yyyy')` |

---

## 🛠️ แก้ปัญหาที่เจอบ่อย

| อาการ | สาเหตุ | วิธีแก้ |
|---|---|---|
| การ์ดขึ้นตัวอักษร `${jobs}` ดิบ ๆ | เอาไฟล์ `01` ไปใช้ใน Flow | ใช้ไฟล์ `04` แทน |
| `The template language expression ... cannot be evaluated` | ชื่อ action ไม่ตรง | เช็คชื่อทุก action ให้ตรง Step ที่ระบุ (เว้นวรรค = `_`) |
| ตารางว่างเปล่า ไม่มีแถว | `Select Rows` ไม่ได้ output | เปิด Run history ดู output ของ `Top Jobs` ว่ามีข้อมูลไหม |
| ช่องข้อมูลบางคอลัมน์ว่าง | ชื่อคอลัมน์สะกดไม่ตรง | เปิด Run history → ดู output ดิบของ `Filter Open` → copy ชื่อ key มาใส่ใน `item()?['...']` ให้ตรงตัวพิมพ์เล็ก/ใหญ่ |
| Create Time ขึ้นเป็นตัวเลข `46265.69` | เซลล์ Excel เป็นชนิด Date | ใช้คอลัมน์ช่วย `=TEXT(...)` ตามหัวข้อ "ขั้นเตรียมไฟล์ Excel" |
| Teams ไม่ส่งการ์ด / error ขนาด | การ์ดเกิน 28 KB | ลดเลขใน `take(...,15)` เหลือ 10 |
| ได้แค่ 256 แถว | ไม่ได้เปิด Pagination | Step 3 → Settings → Pagination On |
| การ์ดแคบ ตารางอัดกัน | Teams ไม่ได้ขยายเต็มความกว้าง | เช็คว่ามีบรรทัด `"msteams": { "width": "Full" }` ในการ์ด |
| `Status Breakdown` ขึ้น 0 หมด | รูปแบบ JSON ที่ `string()` คืนมาไม่ตรง | เปิด Run history ดู output ของ `Filter Open` แล้วแก้คำใน `split()` ให้ตรง |

---

## 🎨 ปรับแต่งเพิ่มเติม

**เปลี่ยนจำนวนแถวที่แสดง** → แก้เลขใน `Top Jobs`: `take(outputs('Sorted_Jobs'),20)`

**ส่งเฉพาะ Critical** → ที่ `Sorted Jobs` เปลี่ยนเป็น `body('Filter_Critical')` เฉย ๆ

**กรองเฉพาะ Zone ตัวเอง** → ที่ `Filter Open` เพิ่มเงื่อนไข:
```
@and(equals(item()?['Zone'],'RC3-UDN'),not(equals(item()?['Status'],'Closed')))
```

**แยกการ์ดตามจังหวัด** → ครอบ Step 5–11 ด้วย **Apply to each** บนรายชื่อจังหวัด

**เพิ่มปุ่มโทรหาช่าง** → เพิ่มใน `actions` ของไฟล์ `04`:
```json
{ "type": "Action.OpenUrl", "title": "📞 โทรหาช่าง", "url": "tel:0657152305" }
```

**เปลี่ยนลิงก์ปุ่ม** → แก้ `url` ใน `actions` ท้ายไฟล์ `04`
(ตอนนี้ชี้ไปที่ dashboard และ path SharePoint ตัวอย่าง `contoso.sharepoint.com` — ต้องเปลี่ยนเป็น URL จริงขององค์กร)

**ส่งเข้า Chat ส่วนตัวแทน Channel** → Step 11 เปลี่ยน `Post in` = `Chat with Flow bot` แล้วใส่ Recipient

---

## 📌 หมายเหตุเรื่องข้อมูล

ตัวเลขใน `02-sample-data.json` มาจากไฟล์ Excel ที่แนบมาจริง (snapshot 15-Aug-26):

- รวม **29 jobs** — Critical 6 · Major 23 · Minor 0
- Zone `RC3-UDN` · Region `NorthEast` ทั้งหมด
- Sub System: Splitter-OSP 13 · Transmission-OSP 11 · Power System 3 · FTTH-OSP 1 · EDS-OSP 1
- Status: Assigned 17 · On-Site 7 · Held 2 · Departed 1 · Accepted 1 · Transfered 1
- จังหวัด: อุดรธานี 17 · บึงกาฬ 5 · หนองบัวลำภู 3 · หนองคาย 3 · เลย 1
