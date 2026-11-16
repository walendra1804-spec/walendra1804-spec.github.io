---
layout: page
title: "BAB 04 — Function, Stored Procedure, Trigger, Commit & Rollback | Modul Laravel 12 CRUD Peminjaman Alat"
description: "Membuat function hitung_denda, stored procedure kembalikan_alat dengan COMMIT dan ROLLBACK, serta trigger kurangi_stok dan tambah_stok di MySQL lewat migration Laravel."
permalink: /modul-laravel-peminjaman-alat/bab-04-function-procedure-trigger/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
Soal poin 6 mewajibkan: **stored procedure, function, trigger, commit dan rollback**. Bab ini membuat semuanya.

---

## 1. Kenalan dulu

| Istilah | Artinya | Dipakai untuk |
|---|---|---|
| **Function** | Rumus di MySQL yang **mengembalikan satu nilai** | `hitung_denda()` → menghitung denda |
| **Stored Procedure** | Kumpulan perintah SQL yang disimpan, dipanggil dengan `CALL` | `kembalikan_alat()` → proses pengembalian |
| **Trigger** | Perintah yang **jalan otomatis** saat tabel di-INSERT/UPDATE/DELETE | `kurangi_stok`, `tambah_stok` |
| **Transaction** | Sekumpulan perintah yang harus berhasil **semua** atau **batal semua** | |
| **COMMIT** | Simpan permanen semua perubahan di transaksi | |
| **ROLLBACK** | Batalkan semua perubahan di transaksi (kalau ada error) | |

Contoh kenapa butuh transaction: saat pengembalian, kita (1) simpan data pengembalian dan (2) ubah status peminjaman. Kalau langkah 1 berhasil tapi langkah 2 error, data jadi kacau. Dengan transaction, kalau ada yang gagal → **ROLLBACK** semuanya.

---

## 2. Buat migration-nya

Kita taruh SQL-nya di migration, supaya ikut terbuat setiap `php artisan migrate`.

```bash
php artisan make:migration create_function_procedure_trigger
```

Isi file `database/migrations/xxxx_create_function_procedure_trigger.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\DB;

return new class extends Migration
{
    public function up(): void
    {
        // hapus dulu kalau sudah ada (supaya migrate:fresh tidak error)
        DB::unprepared('DROP FUNCTION IF EXISTS hitung_denda');
        DB::unprepared('DROP PROCEDURE IF EXISTS kembalikan_alat');

        // 1. FUNCTION: menghitung denda (Rp 5.000 per hari terlambat)
        DB::unprepared("
            CREATE FUNCTION hitung_denda(p_batas DATE, p_kembali DATE)
            RETURNS INT
            DETERMINISTIC
            BEGIN
                DECLARE telat INT;
                SET telat = DATEDIFF(p_kembali, p_batas);
                IF telat > 0 THEN
                    RETURN telat * 5000;
                END IF;
                RETURN 0;
            END
        ");

        // 2. STORED PROCEDURE: proses pengembalian alat (pakai COMMIT & ROLLBACK)
        DB::unprepared("
            CREATE PROCEDURE kembalikan_alat(IN p_peminjaman_id INT, IN p_tanggal DATE)
            BEGIN
                DECLARE v_batas DATE;

                DECLARE EXIT HANDLER FOR SQLEXCEPTION
                BEGIN
                    ROLLBACK;
                    RESIGNAL;
                END;

                START TRANSACTION;

                SELECT batas_kembali INTO v_batas
                FROM peminjaman WHERE id = p_peminjaman_id;

                INSERT INTO pengembalian (peminjaman_id, tanggal_dikembalikan, denda, created_at, updated_at)
                VALUES (p_peminjaman_id, p_tanggal, hitung_denda(v_batas, p_tanggal), NOW(), NOW());

                UPDATE peminjaman SET status = 'dikembalikan', updated_at = NOW()
                WHERE id = p_peminjaman_id;

                COMMIT;
            END
        ");

        // 3. TRIGGER: stok alat berkurang saat peminjaman disetujui
        DB::unprepared("
            CREATE TRIGGER kurangi_stok
            AFTER UPDATE ON peminjaman
            FOR EACH ROW
            BEGIN
                IF OLD.status <> 'disetujui' AND NEW.status = 'disetujui' THEN
                    UPDATE alat SET stok = stok - NEW.jumlah WHERE id = NEW.alat_id;
                END IF;
            END
        ");

        // 4. TRIGGER: stok alat bertambah saat alat dikembalikan
        DB::unprepared("
            CREATE TRIGGER tambah_stok
            AFTER INSERT ON pengembalian
            FOR EACH ROW
            BEGIN
                UPDATE alat
                JOIN peminjaman ON peminjaman.alat_id = alat.id
                SET alat.stok = alat.stok + peminjaman.jumlah
                WHERE peminjaman.id = NEW.peminjaman_id;
            END
        ");
    }

    public function down(): void
    {
        DB::unprepared('DROP TRIGGER IF EXISTS tambah_stok');
        DB::unprepared('DROP TRIGGER IF EXISTS kurangi_stok');
        DB::unprepared('DROP PROCEDURE IF EXISTS kembalikan_alat');
        DB::unprepared('DROP FUNCTION IF EXISTS hitung_denda');
    }
};
```

