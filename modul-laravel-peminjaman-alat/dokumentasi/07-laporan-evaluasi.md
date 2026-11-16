---
layout: page
title: "Dokumentasi UKK Laporan Evaluasi Singkat | Modul Laravel 12 CRUD Peminjaman Alat"
description: "Contoh laporan evaluasi singkat: fitur yang berjalan, bug yang belum diperbaiki, dan rencana pengembangan aplikasi peminjaman alat."
permalink: /modul-laravel-peminjaman-alat/dokumentasi/07-laporan-evaluasi/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
**Judul:** Aplikasi Peminjaman Alat
**Teknologi:** Laravel 12, PHP 8.2, MySQL 8, Bootstrap 5

## a. Fitur yang sudah berjalan dengan baik
1. Login & logout untuk 3 level pengguna (admin, petugas, peminjam), password ter-hash.
2. Pembatasan hak akses per role dengan middleware (akses ilegal → 403).
3. Admin: CRUD user, kategori, alat, data peminjaman, data pengembalian, dan melihat log aktivitas.
4. Petugas: menyetujui/menolak peminjaman (dengan transaksi COMMIT/ROLLBACK), memantau pengembalian dengan tanda terlambat, mencetak laporan per periode beserta total denda.
5. Peminjam: melihat & mencari alat, mengajukan peminjaman, mengembalikan alat.
6. Database: relasi foreign key, function `hitung_denda`, stored procedure `kembalikan_alat` (dengan COMMIT/ROLLBACK), trigger `kurangi_stok` & `tambah_stok`.
7. Semua daftar data memakai pagination dan eager loading.
8. 15/15 test case aplikasi dan 5/5 uji database berhasil.

## b. Bug / keterbatasan yang belum diperbaiki
1. Jika admin **mengubah alat atau jumlah** pada peminjaman yang sudah `disetujui`, stok tidak disesuaikan ulang (trigger hanya berjalan saat status berubah menjadi `disetujui`).
2. Jika admin mengubah status dari `disetujui` kembali ke `menunggu`/`ditolak`, stok tidak dikembalikan.
3. Admin yang mengubah status menjadi `disetujui` lewat menu CRUD tidak dicek stoknya (pengecekan stok hanya di fitur persetujuan petugas), sehingga stok bisa minus.
4. Menghapus data pengembalian tidak mengembalikan status peminjaman dan stok.
5. Menghapus kategori/alat ikut menghapus riwayat peminjamannya (cascade).
6. Pesan validasi masih berbahasa Inggris (bawaan Laravel).

## c. Rencana pengembangan berikutnya
1. Satu peminjaman bisa berisi beberapa alat (tabel detail peminjaman).
2. Trigger tambahan untuk menyesuaikan stok saat data peminjaman/pengembalian diedit atau dihapus.
3. Soft delete untuk alat & kategori agar riwayat tidak hilang.
4. Export laporan ke PDF/Excel.
5. Notifikasi (email/WhatsApp) pengingat batas pengembalian.
6. Upload foto alat dan kondisi alat saat dikembalikan.
7. Pesan validasi berbahasa Indonesia.
{% endraw %}

---

[← 6. Pengujian (Test Case)](/modul-laravel-peminjaman-alat/dokumentasi/06-pengujian/) · [Daftar isi](/modul-laravel-peminjaman-alat/)
