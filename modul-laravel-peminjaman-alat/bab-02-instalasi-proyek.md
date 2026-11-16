---
layout: page
title: "BAB 02 — Instalasi Proyek Laravel | Modul Laravel 12 CRUD Peminjaman Alat"
description: "Cara install Laravel 12, membuat database MySQL db_peminjaman_alat, mengatur file .env dan menjalankan php artisan serve."
permalink: /modul-laravel-peminjaman-alat/bab-02-instalasi-proyek/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
Tujuan bab ini: proyek Laravel baru sudah jalan di browser dan terhubung ke MySQL.

---

## 1. Nyalakan MySQL

Pilih sesuai yang ada di komputermu:

| Tools | Cara |
|---|---|
| DevStack (Mac) | `devstack start` → phpMyAdmin di http://localhost:8080 |
| Laragon (Windows) | klik **Start All** |
| XAMPP | Start **Apache** dan **MySQL** di XAMPP Control Panel |

Cek versi PHP & Composer (minimal PHP 8.2):

```bash
php -v
composer -V
```

---

## 2. Buat proyek Laravel

Buka terminal di folder kerja (misalnya `modulUKK`), lalu:

```bash
composer create-project laravel/laravel peminjaman-alat
```

Tunggu sampai selesai (butuh internet). Lalu masuk ke foldernya dan buka di VS Code:

```bash
cd peminjaman-alat
code .
```

> Modul ini dibuat dengan **Laravel 12**. Kalau versimu beda sedikit, kodenya tetap sama.

---

## 3. Atur file `.env`

Buka file `.env` (di folder paling luar proyek). Ubah bagian ini:

```env
APP_NAME="Peminjaman Alat"
```

Lalu cari bagian `DB_`, ubah jadi seperti ini (hapus tanda `#` di depannya):

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=db_peminjaman_alat
DB_USERNAME=root
DB_PASSWORD=
```

> `DB_PASSWORD` dikosongkan karena default Laragon/XAMPP/DevStack tidak pakai password. Kalau MySQL-mu pakai password, isi di situ.

---

## 4. Atur zona waktu

Denda dihitung dari tanggal, jadi zona waktu harus benar. Buka `config/app.php`, cari baris `'timezone'` lalu ubah:

```php
'timezone' => 'Asia/Jakarta',
```

---

## 5. Buat database

**Cara 1 — phpMyAdmin:** buka phpMyAdmin → tab **Databases** → isi nama `db_peminjaman_alat` → **Create**.

**Cara 2 — terminal:**

```bash
mysql -u root -e "CREATE DATABASE db_peminjaman_alat"
```

---

## 6. Hapus file yang tidak dipakai

Supaya proyek bersih, hapus 3 file ini:

| File | Alasan |
|---|---|
| `database/database.sqlite` | kita pakai MySQL, bukan SQLite |
| `resources/views/welcome.blade.php` | halaman bawaan Laravel, tidak dipakai |
| `database/factories/UserFactory.php` | berisi kolom `email`, sedangkan kita pakai `username` |

---

## 7. Jalankan server

```bash
php artisan serve
```

Buka http://localhost:8000. Untuk sekarang halamannya masih **error** (karena `welcome` sudah dihapus dan tabel belum dibuat) — itu normal, nanti beres di bab berikutnya.

> Biarkan terminal `php artisan serve` tetap menyala. Buka **terminal kedua** untuk mengetik perintah `php artisan ...` lainnya.

---

## ✅ Checklist Bab 02
- [ ] Proyek `peminjaman-alat` sudah dibuat
- [ ] `.env` sudah pakai MySQL `db_peminjaman_alat`
- [ ] Timezone `Asia/Jakarta`
- [ ] Database sudah dibuat di MySQL

Lanjut ke **[BAB 03 — Migration Tabel](/modul-laravel-peminjaman-alat/bab-03-migration-tabel/)**.
{% endraw %}

---

[← BAB 01 — Analisis Kebutuhan & Desain](/modul-laravel-peminjaman-alat/bab-01-analisis-dan-desain/) · [Daftar isi](/modul-laravel-peminjaman-alat/) · [BAB 03 — Membuat Tabel dengan Migration →](/modul-laravel-peminjaman-alat/bab-03-migration-tabel/)
