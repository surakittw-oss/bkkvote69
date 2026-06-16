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

## การย้ายขึ้น Domain จริง

ข้อมูลทั้งหมด (คะแนน / โลโก้ / สถานะ) อยู่ใน Firebase project → **ย้ายตามไปอัตโนมัติ ไม่หาย**

### 🔴 ต้องแก้เมื่อเปลี่ยน domain

1. **Google Cloud Console → OAuth Client → Authorized JavaScript origins**
   เพิ่ม domain ใหม่ (ไม่งั้นปุ่ม login พัง)
   Client ID: `854069548103-...apps.googleusercontent.com`

2. **Firebase Console → Authentication → Settings → Authorized domains**
   เพิ่ม domain ใหม่ (ไม่งั้น `signInWithCredential` ล้มเหลว)

3. **`controller.html`** (ถ้าใช้ไฟล์นี้แยก) — แก้ URL hardcoded:
   ```js
   const OVERLAY_URL='https://bkkvote69.vercel.app/?overlay=live';
   ```

4. **`vercel.json`** — แก้ค่า host `ctrl.thestandard.co` ถ้าเปลี่ยน subdomain

### 🟢 ไม่ต้องแตะ

- `index.html` — ใช้ URL แบบ relative (`location.origin`) ทำงานทุก domain
- `firebaseConfig` — ผูกกับ Firebase project ไม่ใช่ hosting domain

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