### Penjelasan

**`DB::unprepared(...)`** → menjalankan SQL mentah apa adanya. Dipakai karena SQL function/procedure punya banyak `;` di dalamnya.

**Function `hitung_denda`**
- `DATEDIFF(p_kembali, p_batas)` → selisih hari. Contoh `DATEDIFF('2026-11-13', '2026-11-10')` = `3`.
- Kalau telat > 0 → `telat * 5000`, kalau tidak → `0`.
- `DETERMINISTIC` → wajib ditulis di MySQL 8 (artinya: input sama → hasil selalu sama).

**Procedure `kembalikan_alat`**
- `DECLARE EXIT HANDLER FOR SQLEXCEPTION ... ROLLBACK` → "kalau ada error SQL di tengah jalan, batalkan semuanya".
- `START TRANSACTION` → mulai transaksi.
- `SELECT ... INTO v_batas` → ambil batas kembali, simpan ke variabel.
- `INSERT INTO pengembalian` → simpan data, denda dihitung oleh function `hitung_denda`.
- `UPDATE peminjaman SET status = 'dikembalikan'`.
- `COMMIT` → simpan permanen.

**Trigger `kurangi_stok`** (AFTER UPDATE pada `peminjaman`)
- `OLD` = data sebelum diubah, `NEW` = data sesudah diubah.
- Hanya jalan kalau status **berubah menjadi** `disetujui`, supaya stok tidak berkurang dua kali.

**Trigger `tambah_stok`** (AFTER INSERT pada `pengembalian`)
- Setiap ada data pengembalian baru, stok alat ditambah sesuai `jumlah` yang dipinjam.

**Baris `DROP ... IF EXISTS` di awal `up()`**
- `php artisan migrate:fresh` hanya menghapus **tabel**, bukan function/procedure. Tanpa baris ini, migrate ulang akan error *"FUNCTION hitung_denda already exists"*. (Trigger ikut terhapus bersama tabelnya, jadi tidak perlu.)

---

## 3. Jalankan

```bash
php artisan migrate
```

---

## 4. Uji langsung di phpMyAdmin (tab SQL)

Pilih database `db_peminjaman_alat` → tab **SQL**, jalankan satu per satu:

**Uji function:**
```sql
SELECT hitung_denda('2026-11-10', '2026-11-13');   -- hasil: 15000 (telat 3 hari)
SELECT hitung_denda('2026-11-10', '2026-11-09');   -- hasil: 0 (tidak telat)
```

**Lihat semua yang sudah dibuat:**
```sql
SHOW FUNCTION STATUS WHERE Db = 'db_peminjaman_alat';
SHOW PROCEDURE STATUS WHERE Db = 'db_peminjaman_alat';
SHOW TRIGGERS;
```

**Uji ROLLBACK** (panggil procedure dengan id peminjaman yang tidak ada):
```sql
CALL kembalikan_alat(999, '2026-11-13');
-- muncul error foreign key → procedure melakukan ROLLBACK
SELECT * FROM pengembalian;
-- tabel tetap kosong, artinya rollback berhasil
```

> Uji trigger & procedure dengan data asli akan kita lakukan di BAB 12 setelah aplikasi jadi.

---

## 5. Di mana COMMIT/ROLLBACK lainnya?

Selain di procedure, kita juga pakai transaction **dari Laravel** saat petugas menyetujui peminjaman (BAB 10):

```php
DB::beginTransaction();
try {
    // ... proses ...
    DB::commit();
} catch (\Exception $e) {
    DB::rollBack();
}
```

Jadi COMMIT & ROLLBACK ada di **dua tempat**: di SQL (procedure) dan di PHP (controller). Bagus untuk ditunjukkan ke penguji.

---

## ✅ Checklist Bab 04
- [ ] `SELECT hitung_denda(...)` menghasilkan angka yang benar
- [ ] `SHOW TRIGGERS` menampilkan `kurangi_stok` dan `tambah_stok`
- [ ] Procedure `kembalikan_alat` muncul

Lanjut ke **[BAB 05 — Model dan Seeder](/modul-laravel-peminjaman-alat/bab-05-model-dan-seeder/)**.
{% endraw %}

---

[← BAB 03 — Membuat Tabel dengan Migration](/modul-laravel-peminjaman-alat/bab-03-migration-tabel/) · [Daftar isi](/modul-laravel-peminjaman-alat/) · [BAB 05 — Model dan Seeder →](/modul-laravel-peminjaman-alat/bab-05-model-dan-seeder/)
