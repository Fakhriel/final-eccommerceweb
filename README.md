# STRIDE — Kicks Lab

Project e-commerce sepatu yang **masih dalam tahap pengembangan**. Terdiri dari tiga aplikasi terpisah: toko (customer-facing), dashboard admin, dan REST API backend.

> ⚠️ **Status: Work in Progress — belum production-ready.** Sebagian fitur sudah memanggil API backend sungguhan (bukan data dummy), sebagian lain masih UI saja tanpa logic di belakangnya. README ini ditulis berdasarkan pembacaan kode, **belum lewat testing menyeluruh** (manual maupun otomatis). Cek [Status Fitur](#status-fitur) dan [Known Issues](#known-issues--temuan-qa) sebelum dipakai.

---

## Daftar Isi

- [Arsitektur](#arsitektur)
- [Tech Stack](#tech-stack)
- [Struktur Folder](#struktur-folder)
- [Status Fitur](#status-fitur)
- [Setup & Instalasi](#setup--instalasi)
- [Environment Variables](#environment-variables)
- [Skema Database](#skema-database)
- [API Reference](#api-reference)
- [Known Issues](#known-issues--temuan-qa)
- [Konvensi & Catatan Desain](#konvensi--catatan-desain)
- [Roadmap](#roadmap)

---

## Arsitektur

Tiga aplikasi berdiri sendiri (bukan monorepo workspace), saling terhubung lewat HTTP ke satu backend yang sama:

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────┐
│   kicks-lab      │      │  kick-dashboard   │      │    kicks-api     │
│   (storefront)   │      │  (admin panel)    │      │    (backend)     │
│   Astro :4321    │─────▶│   Astro :4322     │─────▶│  Express :4000   │
└─────────────────┘      └──────────────────┘      └────────┬────────┘
                                                              │
                                                    ┌─────────┴─────────┐
                                                    │   MySQL (kicks_lab) │
                                                    └─────────────────────┘
                                                              │
                                          ┌───────────────────┼───────────────────┐
                                          ▼                   ▼                   ▼
                                   Midtrans Snap      RajaOngkir Komerce   (storage gambar:
                                   (pembayaran)         (ongkos kirim)       URL eksternal)
```

Customer dan admin memakai jalur autentikasi JWT yang **dipisahkan lewat klaim `role`** meski berbagi `JWT_SECRET` yang sama — token admin punya klaim `role: "admin"` yang dicek middleware `requireAdmin`, sehingga (sejauh terbaca dari kode) token customer biasa seharusnya tidak bisa dipakai mengakses endpoint admin. Belum diuji langsung dengan percobaan bypass.

---

## Tech Stack

| Layer | Teknologi |
|---|---|
| Storefront | Astro 6, Tailwind CSS v4, GSAP |
| Admin Dashboard | Astro 6, Tailwind CSS v4, GSAP |
| Backend API | Node.js (ESM), Express 4, MySQL2 |
| Database | MySQL (InnoDB) |
| Auth | JWT (`jsonwebtoken`), `bcrypt` untuk hashing password |
| Payment Gateway | Midtrans Snap (sandbox/production toggle) |
| Ongkos Kirim | RajaOngkir Komerce API |
| Rate Limiting | `express-rate-limit` (modulnya ada, **belum terpasang aktif** — lihat Known Issues #1) |

---

## Struktur Folder

```
tokosepatu/
├── kicks-lab/          # Storefront — yang dilihat pembeli
│   └── src/
│       ├── components/ # Navbar, Hero, ProductGrid, ProductCard, Footer, dst
│       ├── data/       # Data statis (poster kategori, dll)
│       ├── lib/        # AuthState, CartWishlist, rajaongkir — semua fetch ke kicks-api
│       └── pages/
│           ├── index.astro
│           ├── pria.astro / wanita.astro
│           ├── koleksi-baru.astro
│           ├── produk/[slug].astro   # halaman detail produk (paling kompleks)
│           ├── profile.astro         # akun, riwayat order, wishlist
│           └── tentang.astro
│
├── kick-dashboard/      # Admin panel — kelola produk, order, pembayaran
│   └── src/
│       ├── components/  # Sidebar, Topbar, StatCard, PageHeader
│       ├── lib/         # auth.ts (admin), export.ts
│       └── pages/
│           ├── index.astro       # overview (placeholder, lihat status)
│           ├── login.astro
│           ├── products.astro    # CRUD produk lengkap
│           ├── orders.astro
│           ├── payments.astro
│           ├── customers.astro   # placeholder
│           ├── categories.astro  # placeholder
│           ├── roi.astro         # placeholder
│           └── admins.astro      # placeholder
│
└── kicks-api/           # Backend — satu-satunya sumber kebenaran data
    ├── sql/             # Migration berurutan (01 → 07)
    ├── scripts/         # seed-admin.js, generate-slugs.js
    └── src/
        ├── config/      # koneksi MySQL pool
        ├── controllers/ # logika bisnis per domain
        ├── db/          # query layer (raw SQL via mysql2)
        ├── middleware/  # requireAuth, requireAdmin, rate limiter, error handler
        ├── routes/      # definisi endpoint per domain
        └── utils/       # helper RajaOngkir sisi server
```

---

## Status Fitur

Legenda: ✅ memanggil endpoint API sungguhan (bukan data dummy) · 🟡 sebagian terhubung, sebagian masih statis · ⬜ UI saja, belum ada logic ke backend

> Status di bawah berasal dari membaca kode (apakah ada pemanggilan `fetch`/`adminApiFetch` ke endpoint nyata atau tidak), **bukan dari hasil testing fungsional**. Sebuah fitur bertanda ✅ berarti kodenya memanggil API yang benar, tapi belum tentu sudah diuji untuk semua skenario (error handling, edge case, dll).

### Storefront (`kicks-lab`)

| Fitur | Status | Catatan |
|---|---|---|
| Navbar (scroll transparency, dropdown mobile, switcher destinasi) | ✅ | |
| Hero & katalog (Koleksi Baru, Pria, Wanita) | ✅ | |
| Detail produk (galeri, warna, size chart, stok per ukuran) | ✅ | Kode memvalidasi ulang stok & harga di server saat checkout |
| Auth (register/login) | ✅ | JWT disimpan di `localStorage` |
| Cart & Wishlist | ✅ | Wajib login, fetch ke `kicks-api` |
| Checkout & pembayaran (Midtrans Snap) | ✅ | Status order diupdate lewat webhook Midtrans, bukan langsung dari respons frontend — belum diuji end-to-end dengan transaksi sandbox sungguhan |
| Cek ongkos kirim (RajaOngkir) | ✅ | |
| Riwayat pembayaran (`/profile`) | ✅ | |

### Admin Dashboard (`kick-dashboard`)

| Fitur | Status | Catatan |
|---|---|---|
| Login admin | ✅ | Terhubung ke `/api/admin-auth/login`, token di `sessionStorage` |
| CRUD Produk | ✅ | Termasuk gambar, warna, size chart, stok |
| Daftar & detail Order | ✅ | |
| Daftar Pembayaran | ✅ | |
| Overview / Dashboard utama | ⬜ | Card revenue, chart, top produk, order terbaru — semua **placeholder statis**, belum fetch data asli |
| Manajemen Kategori | ⬜ | UI ada, belum terhubung API |
| Manajemen Pelanggan | ⬜ | UI ada, belum terhubung API |
| ROI / Analitik | ⬜ | UI ada, belum terhubung API |
| Manajemen Admin (multi-admin) | ⬜ | UI ada, belum terhubung API — backend juga belum punya endpoint CRUD admin selain login |

### Backend (`kicks-api`)

| Domain | Endpoint tersedia |
|---|---|
| Products | List, detail (by id & slug), admin list, create, update, delete |
| Auth (customer) | Register, login, me |
| Admin Auth | Login, me |
| Cart | Get, add, update qty, delete |
| Wishlist | Get, add, delete |
| Ongkir | Cari destinasi, hitung ongkir |
| Payment | Buat transaksi (Snap), webhook notifikasi, cek status order, riwayat order |

---

## Setup & Instalasi

Prasyarat: **Node.js ≥ 22.12**, **MySQL** aktif, akun **Midtrans Sandbox**, API key **RajaOngkir Komerce**.

### 1. Backend (`kicks-api`)

```bash
cd kicks-api
npm install
```

Buat file `.env` (lihat [Environment Variables](#environment-variables)), lalu jalankan migration **berurutan** lewat MySQL client/phpMyAdmin:

```bash
mysql -u root -p < sql/01_schema.sql
mysql -u root -p < sql/02_seed.sql
mysql -u root -p < sql/03_add_slug.sql
mysql -u root -p < sql/04_lock_slug.sql
mysql -u root -p < sql/05_create_users.sql
mysql -u root -p < sql/06_create_wishlist_cart.sql
mysql -u root -p < sql/07_create_admins.sql
```

Buat akun admin pertama:

```bash
node scripts/seed-admin.js
```

> Skrip ini membuat admin dengan username `superadmin` dan password default yang **tertulis langsung di kode** (`scripts/seed-admin.js`). Ganti password tersebut di file sebelum menjalankan skrip pada environment apa pun yang bisa diakses orang lain, dan ganti lagi lewat database setelah login pertama.

Jika ada produk lama tanpa slug:

```bash
node scripts/generate-slugs.js
```

Jalankan server:

```bash
npm run dev     # dengan --watch, untuk development
npm start       # tanpa watch
```

Server jalan di `http://localhost:4000`. Cek `GET /health` untuk verifikasi cepat.

### 2. Storefront (`kicks-lab`)

```bash
cd kicks-lab
npm install
npm run dev      # http://localhost:4321
```

### 3. Admin Dashboard (`kick-dashboard`)

```bash
cd kick-dashboard
npm install
npm run dev      # http://localhost:4322 (port di-set eksplisit di package.json)
```

**Urutan start yang disarankan:** `kicks-api` → `kicks-lab` / `kick-dashboard`. Kedua frontend gagal fetch data (bukan crash) kalau backend belum jalan.

---

## Environment Variables

### `kicks-api/.env`

| Variabel | Wajib | Keterangan |
|---|---|---|
| `PORT` | Tidak | Default `4000` |
| `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` | Ya | Kredensial MySQL. Default fallback: `localhost`, `3306`, `root`, ``, `kicks_lab` |
| `FRONTEND_URL` | Disarankan | Daftar origin yang diizinkan CORS, pisahkan dengan koma. Default mencakup `:4321` dan `:4322` |
| `JWT_SECRET` | **Ya** | Wajib diisi — server akan error saat membuat token kalau kosong |
| `JWT_EXPIRES_IN` | Tidak | Default `7d` |
| `MIDTRANS_SERVER_KEY` | **Ya** (untuk checkout) | Server key dari dashboard Midtrans |
| `MIDTRANS_IS_PRODUCTION` | Tidak | `"true"` untuk pakai endpoint production, default sandbox |
| `RAJAONGKIR_API_KEY` | **Ya** (untuk ongkir) | Dari RajaOngkir Komerce |
| `RAJAONGKIR_ORIGIN_ID` | Ya (untuk ongkir) | ID kota/kecamatan asal pengiriman (gudang toko) |

### `kicks-lab/.env`

| Variabel | Keterangan |
|---|---|
| `PUBLIC_API_URL` | Base URL `kicks-api`, contoh `http://localhost:4000` |
| `PUBLIC_MIDTRANS_CLIENT_KEY` | Client key Midtrans untuk Snap.js di browser |
| `MIDTRANS_IS_PRODUCTION` | Dipakai untuk menentukan script Snap.js yang dimuat (sandbox vs production) |

### `kick-dashboard/.env`

| Variabel | Keterangan |
|---|---|
| `PUBLIC_API_URL` | Base URL `kicks-api`, sama seperti di atas |

> File `.env` **tidak boleh** di-commit ke version control. Pertimbangkan menambahkan `.env.example` di tiap folder berisi nama variabel saja (tanpa value) supaya kontributor lain tidak perlu menebak-nebak — saat ini belum ada file tersebut di repo.

---

## Skema Database

Database: `kicks_lab` (MySQL, `utf8mb4`). Dibuat lewat 7 file migration berurutan di `kicks-api/sql/`.

| Tabel | Fungsi |
|---|---|
| `products` | Data inti produk (nama, kategori, gender, harga, berat, status aktif) |
| `product_images` | Galeri foto produk (multi-row, max 10 divalidasi di level aplikasi) |
| `product_highlights` | Poin fitur singkat di halaman detail |
| `product_colors` | Varian warna + foto representatif per warna |
| `product_sizes` | Stok per ukuran — sumber kebenaran validasi checkout |
| `product_size_chart_headers` / `_rows` | Tabel ukuran custom per produk (disimpan JSON karena kolom beda-beda) |
| `users` | Akun customer. Kolom `google_id`/`facebook_id` disiapkan untuk social login di masa depan |
| `admins` | Akun staff/admin, terpisah dari `users` |
| `wishlists` | Relasi many-to-many user ↔ produk |
| `carts` | Item keranjang per user, unique per kombinasi produk+ukuran+warna |
| `orders` | Transaksi checkout. Status: `pending` → `paid` / `failed` / `expired` / `cancelled`, diupdate via webhook Midtrans |

Relasi kunci yang terbaca dari kode (belum diverifikasi lewat testing langsung):
- `orders.product_id` adalah referensi, tapi harga & nama produk disimpan terpisah (snapshot) di kolom `orders` saat transaksi dibuat — tujuannya supaya histori order tidak ikut berubah kalau produk diedit/dihapus belakangan.
- Pengurangan stok (`product_sizes.stock`) ditulis untuk hanya terjadi **sekali**, saat status order pertama kali berubah jadi `paid` — tujuannya mencegah notifikasi duplikat dari Midtrans mengurangi stok dua kali. Logic ini belum diuji dengan skenario notifikasi duplikat sungguhan.

---

## API Reference

Base URL: `http://localhost:4000` (dev). Semua response sukses berbentuk `{ data: ... }`, semua error berbentuk `{ error: "..." }`.

### Auth — customer (`/api/auth`)
| Method | Endpoint | Auth | Keterangan |
|---|---|---|---|
| POST | `/register` | – | Min. password 8 karakter |
| POST | `/login` | – | Pesan error dibuat sama untuk "email salah" & "password salah" (terlihat dari kode dimaksudkan untuk mencegah email enumeration) |
| GET | `/me` | Bearer (customer) | |

### Admin Auth (`/api/admin-auth`)
| Method | Endpoint | Auth | Keterangan |
|---|---|---|---|
| POST | `/login` | – | |
| GET | `/me` | Bearer (admin) | |

### Products (`/api/products`)
| Method | Endpoint | Auth | Keterangan |
|---|---|---|---|
| GET | `/` | – | Daftar produk publik |
| GET | `/:id` | – | Detail by ID |
| GET | `/slug/:slug` | – | Detail by slug (dipakai storefront) |
| GET | `/admin` | Bearer (admin) | Daftar lengkap untuk dashboard |
| POST | `/` | Bearer (admin) | Buat produk |
| PATCH | `/:id` | Bearer (admin) | Update produk |
| DELETE | `/:id` | Bearer (admin) | Hapus produk |

### Cart (`/api/cart`) — semua wajib login
| Method | Endpoint |
|---|---|
| GET | `/` |
| POST | `/` |
| PATCH | `/:cartItemId` |
| DELETE | `/:cartItemId` |

### Wishlist (`/api/wishlist`) — semua wajib login
| Method | Endpoint |
|---|---|
| GET | `/` |
| POST | `/` |
| DELETE | `/:productId` |

### Ongkir (`/api/ongkir`)
| Method | Endpoint | Keterangan |
|---|---|---|
| GET | `/destinasi` | Autocomplete pencarian kota/kecamatan |
| POST | `/cek` | Hitung ongkos kirim |

### Payment (`/api/payment`)
| Method | Endpoint | Auth | Keterangan |
|---|---|---|---|
| POST | `/transaksi` | Bearer (customer) | Buat transaksi Snap. Harga & stok divalidasi ulang dari DB, bukan dari input frontend |
| POST | `/notifikasi` | – (dipanggil Midtrans) | Webhook status pembayaran. Wajib didaftarkan di Midtrans Dashboard → Settings → Payment Notification URL |
| GET | `/order/:orderId` | – | Cek status order |
| GET | `/riwayat` | Bearer (customer) | Riwayat order user yang login |

---

## Known Issues / Temuan QA

Daftar ini adalah hasil pembacaan kode, bukan hasil pengujian end-to-end — disarankan tetap diverifikasi manual sebelum dianggap final.

1. **Rate limiter di-import tapi tidak aktif.** Di `kicks-api/src/routes/auth.js` dan `routes/adminAuth.js`, `loginLimiter`, `registerLimiter`, dan `adminLoginLimiter` diimpor dari `middleware/rateLimiter.js` tapi **tidak dipasang** sebagai middleware di route (`router.post("/login", login)` — tanpa limiter di antaranya). Endpoint login & register saat ini tidak punya proteksi brute-force aktif walau modulnya sudah ditulis lengkap. Perbaikan: tambahkan limiter sebagai argumen kedua, mis. `router.post("/login", loginLimiter, login)`.
2. **Dashboard Overview, Kategori, Pelanggan, ROI, dan Manajemen Admin belum terhubung ke API.** Halaman-halaman ini menampilkan UI/empty-state yang sudah jadi secara visual, tapi belum melakukan fetch data sungguhan. Backend juga belum punya endpoint untuk kategori, daftar pelanggan, ROI, atau CRUD admin selain login — perlu dirancang dan ditambahkan sebelum frontend-nya bisa disambungkan.
3. **Password admin default tertulis di source code.** `scripts/seed-admin.js` punya password default dalam bentuk plain text di kode (dengan komentar pengingat untuk diganti). Aman untuk dev lokal, tapi pastikan tidak pernah dijalankan dengan password default itu di environment yang bisa diakses publik.
4. **Tidak ada file `.env.example`.** Ketiga aplikasi punya variabel environment yang wajib diisi, tapi belum ada template publik yang aman untuk dibagikan ke kontributor lain.
5. **Tidak ada automated test.** Tidak ditemukan folder/file test (unit, integration, maupun e2e) di ketiga aplikasi. Validasi saat ini murni manual.

---

## Konvensi & Catatan Desain

Pola yang konsisten terlihat di kode saat ini (bukan style guide formal, tapi baik diikuti untuk konsistensi):

- **Mobile-first**: komponen didesain dari layar kecil ke besar.
- **Ikon SVG**, bukan emoji, di seluruh UI.
- Data statis dipisah ke `src/data/`, tidak di-hardcode langsung di komponen.
- Penggunaan `!important` diminimalkan di stylesheet.
- Komentar kode ditulis dalam **Bahasa Indonesia**.
- Arah visual: tema gelap, grid/garis tegas — bukan template UI generik default.

---

## Roadmap

Berdasarkan apa yang sudah disiapkan tapi belum diaktifkan di kode (lihat kolom `google_id`/`facebook_id` di tabel `users`, dan halaman placeholder di dashboard):

- [ ] Pasang rate limiter yang sudah ditulis ke route auth & admin-auth
- [ ] Hubungkan Dashboard Overview ke data revenue/order asli
- [ ] Bangun endpoint + UI Manajemen Kategori
- [ ] Bangun endpoint + UI Manajemen Pelanggan
- [ ] Bangun endpoint + UI ROI/Analitik
- [ ] Bangun endpoint CRUD Admin (multi-admin, bukan cuma login)
- [ ] Social login (Google/Facebook) — kolom DB sudah disiapkan
- [ ] Tambah `.env.example` di tiap aplikasi
- [ ] Tambah automated test (minimal untuk flow checkout & auth)

---

## Lisensi

Copyright © 2026 ByFakhriel. All rights reserved.
Proprietary software — unauthorized use, distribution, or reproduction is prohibited.
