# BKK VOTE 69 — ระบบรายงานผลเลือกตั้ง กทม. 2569

Dashboard + Broadcast Overlay สำหรับรายงานผลนับคะแนนเลือกตั้ง **ผู้ว่าฯ กทม.** และ **ส.ก. (สมาชิกสภากรุงเทพมหานคร) 50 เขต** แบบเรียลไทม์ ออกแบบมาเพื่อออกอากาศบน YouTube Live / Facebook Live ผ่าน OBS / vMix

---

## สถาปัตยกรรมระบบ

```
                   ┌────────────────────┐
                   │   ECT API          │  ← media.election.in.th (ข้อมูลคะแนนจริง)
                   │   (Bearer token)   │
                   └────────┬───────────┘
                            │ auto-refresh 30 วิ
                            ▼
┌─────────────┐  เขียน   ┌──────────────────┐  อ่าน (public)   ┌──────────────┐
│  Controller  │ ───────► │ Firebase RTDB     │ ◄──────────────── │  Dashboard   │
│  ?control=1  │          │  · election/      │                   │  (public)    │
│  @thestandard│          │  · overlay/control│                   └──────────────┘
│  .co only    │          │  · campaign/logo  │                   ┌──────────────┐
└─────────────┘          │  · election/      │ ◄──────────────── │  OBS Overlay │
                          │    snapshots/     │                   │  ?overlay=X  │
  ┌─────────────┐         └──────────────────┘                   └──────────────┘
  │ Google Sheets│ ← สี/รูปผู้สมัคร, โลโก้พรรค (CSV export)
  └─────────────┘
```

### การไหลของข้อมูล

1. **Controller** (ต้อง login `@thestandard.co`) ดึงคะแนนจาก ECT API ทุก 30 วินาที
2. **Auto-save** เขียนลง Firebase ทุก 15 วินาที + snapshot ทุก 30 นาที
3. **Dashboard / Overlay** อ่านจาก Firebase RTDB แบบ realtime listener (ไม่ต้อง login)
4. **รูป/สี ส.ก.** ดึงจาก Google Sheets (CSV export) → จับคู่ผู้สมัครด้วยชื่อ+พรรค

---

## โครงสร้างไฟล์

| ไฟล์ | หน้าที่ |
|---|---|
| `index.html` | ตัวระบบหลักทั้งหมด (Dashboard + Overlay + Controller) — single-file HTML/CSS/JS |
| `controller.html` | แผงควบคุมแบบ standalone สำหรับ deploy แยก subdomain |
| `vercel.json` | กฎ rewrite ของ Vercel (`ctrl.thestandard.co` → `controller.html`) |
| `bg-*.png` | ภาพพื้นหลัง Art สำหรับ Overlay (bottombar, side panel, stat) |
| `logo-bkk-election-2569.webp` | โลโก้แคมเปญ (fallback เมื่อยังไม่มีโลโก้ใน Firebase) |

> **ไม่มีไฟล์ config แยก** — ค่า Firebase, OAuth Client ID, ECT API base URL ฝังในโค้ดโดยตรง

---

## Google Sheets ที่ใช้

### Sheet 1 — ข้อมูลผู้สมัคร ส.ก. (Spreadsheet `1abANAC...`)

| gid | ชื่อ tab | ใช้ทำอะไร | โหลดเมื่อ |
|---|---|---|---|
| `891915583` | Dashboard ภาพ ส.ก. | รูป portrait + สี ลง popup dashboard | ทุก mode (ยกเว้น preview) |
| `916858386` | Overlay Side Panel | ไฟล์ Art composite (Drive) ลง side panel | Overlay mode เท่านั้น |
| `1384942493` | Overlay สี/รูปบาร์ | สี/รูปผู้สมัครผู้ว่าฯ ใน bottombar | Overlay mode เท่านั้น |

### Sheet 2 — โลโก้พรรค (Spreadsheet `1M3_JgQ...`)

| gid | ใช้ทำอะไร |
|---|---|
| `398211398` | โลโก้พรรค ส.ก. ใน bottombar overlay |
| `0` | รูป portrait ผู้ว่าฯ สำหรับ side panel overlay |

