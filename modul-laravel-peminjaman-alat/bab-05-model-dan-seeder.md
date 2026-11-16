---
layout: page
title: "BAB 05 — Model dan Seeder | Modul Laravel 12 CRUD Peminjaman Alat"
description: "Membuat model Eloquent dengan relasi belongsTo dan hasOne, serta seeder akun admin, petugas, peminjam dan data alat awal."
permalink: /modul-laravel-peminjaman-alat/bab-05-model-dan-seeder/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
**Model** = class PHP yang mewakili satu tabel. Lewat model kita bisa `create`, `update`, `delete`, `where`, dll tanpa menulis SQL.

**Seeder** = kode untuk mengisi data awal (akun admin, petugas, contoh alat).

---

## 1. Buat file model

```bash
php artisan make:model Kategori
php artisan make:model Alat
php artisan make:model Peminjaman
php artisan make:model Pengembalian
php artisan make:model LogAktivitas
```

Model `User` sudah ada, tinggal diubah.

Isi setiap model punya 3 hal:
- `$table` → nama tabel (wajib, karena nama tabel kita bahasa Indonesia).
- `$fillable` → kolom yang boleh diisi lewat `create()` / `update()`.
- **Fungsi relasi** → `belongsTo` (milik ...) atau `hasOne` (punya satu ...).

---

## 2. `app/Models/User.php` — ganti seluruh isinya

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable
{
    protected $fillable = ['name', 'username', 'password', 'role'];

    protected $hidden = ['password', 'remember_token'];

    protected function casts(): array
    {
        return [
            'password' => 'hashed',
        ];
    }
}
```

`'password' => 'hashed'` → setiap kali password disimpan, otomatis dienkripsi. Jadi kita tidak perlu menulis `Hash::make()` sendiri.

---

## 3. `app/Models/Kategori.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Kategori extends Model
{
    protected $table = 'kategori';

    protected $fillable = ['nama_kategori'];
}
```

---

## 4. `app/Models/Alat.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Alat extends Model
{
    protected $table = 'alat';

    protected $fillable = ['kategori_id', 'nama_alat', 'stok', 'keterangan'];

    // satu alat milik satu kategori
    public function kategori()
    {
        return $this->belongsTo(Kategori::class);
    }
}
```

Dengan relasi ini, kita bisa menulis `$alat->kategori->nama_kategori` untuk mengambil nama kategori.

---

## 5. `app/Models/Peminjaman.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Peminjaman extends Model
{
    protected $table = 'peminjaman';

    protected $fillable = ['user_id', 'alat_id', 'jumlah', 'tanggal_pinjam', 'batas_kembali', 'status', 'petugas_id'];

    // peminjam (user yang meminjam)
    public function user()
    {
        return $this->belongsTo(User::class);
    }

    public function alat()
    {
        return $this->belongsTo(Alat::class);
    }

    // petugas yang menyetujui / menolak
    public function petugas()
    {
        return $this->belongsTo(User::class, 'petugas_id');
    }

    // data pengembalian (kalau sudah dikembalikan)
    public function pengembalian()
    {
        return $this->hasOne(Pengembalian::class);
    }
}
```

- `belongsTo(User::class, 'petugas_id')` → kolomnya bukan `user_id`, jadi nama kolom ditulis manual.
- `hasOne(Pengembalian::class)` → satu peminjaman punya satu pengembalian.

---

## 6. `app/Models/Pengembalian.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Pengembalian extends Model
{
    protected $table = 'pengembalian';

    protected $fillable = ['peminjaman_id', 'tanggal_dikembalikan', 'denda'];

    public function peminjaman()
    {
        return $this->belongsTo(Peminjaman::class);
    }
}
```

---

## 7. `app/Models/LogAktivitas.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class LogAktivitas extends Model
{
    protected $table = 'log_aktivitas';

    protected $fillable = ['user_id', 'aktivitas'];

    public function user()
    {
        return $this->belongsTo(User::class);
    }

    // fungsi bantu untuk mencatat aktivitas, contoh: LogAktivitas::catat('Login');
    public static function catat($aktivitas)
    {
        LogAktivitas::create([
            'user_id' => auth()->id(),
            'aktivitas' => $aktivitas,
        ]);
    }
}
```

Fungsi `catat()` adalah fungsi bantu buatan kita. Di controller cukup tulis satu baris:
```php
LogAktivitas::catat('Menambah alat Proyektor');
```
`auth()->id()` = id user yang sedang login.

---

## 8. Seeder — data awal

Buka `database/seeders/DatabaseSeeder.php`, **ganti seluruh isinya**:

```php
<?php

namespace Database\Seeders;

use App\Models\Alat;
use App\Models\Kategori;
use App\Models\User;
use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        // 3 akun, satu untuk setiap level
        User::create(['name' => 'Administrator', 'username' => 'admin', 'password' => 'admin123', 'role' => 'admin']);
        User::create(['name' => 'Petugas Lab', 'username' => 'petugas', 'password' => 'petugas123', 'role' => 'petugas']);
        User::create(['name' => 'Budi Siswa', 'username' => 'peminjam', 'password' => 'peminjam123', 'role' => 'peminjam']);

        // kategori
        $elektronik = Kategori::create(['nama_kategori' => 'Elektronik']);
        $olahraga = Kategori::create(['nama_kategori' => 'Olahraga']);
        $perkakas = Kategori::create(['nama_kategori' => 'Perkakas']);

        // alat
        Alat::create(['kategori_id' => $elektronik->id, 'nama_alat' => 'Proyektor', 'stok' => 3, 'keterangan' => 'Epson']);
        Alat::create(['kategori_id' => $elektronik->id, 'nama_alat' => 'Laptop', 'stok' => 5, 'keterangan' => 'Untuk presentasi']);
        Alat::create(['kategori_id' => $olahraga->id, 'nama_alat' => 'Bola Basket', 'stok' => 10, 'keterangan' => null]);
        Alat::create(['kategori_id' => $perkakas->id, 'nama_alat' => 'Obeng Set', 'stok' => 4, 'keterangan' => 'Lengkap']);
    }
}
```

Jalankan migration ulang + seeder:

```bash
php artisan migrate:fresh --seed
```

Cek di phpMyAdmin: tabel `users` berisi 3 akun dan kolom `password` berisi teks acak (`$2y$12$...`) — itu tandanya password sudah ter-hash.

---

## ✅ Checklist Bab 05
- [ ] 6 model sudah dibuat
- [ ] `migrate:fresh --seed` berhasil
- [ ] Tabel `users`, `kategori`, `alat` sudah ada isinya

Lanjut ke **[BAB 06 — Login, Role, Layout](/modul-laravel-peminjaman-alat/bab-06-login-role-layout/)**.
{% endraw %}

---

[← BAB 04 — Function, Stored Procedure, Trigger, Commit & Rollback](/modul-laravel-peminjaman-alat/bab-04-function-procedure-trigger/) · [Daftar isi](/modul-laravel-peminjaman-alat/) · [BAB 06 — Login, Logout, Hak Akses (Role), dan Layout →](/modul-laravel-peminjaman-alat/bab-06-login-role-layout/)
