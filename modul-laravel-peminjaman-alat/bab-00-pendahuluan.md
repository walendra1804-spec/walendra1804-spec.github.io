---
layout: page
title: "BAB 00 — Pendahuluan | Modul Laravel 12 CRUD Peminjaman Alat"
description: "Pengantar modul membuat aplikasi peminjaman alat dengan Laravel 12, MySQL dan Bootstrap 5 untuk UKK RPL 2025/2026 Paket 1: fitur tiap role, alur aplikasi, daftar bab dan akun uji."
permalink: /modul-laravel-peminjaman-alat/bab-00-pendahuluan/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
Modul ini memandu kamu membuat **Aplikasi Peminjaman Alat** (soal UKK RPL 2025/2026 Paket 1) dari nol sampai selesai, memakai **Laravel + MySQL + Bootstrap 5**.

Ketik ulang kodenya sendiri (jangan copy-paste terus), supaya waktu UKK tanganmu sudah hafal polanya.

---

## 1. Apa yang diminta soal?

Aplikasi dengan **3 level pengguna**:

| Fitur | Admin | Petugas | Peminjam |
|---|:-:|:-:|:-:|
| Login / Logout | ✔ | ✔ | ✔ |
| CRUD user, alat, kategori | ✔ | | |
| CRUD data peminjaman & pengembalian | ✔ | | |
| Log aktivitas | ✔ | | |
| Menyetujui peminjaman | | ✔ | |
| Memantau pengembalian | | ✔ | |
| Mencetak laporan | | ✔ | |
| Melihat daftar alat | | | ✔ |
| Mengajukan peminjaman | | | ✔ |
| Mengembalikan alat | | | ✔ |

Di database juga wajib ada: **relasi tabel, stored procedure, function, trigger, commit & rollback**.

Yang dikumpulkan:
1. Folder proyek (kode lengkap)
2. File database `.sql`
3. Dokumentasi: ERD, deskripsi program, dokumentasi fungsi, debugging, pengujian + screenshot
4. Laporan evaluasi singkat

---

## 2. Alur aplikasi (gambaran besar)

```
Peminjam                 Petugas                    Database
--------                 -------                    --------
lihat daftar alat
ajukan pinjam  ───────►  status = "menunggu"
                         klik Setujui ───────────►  TRIGGER kurangi_stok  (stok - jumlah)
                         (atau Tolak)               status = "disetujui"
klik Kembalikan ───────────────────────────────►   PROCEDURE kembalikan_alat
                                                     ├─ FUNCTION hitung_denda (Rp 5.000/hari telat)
                                                     ├─ simpan ke tabel pengembalian
                                                     ├─ TRIGGER tambah_stok (stok + jumlah)
                                                     └─ status = "dikembalikan"  → COMMIT
                         lihat Pantau Pengembalian
                         cetak Laporan
```

---

## 3. Struktur folder akhir

```
modulUKK/
├── peminjaman-alat/          ← proyek Laravel (hasil akhir)
├── modul/                    ← modul ini
└── dokumentasi/              ← ERD, flowchart, pengujian, laporan (contoh jawaban)
```

---

## 4. Daftar bab

| Bab | Isi |
|---|---|
| [01](/modul-laravel-peminjaman-alat/bab-01-analisis-dan-desain/) | Analisis kebutuhan, ERD, flowchart |
| [02](/modul-laravel-peminjaman-alat/bab-02-instalasi-proyek/) | Install Laravel, setting database |
| [03](/modul-laravel-peminjaman-alat/bab-03-migration-tabel/) | Membuat tabel dengan migration |
| [04](/modul-laravel-peminjaman-alat/bab-04-function-procedure-trigger/) | Function, stored procedure, trigger, commit/rollback |
| [05](/modul-laravel-peminjaman-alat/bab-05-model-dan-seeder/) | Model (relasi) dan data awal |
| [06](/modul-laravel-peminjaman-alat/bab-06-login-role-layout/) | Login, logout, hak akses per role, layout |
| [07](/modul-laravel-peminjaman-alat/bab-07-admin-kategori-alat/) | CRUD kategori & alat |
| [08](/modul-laravel-peminjaman-alat/bab-08-admin-user/) | CRUD user |
| [09](/modul-laravel-peminjaman-alat/bab-09-admin-peminjaman-pengembalian-log/) | CRUD peminjaman, pengembalian, log aktivitas |
| [10](/modul-laravel-peminjaman-alat/bab-10-petugas/) | Setujui peminjaman, pantau pengembalian, cetak laporan |
| [11](/modul-laravel-peminjaman-alat/bab-11-peminjam/) | Daftar alat, ajukan pinjam, kembalikan alat |
| [12](/modul-laravel-peminjaman-alat/bab-12-pengujian-debugging/) | Test case, cara debugging |
| [13](/modul-laravel-peminjaman-alat/bab-13-export-sql-dan-pengumpulan/) | Export `.sql`, laporan evaluasi, checklist kumpul |

---

## 5. Akun untuk mencoba

| Role | Username | Password |
|---|---|---|
| Admin | `admin` | `admin123` |
| Petugas | `petugas` | `petugas123` |
| Peminjam | `peminjam` | `peminjam123` |

---

## 6. Pola yang akan terus berulang

Hampir semua fitur di Laravel dibuat dengan 4 langkah yang sama. Hafalkan ini:

1. **Route** (`routes/web.php`) → alamat URL mana memanggil fungsi apa
2. **Controller** (`app/Http/Controllers`) → logika: ambil data, validasi, simpan
3. **Model** (`app/Models`) → wakil tabel database
4. **View** (`resources/views`) → tampilan HTML (Blade)

Kalau bingung "kodenya taruh di mana", balik lagi ke 4 langkah ini.

Lanjut ke **[BAB 01 — Analisis dan Desain](/modul-laravel-peminjaman-alat/bab-01-analisis-dan-desain/)**.
{% endraw %}

---

[Daftar isi](/modul-laravel-peminjaman-alat/) · [BAB 01 — Analisis Kebutuhan & Desain →](/modul-laravel-peminjaman-alat/bab-01-analisis-dan-desain/)
