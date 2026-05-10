# UKPBJ Tracker — Kabupaten Pacitan

Sistem tracking & validasi pemakaian tenaga ahli pada paket pengadaan barang/jasa di lingkungan Pemerintah Kabupaten Pacitan.

**Core Rule:** 1 Tenaga Ahli (berdasarkan NIK) hanya dapat digunakan oleh 1 Penyedia selama paket pengadaan masih berjalan (belum BAST).

---

## Fitur Utama

- **Autentikasi** dengan JWT + bcrypt + httpOnly cookie
- **Role-based access**: Admin UKPBJ (full) + Pejabat Pengadaan PD (scope PD sendiri, read-only cross-PD)
- **Ganti password mandiri** oleh tiap user via menu profil (verifikasi password lama)
- **Manajemen Paket Pengadaan**: CRUD lengkap, status otomatis (berjalan / selesai)
- **Manajemen Penyedia & Tenaga Ahli**
- **Validasi NIK real-time** dengan 3-layer defense (UI debounced check → Prisma $transaction Serializable → PostgreSQL trigger + advisory lock)
- **Dashboard**: ringkasan paket, daftar TA aktif, histori penggunaan
- **Audit Log**: seluruh aktivitas (create/update/delete/login/konflik NIK) tercatat
- **Export Excel**: paket, tenaga ahli aktif, audit log
- **Healthcheck endpoint** (`/api/health`) untuk monitoring & docker healthcheck
- **UI modern-minimalis**: Bahasa Indonesia, format IDR & tanggal Indonesia, responsive

---

## Tech Stack

| Komponen   | Teknologi                              |
|------------|----------------------------------------|
| Frontend   | Next.js 14 (App Router) + TypeScript   |
| UI         | Tailwind CSS + shadcn/ui primitives    |
| Backend    | Next.js API Routes                     |
| ORM        | Prisma 5                               |
| Database   | PostgreSQL 16                          |
| Auth       | `jose` (JWT) + `bcryptjs`              |
| Export     | `exceljs`                              |
| Deployment | Docker + docker-compose                |

---

## Struktur Direktori

```
ukpbj-tracker/
├── prisma/
│   ├── schema.prisma           # model Prisma
│   ├── migrations/             # migrasi SQL + trigger & advisory lock
│   └── seed.ts                 # seed admin & sample PD users
├── src/
│   ├── app/
│   │   ├── (auth)/login/       # halaman login
│   │   ├── (dashboard)/        # dashboard, paket, penyedia, TA, users, audit
│   │   └── api/                # route handlers (auth, CRUD, export)
│   ├── components/
│   │   ├── ui/                 # shadcn-style primitives
│   │   └── dashboard/          # sidebar, topbar, page-header, status-badge
│   ├── lib/                    # prisma, auth, audit, format, validations, utils
│   ├── hooks/                  # use-debounced-value
│   ├── types/                  # shared TypeScript types
│   └── middleware.ts           # route protection
├── docker/entrypoint.sh        # migrate + (opsional) seed sebelum start
├── Dockerfile                  # multi-stage build
├── docker-compose.yml          # app + postgres
└── .env.example                # template environment
```

---

## Quickstart (Local Development)

