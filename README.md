# Alongkorn Gold Display

หน้าแสดงราคาทองของร้านทองอลงกรณ์บน GitHub Pages โดยใช้ Google Apps Script (Gold Fetch) เป็น backend สำหรับข้อมูลราคาโดยตรง

## Architecture

Runtime เดิม:

```text
GoldTraders API
  -> GAS trigger
  -> PriceLog
  -> GitHub latest_gold.json
  -> GitHub Pages frontend
```

Runtime ใหม่:

```text
GoldTraders API
  -> GAS trigger
  -> PriceLog Google Sheet
  -> GAS Web App doGet()
  -> Alongkorn Display (GitHub Pages)
```

GitHub ไม่มีหน้าที่เป็น runtime data store อีกต่อไป ใช้เก็บ source และ host หน้า frontend เท่านั้น

## Frontend

`index.html` เป็น responsive frontend สำหรับ browser รุ่นใหม่:

- CSS Grid / Flex
- `clamp()`, `minmax()` และ responsive units
- รองรับ modern TV, desktop, laptop, tablet และ mobile
- ไม่ใช้ table-layout hack หรือ fixed-resolution layout
- ใช้ `fetch`, `async/await`, optional chaining, `Intl.NumberFormat` และ `AbortController`
- fetch GAS Web App โดยตรง
- รองรับ response wrapper `{ success: true, data: {...} }`
- มี timeout, bounded retry และ malformed-payload handling
- หาก Android browser ดึง JSON ด้วย `fetch()` ไม่สำเร็จ จะลองอ่านข้อมูลสาธารณะชุดเดียวกันจาก GAS ผ่าน JSONP (`<script>`) แล้วใช้วิธีนั้นต่อในหน้านั้น
- แสดงราคาที่เคยโหลดสำเร็จจากเครื่องนี้ทันทีเมื่อเปิดหน้า พร้อมบอกว่ากำลังตรวจราคาใหม่; เปลี่ยนเป็นสถานะเชื่อมต่อสำเร็จเมื่อได้ข้อมูลสด
- สถานะการเชื่อมต่ออ้างอิงผล fetch ล่าสุด ส่วนเวลาอัปเดตราคาแสดงจาก `asTime` เพราะราคาอาจคงเดิมนานโดยไม่มีการประกาศรอบใหม่

GAS endpoint ตั้งอยู่ใน `CONFIG.endpoint` ภายใน `index.html`:

```text
https://script.google.com/macros/s/AKfycbyV_ZEbQO9lXs2yXn6_iINYcBnkkkp8aHhvcIgoLdL3wcFPWXkwEteVnoqI9oGaCMMuyg/exec
```

## Gold Fetch GAS

source ที่ต้อง deploy อยู่ที่:

```text
gas/Code.gs
```

พฤติกรรมหลัก:

- `fetchGoldTraders()` และ validation เดิมยังคงอยู่
- `SPREADSHEET_ID` และ PriceLog header ภาษาไทยเดิมยังคงอยู่
- trigger sync เขียนแถวใหม่เฉพาะเมื่อ `GoldPriceID` เปลี่ยน
- หาก GoldTraders API ล้มเหลว Trigger จะรายงานเป็น execution ที่ล้มเหลว แทนการจบแบบสำเร็จเงียบ ๆ
- cleanup ประวัติเกิน 7 วันยังคงอยู่
- `doGet()` อ่านข้อมูลล่าสุดจาก `PriceLog`
- `doGet()` ใช้ Script Cache สูงสุด 60 วินาทีเพื่อลดการอ่าน Sheet ซ้ำ และล้าง cache เมื่อมี `GoldPriceID` ใหม่
- `doGet()` ตอบ JSON ตามเดิม และตอบ JSONP เฉพาะ callback คงที่สำหรับหน้า TV; ทั้งสองแบบอ่านข้อมูลราคาสาธารณะเท่านั้น
- ไม่มี GitHub API read/write ใน runtime
- ไม่มีการอ่าน `GITHUB_TOKEN`, `GITHUB_OWNER`, `GITHUB_REPO`, `GITHUB_BRANCH` หรือ `LATEST_FILE_PATH`
- `updateGoldHybrid()` ยังคงเป็น entry point เพื่อให้ Trigger เดิมทำงานต่อ
- `forceUpdateGoldHybrid()` ยังเรียกได้สำหรับ manual refresh แต่ไม่มีความหมายเรื่อง force-push GitHub แล้ว

อย่าลบหรือปิด Trigger เดิมโดยไม่ตรวจว่า Trigger ผูกกับ function ใดก่อน

## GAS response contract

```json
{
  "success": true,
  "data": {
    "source": "goldtraders_sheet",
    "generatedAt": "...",
    "recordedAt": "...",
    "goldPriceID": 87665,
    "asTime": "...",
    "seq": 7,
    "goldBar": {
      "buy": 67650,
      "sell": 67850
    },
    "exchange": {
      "usdThb": 33.42
    },
    "change": {
      "fromPrevRound": 50,
      "fromYesterday": -150
    }
  }
}
```

ความหมายของเวลา:

- `generatedAt`: เวลาที่ `doGet()` สร้าง response
- `recordedAt`: เวลาที่มีการ append แถวราคาใหม่ลง PriceLog
- `asTime`: เวลาประกาศราคาจากสมาคมค้าทองคำ

## Deployment checklist

1. ตรวจ Apps Script Trigger และคง `updateGoldHybrid()` ไว้
2. นำ `gas/Code.gs` (หรือ `gas/code.gs.txt` ที่มีเนื้อหาเหมือนกัน) ไปแทน Code.gs ของ Gold Fetch
3. Save และ deploy เวอร์ชันใหม่ของ GAS Web App
4. รัน/ตรวจ Trigger แล้วเปิด endpoint เพื่อยืนยัน `success: true`, `recordedAt` และราคาซื้อขาย
5. ตรวจหน้า GitHub Pages ว่า fetch GAS โดยตรง
6. หลัง production ผ่านแล้ว Script Properties `GITHUB_*` และ `LATEST_FILE_PATH` เดิมสามารถลบได้ เพราะ code ใหม่ไม่ใช้งาน
7. ไม่ต้องสร้างหรืออัปเดต `latest_gold.json` อีกต่อไป

## Source

ข้อมูลราคาหลักมาจาก Gold Traders Association endpoint:

```text
https://www.goldtraders.or.th/api/GoldPrices/Latest
```
