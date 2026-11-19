# Tutorial Daffiyanet RTRWNET Management & Billing

Tutorial cara kerja sistem ini — dari gambaran besar sampai alur operasional sehari-hari.

---

## 1. Gambaran besar

Satu **server Node.js** menjalankan seluruh fungsi: billing, portal pelanggan, admin, teknisi, agen, kolektor, voucher Hotspot, integrasi jaringan (MikroTik/OLT/GenieACS), dan WhatsApp/Telegram — semuanya satu app, dipisah hanya lewat **EJS views** (frontend) dan **routes/services** (backend).

```
┌───────────────────────────────────────────────────────┐
│                   STB host (Armbian)                  │
│  ┌──────────────────────┐   ┌────────────────────┐    │
│  │  Node.js app (web)   │──▶│  SQLite (DB)       │    │
│  │  app-customer.js     │   └────────────────────┘    │
│  │  Express + EJS       │                               │
│  ├─ MikroTik API        │   (PPPoE profile/isolation)  │
│  ├─ GenieACS (TR-069)   │   (ONT/router pelanggan)     │
│  ├─ OLT SNMP            │   (PON port & ONU info)      │
│  ├─ WhatsApp (Baileys)  │   (notif WA otomatis)        │
│  └─ QRIS / WinMe link   │   (link bayar per pelanggan) │
│     port 3000            │                             │
└─────────┬─────────────────────────────────────────────┘
          │ Cloudflare Tunnel (HTTPS)
          ▼
  https://billingservices.daffiyanet.my.id
```

---

## 2. Alur kerja utama (contoh: tagihan bulanan)

1. **Cron** membuat invoice otomatis tiap bulan untuk tiap pelanggan aktif sesuai paket (`packages.price`).
2. Invoice muncul di **Portal Pelanggan** (menu "Cek Tagihan") dan/atau dikirim lewat **WhatsApp**.
3. Pelanggan bayar lewat channel yang Anda sediakan (QRIS / link WinMe per-pelanggan `customers.payment_link` / Kas/manual).
4. Untuk **QRIS**: notifikasi masuk via listener Android → endpoint `POST /api/webhook/v1/payment-notif` → sistem mencocokkan **nominal unik** dengan invoice → **otomatis menandai lunas** + **otomatis membuka isolir**.
5. Kalau pelanggan **lewat jatuh tempo** → **isolir otomatis** (PPPoE dipastikan suspend, MikroTik set profile isolir).
6. Setelah bayar & semua tagihan lunas → **unisolir otomatis** (profile PPPoE dikembalikan ke paket semula) via `markAsPaid()` → `activateCustomer()`.

---

## 3. Struktur repo (map singkat)

| Folder | Isi |
|---|---|
| `app-customer.js` | Entry utama: Express app, session, mounting semua routes, webhook, rate-limit, `/health` |
| `config/` | DB (`database.js`), settings (`settingsManager.js`), enkripsi, logger, i18n |
| `routes/` | Semua endpoint per peran: `adminPortal`, `customerPortal`, `customerAPI` (Android), `agentPortal`, `collectorPortal`, `techPortal`, `financePortal`, `settingsAPI`, `voucherPaymentAPI`, `acsPortal` |
| `services/` | Logic bisnis: `billingService` (markAsPaid, generate invoice), `adminService`, `customerService` (activate/suspend + MikroTik), `mikrotikService`, `cronService`, `voucherService`, `whatsappService`, `uniqueCodeService`, dll |
| `views/` | Frontend EJS per peran (`views/admin/*`, `views/customer/*`, dll) |
| `public/` | CSS/JS statis (login styles, theme), favicon, `manifest.webmanifest` (PWA) |
| `android-app/` | Source aplikasi Android (Jetpack Compose, Kotlin) — Admin panel + Customer app |
| `database/` | (tidak dipush) `billing.db` — SQLite |
| `settings.json` | (tidak dipush) kredensial asli; template: `settings.example.json` |

---

