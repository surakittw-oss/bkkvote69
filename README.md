# BKK VOTE 69 — ระบบรายงานผลเลือกตั้งผู้ว่าฯ กทม. (Live)

Dashboard + Broadcast Overlay สำหรับรายงานผลการนับคะแนนเลือกตั้งผู้ว่าราชการ กทม. และ ส.ก. แบบเรียลไทม์ ออกแบบมาเพื่อใช้ออกอากาศบน YouTube Live / Facebook Live ผ่าน OBS / vMix

---

## โครงสร้างไฟล์

| ไฟล์ | หน้าที่ |
|---|---|
| `index.html` | ตัวระบบหลักทั้งหมด (dashboard + overlay + controller) — single-file รวม HTML/CSS/JS |
| `controller.html` | แผงควบคุมแบบ standalone สำหรับ deploy แยก subdomain |
| `vercel.json` | กฎ rewrite ของ Vercel (`ctrl.thestandard.co` → `controller.html`) |
| `logo-bkk-election-2569.webp` | โลโก้แคมเปญ (fallback เมื่อยังไม่มีโลโก้ใน Firebase) |

> **ไม่มีไฟล์ config JSON แยก** — ค่า Firebase และ OAuth Client ID ฝังอยู่ในโค้ดโดยตรง

---

## ลิงค์ทั้งหมด

### 🎛 Controller — แผงควบคุมออกอากาศ
```
https://bkkvote69.vercel.app/?control=1
```
- ต้อง login ด้วย Google บัญชี `@thestandard.co` เท่านั้น
- Studio Mode: **Preview → TAKE → On Air**
- subdomain (ถ้าตั้ง DNS): `https://ctrl.thestandard.co`

### 📺 Overlay สำหรับ OBS / vMix

| URL | แสดงผล | ขนาด |
|---|---|---|
| `?overlay=footer` | Top 3 ผู้สมัคร ↔ สถิติการนับ (วนสลับทุก 10 วิ) | lower-third |
| `?overlay=lower3` | Top 3 ผู้สมัคร | lower-third |
| `?overlay=stat` | สถิติการนับ + ผู้มาใช้สิทธิ | lower-third |
| `?overlay=bottombar` | แถบล่าง (ผู้ว่าฯ / ส.ก. สลับได้) | 1264×209 |
| `?overlay=side` | แผงข้าง (วนรายเขต / แผนที่) | 576×1080 |
| `?overlay=totals` | ตารางคะแนนรวม | fullscreen |
| `?overlay=map` | แผนที่ผู้ชนะ 50 เขต | fullscreen |
| `?overlay=bubble` | Bubble chart สัดส่วนคะแนน | fullscreen |
| `?overlay=live` | Composite 3 เลเยอร์ตามที่ controller สั่ง TAKE | — |

> overlay ทุกตัว (ยกเว้น `live`) ทำงานอิสระ ไม่ต้องรอ controller — ดึงข้อมูลจาก Firebase มาแสดงทันที

### 📊 Dashboard สาธารณะ
```
https://bkkvote69.vercel.app/
```

---

## สถาปัตยกรรม

```
┌─────────────┐     เขียน (ต้อง auth)    ┌──────────────────┐
│  Controller  │ ───────────────────────► │ Firebase RTDB     │
│  ?control=1  │                          │  · election/      │
└─────────────┘                          │  · overlay/control│
                                          │  · campaign/logo  │
┌─────────────┐      อ่าน (public)        └──────────────────┘
│  Overlays    │ ◄────────────────────────────────┘
│  Dashboard   │
└─────────────┘
```

- **Controller** สั่งงานผ่าน path `overlay/control` (footer / side / full แยกอิสระ)
- **Overlay `live`** ฟัง `overlay/control` แล้วประกอบ 3 เลเยอร์ตามสถานะ
- **คะแนน** อยู่ที่ `election/` — controller แก้ได้, overlay/dashboard อ่านอย่างเดียว

---

## ความปลอดภัย (Firebase Rules)

- **อ่าน:** สาธารณะ (จำเป็น เพราะ OBS overlay ไม่ได้ login)
- **เขียน:** เฉพาะ `auth.token.email` ลงท้าย `@thestandard.co` + `email_verified == true`

```json
".write": "auth != null && auth.token.email_verified == true && auth.token.email.matches(/.*@thestandard[.]co$/)"
```

- Login ผ่าน **Google Identity Services** → bridge เป็น **Firebase Auth** (`signInWithCredential`)
- การตรวจ `@thestandard.co` ฝั่ง JS เป็นแค่ UX — กำแพงจริงคือ Firebase Rules

---

## การย้ายขึ้น Domain จริง (thestandard.co)

ข้อมูลทั้งหมด (คะแนน / โลโก้ / สถานะ) อยู่ใน Firebase project → **ย้ายตามไปอัตโนมัติ ไม่หาย**

