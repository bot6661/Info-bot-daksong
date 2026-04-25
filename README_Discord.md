# 🎯 ระบบดักซอง TrueMoney — Discord Version
> ระบบดักซอง TrueMoney อัตโนมัติ ผ่าน Discord Selfbot รองรับผู้ใช้สูงสุด 50 คนพร้อมกัน

---

## ⚡ ภาพรวมระบบ

ระบบดักซอง TrueMoney ผ่าน Discord Selfbot ที่ออกแบบมาเพื่อรองรับผู้ใช้หลายคนพร้อมกัน โดยที่แต่ละคนมี Selfbot Account ของตัวเองคอยตรวจจับซองในทุก Server ที่ Account นั้นอยู่ เมื่อเจอซองจะยิงรับทันทีโดยไม่รอ พร้อมระบบกระจายสิทธิ์ตามชั้น

---

## 🧠 ระบบตรวจจับซอง (3 ช่องทาง)

### 1. Text Detection
ตรวจจับ URL ซองจาก Content และ Embed ของ Message ทุกอัน ทันทีที่ message เข้ามาจะถูกตรวจสอบใน microsecond ไม่มี delay ใดๆ

### 2. QR Code Detection
ตรวจจับซองผ่านรูปภาพที่แนบมากับ message โดยอัตโนมัติ
- ใช้ **sharp** (native C++) + **jsQR** decode ภาพ เร็วกว่า jimp 3-4 เท่า กิน RAM น้อยกว่า
- รองรับ QR code ทุกรูปแบบ รวมถึงภาพที่สีกลับ (inverted) ด้วย `inversionAttempts: 'attemptBoth'`
- จำกัดขนาดภาพ 500KB ป้องกันการดาวน์โหลดไฟล์ขนาดใหญ่โดยไม่จำเป็น
- **Image Dedup System:** คำนวณ MD5 hash ของทุกภาพ บันทึกลง `qr_hashes.txt` — ภาพที่เคยสแกนแล้วไม่สแกนซ้ำอีก ทั้งภาพที่ติดและไม่ติด

### 3. URL Redirect Detection
ตรวจจับซองที่ซ่อนอยู่เบื้องหลังลิงก์ย่อ/redirect ทุกค่ายโดยไม่ต้อง whitelist domain
- Follow redirect chain ได้สูงสุด 10 ขั้น
- ตรวจสอบ Final URL + Body 1KB หลัง redirect
- URL ที่ไม่ใช่ voucher → บันทึกลง `url_blacklist.txt` (blacklist เฉพาะ path ไม่ block ทั้ง domain)
- ถ้าใน message มีมากกว่า 1 URL → ข้ามทั้งหมด (ซองจริงส่งลิงก์เดียว)

---

## 🏆 ระบบชั้นการกระจายซอง (Priority System)

เมื่อเจอซอง ระบบจะกระจายสิทธิ์ตามลำดับดังนี้:

```
#1  คนที่เจอซอง        — ได้เสมอ ไม่ว่าโหมดไหน (dynamic)
#2  OWNER              — เบอร์และ webhook ของเจ้าของระบบ (ถาวร)
#3+ VIP (Premium)      — เรียงตาม Server count มากไปน้อย
#N+ Normal             — เรียงตาม Server count มากไปน้อย
```

**กฎพิเศษโหมด Premium:**
ถ้าคนที่เจอซองเป็น Premium → รับเงียบๆ คนเดียว ไม่กระจายให้ใคร ไม่มี OWNER แย่ง

---

## 🔰 โหมดการทำงาน (Mode System)

| | Normal | Premium |
|---|---|---|
| จำนวนเบอร์ | 1 เบอร์ | สูงสุด 3 เบอร์ |
| กระจายซอง | ✅ ตาม priority | ❌ รับคนเดียว |
| OWNER แย่ง | ✅ มี | ❌ ไม่มี |
| ตรวจสอบอัตโนมัติ | ✅ | ✅ |

ระบบตรวจสอบยศอัตโนมัติทุก 5 นาที:
- ยศหาย → Downgrade เป็น Normal ทันที + แจ้งเตือน webhook
- ได้ยศใหม่ → Upgrade เป็น Premium ทันที + แจ้งเตือน webhook

---

## 🚀 ระบบ Batch Shooting

ยิงทีละ **10 บัญชีพร้อมกัน** (parallel batch)

- ถ้า batch ใดไม่มีใครได้เลย → **หยุดยิงทันที** (ไม่เสียเวลายิงต่อ)
- แต่ละบัญชีมี delay เล็กน้อยตาม config ป้องกัน API rate limit
- Premium user ยิง 3 เบอร์พร้อมกันใน batch เดียว

---

## 💾 ระบบจัดการ RAM (Memory Optimization)

ปัญหาหลักของ discord.js-selfbot-v13 คือ cache message ไม่มีขีดจำกัด → RAM โตเรื่อยๆ

**แก้ด้วย makeCache:**
```
MessageManager: 0       — ไม่เก็บ message ใน RAM เลย (event messageCreate ยังทำงานปกติ)
PresenceManager: 0      — ไม่ track online status ที่ไม่จำเป็น
VoiceStateManager: 0    — ไม่ track voice channel
GuildMemberManager: 50  — เก็บแค่ 50 member ต่อ server
```

