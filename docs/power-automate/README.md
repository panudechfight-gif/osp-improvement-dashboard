# CM-TRS Job Tracking · Power Automate Daily Card

เอกสารชุดนี้แก้ปัญหา **Condition "Has Open Jobs" ตกไปทาง `false` เสมอ** ทำให้การ์ดใน Teams
ขึ้นว่า "ไม่มีงานค้าง" ทุกวัน และไม่เคยส่งรายการ Job ออกมาเลย

| ไฟล์ | ใช้ทำอะไร |
|---|---|
| `README.md` (ไฟล์นี้) | สาเหตุ · วิธีตรวจสอบใน Run history · วิธีแก้ทีละขั้น · expression ทั้งหมดให้ copy |
| `flow-definition.json` | โครง Flow ที่ต่อสายถูกต้องแล้ว ใช้เทียบกับ Flow ปัจจุบัน (Peek code) |
| `card-open-jobs.json` | Adaptive Card สาขา **true** — การ์ดที่ "ส่งข้อมูล" รายการงานค้างจริง |
| `card-no-open-jobs.json` | Adaptive Card สาขา **false** — การ์ดเขียวแบบที่เห็นอยู่ตอนนี้ |

> เอกสารนี้เขียนจาก schema จริงของ CM-TRS ที่ dashboard ใช้อยู่ (`index.html` → `CM_TRS_EMBED`)
> ไม่ได้เดาชื่อคอลัมน์

---

## 1. นิยาม "งานค้าง" ที่ต้องตรงกับ Dashboard

Dashboard ตัดสินว่า Job ปิดแล้วหรือยัง ด้วยกฎเดียวคือ *ข้อความใน `Status`* ไม่ใช่ค่าตายตัว:

```js
const isDone      = s => s.toLowerCase().includes('done') || s.toLowerCase().includes('closed');
const outstanding = r => !isDone(r['Status']);   // งานคงค้าง = ยังไม่ปิดงาน
```

ค่าที่โผล่จริงในชุดข้อมูลล่าสุด (107 rows):

| Status | จำนวน | นับเป็น |
|---|---:|---|
| `Assigned` | 48 | **ค้าง** |
| `Transfered` | 16 | **ค้าง** |
| `Interupted` | 13 | **ค้าง** |
| `Done(Not Leave)` | 12 | ปิดแล้ว |
| `Held` | 8 | **ค้าง** |
| `On-Site` | 5 | **ค้าง** |
| `Departed` | 3 | **ค้าง** |
| `Accepted` | 2 | **ค้าง** |

ข้อควรระวัง 3 ข้อที่ทำให้ Flow เขียนเงื่อนไขผิดกันบ่อย:

1. **ห้ามใช้ `Status ne 'Done'`** — ค่าจริงคือ `Done(Not Leave)` ไม่ใช่ `Done` เป๊ะ ๆ
   ต้องใช้ *contains* ไม่ใช่ *equals*
2. **`Transfered` / `Interupted` สะกดแบบนี้จริงในต้นทาง** (ไม่ใช่ `Transferred` / `Interrupted`)
   ถ้าไป match ตัวสะกดที่ถูกหลักภาษา จะได้ 0 แถว
3. `Create Time` เป็น **ข้อความ** รูปแบบ `28-Jul-26 10:58` ไม่ใช่ date ของ Excel
   ถ้า Filter Query ไปเทียบกับวันที่แบบ `yyyy-MM-dd` จะไม่ match อะไรเลย

จากข้อมูลชุดนี้ 95 จาก 107 งานคือ "ค้าง" — การ์ดที่ขึ้น "ไม่มีงานค้าง" จึงเป็นไปไม่ได้
เว้นแต่ Flow อ่านข้อมูลมาได้ 0 แถว หรือ Filter คัดทิ้งหมด

---

## 2. หาสาเหตุจาก Run history ก่อนแก้ (5 นาที)

