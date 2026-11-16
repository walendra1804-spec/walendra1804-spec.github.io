---
layout: page
title: "BAB 03 — Membuat Tabel dengan Migration | Modul Laravel 12 CRUD Peminjaman Alat"
description: "Membuat tabel users, kategori, alat, peminjaman, pengembalian dan log_aktivitas dengan migration Laravel 12 lengkap dengan foreign key."
permalink: /modul-laravel-peminjaman-alat/bab-03-migration-tabel/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
**Migration** = file PHP yang berisi perintah membuat tabel. Kelebihannya: tabel bisa dibuat ulang kapan saja dengan satu perintah.

Urutan membuat tabel **penting**: tabel yang dirujuk (parent) harus dibuat duluan.
`users` → `kategori` → `alat` → `peminjaman` → `pengembalian` → `log_aktivitas`

---

## 1. Ubah tabel `users`

Laravel sudah punya migration users, tapi pakai `email`. Kita ganti jadi `username` + `role`.

Buka `database/migrations/0001_01_01_000000_create_users_table.php`, **ganti seluruh isinya**:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('username')->unique();
            $table->string('password');
            $table->enum('role', ['admin', 'petugas', 'peminjam'])->default('peminjam');
            $table->rememberToken();
            $table->timestamps();
        });

        Schema::create('sessions', function (Blueprint $table) {
            $table->string('id')->primary();
            $table->foreignId('user_id')->nullable()->index();
            $table->string('ip_address', 45)->nullable();
            $table->text('user_agent')->nullable();
            $table->longText('payload');
            $table->integer('last_activity')->index();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('users');
        Schema::dropIfExists('sessions');
    }
};
```

Penjelasan:
- `->unique()` → username tidak boleh kembar.
- `enum('role', [...])` → isi kolom hanya boleh salah satu dari 3 pilihan.
- Tabel `sessions` dibiarkan karena Laravel menyimpan sesi login di sana.

---

## 2. Buat file migration baru

Jalankan satu per satu (urutan ini menentukan urutan pembuatan tabel):

```bash
php artisan make:migration create_kategori_table
php artisan make:migration create_alat_table
php artisan make:migration create_peminjaman_table
php artisan make:migration create_pengembalian_table
php artisan make:migration create_log_aktivitas_table
```

File baru muncul di `database/migrations/` dengan awalan tanggal, misalnya `2026_11_16_183509_create_kategori_table.php`.

---

## 3. Isi migration `kategori`

File: `database/migrations/xxxx_create_kategori_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('kategori', function (Blueprint $table) {
            $table->id();
            $table->string('nama_kategori');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('kategori');
    }
};
```

> **Penting:** Laravel otomatis menamai tabel dalam bahasa Inggris jamak (misal `kategoris`). Karena itu kita tulis nama tabel secara jelas: `Schema::create('kategori', ...)`. Nanti di Model juga kita tulis `$table = 'kategori'`.

---

## 4. Isi migration `alat`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('alat', function (Blueprint $table) {
            $table->id();
            $table->foreignId('kategori_id')->constrained('kategori')->cascadeOnDelete();
            $table->string('nama_alat');
            $table->integer('stok')->default(0);
            $table->string('keterangan')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('alat');
    }
};
```

- `foreignId('kategori_id')->constrained('kategori')` → membuat kolom `kategori_id` + relasi (foreign key) ke tabel `kategori`.
- `cascadeOnDelete()` → kalau kategori dihapus, alat di dalamnya ikut terhapus.

---

## 5. Isi migration `peminjaman`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('peminjaman', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained('users')->cascadeOnDelete();
            $table->foreignId('alat_id')->constrained('alat')->cascadeOnDelete();
            $table->integer('jumlah');
            $table->date('tanggal_pinjam');
            $table->date('batas_kembali');
            $table->enum('status', ['menunggu', 'disetujui', 'ditolak', 'dikembalikan'])->default('menunggu');
            $table->foreignId('petugas_id')->nullable()->constrained('users')->nullOnDelete();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('peminjaman');
    }
};
```

- `petugas_id` boleh kosong (`nullable`) karena saat diajukan belum ada petugas yang memproses.
- `nullOnDelete()` → kalau akun petugas dihapus, kolom ini jadi `NULL` (datanya tidak ikut hilang).

---

## 6. Isi migration `pengembalian`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('pengembalian', function (Blueprint $table) {
            $table->id();
            $table->foreignId('peminjaman_id')->constrained('peminjaman')->cascadeOnDelete();
            $table->date('tanggal_dikembalikan');
            $table->integer('denda')->default(0);
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('pengembalian');
    }
};
```

---

## 7. Isi migration `log_aktivitas`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('log_aktivitas', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->nullable()->constrained('users')->nullOnDelete();
            $table->string('aktivitas');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('log_aktivitas');
    }
};
```

---

## 8. Jalankan migration

```bash
php artisan migrate
```

Buka phpMyAdmin → database `db_peminjaman_alat`. Harus ada tabel: `users`, `kategori`, `alat`, `peminjaman`, `pengembalian`, `log_aktivitas` (+ tabel bawaan Laravel: `sessions`, `cache`, `jobs`, `migrations`, dll).

Cek relasinya: klik tabel `alat` → **Structure** → **Relation view**. Harus ada foreign key ke `kategori`.

> Kalau salah ketik dan mau mengulang dari awal:
> ```bash
> php artisan migrate:fresh
> ```
> Perintah ini **menghapus semua tabel lalu membuat ulang**. Data ikut hilang.

---

## ✅ Checklist Bab 03
- [ ] 6 tabel utama sudah ada di phpMyAdmin
- [ ] Foreign key `alat → kategori`, `peminjaman → users/alat`, `pengembalian → peminjaman` sudah terlihat

Lanjut ke **[BAB 04 — Function, Procedure, Trigger](/modul-laravel-peminjaman-alat/bab-04-function-procedure-trigger/)**.
{% endraw %}

---

[← BAB 02 — Instalasi Proyek Laravel](/modul-laravel-peminjaman-alat/bab-02-instalasi-proyek/) · [Daftar isi](/modul-laravel-peminjaman-alat/) · [BAB 04 — Function, Stored Procedure, Trigger, Commit & Rollback →](/modul-laravel-peminjaman-alat/bab-04-function-procedure-trigger/)
