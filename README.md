# Gantt Project — Aplikasi Manajemen Proyek

Aplikasi manajemen proyek berbasis Gantt Chart: penjadwalan hari kerja, dependensi antar tugas,
alokasi tim, kalender proyek, serta analisis Critical Path (CPM) dan diagram PERT.
Seluruh antarmuka berbahasa Indonesia. 
<img width="1338" height="622" alt="image" src="https://github.com/user-attachments/assets/cdc71ee8-c5ce-40da-8fd8-0c1ae1a822ac" />
<img width="1339" height="625" alt="image" src="https://github.com/user-attachments/assets/a44b0cb0-2c5b-4aa2-bc36-a79b9f71f557" />




Monorepo npm workspaces: mesin penjadwalan hidup di paket `shared` dan dipakai **sama persis** oleh
server (sumber kebenaran, disimpan ke database) dan client (pratinjau saat bar di-drag).

## Fitur

- **Autentikasi** — register, login, logout. Token JWT (Bearer) disimpan di `localStorage`,
  kata sandi di-hash dengan bcryptjs, rute auth dibatasi rate limit.
- **Tugas** — CRUD tugas dengan durasi (hari kerja), tanggal mulai paling awal, prioritas, warna,
  progres, dan estimasi PERT (optimistis/pesimistis). Sub-tugas berjenjang (parent-child) sampai
  kedalaman tertentu; tugas induk otomatis menghitung rentang, durasi, dan progres dari anaknya.
- **Dependensi** — FS (Finish-to-Start) dan SS (Start-to-Start) dengan jeda hari (lag); siklus
  ditolak dengan pesan jelas.
- **Gantt** — tabel WBS di kiri (bisa dilipat) + timeline SVG di kanan: zoom Hari/Minggu/Bulan,
  tombol "Hari ini", garis hari ini, arsiran akhir pekan dan hari libur, bar berwarna dengan
  lapisan progres, bar induk bergaya bracket, panah dependensi, drag untuk memindahkan dan
  resize untuk mengubah durasi, serta toggle "Sorot jalur kritis".
- **Resource** — anggota tim (nama, email, peran, gaji), alokasi ke tugas dalam persen, biaya
  proyek, dan peringatan over-alokasi bila total alokasi seorang anggota melebihi 100%.
- **Kalender** — atur hari akhir pekan per proyek dan tambahkan hari libur; durasi hanya menghitung
  hari kerja sehingga jadwal bergeser otomatis.
- **Analisis** — ringkasan proyek, tabel CPM (ES, EF, LS, LF, float, kritis), jalur kritis, dan
  diagram PERT activity-on-node yang bisa di-zoom dan di-scroll (node kritis berwarna merah).

## Tech stack

| Lapis | Teknologi |
| --- | --- |
| Frontend | React 18, Vite 6, TypeScript, Tailwind CSS 4, React Router, TanStack Query, React Hook Form, lucide-react |
| Backend | Node.js, Express 4, TypeScript (tsx untuk dev), zod, jsonwebtoken, bcryptjs, helmet, cors, express-rate-limit |
| Database | SQLite + Prisma ORM (`server/prisma/dev.db`, tidak ikut di-commit) |
| Logika inti | Paket `shared` (TypeScript murni, tanpa dependensi UI) |
| Gantt & PERT | Digambar sendiri dengan SVG, tanpa library Gantt eksternal |
| Test | Vitest, Supertest, Testing Library |

## Prasyarat

- Node.js >= 20 dan npm (npm workspaces).
- Tanpa layanan eksternal: database berupa file SQLite.

## Instalasi

```bash
npm install                                   # memasang seluruh workspace
cp .env.example .env                          # Windows: copy .env.example .env
#   isi JWT_SECRET dengan string acak, contoh:
#   node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
npm run db:migrate                            # membuat server/prisma/dev.db + skema
npm run db:seed                               # mengisi akun demo dan proyek contoh
```

`npm run db:seed` bersifat idempoten, aman dijalankan berulang. `npm run db:reset` mengulang
migrasi dari nol lalu menjalankan seed.

Berkas `.env` berisi nilai khusus lingkungan dan tidak boleh di-commit. Variabel yang dibaca server:

| Variabel | Contoh | Keterangan |
| --- | --- | --- |
| `DATABASE_URL` | `file:./dev.db` | Relatif terhadap `server/prisma`, jadi menghasilkan `server/prisma/dev.db` |
| `JWT_SECRET` | string acak | **Wajib.** Server menolak start bila kosong |
| `PORT` | `3001` | Port API (default 3001) |

## Menjalankan

```bash
npm run dev
```

- Aplikasi web: <http://localhost:5173> (Vite dev server; permintaan `/api` di-proxy ke port 3001)
- API: <http://localhost:3001/api/health>
- Bila port 5173 sedang terpakai, Vite otomatis memakai port berikutnya (5174, dan seterusnya) —
  perhatikan alamat yang dicetak di terminal.

**Akun demo** (dari seed): `demo@example.com` / `Demo12345`.
Proyek contoh: *Pengembangan Aplikasi Mobile Toko Online* — 17 tugas, 4 anggota, 13 dependensi,
hari libur 2026-11-11 dan 2026-12-25.

## Test dan build

```bash
npm run typecheck   # tsc --noEmit di ketiga workspace
npm test            # Vitest: shared, server (Supertest), client (Testing Library)
npm run build       # build shared, server, lalu bundle produksi client (client/dist)
```

Hasil terakhir dijalankan pada 16 Oktober 2026: typecheck lolos, **97 test lolos** (36 `shared`,
43 `server`, 18 `client`), build lolos.