เปิด flow → **28-day run history** → เลือก run ที่ส่งการ์ดเขียว → กางทีละ action
แล้วอ่านค่าตามตารางนี้ จะรู้ทันทีว่าพังตรงไหน:

| กาง action นี้ | ดูค่า | ถ้าเจอแบบนี้ = สาเหตุคือ |
|---|---|---|
| `List rows present in a table` | `body.value` มีกี่ item | **0 item** → ปัญหาอยู่ที่ต้นทาง/Filter Query ไม่ใช่ที่ Condition → ไปข้อ 3.1 |
| `Filter array` | `body` มีกี่ item | ต้นทางมีของ แต่ **ตรงนี้เหลือ 0** → เงื่อนไขใน Filter ผิด → ไปข้อ 3.2 |
| `Condition` (Has Open Jobs) | `inputs` ที่ประเมินได้จริง | เห็น `"expression": { "greater": [ 0, 0 ] }` หรือเทียบ int กับ `"0"` ที่เป็น string → ไปข้อ 3.3 |

**สาเหตุที่พบบ่อยที่สุดคือกรณีที่สอง**: ต้นทางมีข้อมูล แต่ `Filter array` คืน `[]`
แล้ว Condition ก็ทำงานถูกต้องตามหน้าที่ของมัน คือตอบ `false` — ตัว Condition ไม่ได้เสีย
ของที่ป้อนเข้ามาต่างหากที่ว่างเปล่า

---

## 3. วิธีแก้

### 3.1 อย่ากรองที่ connector — ดึงมาให้ครบก่อน

ใน `List rows present in a table` (หรือ SharePoint `Get items`):

- **ล้างช่อง `Filter Query` ให้ว่าง** OData ของ Excel connector รองรับ operator จำกัดมาก
  และ `Create Time` / `Status` เป็น text ทำให้เงื่อนไขที่ดูถูกต้องคืน 0 แถวเงียบ ๆ
- เปิด **Settings → Pagination = On**, `Threshold` = `5000`
  ค่า default คือ 256 แถว ถ้าไฟล์ยาวกว่านั้นงานที่ค้างจริงอาจถูกตัดทิ้งทั้งหมด
- ถ้าตาราง Excel มีแถวหัวเรื่องคร่อมอยู่ ให้ชี้ไปที่ **named table** ไม่ใช่ทั้ง worksheet
  ไม่งั้น `value` จะออกมาเป็น `[]`

### 3.2 เขียน Filter array ใหม่ในโหมด Advanced

กดปุ่ม **Edit in advanced mode** ของ `Filter array` (อย่าเลือกจาก dynamic content
เพราะการเลือกจากรายการมักได้ token ของทั้งคอลัมน์ ไม่ใช่ `item()`) แล้ววางทั้งก้อนนี้:

**From**

```
@body('List_rows_present_in_a_table')?['value']
```

**Where** — ตรงกับ `isDone()` ของ dashboard แบบ 1:1

```
@and(
  not(equals(trim(coalesce(item()?['Job ID'], '')), '')),
  not(contains(toLower(coalesce(item()?['Status'], '')), 'done')),
  not(contains(toLower(coalesce(item()?['Status'], '')), 'closed'))
)
```

`coalesce(..., '')` สำคัญมาก — แถวที่ `Status` เป็น null จะทำให้ `toLower()` โยน error
และทั้ง Filter array จะ fail ทั้งก้อน กลายเป็นว่า Condition ไม่ได้ค่าอะไรเลย

### 3.3 เขียน Condition ใหม่ให้เทียบ integer กับ integer

Condition card แบบ basic เก็บค่าที่พิมพ์ในช่องขวาเป็น **string** เสมอ
`length(...)` คืน **integer** การเทียบข้ามชนิดจึงได้ `false` โดยไม่มี error ให้เห็น
ให้กด **Edit in advanced mode** แล้ววาง:

```
@greater(length(body('Filter_open_jobs')), 0)
```

เช็คเพิ่มอีกสองจุดที่ทำให้ผลกลับด้าน:

- ต้องเป็น `body('Filter_open_jobs')` ไม่ใช่ `outputs('Filter_open_jobs')`
  ตัวหลังคือ object `{ "body": [...] }` ซึ่ง `length()` ใช้ไม่ได้
- ถ้าเดิมใช้ `empty(...)` อยู่ ความหมายจะ **กลับด้าน** — `empty()` เป็น true ตอน "ไม่มีงาน"
  ถ้าจะใช้ต่อ ต้องสลับสาขา true/false ให้ครบทั้งคู่

### 3.4 ทำให้สาขา true "ส่งข้อมูล" จริง

ปัญหาที่ผู้ใช้เจอมีสองชั้น: condition ไม่เคยเป็น true **และ** ต่อให้เป็น true การ์ดก็ไม่มีรายการงาน
Adaptive Card ใน Teams วน loop เองไม่ได้ ต้องแปลงเป็นข้อความก่อนด้วย `Select` แล้วค่อย `join`

เพิ่ม action **Select** (ชื่อ `Select_job_lines`) ไว้ก่อน Condition:

**From** — เรียง Critical → Major → Minor แล้วตัด 10 รายการแรก

```
@take(union(union(body('Filter_critical'), body('Filter_major')), body('Filter_minor')), 10)
```

**Map** (กดสลับเป็นโหมด text ด้วยปุ่มขวาบน)

```
@concat('- **', coalesce(item()?['Job ID'],'-'), '** · ', coalesce(item()?['Priority'],'-'), ' · ', coalesce(item()?['Province Name'],'-'), ' · ', coalesce(item()?['Status'],'-'), ' · ', coalesce(item()?['Assign to'],'ยังไม่ระบุผู้รับผิดชอบ'))
```

แล้วในการ์ดใช้:

```
@{join(body('Select_job_lines'), decodeUriComponent('%0A'))}
```

ขึ้นต้นทุกบรรทัดด้วย `- ` แล้ว Teams จะ render เป็น bullet list ให้เอง

ใช้ `decodeUriComponent('%0A')` แทนการพิมพ์ `\n` ตรง ๆ เพราะ `\n` ที่อยู่ใน JSON ของการ์ด
จะถูกแปลงเป็นขึ้นบรรทัดใหม่จริงตั้งแต่ตอน parse JSON ทำให้ตัวนิพจน์มีบรรทัดใหม่คั่นกลางและ compile ไม่ผ่าน

### 3.5 ตัวเลขสรุปหัวการ์ด

WDL ไม่มี lambda ให้นับแบบมีเงื่อนไขในนิพจน์เดียว วิธีที่เชื่อถือได้คือแตก `Filter array` เพิ่ม
โดยรับ input จาก `body('Filter_open_jobs')` (ไม่ใช่จาก connector อีกรอบ):

| Action | Where | ใช้ในการ์ด |
|---|---|---|
| `Filter_critical` | `@equals(item()?['Priority'], 'Critical')` | `@{length(body('Filter_critical'))}` |
| `Filter_major` | `@equals(item()?['Priority'], 'Major')` | `@{length(body('Filter_major'))}` |
| `Filter_minor` | `@equals(item()?['Priority'], 'Minor')` | `@{length(body('Filter_minor'))}` |

### 3.6 (ทางเลือก) นับงานเกิน SLA

SLA ของ dashboard คือ Critical 3 ชม. · Major 10 ชม. · Minor 72 ชม. นับจาก `Create Time`

เนื่องจาก `if()` ของ WDL ประเมินทุก argument แถวที่ `Create Time` ว่างจะทำให้ `parseDateTime`
พังทั้ง action จึงต้องกรองแถวที่มีเวลาออกมาก่อนหนึ่งชั้น:

`Filter_open_with_time` — From `@body('Filter_open_jobs')`, Where:

```
@not(equals(trim(coalesce(item()?['Create Time'], '')), ''))
```

