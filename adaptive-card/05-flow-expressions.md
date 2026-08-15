# 05 · Power Automate Expressions Cheat Sheet

รวมสูตร (Expression) ทุกตัวที่ต้องใช้ใน Flow — คัดลอกไปวางใน **Expression tab** ของแต่ละ Action

> ชื่อ Action ในสูตรต้องตรงกับชื่อจริงใน Flow เป๊ะ ๆ (เว้นวรรค → `_`)
> เช่น Action ชื่อ `Filter Open Jobs` จะอ้างด้วย `body('Filter_Open_Jobs')`

---

## 1. Trigger — Recurrence 08:00 ทุกวัน

| Field | Value |
|---|---|
| Frequency | `Day` |
| Interval | `1` |
| Time zone | `(UTC+07:00) Bangkok, Hanoi, Jakarta` |
| At these hours | `8` |
| At these minutes | `0` |

> ตั้ง Time zone ที่ตัว Trigger ให้เรียบร้อย จะได้ไม่ต้องบวก 7 ชั่วโมงเองในทุกสูตร

---

## 2. วันที่/เวลาแบบไทย (Asia/Bangkok)

| ชื่อ Compose | Expression | ผลลัพธ์ |
|---|---|---|
| `Compose_ReportDate` | `formatDateTime(convertTimeZone(utcNow(),'UTC','SE Asia Standard Time'),'dd MMM yyyy')` | `15 Aug 2026` |
| `Compose_RunTime` | `formatDateTime(convertTimeZone(utcNow(),'UTC','SE Asia Standard Time'),'dd-MMM-yy HH:mm')` | `15-Aug-26 08:00` |
| `Compose_Today` | `formatDateTime(convertTimeZone(utcNow(),'UTC','SE Asia Standard Time'),'dd-MMM-yy')` | `15-Aug-26` |

---

## 3. Filter array — คัดเฉพาะงานที่ยังไม่ปิด

**Action:** `Filter Open Jobs` → From = `value` (output ของ List rows / Get items)

Advanced mode:

```
@not(contains(createArray('Closed','Completed','Cancelled','Resolved'), string(item()?['Status'])))
```

ถ้าอยากได้เฉพาะงานที่สร้าง **วันนี้** เพิ่มเงื่อนไข:

```
@and(
  not(contains(createArray('Closed','Completed','Cancelled','Resolved'), string(item()?['Status']))),
  startsWith(string(item()?['Create Time']), outputs('Compose_Today'))
)
```

---

## 4. Filter array แยกตาม Priority (ใช้ทั้งนับ KPI และเรียงลำดับ)

| Action name | From | Advanced condition |
|---|---|---|
| `Filter Critical` | `body('Filter_Open_Jobs')` | `@equals(item()?['Priority'], 'Critical')` |
| `Filter Major` | `body('Filter_Open_Jobs')` | `@equals(item()?['Priority'], 'Major')` |
| `Filter Minor` | `body('Filter_Open_Jobs')` | `@not(contains(createArray('Critical','Major'), string(item()?['Priority'])))` |

> `Filter Minor` ใช้ `not(contains(...))` แทน `equals(...,'Minor')` เพื่อกวาดแถวที่ `Priority` ว่างหรือเป็น `null` เข้ามาด้วย (ในไฟล์ตัวอย่างมี 1 แถว)

**นับ KPI:**

```
length(body('Filter_Critical'))
length(body('Filter_Major'))
length(body('Filter_Minor'))
length(body('Filter_Open_Jobs'))
```

---

## 5. เรียงลำดับ Critical → Major → Minor แล้วตัดเหลือ 15 แถว

Power Automate ไม่มี Sort action — ใช้วิธีต่ออาร์เรย์ตามลำดับที่ต้องการแทน

**Compose ชื่อ `Compose_Top_Jobs`:**

```
take(union(body('Filter_Critical'), body('Filter_Major'), body('Filter_Minor')), 12)
```

> **ทำไมต้อง 12?** Adaptive Card ที่ Teams รับได้มีขนาดจำกัด **28 KB** ต่อใบ
> วัดจาก template จริงในโฟลเดอร์นี้: โครงการ์ดเปล่า 4.7 KB + แถวละ ~1.24 KB
>
> | จำนวนแถว | ขนาดการ์ด |
> |---|---|
> | 10 | 17.1 KB |
> | **12** | **19.5 KB** ← แนะนำ |
> | 15 | 23.2 KB |
> | 18 | 26.9 KB |
> | 20 | 29.4 KB ❌ เกินลิมิต |
>
> ชื่อ Site Name / ชื่อคนภาษาไทยยาว ๆ ทำให้แต่ละแถวโตกว่านี้ได้ จึงเผื่อไว้ที่ 12 แถว
> ถ้าจะดันถึง 15–18 ให้เช็กขนาดจริงด้วย `preview.html` ก่อนทุกครั้ง
> เกินลิมิตแล้ว Flow จะขึ้น Succeeded ตามปกติ แต่การ์ดจะไม่โผล่ใน Teams เลย
> ไฟล์ตัวอย่างมี 117 งาน จึงต้องตัดยอด แล้วลิงก์ไป Dashboard ดูตัวเต็ม
>
> `union()` ตัดรายการที่ object ซ้ำกันทุก field ออก — Job ID ไม่ซ้ำอยู่แล้วจึงปลอดภัย
> ถ้าไม่อยากให้ตัดเลย ใช้ `take(concat(body('Filter_Critical'), body('Filter_Major'), body('Filter_Minor')), 15)`

