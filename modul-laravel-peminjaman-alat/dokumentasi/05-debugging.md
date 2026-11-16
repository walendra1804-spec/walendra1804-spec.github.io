---
layout: page
title: "Dokumentasi UKK Catatan Debugging | Modul Laravel 12 CRUD Peminjaman Alat"
description: "Contoh catatan debugging aplikasi Laravel peminjaman alat: gejala, cara menemukan, penyebab dan perbaikan."
permalink: /modul-laravel-peminjaman-alat/dokumentasi/05-debugging/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
Berikut error yang ditemui (no. 1–4) dan yang diantisipasi (no. 5–6) selama pembuatan aplikasi, beserta cara menemukan dan memperbaikinya.

| No | Gejala | Cara menemukan | Penyebab | Perbaikan |
|---|---|---|---|---|
| 1 | `php artisan migrate:fresh` gagal: *SQLSTATE[42000] 1304 FUNCTION hitung_denda already exists* | Pesan error di terminal + `storage/logs/laravel.log` | `migrate:fresh` hanya menghapus **tabel**; function & procedure tetap ada di database | Menambahkan `DROP FUNCTION IF EXISTS hitung_denda` dan `DROP PROCEDURE IF EXISTS kembalikan_alat` di awal method `up()` migration |
| 2 | Semua halaman admin/petugas/peminjam error 500 | Membaca `storage/logs/laravel.log`: *Target class [App\Http\Middleware\CekRole] does not exist* | File middleware belum tersimpan (perintah sebelumnya gagal) | Membuat ulang dengan `php artisan make:middleware CekRole` lalu mengisi kodenya |
| 3 | Setelah `migrate:fresh`, tombol Login menghasilkan **419 Page Expired** | Halaman error 419 di browser | Tabel `sessions` ikut di-reset, token CSRF di halaman lama sudah tidak berlaku | Refresh halaman login (bukan bug kode) |
| 4 | Setelah login gagal, username yang sudah diketik hilang | Uji manual TC01 | `back()` tidak membawa input lama | Menambah `->withInput()` pada `back()` di `AuthController@login` |
| 5 | Model `Kategori` mencari tabel `kategoris` (*Table doesn't exist*) | Pesan error SQL | Laravel otomatis memakai nama tabel bahasa Inggris jamak | Menambah `protected $table = 'kategori';` di setiap model |
| 6 | Tombol **Setujui** diklik dua kali → stok berkurang dua kali (potensi) | Analisis kode saat pengujian TC07 | Tidak ada cek status sebelum update | Menambah cek `status != 'menunggu'` dan kondisi `OLD.status <> 'disetujui'` di trigger |

## Teknik debugging yang dipakai
1. **Halaman error Laravel** (`APP_DEBUG=true`) → menunjukkan file & baris error.
2. **`storage/logs/laravel.log`** → riwayat semua error.
3. **`dd($variabel)`** → melihat isi variabel/request di tengah proses.
4. **`php artisan route:list`** → memastikan route & nama route benar.
5. **Uji SQL langsung di phpMyAdmin** → memastikan function/procedure/trigger benar sebelum dipakai di Laravel:
   ```sql
   SELECT hitung_denda('2026-11-10', '2026-11-13');   -- 15000
   CALL kembalikan_alat(999, '2026-11-13');           -- error FK → ROLLBACK, tabel pengembalian tetap kosong
   ```
{% endraw %}

---

[← 4. Deskripsi Program dan Dokumentasi Fungsi / Prosedur](/modul-laravel-peminjaman-alat/dokumentasi/04-dokumentasi-fungsi/) · [Daftar isi](/modul-laravel-peminjaman-alat/) · [6. Pengujian (Test Case) →](/modul-laravel-peminjaman-alat/dokumentasi/06-pengujian/)