**ผลลัพธ์จริง:**

| | ก่อนแก้ | หลังแก้ |
|---|---|---|
| 45 user, 2h 49m | 3,299MB | ~700-900MB |
| พฤติกรรม RAM | โตเรื่อยๆ ไม่หยุด | เสถียร |

**Auto-Restart:** ถ้า RAM เกิน 7GB (config) ระบบบันทึกข้อมูลและ restart อัตโนมัติ

---

## 🔒 ระบบ Dedup (ป้องกันยิงซ้ำ)

**สำหรับ Text/URL voucher — 2 Layer:**
- `processedVouchers` Map + TTL 1 นาที — ตรวจ URL ซองที่เจอแล้ว
- `voucherProcessing` Set (lock) — ป้องกัน 50 account แย่งกัน double-check หลัง lock

**สำหรับ QR — 2 Layer:**
- `scanLock` ด้วย messageId — 50 account เจอรูปเดียวกัน สแกนแค่ 1 ครั้ง
- `qr_hashes.txt` + Set — ภาพที่สแกนแล้วไม่สแกนซ้ำข้ามวัน/ข้าม session

**สำหรับ URL Redirect:**
- `urlBlacklistSet` + `url_blacklist.txt` — URL ที่ไม่ใช่ voucher บันทึกถาวร
- `urlPendingSet` — ป้องกัน 50 account follow redirect URL เดียวกันพร้อมกัน

---

## 📊 ระบบ Ranking Cache

คำนวณลำดับกระจายซองครั้งเดียว แล้ว cache ไว้ 24 ชั่วโมง
- อัพเดทอัตโนมัติเมื่อมี user เปิด/ปิดระบบ
- แสดงผ่าน `/user_ranking` command พร้อม server count และยอดเงินสะสม

---

## 📢 ระบบแจ้งเตือน

**Personal Webhook:** แจ้งทุกครั้งที่รับซองสำเร็จ พร้อม: จำนวนเงิน, ความเร็ว (ms), โหมด, เบอร์ที่ใช้, ลิงก์ต้นทาง (กดดูได้)

**Grouped Webhook:** รวมแจ้งเตือนจากทุก user ส่งทีละคน delay 3 วินาที ป้องกัน webhook rate limit

**การแจ้งเตือนอัตโนมัติ:**
- Token หมดอายุ
- Session หมดอายุ
- ยศ Premium เปลี่ยน (up/down)
- Server ไม่ถึงขั้นต่ำ

---

## 🛡️ ระบบความปลอดภัย

- ตรวจสอบ Token ด้วย Discord API ก่อนบันทึก
- บัญชีที่มี Server น้อยกว่า 25 (config) → ระบบปิดอัตโนมัติ
- Role checker ทุก 5 นาที ป้องกันใช้ Premium โดยไม่มียศ
- `unhandledRejection` และ `uncaughtException` handler ป้องกัน crash
- Graceful shutdown บันทึกข้อมูลก่อนปิดทุกครั้ง

---

## 🖥️ UI & Commands

**Discord Slash Commands:**
| Command | สิทธิ์ | ฟังก์ชัน |
|---|---|---|
| `/setup_gift` | Admin | ติดตั้งเมนูในช่อง |
| `/manage_users` | Admin | ดูรายชื่อและสถานะผู้ใช้ |
| `/system_info` | Admin | RAM, uptime, จำนวน user, เบอร์ |
| `/user_ranking` | Admin | ลำดับกระจายซองพร้อมข้อมูลครบ |

**Button UI (Ephemeral — เห็นคนเดียว):**
- ตั้งค่า Token / เบอร์ / Webhook ผ่าน Modal
- เลือกโหมด Normal / Premium
- เปิด/ปิด/ลบข้อมูล
- เช็คยอดสะสม

---

## 📦 Dependencies

```json
"@opecgame/twapi": "^1.2.1"
"axios": "^1.6.0"
"discord.js": "^14.14.1"
"discord.js-selfbot-v13": "^3.2.0"
"jsqr": "^1.4.0"
"sharp": "^0.33.4"
```

---

## 🗂️ โครงสร้างไฟล์

```
📁 botmeme790/
├── index.js          — ระบบหลักทั้งหมด
├── database.js       — จัดการข้อมูล user, balance, session
├── bot_config.js     — config ทั้งหมด (token, webhook, limit)
├── package.json
├── start.bat
│
📁 data/              — สร้างอัตโนมัติ (ไม่แนบมา)
├── users/            — ข้อมูล user แต่ละคน
│
📄 qr_hashes.txt      — สร้างอัตโนมัติเมื่อเจอ QR ครั้งแรก
📄 url_blacklist.txt  — สร้างอัตโนมัติเมื่อเจอ URL ที่ไม่ใช่ voucher
```

---

## ⚙️ การติดตั้ง

```bash
npm install
node index.js
```

แนะนำใช้ **PM2** พร้อม `--max-memory-restart 3000M`