`Filter_over_sla` — From `@body('Filter_open_with_time')`, Where:

```
@less(
  formatDateTime(
    addHours(
      parseDateTime(item()?['Create Time'], 'en-US', 'dd-MMM-yy HH:mm'),
      if(equals(item()?['Priority'], 'Critical'), 3, if(equals(item()?['Priority'], 'Major'), 10, 72))
    ),
    'yyyy-MM-ddTHH:mm:ss'
  ),
  formatDateTime(convertTimeZone(utcNow(), 'UTC', 'SE Asia Standard Time'), 'yyyy-MM-ddTHH:mm:ss')
)
```

ทั้งสองฝั่ง format เป็น string รูปแบบเดียวกันก่อนเทียบ เพื่อไม่ให้ติดปัญหา `Z` / เศษ tick ไม่เท่ากัน

### 3.7 วันที่และเวลาบนการ์ด

`utcNow()` ไม่ใช่เวลาไทย ถ้า flow ยิงช่วงหัวค่ำถึงเที่ยงคืน วันที่บนการ์ดจะเพี้ยนไปหนึ่งวัน
ใช้แบบนี้ทุกจุดที่แสดงเวลา:

```
@{formatDateTime(convertTimeZone(utcNow(), 'UTC', 'SE Asia Standard Time'), 'dd-MMM-yy HH:mm')}
@{formatDateTime(convertTimeZone(utcNow(), 'UTC', 'SE Asia Standard Time'), 'dd MMM yyyy')}
```

และตั้ง **Recurrence → Time zone = (UTC+07:00) Bangkok, Hanoi, Jakarta** ที่ตัว trigger ด้วย

---

## 4. ตรวจว่าแก้แล้วจริง

1. กด **Test → Manually** ไม่ต้องรอรอบถัดไป
2. เปิด run ที่เพิ่งจบ กาง `Filter_open_jobs` — ต้องเห็นจำนวน item > 0
3. กาง `Condition` — ช่อง inputs ต้องอ่านได้ว่า `true` และ flow ต้องลงสาขาซ้าย
4. เทียบตัวเลขบนการ์ดกับ KPI **"งานคงค้าง"** บน dashboard — ต้องตรงกันเป๊ะ
   ถ้าไม่ตรง แปลว่า Filter ยังไม่ตรงกับ `isDone()` ให้ย้อนไปข้อ 3.2
5. ทดสอบสาขา false ด้วย: แก้ Where ของ `Filter_open_jobs` เป็น `@false` ชั่วคราว
   ต้องได้การ์ดเขียวใบเดิม แล้วอย่าลืมแก้กลับ

การ์ดเขียวใน `card-no-open-jobs.json` เพิ่มบรรทัด "อ่านข้อมูลมาทั้งหมด N รายการ" ไว้ให้
ถ้าวันไหนขึ้น "ไม่มีงานค้าง" อีก จะเห็นทันทีบนการ์ดว่า N เป็น 0 หรือไม่
คือแยกออกได้เลยว่า *อ่านข้อมูลไม่ได้* หรือ *ปิดงานครบจริง* โดยไม่ต้องเปิด Run history

---

## 5. หมายเหตุการใช้ไฟล์ Adaptive Card

- วาง JSON ทั้งไฟล์ลงใน action **Post adaptive card in a chat or channel** ช่อง *Adaptive Card*
- `@{...}` ในไฟล์คือ expression ที่ Power Automate จะประเมินให้ตอน runtime
- ถ้าต้องใส่ตัวอักษร `@` จริง ๆ ในข้อความ ต้องพิมพ์ `@@` ไม่งั้นจะถูกตีความเป็น expression
- ชื่อ action ในไฟล์อ้างแบบมี `_` (`Filter_open_jobs`) ตามที่ Power Automate แปลงจาก
  ชื่อที่มีช่องว่าง ถ้าตั้งชื่อ action ต่างจากนี้ ต้องแก้ชื่อในนิพจน์ให้ตรงด้วย
