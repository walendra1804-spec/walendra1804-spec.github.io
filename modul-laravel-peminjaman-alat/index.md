---
layout: page
title: "Daftar Isi: Modul Laravel 12 CRUD Peminjaman Alat"
description: "Daftar isi modul lengkap membuat aplikasi peminjaman alat dengan Laravel 12, MySQL dan Bootstrap 5 untuk UKK RPL 2025/2026: 14 bab langkah demi langkah dan 7 contoh dokumen UKK."
permalink: /modul-laravel-peminjaman-alat/
lang: id
---

Modul ini adalah bagian dari artikel **[Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/)**. Ikuti bab satu per satu dari BAB 00. Kode di setiap bab sama persis dengan proyek yang sudah diuji.

**[Download semua (modul + kode + .sql + dokumentasi), zip](/downloads/modulUKK-peminjaman-alat-laravel12.zip)**

## Modul (14 bab)

1. [BAB 00 — Pendahuluan](/modul-laravel-peminjaman-alat/bab-00-pendahuluan/): Pengantar modul membuat aplikasi peminjaman alat dengan Laravel 12, MySQL dan Bootstrap 5 untuk UKK RPL 2025/2026 Paket 1: fitur tiap role, alur aplikasi, daftar bab dan akun uji.
2. [BAB 01 — Analisis Kebutuhan & Desain](/modul-laravel-peminjaman-alat/bab-01-analisis-dan-desain/): Analisis kebutuhan, desain ERD, flowchart dan pseudocode login, peminjaman serta pengembalian dengan denda untuk aplikasi peminjaman alat Laravel 12.
3. [BAB 02 — Instalasi Proyek Laravel](/modul-laravel-peminjaman-alat/bab-02-instalasi-proyek/): Cara install Laravel 12, membuat database MySQL db_peminjaman_alat, mengatur file .env dan menjalankan php artisan serve.
4. [BAB 03 — Membuat Tabel dengan Migration](/modul-laravel-peminjaman-alat/bab-03-migration-tabel/): Membuat tabel users, kategori, alat, peminjaman, pengembalian dan log_aktivitas dengan migration Laravel 12 lengkap dengan foreign key.
5. [BAB 04 — Function, Stored Procedure, Trigger, Commit & Rollback](/modul-laravel-peminjaman-alat/bab-04-function-procedure-trigger/): Membuat function hitung_denda, stored procedure kembalikan_alat dengan COMMIT dan ROLLBACK, serta trigger kurangi_stok dan tambah_stok di MySQL lewat migration Laravel.
6. [BAB 05 — Model dan Seeder](/modul-laravel-peminjaman-alat/bab-05-model-dan-seeder/): Membuat model Eloquent dengan relasi belongsTo dan hasOne, serta seeder akun admin, petugas, peminjam dan data alat awal.
7. [BAB 06 — Login, Logout, Hak Akses (Role), dan Layout](/modul-laravel-peminjaman-alat/bab-06-login-role-layout/): Membuat login dan logout pakai username, middleware hak akses per role (admin, petugas, peminjam) dan layout Bootstrap 5 di Laravel 12.
8. [BAB 07 — Admin: CRUD Kategori & Alat](/modul-laravel-peminjaman-alat/bab-07-admin-kategori-alat/): CRUD kategori dan alat di Laravel 12: Route::resource, controller, validasi, pagination dan view Bootstrap.
9. [BAB 08 — Admin: CRUD User](/modul-laravel-peminjaman-alat/bab-08-admin-user/): CRUD user dengan role di Laravel 12: username unik, password di-hash, password boleh dikosongkan saat edit, dan larangan menghapus akun sendiri.
10. [BAB 09 — Admin: CRUD Peminjaman, Pengembalian, dan Log Aktivitas](/modul-laravel-peminjaman-alat/bab-09-admin-peminjaman-pengembalian-log/): CRUD peminjaman dan pengembalian yang memanggil stored procedure dan function MySQL dari Laravel, plus halaman log aktivitas.
11. [BAB 10 — Fitur Petugas](/modul-laravel-peminjaman-alat/bab-10-petugas/): Fitur petugas: menyetujui peminjaman dengan DB::beginTransaction, commit dan rollBack, memantau pengembalian yang terlambat, dan mencetak laporan.
12. [BAB 11 — Fitur Peminjam](/modul-laravel-peminjaman-alat/bab-11-peminjam/): Fitur peminjam: mencari alat, mengajukan peminjaman dengan validasi stok, dan mengembalikan alat dengan perhitungan denda otomatis.
13. [BAB 12 — Pengujian & Debugging](/modul-laravel-peminjaman-alat/bab-12-pengujian-debugging/): 15 test case aplikasi peminjaman alat dan cara debugging Laravel: halaman error, laravel.log, dd(), route:list, dan tabel error yang sering muncul.
14. [BAB 13 — Export Database, Dokumentasi, dan Pengumpulan](/modul-laravel-peminjaman-alat/bab-13-export-sql-dan-pengumpulan/): Cara export database MySQL beserta function, procedure dan trigger ke file .sql, menyusun dokumentasi UKK, checklist pengumpulan dan tips waktu 23 jam.

## Contoh dokumen UKK

- [1. Analisis Kebutuhan — Aplikasi Peminjaman Alat](/modul-laravel-peminjaman-alat/dokumentasi/01-analisis-kebutuhan/)
- [2. Desain Database (ERD)](/modul-laravel-peminjaman-alat/dokumentasi/02-erd/)
- [3. Flowchart dan Pseudocode](/modul-laravel-peminjaman-alat/dokumentasi/03-flowchart-pseudocode/)
- [4. Deskripsi Program dan Dokumentasi Fungsi / Prosedur](/modul-laravel-peminjaman-alat/dokumentasi/04-dokumentasi-fungsi/)
- [5. Catatan Debugging](/modul-laravel-peminjaman-alat/dokumentasi/05-debugging/)
- [6. Pengujian (Test Case)](/modul-laravel-peminjaman-alat/dokumentasi/06-pengujian/)
- [7. Laporan Evaluasi Singkat](/modul-laravel-peminjaman-alat/dokumentasi/07-laporan-evaluasi/)
