---
layout: page
title: "Dokumentasi UKK Deskripsi Program dan Dokumentasi Fungsi / Prosedur | Modul Laravel 12 CRUD Peminjaman Alat"
description: "Contoh deskripsi program dan dokumentasi fungsi (input, proses, output) untuk controller Laravel serta function, procedure dan trigger MySQL."
permalink: /modul-laravel-peminjaman-alat/dokumentasi/04-dokumentasi-fungsi/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
## 4.1 Deskripsi Program
**Aplikasi Peminjaman Alat** adalah aplikasi web berbasis Laravel 12 (PHP 8.2) dan MySQL 8 dengan tampilan Bootstrap 5. Aplikasi memakai pola **MVC**:

| Lapisan | Lokasi | Isi |
|---|---|---|
| Route | `routes/web.php` | daftar URL + middleware hak akses |
| Middleware | `app/Http/Middleware/CekRole.php` | membatasi halaman per role |
| Controller | `app/Http/Controllers/` | logika setiap fitur |
| Model | `app/Models/` | wakil tabel + relasi |
| View | `resources/views/` | tampilan Blade |
| Database | `database/migrations/` | struktur tabel, function, procedure, trigger |

Alur umum: **Browser → Route → Middleware (auth, role) → Controller → Model/DB → View → Browser**.

---

## 4.2 Modul Autentikasi — `AuthController`

| Method | Input | Proses | Output |
|---|---|---|---|
| `index()` | – | menampilkan form login | view `auth.login` |
| `login(Request)` | `username`, `password` | validasi wajib isi → `Auth::attempt()` → regenerate session → catat log | sukses: redirect `/dashboard`; gagal: kembali ke login + pesan error |
| `logout(Request)` | sesi user | catat log → `Auth::logout()` → hapus sesi & token | redirect `/login` |

## 4.3 Middleware — `CekRole`

| Method | Input | Proses | Output |
|---|---|---|---|
| `handle($request, $next, ...$roles)` | request, daftar role yang diizinkan | cek `auth()->user()->role` ada di `$roles` | lanjut ke controller, atau **403** |

## 4.4 Modul Dashboard — `DashboardController`

| Method | Input | Proses | Output |
|---|---|---|---|
| `index()` | – | hitung jumlah alat, user, peminjaman `menunggu`, `disetujui` | view `dashboard` berisi 4 kartu statistik |

## 4.5 Modul Master Data (Admin)

`KategoriController`, `AlatController`, `UserController` memakai pola CRUD yang sama:

| Method | Input | Proses | Output |
|---|---|---|---|
| `index()` | nomor halaman (`?page=`) | ambil data terbaru, 10 per halaman (`paginate`), alat memakai `with('kategori')` | view tabel + pagination |
| `create()` | – | (alat: ambil daftar kategori) | view form tambah |
| `store(Request)` | data form | validasi → `Model::create()` → catat log | redirect ke index + pesan sukses / kembali dengan error |
| `edit(Model)` | id dari URL | route model binding mencari data | view form edit terisi |
| `update(Request, Model)` | id + data form | validasi → `update()` → catat log | redirect ke index + pesan sukses |
| `destroy(Model)` | id dari URL | `delete()` → catat log | redirect ke index + pesan sukses |

Aturan validasi khusus:
- Alat: `stok` integer ≥ 0.
- User: `username` unik; saat edit `password` boleh kosong (tidak diubah); admin tidak boleh menghapus akunnya sendiri.

## 4.6 Modul Transaksi (Admin)

**`PeminjamanController`** — CRUD data peminjaman

| Method | Input | Proses | Output |
|---|---|---|---|
| `index()` | – | ambil peminjaman `with('user','alat')`, paginate 10 | view tabel |
| `create()` / `edit()` | – / id | ambil user role peminjam & daftar alat | view form |
| `store(Request)` | user_id, alat_id, jumlah, tanggal_pinjam, batas_kembali | validasi (batas ≥ tanggal pinjam) → simpan status `menunggu` → log | redirect + pesan |
| `update(Request, Peminjaman)` | + status | validasi → update (jika status jadi `disetujui`, trigger `kurangi_stok` jalan) → log | redirect + pesan |
| `destroy(Peminjaman)` | id | hapus → log | redirect + pesan |

**`PengembalianController`** — CRUD pengembalian

| Method | Input | Proses | Output |
|---|---|---|---|
| `index()` | – | ambil pengembalian `with('peminjaman.user','peminjaman.alat')` | view tabel + denda |
| `create()` | – | ambil peminjaman berstatus `disetujui` | view form |
| `store(Request)` | peminjaman_id, tanggal_dikembalikan | validasi → `CALL kembalikan_alat(?, ?)` → log | redirect + pesan |
| `edit(Pengembalian)` | id | – | view form tanggal |
| `update(Request, Pengembalian)` | tanggal_dikembalikan | `SELECT hitung_denda(?, ?)` → update tanggal & denda → log | redirect + pesan |
| `destroy(Pengembalian)` | id | hapus → log | redirect + pesan |

**`LogController`**

