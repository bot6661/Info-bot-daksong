# 📱 ระบบดักซอง TrueMoney — Telegram Version
> ระบบดักซอง TrueMoney อัตโนมัติ ผ่าน Telegram MTProto รองรับผู้ใช้สูงสุด 50 คนพร้อมกัน RAM ต่ำกว่า Discord version มาก

---

## ⚡ ภาพรวมระบบ

ระบบดักซอง TrueMoney ผ่าน Telegram Client (GramJS/MTProto) ที่มีประสิทธิภาพสูงกว่า Discord Selfbot ในด้าน RAM เนื่องจาก MTProto Protocol ไม่มี message cache ทำให้รันได้นานกว่ามากโดยไม่ต้องกังวลเรื่อง RAM พุ่ง

---

## 📊 เปรียบเทียบ RAM จริง (รันพร้อมกัน)

| | Discord Version | Telegram Version |
|---|---|---|
| 45-50 user | ~3,299MB ใน 2h 49m | **602MB ใน 44h 28m** |
| พฤติกรรม | โตเรื่อยๆ | เสถียรมาก |
| สาเหตุ | discord.js cache guild/member | MTProto ไม่มี message cache |

Telegram version กิน RAM น้อยกว่า **5 เท่า** และรันได้นานกว่า 15 เท่าโดยไม่มีปัญหา

---

## 🧠 ระบบตรวจจับซอง (3 ช่องทาง)

### 1. Text Detection พร้อม Smart Validation
ตรวจจับ URL ซองและ voucher code จาก message text

ระบบมี `isValidVoucherCode()` ตรวจสอบหลายชั้น:
- ขึ้นต้น `019` เท่านั้น
- ตัวอักษรผสม alphanumeric (ป้องกัน false positive จาก hash ล้วน)
- **Entropy check:** ตัวอักษรไม่ซ้ำต้องมีอย่างน้อย 8 ชนิด
- Blacklist คำทั่วไป เช่น `telegram`, `password`, `facebook` ป้องกันการ match ผิด

### 2. QR Code Detection
ตรวจจับซองผ่านรูปภาพที่ส่งเข้ากลุ่ม Telegram อัตโนมัติ
- เช็คขนาดภาพจาก **metadata ก่อน download** (ไม่เปลืองแบนด์วิดท์)
- จำกัดขนาด 500KB ทั้ง metadata และ buffer จริงหลัง download
- ใช้ **sharp** (native C++) + **jsQR** พร้อม `inversionAttempts: 'attemptBoth'` — decode ใน 1 pass ไม่ต้องสร้าง buffer แยก
- **Image Dedup System:** MD5 hash บันทึกลง `qr_hashes.txt` — ไม่สแกนซ้ำทั้ง session และข้ามวัน

### 3. URL Redirect Detection
ตรวจจับซองที่ซ่อนเบื้องหลังลิงก์ย่อ/redirect ทุกค่าย
- Filter `t.me/` ออกอัตโนมัติ (ลิงก์ Telegram เอง)
- Follow redirect chain สูงสุด 10 ขั้น
- เช็ค Final URL + Body 1KB หลัง redirect
- บันทึก `url_blacklist.txt` เฉพาะ path ไม่ block ทั้ง domain

---

## 🏆 ระบบชั้นการกระจายซอง (Priority System)

```
#1  คนที่เจอซอง        — ได้เสมอ ไม่ว่าโหมดไหน (dynamic)
#2  OWNER              — เบอร์และ webhook ของเจ้าของระบบ (ถาวร)
#3+ VIP (Premium)      — เรียงตาม Group count มากไปน้อย
#N+ Normal             — เรียงตาม Group count มากไปน้อย
```

**กฎพิเศษโหมด Premium:**
ถ้าคนที่เจอซองเป็น Premium → รับเงียบๆ คนเดียว ไม่กระจาย ไม่มี OWNER แย่ง

---

## 🔰 โหมดการทำงาน (Mode System)

| | Normal | Premium |
|---|---|---|
| จำนวนเบอร์ | 1 เบอร์ | สูงสุด 3 เบอร์ |
| กระจายซอง | ✅ ตาม priority | ❌ รับคนเดียว |
| OWNER แย่ง | ✅ มี | ❌ ไม่มี |

ระบบตรวจสอบยศอัตโนมัติทุก 5 นาที ผ่าน Discord guild:
- ยศหาย → Downgrade + แจ้งเตือน webhook
- ได้ยศ → Upgrade ทันที + แจ้งเตือน webhook

---

## 🚀 ระบบ Batch Shooting

ยิงทีละ **10 บัญชีพร้อมกัน** (parallel batch)
- ถ้า batch ใดไม่มีใครได้เลย → **หยุดยิงทันที**
- Premium user ยิง 3 เบอร์พร้อมกัน
- delay ต่อ account config ได้

---

## 🔒 ระบบ Dedup (ป้องกันยิงซ้ำ)

**Text/URL voucher — 2 Layer:**
- `processedVouchers` Map + TTL 1 นาที
- `voucherProcessing` Set (lock) + double-check หลัง lock