### ข้อกำหนดของรูปใน Sheet

- **ต้องวาง URL เป็น text ธรรมดา** ในเซลล์ (ห้ามแนบไฟล์ / file attachment)
- ถ้าเป็น Google Drive ต้อง share เป็น **"Anyone with the link" (Viewer)** ไม่งั้น browser จะ block ด้วย `ERR_BLOCKED_BY_ORB`
- ระบบจะแปลง Drive link เป็น thumbnail URL อัตโนมัติ (`drive.google.com/thumbnail?id=...&sz=s400`)

### การจับคู่ชื่อผู้สมัคร (Sheet ↔ ECT API)

ECT API มีคำนำหน้า (นาย/นาง/นางสาว/ม.ร.ว./ม.ล.) แต่ Sheet ไม่มี ระบบจับคู่ 5 ขั้น:

1. ชื่อเต็ม (ตัดคำนำหน้าแล้ว) + พรรคตรง
2. ชื่อเต็ม (ไม่เช็คพรรค)
3. นามสกุล + พรรคตรง
4. ชื่อต้น + พรรคตรง
5. ชื่อต้น + นามสกุล 4 ตัวแรก + พรรคตรง

สีใช้การจับคู่ตาม **ชื่อพรรค** (normalize ตัดคำว่า "พรรค"/"กลุ่ม" ออก)

---

## ECT API (Election Commission of Thailand)

| ค่า | |
|---|---|
| Base URL | `https://media.election.in.th` |
| ผู้ว่าฯ slug | `bkk-governor-2026` |
| ส.ก. slug | `bkk-council-2026` |
| Auth | `Authorization: Bearer <token>` (ต้องขอจาก ECT) |

### Endpoints ที่ใช้

| Endpoint | ข้อมูล |
|---|---|
| `/api/media/elections/{slug}/auto/statistics?level=province` | สถิติรวม (total_votes, good_votes, coverage) |
| `/api/media/elections/{slug}/auto/candidates?level=...` | รายชื่อผู้สมัคร + คะแนน |
| `/api/media/elections/{slug}/auto/electionAreas?level=area` | ผลรายเขต (ผู้มาใช้สิทธิ, คะแนน) |

### สิ่งที่คำนวณจาก API

| ค่าที่แสดง | วิธีคำนวณ |
|---|---|
| **นับคะแนนแล้ว %** | `coverage.percentage` (% หน่วยเลือกตั้งที่รายงานผล) |
| **ผู้มาใช้สิทธิ** | `statistics.total_votes` หรือ `sum(DISTRICTS.cast)` |
| **รวมคะแนน ส.ก. รายเขต** | `electionArea.total_votes` (รวมบัตรเสีย) ไม่ใช่ sum คะแนนผู้สมัคร |

---

## ลิงค์ทั้งหมด

### Controller — แผงควบคุมออกอากาศ
```
https://bkkvote69.vercel.app/?control=1
```
- ต้อง login ด้วย Google บัญชี `@thestandard.co` เท่านั้น
- Studio Mode: **Preview → TAKE → On Air**
- subdomain: `https://ctrl.thestandard.co`

### Overlay สำหรับ OBS / vMix

| URL | แสดงผล | ขนาด |
|---|---|---|
| `?overlay=footer` | Top 3 ผู้สมัคร ↔ สถิติการนับ (วนสลับ) | lower-third |
| `?overlay=lower3` | Top 3 ผู้สมัคร | lower-third |
| `?overlay=stat` | สถิติการนับ + ผู้มาใช้สิทธิ | lower-third |
| `?overlay=bottombar` | แถบล่าง (ผู้ว่าฯ / ส.ก. สลับได้) | 1264×209 |
| `?overlay=side` | แผงข้าง (วนรายเขต) | 576×1080 |
| `?overlay=totals` | ตารางคะแนนรวม | fullscreen |
| `?overlay=map` | แผนที่ผู้ชนะ 50 เขต | fullscreen |
| `?overlay=bubble` | Bubble chart สัดส่วนคะแนน | fullscreen |
| `?overlay=live` | Composite 3 เลเยอร์ตามที่ controller สั่ง TAKE | — |