## 4. Cara menjalankan (lokal / paham projek)

Prasyarat: **Node.js ≥ 20**, **npm**, **SQLite3** (opsional untuk inspeksi).

```bash
# 1) clone
git clone https://github.com/Indra725/billing-rtrw.git
cd billing-rtrw
npm install

# 2) siapkan setting
cp settings.example.json settings.json   # lalu edit nilai-nilainya
```

Minimal yang harus diisi di `settings.json`:
- `admin_username`, `admin_password` (ganti dari default!)
- `mikrotik_host` (IP MikroTik), `mikrotik_user`, `mikrotik_password`
- `company_name`, `company_phone` (opsional)
- `webhook_secret` (token webhook pembayaran)
- `server_port` (default 3000)

```bash
# 3) jalankan
node app-customer.js
# app tersedia di http://localhost:3000
```

Jalur pertama:
- Portal admin: **http://localhost:3000/admin** → login `admin` / password dari `settings.json`
- Portal pelanggan: **http://localhost:3000/customer/login**
- Health check: **http://localhost:3000/health**

---

## 5. Cara kerja fitur (ringkasan)

### 5.1 Isolir otomatis (peraturan & alur)
- Setiap pelanggan default tanggal isolir (`isolir_day` di `settings.json`, default 10) atau override per-pelanggan di kolom `isolate_day`.
- Cron `CRON isolir` jalan tiap hari: pelanggan yang lewat `isolate_day` & masih `unpaid` → panggil `customerSvc.suspendCustomer()` → MikroTik ubah profile PPPoE ke `isolir_profile` → pelanggan diarahkan ke halaman isolir.
- Ketika invoice pelanggan `unpaid` → di admin muncul tombol "Isolir" dan customer langsung terisolir.

### 5.2 Unisolir otomatis
- Di `markAsPaid()` (ketika invoice ditandai lunas): cek apakah masih ada tagihan `unpaid` pelanggan → kalau 0 → cari customer dengan `status='suspended'|'isolated'` → panggil `activateCustomer()` → MikroTik kembalikan profile ke paket semula → pelanggan bisa browsing lagi.

### 5.3 QRIS otomatis via nominal unik
- Setiap tagihan bisa mendapat **kode unik** (0-99) → total = `amount + unique_code`.
- Pelanggan scan QR/transfer ke nominal tepat (mis. 105.005).
- Listener Android mendeteksi notifikasi masuk (QRIS/e-wallet/bank) → diteruskan ke `POST /api/webhook/v1/payment-notif`.
- Server mencocokkan nominal ↔ invoice (QRIS unique) → otomatis markAsPaid + unisolir.

### 5.4 GenieACS (TR-069)
- ONT/router pelanggan yang di-managed lewat TR-069 akan inform ke GenieACS.
- Server pakai GenieACS untuk reboot, ubah SSID/password WiFi, cek status ONU.

### 5.5 `android-app`
- Aplikasi Kotlin + Compose sebagai shell customer + admin panel. Admin juga bisa dibuka dari web.
- Build via Android Studio / gradle: `./gradlew :app:assembleDebug` (sinyal release membutuhkan keystore terpisah yang **tidak** disertakan di repo).

### 5.6 PWA
- `public/manifest.webmanifest` + `public/sw.js` → halaman bisa di-install ke HP.

---

## 6. Catatan operasional

- **Jangan** commit `settings.json`, `database/`, file keystore, atau dokumen customer.
- **Jangan** jalankan `npm install` di production STB tanpa alasan — `node_modules/` tidak disertakan di repo; Anda `npm install` di mesin deploy production, atau gunakan backup node_modules yang sudah ada di STB.
- Update produksi: edit → `node --check` → restart `billing-rtrw.service` → cek `/health` dan halaman publik.
- WHOOPS / ERROR: cek `journalctl -u billing-rtrw -n 50 --no-pager` untuk error log.

---

## 7. Lisensi

MIT (lihat `LICENSE`).