### Prasyarat
- Node.js 20+ ([unduh](https://nodejs.org/))
- PostgreSQL 15+ (lokal, remote, atau via Docker)
- npm 10+ (bundled dengan Node.js 20)

### Cek prasyarat

```bash
node --version    # harus v20.x atau lebih baru
npm --version     # harus 10.x atau lebih baru
psql --version    # opsional, untuk cek PostgreSQL client
```

### Langkah

```bash
# 1) Install dependencies
npm install

# 2) Siapkan environment
cp .env.example .env
# Edit DATABASE_URL & JWT_SECRET sesuai lingkungan Anda

# 3) Jalankan migrasi (buat tabel + trigger)
npm run db:migrate

# 4) Seed admin default
npm run db:seed

# 5) Jalankan dev server
npm run dev
```

Aplikasi berjalan di `http://localhost:3000`.

**Kredensial default (dari seed):**
- Admin: `admin` / `admin123`
- Sample Pejabat PD: `pd.dinkes` / `pejabat123` (dan variasi lainnya, lihat `prisma/seed.ts`)

> **Ubah password setelah login pertama kali.**

---

## Deployment (Docker)

### Persiapan Server

```bash
# 1) Clone / copy source code ke server
cd /opt/ukpbj-tracker

# 2) Siapkan .env
cp .env.example .env
nano .env
```

Minimal yang HARUS diisi di `.env`:

```env
# DB credentials (dipakai oleh docker-compose)
POSTGRES_DB=ukpbj_tracker
POSTGRES_USER=ukpbj
POSTGRES_PASSWORD=<password-kuat>

# JWT secret — generate dengan: openssl rand -base64 32
JWT_SECRET=<minimal-32-karakter-random>

# Port app (default 3000)
APP_PORT=3000

# Seed admin sekali saja di first deploy
RUN_SEED=true
SEED_ADMIN_USERNAME=admin
SEED_ADMIN_PASSWORD=<password-kuat-untuk-admin>
SEED_ADMIN_NAMA=Administrator UKPBJ
```

### Build & Start

```bash
docker compose up -d --build
```

Setelah container berjalan dan seed selesai, **ubah `RUN_SEED=false`** di `.env` dan restart:

```bash
docker compose up -d
```

### Upgrade (deploy versi baru)

```bash
git pull   # atau copy source terbaru
docker compose up -d --build
```

Migrasi Prisma akan otomatis dijalankan oleh entrypoint.

### Backup Database

```bash
docker exec ukpbj-db pg_dump -U ukpbj ukpbj_tracker > backup-$(date +%Y%m%d).sql
```

### Restore Database

```bash
cat backup-YYYYMMDD.sql | docker exec -i ukpbj-db psql -U ukpbj -d ukpbj_tracker
```

---

## Arsitektur Keamanan NIK

Mencegah 1 NIK dipakai oleh lebih dari 1 penyedia pada paket yang masih berjalan:

1. **UI Layer** (`src/app/(dashboard)/paket/[id]/add-tenaga-ahli-form.tsx`)  
   Debounced 400ms ke `/api/tenaga-ahli/check-nik` → UX cepat, tombol submit dinonaktifkan saat konflik.

2. **Application Layer** (`src/app/api/paket/[id]/tenaga-ahli/route.ts`)  
   `prisma.$transaction` dengan `isolationLevel: Serializable`. Proses find-or-create TA + assignment dalam satu transaksi atomik.

3. **Database Layer** (`prisma/migrations/.../migration.sql`)  
   - Fungsi `check_tenaga_ahli_availability()` sebagai trigger `BEFORE INSERT/UPDATE` pada `paket_tenaga_ahli`.  
   - `pg_advisory_xact_lock(hashtext('ukpbj_ta_lock'), tenaga_ahli_id)` memastikan serialisasi per-NIK walau ada ribuan request konkuren.  
   - Helper function `assign_tenaga_ahli_safe()` dipanggil dari application code via raw SQL untuk lock eksplisit.

Jika trigger mendeteksi konflik, ia melempar exception dengan pesan yang mudah dibaca (contoh: *"NIK sedang digunakan oleh PT. ABC pada paket X (nomor kontrak: 027/xxx)"*) dan SQLSTATE `23505`. Route handler menerjemahkan ini menjadi HTTP 409 Conflict.

---

## Otorisasi

| Role          | Paket                          | Penyedia    | TA        | Users    | Audit     |
|---------------|--------------------------------|-------------|-----------|----------|-----------|
| admin         | Full CRUD semua PD             | Full        | Full      | Full     | Full view |
| pejabat_pd    | Full CRUD PD sendiri, read cross-PD | Full   | Full      | -        | -         |

Diatur via `canMutatePaket()` di `src/lib/paket-helpers.ts`.

---

## Scripts

```bash
npm run dev          # dev server
npm run build        # production build (prisma generate + next build)
npm run start        # start production server
npm run typecheck    # TypeScript check tanpa emit
npm run lint         # ESLint
npm run test         # jalankan semua unit test (Vitest)
npm run test:watch   # Vitest mode watch
npm run test:ui      # Vitest UI (dashboard interaktif di browser)
npm run test:coverage # jalankan test + laporan coverage
npm run db:migrate   # prisma migrate dev (pembuatan migrasi baru)
npm run db:deploy    # prisma migrate deploy (production)
npm run db:seed      # jalankan seed
npm run db:studio    # Prisma Studio
```

---

## Testing

Stack: **Vitest 2.x** + **jsdom** (untuk komponen React bila diperlukan).

### Struktur test

```
tests/
├── setup.ts                 # env vars dummy + stub next/headers (cookies/headers)
├── lib/
│   ├── format.test.ts       # format tanggal, IDR, angka, label dictionary
│   ├── validations.test.ts  # semua zod schema (login, paket, TA, user, dll)
│   ├── paket-helpers.test.ts # canMutatePaket, paketStatus, toPaketListItem, toPaketDetail
│   └── auth.test.ts         # JWT sign/verify, guard, HttpError
└── api/
    └── change-password.test.ts  # contoh integration test API (prisma + audit mocked)
```

### Menjalankan

```bash
# Sekali jalan (CI-friendly)
npm run test

# Watch mode saat development
npm run test:watch

# Dashboard UI
npm run test:ui

# Dengan coverage report (threshold 70% di src/lib/**)
npm run test:coverage
```

Laporan coverage HTML tersimpan di `coverage/index.html`.

### Menulis test baru

- File lib / util: taruh di `tests/lib/*.test.ts`, environment `node` (default, cepat).
- File React component: pakai `// @vitest-environment jsdom` di atas file + import `@testing-library/react` bila nanti ditambahkan.
- Test API route: mock `@/lib/prisma` dan `@/lib/audit` via `vi.mock(...)`, lihat `tests/api/change-password.test.ts` sebagai template.
- Untuk simulasi user login, gunakan helper dari `tests/setup.ts`:

```ts
import { __testCookieJar } from "../setup";
import { signSession, COOKIE_NAME } from "@/lib/auth";

const token = await signSession({ uid: 1, username: "admin", nama: "A", role: "admin", pdNama: null });
__testCookieJar._inject(COOKIE_NAME, token);
```

### Catatan

- `tests/setup.ts` otomatis membersihkan cookie jar sebelum setiap test.
- `JWT_SECRET` di test pakai nilai dummy yang aman — tidak perlu `.env` terpisah.
- Test tidak butuh database sungguhan; `prisma` di-mock per test sesuai kebutuhan.

---

## Troubleshooting

### Cek status aplikasi
Endpoint `GET /api/health` (public, tanpa auth) mengembalikan:
- `200` jika app + database sehat → `{ "status": "ok", "db": "up", "uptime": ..., "latencyMs": ... }`
- `503` jika database tidak terjangkau → `{ "status": "degraded", "db": "down", "error": ... }`

Docker compose otomatis memakai endpoint ini sebagai container healthcheck.

### `JWT_SECRET is not set or too short`
`.env` belum di-isi. Jalankan `openssl rand -base64 32` dan pasang sebagai `JWT_SECRET`.

### `Error: P1001: Can't reach database server`
- Pastikan PostgreSQL berjalan & `DATABASE_URL` benar.
- Di Docker: cek `docker compose logs db`.

### Migrasi trigger error di Prisma migrate
Semua trigger/advisory lock dimasukkan ke file SQL di `prisma/migrations/20260510000000_init/migration.sql`. Jika Anda perlu mengubahnya, buat migrasi baru dan JANGAN edit migration yang sudah di-deploy ke production.

### Logo Pacitan tidak muncul
Salin file PNG transparan logo ke `public/logo-pacitan.png`. Jika belum tersedia, tampilan tetap berfungsi (Image onError akan menyembunyikannya).

### Konflik NIK tidak terdeteksi
Pastikan migration dengan trigger sudah di-deploy. Verifikasi via `psql`:
```sql
\df check_tenaga_ahli_availability
\df assign_tenaga_ahli_safe
```

---

## Lisensi

Internal — Pemerintah Kabupaten Pacitan. Tidak untuk distribusi publik.
#   t e s t u k p b j  
 