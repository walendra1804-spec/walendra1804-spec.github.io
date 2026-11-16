---
layout: page
title: "BAB 13 — Export Database, Dokumentasi, dan Pengumpulan | Modul Laravel 12 CRUD Peminjaman Alat"
description: "Cara export database MySQL beserta function, procedure dan trigger ke file .sql, menyusun dokumentasi UKK, checklist pengumpulan dan tips waktu 23 jam."
permalink: /modul-laravel-peminjaman-alat/bab-13-export-sql-dan-pengumpulan/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
---

## 1. Export database ke `.sql`

Yang penting: **function, procedure, dan trigger harus ikut ter-export**.

### Cara 1 — phpMyAdmin
1. Klik database `db_peminjaman_alat` → tab **Export**.
2. Pilih **Custom**.
3. Di bagian *Output*, pastikan format **SQL**.
4. Di bagian *Object creation options*, centang:
   - ✅ **Add CREATE PROCEDURE / FUNCTION / EVENT statement**
   - ✅ **Add CREATE TRIGGER statement** (biasanya sudah tercentang)
5. Klik **Export** → simpan sebagai `db_peminjaman_alat.sql`.

### Cara 2 — terminal
```bash
mysqldump -u root --routines --triggers --databases db_peminjaman_alat > database/db_peminjaman_alat.sql
```
- `--routines` → ikutkan function & procedure
- `--triggers` → ikutkan trigger
- `--databases` → ikutkan perintah `CREATE DATABASE`, jadi tinggal import

Cek hasilnya: buka file `.sql`, cari kata `CREATE FUNCTION`, `CREATE PROCEDURE`, `TRIGGER`. Harus ada semua.

---

## 2. Menjalankan proyek di komputer lain (misal komputer penguji)

```bash
composer install
cp .env.example .env
php artisan key:generate
```
Lalu atur `DB_*` di `.env` (lihat BAB 02), kemudian pilih salah satu:
- **Import** file `db_peminjaman_alat.sql` lewat phpMyAdmin, **atau**
- `php artisan migrate --seed`

Terakhir `php artisan serve`.

> Folder `vendor/` tidak perlu ikut dikumpulkan kalau ukurannya jadi besar — bisa dibuat ulang dengan `composer install`. Tanyakan ke penguji apakah boleh. Kalau komputer penguji **tidak ada internet**, sertakan saja folder `vendor/`.

---

## 3. Dokumentasi

Contoh lengkap semua dokumen ada di folder `dokumentasi/`. Pindahkan ke Word kalau diminta dalam bentuk dokumen:

| Dokumen (sesuai soal) | File contoh |
|---|---|
| Analisis kebutuhan | `dokumentasi/01-analisis-kebutuhan.md` |
| ERD + kamus data | `dokumentasi/02-erd.md` |
| Flowchart / pseudocode | `dokumentasi/03-flowchart-pseudocode.md` |
| Deskripsi program + dokumentasi fungsi/prosedur (input–proses–output) | `dokumentasi/04-dokumentasi-fungsi.md` |
| Debugging | `dokumentasi/05-debugging.md` |
| Pengujian + screenshot | `dokumentasi/06-pengujian.md` |
| Laporan evaluasi singkat | `dokumentasi/07-laporan-evaluasi.md` |

> **Jangan cuma copy.** Penguji bisa bertanya. Pastikan kamu bisa menjelaskan setiap baris di ERD dan dokumentasi fungsi.

---

## 4. Laporan evaluasi singkat (soal poin 11)

Isinya 3 bagian:
- **a. Fitur yang sudah berjalan baik** → ambil dari test case yang statusnya Berhasil.
- **b. Bug yang belum diperbaiki** → tulis jujur, misal keterbatasan yang kamu tahu.
- **c. Rencana pengembangan** → fitur yang ingin ditambah.

Contohnya ada di `dokumentasi/07-laporan-evaluasi.md`.

---

## 5. Checklist pengumpulan

```
📁 NamaKamu_UKK_PeminjamanAlat/
├── 📁 peminjaman-alat/            ← 1. folder proyek (kode lengkap)
│   └── database/db_peminjaman_alat.sql
├── 📄 db_peminjaman_alat.sql       ← 2. database (.sql) — salin juga ke luar biar mudah dicari
└── 📁 dokumentasi/                 ← 3 & 4. dokumentasi + laporan evaluasi
    ├── ERD (gambar)
    ├── deskripsi program & dokumentasi fungsi
    ├── debugging
    ├── pengujian + 📁 screenshot/
    └── laporan evaluasi
```

- [ ] Aplikasi jalan, 3 role bisa login
- [ ] File `.sql` berisi tabel, data, function, procedure, trigger
- [ ] ERD sesuai dengan tabel di database
- [ ] Flowchart login, peminjaman, pengembalian & denda
- [ ] Dokumentasi fungsi (input – proses – output)
- [ ] Catatan debugging
- [ ] Test case + screenshot
- [ ] Laporan evaluasi

---

## 6. Tips saat UKK (23 jam)

1. **Jam 1–3:** analisis, ERD, flowchart (BAB 01). Jangan langsung ngoding.
2. **Jam 3–5:** install, migration, function/procedure/trigger, model, seeder (BAB 02–05).
3. **Jam 5–14:** login, CRUD admin, petugas, peminjam (BAB 06–11). Setelah setiap bab, langsung coba di browser.
4. **Jam 14–18:** pengujian + screenshot (BAB 12).
5. **Jam 18–21:** dokumentasi & laporan (BAB 13).
6. **Sisa waktu:** cadangan untuk error. Export `.sql` **paling akhir** supaya datanya terbaru.

Simpan pekerjaan sesering mungkin. **Selamat & sukses!** 🎓
{% endraw %}

---

[← BAB 12 — Pengujian & Debugging](/modul-laravel-peminjaman-alat/bab-12-pengujian-debugging/) · [Daftar isi](/modul-laravel-peminjaman-alat/)