### ขั้นตอนที่ 1 — ส่งไฟล์ให้ทีม Web/IT

ส่งไฟล์ทั้งหมดต่อไปนี้ให้วางไว้ใน **folder เดียวกัน** บน web server:

```
index.html
controller.html
bg-bottombar.png
bg-side-gov.png
bg-side-skg.png
bg-stat.png
logo-bkk-election-2569.webp
```

**สิ่งที่ต้องแจ้งทีม web:**
- เป็น **Static HTML** — ไม่ต้อง build, ไม่ต้อง Node.js
- ทุกไฟล์ต้องอยู่ **ระดับเดียวกัน** (ห้ามแยก subfolder) เพราะโค้ดอ้างไฟล์แบบ relative path
- ต้องเป็น **HTTPS เท่านั้น** — Firebase Auth ไม่ทำงานบน HTTP

### ขั้นตอนที่ 2 — Firebase Console (ต้องทำทุกครั้งที่เปลี่ยน domain)

#### 2.1 เพิ่ม Authorized domain
**Firebase Console** → Project `bkk-election-vote` → **Authentication** → **Settings** → **Authorized domains** → **Add domain**

เพิ่ม domain ที่จะใช้จริง เช่น:
- `vote.thestandard.co`
- `thestandard.co`

> ถ้าไม่เพิ่ม → ปุ่ม Login Google จะ error `auth/unauthorized-domain`

#### 2.2 ตรวจ Google OAuth Provider
**Authentication** → **Sign-in method** → **Google** → ต้องเป็น **Enabled**

#### 2.3 ตรวจ Realtime Database Rules
**Realtime Database** → **Rules** ต้องเป็น:
```json
{
  "rules": {
    ".read": true,
    ".write": "auth != null && auth.token.email_verified == true && auth.token.email.matches(/.*@thestandard[.]co$/)"
  }
}
```

#### 2.4 Google Cloud Console — Authorized JavaScript origins
**Google Cloud Console** → **APIs & Services** → **Credentials** → เลือก OAuth Client ID → เพิ่มใน **Authorized JavaScript origins**:
- `https://vote.thestandard.co` (หรือ domain จริงที่ใช้)

Client ID ที่ใช้: `854069548103-...apps.googleusercontent.com`

> ถ้าไม่เพิ่ม → ปุ่ม Login พัง แม้ Firebase จะ authorize แล้ว

### ขั้นตอนที่ 3 — แก้ไขโค้ด (ถ้าจำเป็น)

| ไฟล์ | แก้อะไร | เมื่อไหร่ |
|---|---|---|
| `controller.html` | URL hardcoded `OVERLAY_URL='https://bkkvote69.vercel.app/?overlay=live'` | ถ้าใช้ไฟล์นี้แยก |
| `vercel.json` | host `ctrl.thestandard.co` | ถ้าเปลี่ยน subdomain |
| `index.html` | **ไม่ต้องแก้** ใช้ relative URL | — |
| `firebaseConfig` | **ไม่ต้องแก้** ผูก Firebase project ไม่ใช่ domain | — |

### ขั้นตอนที่ 4 — ตรวจสอบหลัง deploy

เปิดบน domain จริงแล้วเช็ค:

1. `https://vote.thestandard.co/` → Dashboard โหลด + Firebase Live status ขึ้น
2. `https://vote.thestandard.co/?control=1` → กด Login Google ด้วย email `@thestandard.co` ต้องเข้าได้
3. `https://vote.thestandard.co/?overlay=live` → Overlay แสดงผลไม่มี error
4. เปิด **F12 → Console** → ไม่มี error `unauthorized-domain` หรือ `CORS`

### Checklist สรุป

| รายการ | ทำที่ไหน | สถานะ |
|---|---|---|
| Upload ไฟล์ทั้งหมดในระดับเดียวกัน | Web server | ☐ |
| HTTPS เท่านั้น | Web server | ☐ |
| เพิ่ม Authorized domain | Firebase Console → Auth → Settings | ☐ |
| Google OAuth enabled | Firebase Console → Auth → Sign-in method | ☐ |
| Database Rules `@thestandard.co` | Firebase Console → Realtime DB → Rules | ☐ |
| เพิ่ม JS origin | Google Cloud Console → OAuth Credentials | ☐ |
| Google Sheets เปิด public view | Google Sheets (ECT + ส.ก. image sheet) | ☐ |
| แก้ `OVERLAY_URL` ใน controller.html | Code editor | ☐ (ถ้าใช้) |

---

## Firebase Project

| ค่า | |
|---|---|
| Project ID | `bkk-election-vote` |
| Region (RTDB) | `asia-southeast1` |
| Auth | Google provider (Safelist external Client ID) |

---

## Deploy

Push เข้า `main` → Vercel auto-deploy

```bash
git add -A
git commit -m "..."
git push
```
