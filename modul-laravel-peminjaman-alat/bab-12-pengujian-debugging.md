---
layout: page
title: "BAB 12 — Pengujian & Debugging | Modul Laravel 12 CRUD Peminjaman Alat"
description: "15 test case aplikasi peminjaman alat dan cara debugging Laravel: halaman error, laravel.log, dd(), route:list, dan tabel error yang sering muncul."
permalink: /modul-laravel-peminjaman-alat/bab-12-pengujian-debugging/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
Soal poin 9–10: tentukan kebutuhan uji, lalu lakukan pengujian. Minimal 5 skenario:
**login, tambah alat, pinjam alat, kembalikan alat (denda), cek privilege.**

---

## 1. Siapkan data bersih

```bash
php artisan migrate:fresh --seed
```

> Setelah perintah ini, sesi login lama ikut terhapus. Kalau browser menampilkan **419 Page Expired**, cukup refresh halaman login.

---

## 2. Tabel Test Case

Kerjakan satu per satu, **screenshot hasilnya** (Mac: `Cmd + Shift + 4`, Windows: `Win + Shift + S`), simpan ke folder `dokumentasi/screenshot/` dengan nama sesuai kode uji (`TC01.png`, dst).

| Kode | Skenario | Langkah | Hasil yang diharapkan |
|---|---|---|---|
| TC01 | Login salah | username `admin`, password `salah123` | Gagal, muncul "Username atau password salah!" |
| TC02 | Login benar | `admin` / `admin123` | Berhasil, masuk dashboard admin |
| TC03 | Tambah alat (valid) | Admin → Alat → Tambah: Kamera, Elektronik, stok 2 | Data tersimpan, muncul "Alat berhasil ditambah" |
| TC04 | Tambah alat (kosong) | Klik Simpan tanpa mengisi | Gagal, muncul pesan error validasi |
| TC05 | Ajukan pinjam | Peminjam → Daftar Alat → Proyektor → jumlah 2 | Tersimpan, status `menunggu` |
| TC06 | Pinjam melebihi stok | Jumlah 99 | Gagal, error validasi jumlah |
| TC07 | Setujui pinjaman | Petugas → Persetujuan → Setujui | Status `disetujui`, stok Proyektor 3 → 1 |
| TC08 | Setujui stok kurang | Admin buat pinjaman jumlah > stok → Petugas Setujui | Gagal "Stok alat tidak cukup", stok tidak berubah (rollback) |
| TC09 | Kembalikan tepat waktu | Peminjam → Peminjaman Saya → Kembalikan | Status `dikembalikan`, denda Rp 0, stok kembali 3 |
| TC10 | Kembalikan terlambat | Peminjam pinjam Bola Basket → petugas Setujui → admin edit peminjaman itu: Tanggal Pinjam = 7 hari lalu, Batas Kembali = 4 hari lalu → peminjam Kembalikan | Denda Rp 20.000 |
| TC11 | Privilege peminjam | Login peminjam, buka `/admin/user` | Ditolak, halaman 403 |
| TC12 | Privilege petugas | Login petugas, buka `/admin/alat` | Ditolak, halaman 403 |
| TC13 | Akses tanpa login | Logout, buka `/dashboard` | Diarahkan ke halaman login |
| TC14 | Cetak laporan | Petugas → Laporan → Cetak | Muncul dialog cetak, navbar tidak ikut tercetak |
| TC15 | Log aktivitas | Admin → Log Aktivitas | Semua aksi di atas tercatat |

Setelah diuji, isi kolom **Hasil Sebenarnya** dan **Status (Berhasil/Gagal)** di [`dokumentasi/06-pengujian.md`](/modul-laravel-peminjaman-alat/dokumentasi/06-pengujian/).

---

## 3. Cara Debugging

**Debugging** = mencari dan memperbaiki kesalahan. Ini juga diminta di dokumentasi (poin 3d), jadi **catat setiap error yang kamu temui** dan cara memperbaikinya.

### a. Baca halaman error Laravel
Karena `APP_DEBUG=true` di `.env`, setiap error menampilkan halaman detail: **pesan error**, **nama file**, dan **nomor baris**. Baca baris paling atas dulu.

### b. Lihat file log
Semua error juga disimpan di `storage/logs/laravel.log`. Buka file itu, lihat bagian paling bawah.

### c. Pakai `dd()` (dump and die)
Untuk melihat isi variabel. Contoh, di controller:
```php
public function store(Request $request)
{
    dd($request->all());   // tampilkan semua isian form lalu berhenti
    ...
}
```
**Jangan lupa hapus `dd()` setelah selesai.**

### d. Cek daftar route
```bash
php artisan route:list
```

### e. Error yang sering muncul

| Pesan error | Penyebab | Solusi |
|---|---|---|
| `419 Page Expired` | Lupa `@csrf` di form, atau sesi sudah kedaluwarsa | Tambah `@csrf`, refresh halaman |
| `403 Forbidden` | Role tidak punya akses | Memang sengaja (middleware). Kalau tidak sengaja, cek `role:` di route |
| `404 Not Found` | URL salah / data dengan id itu tidak ada | Cek `php artisan route:list` |
| `Target class [XxxController] does not exist` | Controller belum dibuat / salah nama / lupa `use` di `web.php` | Buat controller, cek ejaan |
| `Route [xxx] not defined` | Nama route di `route('...')` salah | Cek nama di `php artisan route:list` |
| `Add [xxx] to fillable property` | Kolom belum masuk `$fillable` di model | Tambahkan ke `$fillable` |
| `SQLSTATE[42S02]: Table 'xxx' doesn't exist` | Lupa `$table` di model, atau belum migrate | Tambah `protected $table = '...'` / jalankan migrate |
| `SQLSTATE[42S22]: Column not found` | Salah ketik nama kolom | Samakan dengan migration |
| `FUNCTION hitung_denda already exists` | `migrate:fresh` tidak menghapus function | Pastikan ada `DROP FUNCTION IF EXISTS` di awal migration |
| `Undefined variable $xxx` di view | Variabel tidak dikirim lewat `compact()` | Tambahkan di `compact(...)` |
| `SQLSTATE[HY000] [2002] Connection refused` | MySQL belum dinyalakan | Start MySQL (Laragon/XAMPP/DevStack) |

Contoh catatan debugging yang sudah terjadi saat membuat aplikasi ini ada di [`dokumentasi/05-debugging.md`](/modul-laravel-peminjaman-alat/dokumentasi/05-debugging/).

---

## ✅ Checklist Bab 12
- [ ] 15 test case sudah dijalankan
- [ ] Screenshot tersimpan di `dokumentasi/screenshot/`
- [ ] Error yang ditemui sudah dicatat

Lanjut ke **[BAB 13 — Export SQL & Pengumpulan](/modul-laravel-peminjaman-alat/bab-13-export-sql-dan-pengumpulan/)**.
{% endraw %}

---

[← BAB 11 — Fitur Peminjam](/modul-laravel-peminjaman-alat/bab-11-peminjam/) · [Daftar isi](/modul-laravel-peminjaman-alat/) · [BAB 13 — Export Database, Dokumentasi, dan Pengumpulan →](/modul-laravel-peminjaman-alat/bab-13-export-sql-dan-pengumpulan/)