## Struktur repository

```
shared/src/      dates calendar scheduler cpm pert cost schemas types index  (+ *.test.ts)
server/
  prisma/        schema.prisma  seed.ts  migrations/
  src/           app.ts index.ts config.ts db.ts
                 middleware/ auth error validate
                 routes/     auth projects tasks dependencies resources calendar analysis
                 services/   access dto recompute snapshot
                 tests/      auth tasks calendar team analysis seed (Supertest)
client/src/      main.tsx App.tsx
                 api/ hooks/ lib/ styles/
                 pages/       Login Register Projects Gantt Team Calendar Analysis NotFound
                 components/  layout auth gantt pert calendar ui
```

## Ringkasan API

Prefiks `/api`. Semua rute selain `/api/health`, `/api/auth/register`, dan `/api/auth/login`
memerlukan header `Authorization: Bearer <token>`. Setiap data diperiksa kepemilikannya; data milik
pengguna lain dijawab `404`.

Format error: `{ "error": { "code": "STRING", "message": "pesan bahasa Indonesia", "details": {} } }`.
Status: `400`/`422` validasi, `401` belum login, `404` tidak ditemukan, `409` konflik.

| Method dan path | Fungsi |
| --- | --- |
| GET `/api/health` | Cek server hidup |
| POST `/api/auth/register` | Daftar (name, email, password min 8 karakter), mengembalikan token dan user |
| POST `/api/auth/login` | Masuk, mengembalikan token dan user |
| GET `/api/auth/me` | Data user saat ini |
| GET, POST `/api/projects` | Daftar dan buat proyek |
| GET, PATCH, DELETE `/api/projects/:id` | Detail (snapshot lengkap), ubah, hapus |
| POST `/api/projects/:id/tasks` | Buat tugas (boleh dengan `parentId`) |
| PATCH, DELETE `/api/tasks/:id` | Ubah atau hapus tugas (anak ikut terhapus) |
| POST `/api/tasks/:id/move` | Ubah parent dan posisi |
| POST `/api/tasks/:id/dependencies` | Tambah predecessor (`predecessorId`, `type`, `lagDays`) |
| PATCH, DELETE `/api/dependencies/:id` | Ubah atau hapus dependensi |
| GET, POST `/api/projects/:id/resources` | Daftar dan tambah anggota tim |
| PATCH, DELETE `/api/resources/:id` | Ubah atau hapus anggota |
| POST `/api/tasks/:id/assignments` | Alokasikan anggota (`resourceId`, `allocationPercent`) |
| PATCH, DELETE `/api/assignments/:id` | Ubah atau hapus alokasi |
| PUT `/api/projects/:id/calendar` | Atur `weekendDays`, jadwal dihitung ulang |
| POST `/api/projects/:id/holidays` | Tambah hari libur (`date`, `name`) |
| DELETE `/api/holidays/:id` | Hapus hari libur |
| GET `/api/projects/:id/analysis` | Hasil CPM, jalur kritis, data PERT, biaya, beban resource |

Setiap mutasi yang mengubah jadwal mengembalikan `{ tasks: [...] }` berisi seluruh tugas terbaru,
sehingga client cukup memperbarui cache.

## Asumsi

1. **Logout** dilakukan di client dengan menghapus token dari `localStorage`; tidak ada endpoint
   logout server karena token bersifat stateless dan hanya dikirim lewat header Bearer.
2. **Tanggal** di API selalu format ISO `YYYY-MM-DD`; tampilan memakai format Indonesia
   (`19 Oktober 2026`). Mata uang ditampilkan dalam Rupiah tanpa desimal.
3. **Durasi** adalah jumlah hari kerja (bukan selisih hari kalender). Tugas berdurasi 1 hari selesai
   pada hari mulai.
4. **Tugas induk** tidak punya durasi/tanggal mulai sendiri: nilainya dihitung dari anak
   (rentang = hari kerja terawal mulai sampai terakhir selesai, durasi = jumlah hari kerja rentang,
   progres = rata-rata berbobot durasi anak). Kolom ini read-only di form, dan tugas induk tidak
   boleh memiliki dependensi atau alokasi.
5. **Kedalaman sub-tugas** dibatasi (`MAX_TASK_DEPTH` di `shared`) agar tampilan Gantt tetap
   terbaca.
6. **Nilai PERT** (`expected`, `variance`, `stdDev`) dibulatkan 2 desimal di mesin `shared`.
   `pertPath` merangkum tugas-tugas di jalur kritis yang punya estimasi; nilainya `null` bila tidak
   ada satu pun yang punya estimasi.
7. **Rate limit** rute auth: 30 permintaan per 15 menit per alamat IP.
8. **CORS** dibuka hanya untuk origin `http://localhost:<port>` dan `http://127.0.0.1:<port>`
   (pengembangan lokal). Sesi memakai token Bearer, bukan cookie.
9. **Database** SQLite per lingkungan: file `server/prisma/dev.db` (di-`.gitignore`). Test server
   memakai database SQLite terpisah yang dibuang setelah dijalankan.
10. **Drag dan resize** pada timeline mengubah `startDate`/`durationDays` tugas daun lewat `PATCH`
    `/api/tasks/:id`, lalu jadwal dihitung ulang oleh server.

## Batasan

- Di luar cakupan: kolaborasi real-time, ekspor PDF, notifikasi email, multi-bahasa, deployment cloud.
- Test client berjalan di jsdom; interaksi drag dan tampilan SVG diverifikasi melalui test komponen
  dan build produksi, bukan melalui browser otomatis.
