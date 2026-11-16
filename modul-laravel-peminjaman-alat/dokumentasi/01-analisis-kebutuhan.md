---
layout: page
title: "Dokumentasi UKK Analisis Kebutuhan — Aplikasi Peminjaman Alat | Modul Laravel 12 CRUD Peminjaman Alat"
description: "Contoh dokumen analisis kebutuhan aplikasi peminjaman alat: kebutuhan fungsional per role dan kebutuhan non-fungsional."
permalink: /modul-laravel-peminjaman-alat/dokumentasi/01-analisis-kebutuhan/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
## 1.1 Latar Belakang
Peminjaman alat (elektronik, olahraga, perkakas) masih dicatat manual sehingga stok sering tidak sesuai, keterlambatan sulit dipantau, dan denda dihitung manual. Dibutuhkan aplikasi web yang mencatat pengajuan, persetujuan, pengembalian, dan denda secara otomatis.

## 1.2 Metode Pengembangan
Waterfall sederhana: Analisis → Desain → Implementasi → Pengujian → Dokumentasi.

## 1.3 Pengguna Sistem
| Level | Deskripsi |
|---|---|
| Admin | Pengelola sistem. Mengelola user, kategori, alat, data peminjaman & pengembalian, melihat log aktivitas. |
| Petugas | Petugas lab/gudang. Menyetujui atau menolak peminjaman, memantau pengembalian, mencetak laporan. |
| Peminjam | Siswa/karyawan. Melihat daftar alat, mengajukan peminjaman, mengembalikan alat. |

## 1.4 Kebutuhan Fungsional
| No | Kebutuhan | Admin | Petugas | Peminjam |
|---|---|:-:|:-:|:-:|
| F01 | Login dan logout | ✔ | ✔ | ✔ |
| F02 | CRUD user | ✔ | | |
| F03 | CRUD kategori | ✔ | | |
| F04 | CRUD alat | ✔ | | |
| F05 | CRUD data peminjaman | ✔ | | |
| F06 | CRUD pengembalian (denda dihitung otomatis) | ✔ | | |
| F07 | Melihat log aktivitas | ✔ | | |
| F08 | Menyetujui / menolak peminjaman | | ✔ | |
| F09 | Memantau pengembalian (tanda terlambat) | | ✔ | |
| F10 | Mencetak laporan per periode | | ✔ | |
| F11 | Melihat & mencari daftar alat | | | ✔ |
| F12 | Mengajukan peminjaman | | | ✔ |
| F13 | Mengembalikan alat | | | ✔ |

## 1.5 Aturan Bisnis
1. Status peminjaman: `menunggu` → `disetujui` / `ditolak` → `dikembalikan`.
2. Jumlah pinjam tidak boleh melebihi stok.
3. Stok berkurang saat peminjaman **disetujui** dan bertambah saat alat **dikembalikan** (trigger database).
4. Denda keterlambatan **Rp 5.000 per hari** (function database `hitung_denda`).
5. Setiap aktivitas penting dicatat di log.

## 1.6 Kebutuhan Non-Fungsional
| No | Kebutuhan |
|---|---|
| NF01 | Password disimpan dalam bentuk hash (bcrypt). |
| NF02 | Setiap halaman dilindungi sesuai role (middleware). Akses ilegal → 403. |
| NF03 | Form dilindungi token CSRF. |
| NF04 | Query efisien: eager loading `with()` dan pagination `paginate()` (LIMIT). |
| NF05 | Tampilan responsif menggunakan Bootstrap 5. |

## 1.7 Kebutuhan Perangkat
| Jenis | Spesifikasi |
|---|---|
| Perangkat keras | Laptop/PC prosesor 2 GHz, RAM 4 GB |
| Sistem operasi | Windows / macOS / Linux |
| Bahasa & framework | PHP 8.2, Laravel 12 |
| Basis data | MySQL 8 |
| Tools | Composer, VS Code, phpMyAdmin, browser |
{% endraw %}

---

[Daftar isi](/modul-laravel-peminjaman-alat/) · [2. Desain Database (ERD) →](/modul-laravel-peminjaman-alat/dokumentasi/02-erd/)