> Overlay ทุกตัว (ยกเว้น `live`) ทำงานอิสระ ไม่ต้องรอ controller — ดึงข้อมูลจาก Firebase มาแสดงทันที

### Dashboard สาธารณะ
```
https://bkkvote69.vercel.app/
```

---

## ความปลอดภัย

### Firebase Rules
- **อ่าน:** สาธารณะ (จำเป็น — OBS overlay ไม่ได้ login)
- **เขียน:** เฉพาะ `auth.token.email` ลงท้าย `@thestandard.co` + `email_verified == true`

```json
{
  "rules": {
    ".read": true,
    ".write": "auth != null && auth.token.email_verified == true && auth.token.email.matches(/.*@thestandard[.]co$/)"
  }
}
```

### Auth flow
1. Login ผ่าน **Google Identity Services** (client-side)
2. Bridge credential เป็น **Firebase Auth** (`signInWithCredential`)
3. ตรวจ `@thestandard.co` ฝั่ง JS เป็นแค่ UX — กำแพงจริงคือ Firebase Rules

---

## Firebase Project

| ค่า | |
|---|---|
| Project ID | `bkk-election-vote` |
| Region (RTDB) | `asia-southeast1` |
| Database URL | `bkk-election-vote-default-rtdb.asia-southeast1.firebasedatabase.app` |
| Auth Provider | Google (OAuth Client ID: `854069548103-...`) |

### Firebase data paths

| Path | ข้อมูล | เขียนโดย |
|---|---|---|
| `election/candidates` | ผู้สมัครผู้ว่าฯ + คะแนน | Controller (auto-save) |
| `election/districts` | ผลรายเขต 50 เขต | Controller (auto-save) |
| `election/skg` | ผล ส.ก. รายเขต | Controller (auto-save) |
| `election/mbstats` | สถิติจาก ECT API (coverage, statistics) | Controller (auto-save) |
| `election/meta` | timestamp อัปเดตล่าสุด | Controller (auto-save) |
| `election/snapshots/` | Snapshot คะแนนทุก 30 นาที (เก็บ 30 อัน) | Controller (auto) |
| `overlay/control` | สถานะ overlay (footer/side/full) | Controller (manual TAKE) |
| `campaign/logo` | โลโก้แคมเปญ (base64 หรือ URL) | Controller (upload) |

---

## การ Host / Deploy

### วิธีที่ 1 — Vercel (ใช้อยู่ปัจจุบัน)

Push เข้า `main` → Vercel auto-deploy

```bash
git add index.html controller.html vercel.json *.png *.webp
git commit -m "update"
git push
```

### วิธีที่ 2 — Static Web Server ใดก็ได้

**ข้อกำหนด:**
- **Static HTML** — ไม่ต้อง build, ไม่ต้อง Node.js, ไม่ต้อง backend
- ทุกไฟล์ต้องอยู่ **ระดับเดียวกัน** (ห้ามแยก subfolder) เพราะโค้ดอ้างไฟล์แบบ relative path
- ต้องเป็น **HTTPS เท่านั้น** — Firebase Auth ไม่ทำงานบน HTTP

**ไฟล์ที่ต้อง deploy:**
```
index.html
controller.html
bg-bottombar.png
bg-bottombar-headline.png
bg-side-gov.png
bg-side-skg.png
bg-skg.png
bg-skg-headline.png
bg-stat.png
logo-bkk-election-2569.webp
```

### ขั้นตอนตั้งค่า (ทุกครั้งที่เปลี่ยน domain)

#### 1. Firebase Console → Authentication → Settings → Authorized domains
เพิ่ม domain ที่ใช้จริง เช่น `vote.thestandard.co`
> ถ้าไม่เพิ่ม → Login error `auth/unauthorized-domain`

#### 2. Firebase Console → Authentication → Sign-in method → Google
ต้องเป็น **Enabled**