| Method | Input | Proses | Output |
|---|---|---|---|
| `index()` | – | ambil log `with('user')`, terbaru, 20 per halaman | view tabel log |

## 4.7 Modul Petugas — `PetugasController`

| Method | Input | Proses | Output |
|---|---|---|---|
| `peminjaman()` | – | ambil peminjaman status `menunggu` (terlama dulu) | view daftar + tombol Setujui/Tolak |
| `setujui(Peminjaman)` | id | **BEGIN TRANSACTION** → cek status `menunggu` → cek stok ≥ jumlah → update status `disetujui` + petugas_id (trigger kurangi stok) → log → **COMMIT**; jika gagal **ROLLBACK** | kembali + pesan sukses / "Gagal: ..." |
| `tolak(Peminjaman)` | id | update status `ditolak` + petugas_id → log | kembali + pesan |
| `pengembalian()` | – | ambil peminjaman `disetujui` urut batas kembali + riwayat pengembalian | view: tabel sedang dipinjam (label Terlambat) + riwayat |
| `laporan(Request)` | `dari`, `sampai` (default awal bulan s/d hari ini) | `whereBetween('tanggal_pinjam')` dengan `with('user','alat','pengembalian')` | view laporan + total denda + tombol cetak |

## 4.8 Modul Peminjam — `PeminjamController`

| Method | Input | Proses | Output |
|---|---|---|---|
| `alat(Request)` | `cari` (opsional) | cari `nama_alat LIKE %cari%`, paginate 10 | view daftar alat + tombol Pinjam |
| `formPinjam(Alat)` | id alat | – | view form pengajuan |
| `pinjam(Request, Alat)` | jumlah, batas_kembali | validasi (1 ≤ jumlah ≤ stok, batas ≥ hari ini) → simpan status `menunggu` → log | redirect riwayat + pesan |
| `riwayat()` | user login | ambil peminjaman milik user `with('alat','pengembalian')` | view riwayat + tombol Kembalikan |
| `kembalikan(Peminjaman)` | id | cek milik sendiri & status `disetujui` → `CALL kembalikan_alat(id, hari_ini)` → log → ambil denda | kembali + pesan "Denda: Rp ..." |

## 4.9 Model

| Model | Tabel | Relasi / fungsi |
|---|---|---|
| `User` | users | cast `password => hashed` |
| `Kategori` | kategori | – |
| `Alat` | alat | `kategori()` belongsTo |
| `Peminjaman` | peminjaman | `user()`, `alat()`, `petugas()` belongsTo; `pengembalian()` hasOne |
| `Pengembalian` | pengembalian | `peminjaman()` belongsTo |
| `LogAktivitas` | log_aktivitas | `user()` belongsTo; static `catat($aktivitas)` → **input:** teks aktivitas, **proses:** insert dengan user_id login, **output:** 1 baris log baru |

## 4.10 Objek Database

**FUNCTION `hitung_denda(p_batas DATE, p_kembali DATE) RETURNS INT`**
- Input: batas kembali, tanggal dikembalikan
- Proses: `telat = DATEDIFF(p_kembali, p_batas)`; jika `telat > 0` → `telat × 5000`
- Output: nominal denda (0 jika tidak terlambat)

**PROCEDURE `kembalikan_alat(IN p_peminjaman_id INT, IN p_tanggal DATE)`**
- Input: id peminjaman, tanggal pengembalian
- Proses: EXIT HANDLER (ROLLBACK + RESIGNAL) → START TRANSACTION → ambil batas_kembali → INSERT pengembalian dengan `hitung_denda()` → UPDATE status `dikembalikan` → COMMIT
- Output: 1 baris pengembalian baru, status peminjaman berubah; jika error semua dibatalkan

**TRIGGER `kurangi_stok` — AFTER UPDATE ON peminjaman**
- Input: baris OLD & NEW
- Proses: jika `OLD.status <> 'disetujui'` dan `NEW.status = 'disetujui'` → `alat.stok -= NEW.jumlah`
- Output: stok alat berkurang

**TRIGGER `tambah_stok` — AFTER INSERT ON pengembalian**
- Input: baris NEW
- Proses: `UPDATE alat JOIN peminjaman ... SET stok = stok + jumlah`
- Output: stok alat bertambah

## 4.11 Penerapan Best Practice (soal poin 8)
| Ketentuan | Penerapan |
|---|---|
| Query efisien | eager loading `with()` mencegah N+1 query |
| Hindari looping tidak perlu | perhitungan statistik pakai `count()` di SQL, bukan loop PHP; denda dihitung di database |
| Gunakan limit | semua daftar memakai `paginate(10)` / `paginate(20)` |
| Keamanan | password hash, CSRF, middleware role, query binding `?` |
| Penamaan | nama tabel/kolom bahasa Indonesia konsisten, controller per fitur |
{% endraw %}

---

[← 3. Flowchart dan Pseudocode](/modul-laravel-peminjaman-alat/dokumentasi/03-flowchart-pseudocode/) · [Daftar isi](/modul-laravel-peminjaman-alat/) · [5. Catatan Debugging →](/modul-laravel-peminjaman-alat/dokumentasi/05-debugging/)