**จำนวนที่เหลือ (ใช้ในบรรทัด "ยังมีอีก N รายการ"):**

```
sub(length(body('Filter_Open_Jobs')), length(outputs('Compose_Top_Jobs')))
```

---

## 6. Select — แปลงข้อมูลเป็นแถวตาราง Adaptive Card

**Action:** `Select Job Rows`
**From:** `outputs('Compose_Top_Jobs')`
**Map:** กด ⇄ สลับเป็น **code view** (ปุ่มขวาบนของช่อง Map) แล้ววาง `02-row-template.json` ทั้งก้อน

output ที่ได้คืออาร์เรย์ของ `ColumnSet` พร้อมยัดเข้า `body` ของการ์ดโดยตรง

---

## 7. สรุปจำนวนแยกตาม Status (บรรทัด statusBreakdown)

**Select `Select_Status`** → From `body('Filter_Open_Jobs')`, Map (text mode): `item()?['Status']`

**Compose `Compose_StatusBreakdown`:**

```
join(
  select(
    union(body('Select_Status'), body('Select_Status')),
    concat(item(), ' ', string(length(split(concat('#', join(body('Select_Status'), '#')), concat('#', item())))))
  ),
  ' · '
)
```

> สูตรนี้อ่านยากและเปราะ — ถ้าอยากได้ผลนิ่งกว่า ให้ทำแยกเป็น Filter array ต่อ Status ที่สนใจ
> (`Interupted`, `Assigned`, `On-Site`) แล้ว `length()` ทีละตัว เหมือนหัวข้อ 4

ทางที่แนะนำจริง ๆ — Compose ธรรมดา:

```
concat(
  'Interupted ', length(body('Filter_Interupted')),
  ' · Assigned ', length(body('Filter_Assigned')),
  ' · On-Site ',  length(body('Filter_OnSite'))
)
```

---

## 8. ถ้า Create Time ออกมาเป็นตัวเลข (Excel serial date)

Excel เก็บวันที่เป็นตัวเลข เช่น `46249.2618` — connector บางกรณีจะส่งเลขดิบมา
แปลงกลับด้วย (ฐานของ Excel คือ 1899-12-30):

```
formatDateTime(
  addSeconds('1899-12-30', int(mul(float(item()?['Create Time']), 86400))),
  'dd-MMM-yy HH:mm'
)
```

ในไฟล์ `JobMonitor_CMTRS_Job_Tracking.xlsx` คอลัมน์นี้เป็น **ข้อความ** (`15-Aug-26 06:17`) อยู่แล้ว
จึงใช้ `item()?['Create Time']` ตรง ๆ ได้เลย — เก็บสูตรนี้ไว้เผื่อเปลี่ยนไปใช้ SharePoint (ซึ่งส่งมาเป็น ISO datetime):

```
formatDateTime(convertTimeZone(item()?['Create_x0020_Time'],'UTC','SE Asia Standard Time'),'dd-MMM-yy HH:mm')
```

---

## 9. ชื่อคอลัมน์: Excel vs SharePoint

| หัวตาราง | Excel connector | SharePoint connector (internal name) |
|---|---|---|
| Sub System | `item()?['Sub System']` | `item()?['Sub_x0020_System']` |
| Job ID | `item()?['Job ID']` | `item()?['Job_x0020_ID']` |
| Priority | `item()?['Priority']` | `item()?['Priority']` |
| Create Time | `item()?['Create Time']` | `item()?['Create_x0020_Time']` |
| Status | `item()?['Status']` | `item()?['Status']` |
| Province Name | `item()?['Province Name']` | `item()?['Province_x0020_Name']` |
| Site Name | `item()?['Site Name']` | `item()?['Site_x0020_Name']` |
| Assign to | `item()?['Assign to']` | `item()?['Assign_x0020_to']` |

> หา internal name จริงได้ที่ List settings → คลิกชื่อคอลัมน์ → ดูท้าย URL ที่ `&Field=`
> ถ้าคอลัมน์เป็น **Person** ต้องใช้ `item()?['Assign_x0020_to']?['DisplayName']`
> ถ้าเป็น **Choice** ต้องใช้ `item()?['Priority']?['Value']`

---

## 10. เงื่อนไขกันการ์ดเปล่า

ครอบ Action "Post adaptive card" ด้วย **Condition**:

```
length(body('Filter_Open_Jobs'))  is greater than  0
```

ไม่มีงานค้าง → ส่งการ์ดสั้น ๆ แทน (ดู `06-adaptive-card-empty-state.json`)