#### 3. Firebase Console → Realtime Database → Rules
ต้องเป็น rules ตามด้านบน (read: true, write: `@thestandard.co`)

#### 4. Google Cloud Console → APIs & Services → Credentials → OAuth Client ID
เพิ่มใน **Authorized JavaScript origins**: `https://vote.thestandard.co`
> ถ้าไม่เพิ่ม → ปุ่ม Login พัง แม้ Firebase จะ authorize แล้ว

#### 5. Google Sheets
Sheet ทั้งหมดต้อง share เป็น **"Anyone with the link" (Viewer)** — ทั้ง Sheet และไฟล์รูปใน Drive

### ตรวจสอบหลัง deploy

1. `https://<domain>/` → Dashboard โหลด + Firebase Live status ขึ้น
2. `https://<domain>/?control=1` → Login Google ด้วย `@thestandard.co` ต้องเข้าได้
3. `https://<domain>/?overlay=live` → Overlay แสดงผล ไม่มี error
4. เปิด **F12 → Console** → ไม่มี error `unauthorized-domain` หรือ `CORS`
5. เปิด popup ส.ก. → รูปผู้สมัครที่มี Drive link ต้องขึ้น (ไม่ใช่ตัวอักษร fallback)

### Checklist

| รายการ | ทำที่ไหน | ☐ |
|---|---|---|
| Upload ไฟล์ทั้งหมดในระดับเดียวกัน | Web server | |
| HTTPS เท่านั้น | Web server | |
| เพิ่ม Authorized domain | Firebase Console → Auth → Settings | |
| Google OAuth enabled | Firebase Console → Auth → Sign-in method | |
| Database Rules `@thestandard.co` | Firebase Console → Realtime DB → Rules | |
| เพิ่ม JS origin | Google Cloud Console → OAuth Credentials | |
| Google Sheets share "Anyone with link" | Google Sheets + Drive files | |
| ECT API key พร้อมใช้ | ขอจาก ECT / ทีม media | |
| แก้ `OVERLAY_URL` ใน controller.html | Code editor (ถ้าใช้ standalone controller) | |

---

## สรุปการทำงานของระบบ

### Dashboard (หน้าหลัก)
- แสดงผลคะแนนผู้ว่าฯ (Top 5 + bar chart) และ ส.ก. 50 เขต
- แผนที่ 50 เขตระบายสีตามผู้ชนะ (คลิกดูรายเขตได้)
- Popup ส.ก. แยกตามพรรค พร้อมรูป portrait จาก Google Sheet
- ดึงข้อมูลจาก Firebase RTDB แบบ realtime (auto-sync ทันที)

### Controller (แผงควบคุม)
- ดึงคะแนนจาก ECT API ทุก 30 วินาที (ต้องมี Bearer token)
- Auto-save ลง Firebase ทุก 15 วินาที
- Snapshot คะแนนทุก 30 นาที (เก็บไว้ 30 อัน สำหรับ fallback)
- Studio Mode: Preview → TAKE → On Air (ควบคุม overlay ที่ออกอากาศ)

### Overlay (สำหรับ OBS)
- หลายรูปแบบ: bottombar, side panel, stat, map, totals, bubble
- Side panel ส.ก. ใช้ไฟล์ Art composite จาก Sheet gid=916858386 (ถ้ามี) หรือ fallback เป็น portrait + ชื่อ
- Count Stats (นับคะแนนแล้ว %) คำนวณจาก `coverage.percentage` ของ ECT API (% หน่วยที่รายงานผล)
- ผู้มาใช้สิทธิ คำนวณจาก `sum(DISTRICTS.cast)` (ผลรวม cast ทุกเขต)

### Auto-recovery
- ถ้า Firebase มีข้อมูลแต่คะแนนเป็น 0 → fallback ดึง snapshot ล่าสุดที่มีคะแนนจริง
- ถ้า Sheet โหลดไม่ได้ → ใช้สี/รูปจาก ECT API หรือ fallback เป็นตัวอักษรย่อ