**QR — 2 Layer:**
- `scanLock` ด้วย msgId — 50 account เจอรูปเดียวกัน สแกนแค่ 1 ครั้ง
- `qr_hashes.txt` — ไม่สแกนซ้ำข้ามวัน

**URL Redirect — 2 Layer:**
- `urlBlacklistSet` + `url_blacklist.txt`
- `urlPendingSet` — ป้องกัน follow redirect ซ้ำพร้อมกัน

---

## 🔑 ระบบ Login (OTP Flow)

Telegram ต้องการ login ผ่าน OTP ไม่ใช่ token — ระบบจัดการให้อัตโนมัติ:

```
กดเปิดระบบ
    ↓
ระบบส่ง OTP ไปยังเบอร์ Telegram
    ↓
ใส่ OTP ผ่าน Modal บน Discord
    ↓
บันทึก Session ลง database
    ↓
ครั้งต่อไปไม่ต้อง OTP อีก (ใช้ session ที่บันทึกไว้)
```

**OTP Timeout:** 5 นาที ถ้าไม่ใส่ → cleanup อัตโนมัติ

**Session Validation:** ตรวจสอบ session ก่อนเปิดทุกครั้ง ถ้า invalid → แจ้งเตือน + ล้าง session อัตโนมัติ

---

## ⚠️ ระบบจัดการ Error อัจฉริยะ

`createMessageHandler` มี error tracking:
- **Critical Errors** (`AUTH_KEY`, `SESSION`, `UNREGISTERED`) → หยุดทันที + แจ้งเตือน
- **Connection Errors** (`TIMEOUT`, `CONNECTION`, `NETWORK`) → นับ error ถ้าเกิน 3 ครั้ง → แจ้งเตือน
- Error อื่นๆ → log และทำงานต่อ ไม่ crash

---

## 🌐 ระบบนับกลุ่ม Telegram

ใช้ `iterDialogs()` นับกลุ่มและ Channel ทั้งหมดของแต่ละ account:
- ใช้สำหรับเรียงลำดับ VIP/Normal (คนที่มีกลุ่มมาก = เจอซองมากกว่า = ได้สิทธิ์ก่อน)
- อัพเดทอัตโนมัติทุกครั้งที่ login

---

## 📢 ระบบแจ้งเตือน

**Personal Webhook:** จำนวนเงิน, ความเร็ว, โหมด, เบอร์, แหล่งที่มา (Text/QR Code/ลิงก์ย่อ)

**Grouped Webhook:** รวมแจ้งเตือนทุก user ส่งทีละคน delay 3 วินาที

**การแจ้งเตือนอัตโนมัติ:**
- Session หมดอายุ / ไม่ถูกต้อง
- การเชื่อมต่อมีปัญหา
- ยศ Premium เปลี่ยน (up/down)

---

## 💾 ระบบ Memory Monitoring

- ตรวจสอบ RAM ทุก 2 นาที (config)
- RAM เกิน limit → บันทึกข้อมูล + restart อัตโนมัติ
- `roleCheckCache` cleanup ทุก 30 นาที ป้องกัน memory leak จาก user ที่ไม่ active

---

## 📊 ระบบ Ranking Cache

คำนวณลำดับ 1 ครั้ง cache 24 ชั่วโมง อัพเดทเมื่อมี user เปิด/ปิด

---

## 🖥️ UI & Commands (ผ่าน Discord Bot)

**Discord Slash Commands:**
| Command | ฟังก์ชัน |
|---|---|
| `/setup_gift` | ติดตั้งเมนูในช่อง |
| `/manage_users` | ดูรายชื่อและสถานะ |
| `/system_info` | RAM, uptime, จำนวน user |
| `/user_ranking` | ลำดับกระจายซอง + group count |

**Button UI:** ตั้งค่า API ID / API Hash / เบอร์ Telegram / เบอร์รับตัง / Webhook / OTP / เปิด-ปิดระบบ

---

## 📦 Dependencies

```json
"@opecgame/twapi": "^1.2.1"
"axios": "^1.6.0"
"discord.js": "^14.14.1"
"telegram": "^2.25.19"
"jsqr": "^1.4.0"
"sharp": "^0.33.4"
```

---

## 🗂️ โครงสร้างไฟล์

```
📁 telegrambot/
├── index.js          — ระบบหลักทั้งหมด
├── database.js       — จัดการข้อมูล user, balance, session
├── bot_config.js     — config ทั้งหมด
├── package.json
├── start.bat
│
📁 data/              — สร้างอัตโนมัติ (ไม่แนบมา)
├── users/            — ข้อมูล user แต่ละคน
├── sessions/         — Telegram session แต่ละคน
│
📄 qr_hashes.txt      — สร้างอัตโนมัติ
📄 url_blacklist.txt  — สร้างอัตโนมัติ
```

---

## ⚙️ การติดตั้ง

```bash
npm install
node index.js
```

แนะนำใช้ **PM2** — RAM เสถียรมาก ไม่จำเป็นต้องตั้ง memory restart บ่อย
