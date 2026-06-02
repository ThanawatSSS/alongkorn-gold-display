# Gold Price Display - ร้านทองอลงกรณ์ ✨

เว็บแอปพลิเคชันแสดงราคาทองคำแบบ Real-time ออกแบบมาเป็นพิเศษเพื่อความเสถียรสูงสุดบน **Smart TV รุ่นเก่า (Samsung, Tizen, LG WebOS ฯลฯ)** โดยเฉพาะที่ร้านทองอลงกรณ์

---

## 🚀 คุณสมบัติเด่น (Key Features)

- **TV-Optimized Layout:** ปรับแต่งเลย์เอาต์แบบ Wide Screen ให้แสดงผลได้เต็มหน้าจอที่ 100% Zoom โดยไม่ต้องปรับแต่งที่ตัวเครื่องทีวี
- **Hybrid Data Fetching:** ใช้ระบบ **GitHub Actions** ดึงราคามาเก็บไว้ที่ฝั่งตัวเองทุก 5 นาที เพื่อแก้ปัญหาการเชื่อมต่อ (SSL/CORS) บนทีวีรุ่นเก่าได้ 100%
- **Hero Price Display:** เน้นราคาขายหน้าร้าน (บวกเพิ่มจากสมาคมอัตโนมัติ) ให้โดดเด่นและชัดเจนที่สุด
- **Price Trend Indicators:** แสดงสัญลักษณ์ ▲ (ขึ้น) / ▼ (ลง) พร้อมสีสันที่ชัดเจนตามสถานะตลาด
- **Time-Ago Logic:** บอกระยะเวลาที่อัปเดตล่าสุดอย่างแม่นยำ (เช่น "อัปเดตเมื่อ 10 นาทีที่แล้ว")
- **Legacy Browser Support:** เขียนด้วยมาตรฐาน **ES5** และ Pure CSS เพื่อความลื่นไหลในเบราว์เซอร์รุ่นเก่า

---

## 🛠️ วิธีการตั้งค่าสำหรับ GitHub Pages

เพื่อให้ระบบทำงานได้สมบูรณ์แบบบนทีวีรุ่นเก่า:

1. **เปิดใช้งาน GitHub Pages:** ไปที่ `Settings > Pages` ของ Repository นี้ แล้วเลือก Branch `master`
2. **ระบบอัปเดตราคา:** ปัจจุบันย้ายระบบดึงและอัปเดตราคาทองคำไปรันบน **Google Apps Script (GAS)** เพื่อหลีกเลี่ยงการติด Cloudflare 403 Challenge จากฝั่ง GitHub Actions

---

## ⏪ การย้อนกลับ (Rollback)

หากวันใดเกิดปัญหากับระบบอัปเดตผ่าน GAS และต้องการย้อนกลับมาใช้ระบบ GitHub Actions ในการดึงราคาแบบเดิม สามารถตรวจสอบโค้ดหรือกู้คืนได้ทันที:

- **ตรวจสอบโค้ดเดิมใน Branch สำรอง:**
  ```bash
  git checkout backup/github-actions-chnwt
  ```
- **หรือดึงโค้ด GitHub Actions กลับมาใช้ใน master:**
  ใช้คำสั่ง `git cherry-pick` ดึง commit หรือก๊อปปี้ไฟล์ `.github/workflows/update_gold.yml` จาก branch สำรองกลับมาใช้งานได้ทันที

---

## 🙏 ขอบคุณแหล่งข้อมูล (Credits)

โครงการนี้สำเร็จได้ด้วยข้อมูลราคาทองคำคุณภาพสูงจาก:

- **Thai Gold API** พัฒนาโดยคุณ **@chnwt** | [Website](https://api.chnwt.dev/thai-gold-api)
- **API Developer GitHub:** [chnwt](https://github.com/chnwt)

---
