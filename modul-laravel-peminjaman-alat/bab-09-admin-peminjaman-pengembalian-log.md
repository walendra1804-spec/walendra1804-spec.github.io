---
layout: page
title: "BAB 09 — Admin: CRUD Peminjaman, Pengembalian, dan Log Aktivitas | Modul Laravel 12 CRUD Peminjaman Alat"
description: "CRUD peminjaman dan pengembalian yang memanggil stored procedure dan function MySQL dari Laravel, plus halaman log aktivitas."
permalink: /modul-laravel-peminjaman-alat/bab-09-admin-peminjaman-pengembalian-log/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
Di bab ini kita mulai **memakai** function, procedure, dan trigger yang dibuat di BAB 04.

---

## A. CRUD Data Peminjaman

Admin bisa menambah peminjaman atas nama peminjam, mengubah (termasuk status), dan menghapus.

### 1. Controller

```bash
php artisan make:controller PeminjamanController
```

```php
<?php

namespace App\Http\Controllers;

use App\Models\Alat;
use App\Models\LogAktivitas;
use App\Models\Peminjaman;
use App\Models\User;
use Illuminate\Http\Request;

class PeminjamanController extends Controller
{
    public function index()
    {
        $peminjaman = Peminjaman::with('user', 'alat')->latest()->paginate(10);
        return view('peminjaman.index', compact('peminjaman'));
    }

    public function create()
    {
        $users = User::where('role', 'peminjam')->get();
        $alat = Alat::all();
        return view('peminjaman.create', compact('users', 'alat'));
    }

    public function store(Request $request)
    {
        $data = $request->validate([
            'user_id' => 'required',
            'alat_id' => 'required',
            'jumlah' => 'required|integer|min:1',
            'tanggal_pinjam' => 'required|date',
            'batas_kembali' => 'required|date|after_or_equal:tanggal_pinjam',
        ]);

        Peminjaman::create($data);
        LogAktivitas::catat('Menambah data peminjaman');

        return redirect()->route('peminjaman.index')->with('success', 'Peminjaman berhasil ditambah');
    }

    public function edit(Peminjaman $peminjaman)
    {
        $users = User::where('role', 'peminjam')->get();
        $alat = Alat::all();
        return view('peminjaman.edit', compact('peminjaman', 'users', 'alat'));
    }

    public function update(Request $request, Peminjaman $peminjaman)
    {
        $data = $request->validate([
            'user_id' => 'required',
            'alat_id' => 'required',
            'jumlah' => 'required|integer|min:1',
            'tanggal_pinjam' => 'required|date',
            'batas_kembali' => 'required|date|after_or_equal:tanggal_pinjam',
            'status' => 'required',
        ]);

        $peminjaman->update($data);
        LogAktivitas::catat('Mengubah data peminjaman #' . $peminjaman->id);

        return redirect()->route('peminjaman.index')->with('success', 'Peminjaman berhasil diubah');
    }

    public function destroy(Peminjaman $peminjaman)
    {
        $peminjaman->delete();
        LogAktivitas::catat('Menghapus data peminjaman #' . $peminjaman->id);

        return redirect()->route('peminjaman.index')->with('success', 'Peminjaman berhasil dihapus');
    }
}
```

- `User::where('role', 'peminjam')->get()` → dropdown hanya berisi user ber-role peminjam.
- `'batas_kembali' => '...|after_or_equal:tanggal_pinjam'` → batas kembali tidak boleh sebelum tanggal pinjam.
- Data baru otomatis berstatus `menunggu` (default di migration).
- Kalau admin mengubah status menjadi **disetujui**, **trigger `kurangi_stok`** otomatis mengurangi stok alat.

### 2. View

Buat folder `resources/views/peminjaman`.

**`resources/views/peminjaman/index.blade.php`**

```blade
@extends('layout')

@section('content')
<h3>Data Peminjaman</h3>
<a href="{{ route('peminjaman.create') }}" class="btn btn-primary mb-3">+ Tambah Peminjaman</a>

<table class="table table-bordered bg-white">
    <tr>
        <th>No</th>
        <th>Peminjam</th>
        <th>Alat</th>
        <th>Jumlah</th>
        <th>Tgl Pinjam</th>
        <th>Batas Kembali</th>
        <th>Status</th>
        <th>Aksi</th>
    </tr>
    @foreach ($peminjaman as $p)
    <tr>
        <td>{{ $peminjaman->firstItem() + $loop->index }}</td>
        <td>{{ $p->user->name }}</td>
        <td>{{ $p->alat->nama_alat }}</td>
        <td>{{ $p->jumlah }}</td>
        <td>{{ $p->tanggal_pinjam }}</td>
        <td>{{ $p->batas_kembali }}</td>
        <td><span class="badge bg-secondary">{{ $p->status }}</span></td>
        <td>
            <a href="{{ route('peminjaman.edit', $p->id) }}" class="btn btn-warning btn-sm">Edit</a>
            <form action="{{ route('peminjaman.destroy', $p->id) }}" method="POST" class="d-inline">
                @csrf
                @method('DELETE')
                <button class="btn btn-danger btn-sm" onclick="return confirm('Yakin hapus?')">Hapus</button>
            </form>
        </td>
    </tr>
    @endforeach
</table>

{{ $peminjaman->links('pagination::bootstrap-5') }}
@endsection
```

**`resources/views/peminjaman/create.blade.php`**

```blade
@extends('layout')

@section('content')
<h3>Tambah Peminjaman</h3>

<form action="{{ route('peminjaman.store') }}" method="POST" class="card card-body">
    @csrf
    <div class="mb-3">
        <label class="form-label">Peminjam</label>
        <select name="user_id" class="form-select">
            @foreach ($users as $u)
                <option value="{{ $u->id }}">{{ $u->name }}</option>
            @endforeach
        </select>
    </div>
    <div class="mb-3">
        <label class="form-label">Alat</label>
        <select name="alat_id" class="form-select">
            @foreach ($alat as $a)
                <option value="{{ $a->id }}">{{ $a->nama_alat }} (stok: {{ $a->stok }})</option>
            @endforeach
        </select>
    </div>
    <div class="mb-3">
        <label class="form-label">Jumlah</label>
        <input type="number" name="jumlah" class="form-control" value="{{ old('jumlah', 1) }}">
    </div>
    <div class="mb-3">
        <label class="form-label">Tanggal Pinjam</label>
        <input type="date" name="tanggal_pinjam" class="form-control" value="{{ old('tanggal_pinjam', date('Y-m-d')) }}">
    </div>
    <div class="mb-3">
        <label class="form-label">Batas Kembali</label>
        <input type="date" name="batas_kembali" class="form-control" value="{{ old('batas_kembali') }}">
    </div>
    <div>
        <button class="btn btn-primary">Simpan</button>
        <a href="{{ route('peminjaman.index') }}" class="btn btn-secondary">Kembali</a>
    </div>
</form>
@endsection
```

`date('Y-m-d')` → tanggal hari ini sebagai nilai awal.

**`resources/views/peminjaman/edit.blade.php`**

```blade
@extends('layout')

@section('content')
<h3>Edit Peminjaman</h3>

<form action="{{ route('peminjaman.update', $peminjaman->id) }}" method="POST" class="card card-body">
    @csrf
    @method('PUT')
    <div class="mb-3">
        <label class="form-label">Peminjam</label>
        <select name="user_id" class="form-select">
            @foreach ($users as $u)
                <option value="{{ $u->id }}" @selected($u->id == $peminjaman->user_id)>{{ $u->name }}</option>
            @endforeach
        </select>
    </div>
    <div class="mb-3">
        <label class="form-label">Alat</label>
        <select name="alat_id" class="form-select">
            @foreach ($alat as $a)
                <option value="{{ $a->id }}" @selected($a->id == $peminjaman->alat_id)>{{ $a->nama_alat }} (stok: {{ $a->stok }})</option>
            @endforeach
        </select>
    </div>
    <div class="mb-3">
        <label class="form-label">Jumlah</label>
        <input type="number" name="jumlah" class="form-control" value="{{ old('jumlah', $peminjaman->jumlah) }}">
    </div>
    <div class="mb-3">
        <label class="form-label">Tanggal Pinjam</label>
        <input type="date" name="tanggal_pinjam" class="form-control" value="{{ old('tanggal_pinjam', $peminjaman->tanggal_pinjam) }}">
    </div>
    <div class="mb-3">
        <label class="form-label">Batas Kembali</label>
        <input type="date" name="batas_kembali" class="form-control" value="{{ old('batas_kembali', $peminjaman->batas_kembali) }}">
    </div>
    <div class="mb-3">
        <label class="form-label">Status</label>
        <select name="status" class="form-select">
            <option value="menunggu" @selected($peminjaman->status == 'menunggu')>Menunggu</option>
            <option value="disetujui" @selected($peminjaman->status == 'disetujui')>Disetujui</option>
            <option value="ditolak" @selected($peminjaman->status == 'ditolak')>Ditolak</option>
            <option value="dikembalikan" @selected($peminjaman->status == 'dikembalikan')>Dikembalikan</option>
        </select>
    </div>
    <div>
        <button class="btn btn-primary">Update</button>
        <a href="{{ route('peminjaman.index') }}" class="btn btn-secondary">Kembali</a>
    </div>
</form>
@endsection
```

---

## B. CRUD Pengembalian

- **Tambah** → memanggil **stored procedure** `kembalikan_alat`.
- **Edit** tanggal → denda dihitung ulang dengan **function** `hitung_denda`.

### 1. Controller

```bash
php artisan make:controller PengembalianController
```

```php
<?php

namespace App\Http\Controllers;

use App\Models\LogAktivitas;
use App\Models\Peminjaman;
use App\Models\Pengembalian;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class PengembalianController extends Controller
{
    public function index()
    {
        $pengembalian = Pengembalian::with('peminjaman.user', 'peminjaman.alat')->latest()->paginate(10);
        return view('pengembalian.index', compact('pengembalian'));
    }

    public function create()
    {
        // hanya peminjaman yang sedang dipinjam (disetujui) yang bisa dikembalikan
        $peminjaman = Peminjaman::with('user', 'alat')->where('status', 'disetujui')->get();
        return view('pengembalian.create', compact('peminjaman'));
    }

    public function store(Request $request)
    {
        $data = $request->validate([
            'peminjaman_id' => 'required',
            'tanggal_dikembalikan' => 'required|date',
        ]);

        // panggil STORED PROCEDURE kembalikan_alat
        DB::statement('CALL kembalikan_alat(?, ?)', [$data['peminjaman_id'], $data['tanggal_dikembalikan']]);
        LogAktivitas::catat('Menambah data pengembalian peminjaman #' . $data['peminjaman_id']);

        return redirect()->route('pengembalian.index')->with('success', 'Pengembalian berhasil disimpan');
    }

    public function edit(Pengembalian $pengembalian)
    {
        return view('pengembalian.edit', compact('pengembalian'));
    }

    public function update(Request $request, Pengembalian $pengembalian)
    {
        $data = $request->validate([
            'tanggal_dikembalikan' => 'required|date',
        ]);

        // hitung ulang denda pakai FUNCTION hitung_denda
        $batas = $pengembalian->peminjaman->batas_kembali;
        $hasil = DB::selectOne('SELECT hitung_denda(?, ?) AS denda', [$batas, $data['tanggal_dikembalikan']]);

        $pengembalian->update([
            'tanggal_dikembalikan' => $data['tanggal_dikembalikan'],
            'denda' => $hasil->denda,
        ]);
        LogAktivitas::catat('Mengubah data pengembalian #' . $pengembalian->id);

        return redirect()->route('pengembalian.index')->with('success', 'Pengembalian berhasil diubah');
    }

    public function destroy(Pengembalian $pengembalian)
    {
        $pengembalian->delete();
        LogAktivitas::catat('Menghapus data pengembalian #' . $pengembalian->id);

        return redirect()->route('pengembalian.index')->with('success', 'Pengembalian berhasil dihapus');
    }
}
```

Penjelasan:
- `with('peminjaman.user', 'peminjaman.alat')` → eager loading **bertingkat**: pengembalian → peminjaman → user & alat.
- `DB::statement('CALL kembalikan_alat(?, ?)', [...])` → memanggil stored procedure. Tanda `?` diganti nilai dari array (aman dari SQL injection).
- `DB::selectOne('SELECT hitung_denda(?, ?) AS denda', [...])` → memanggil function MySQL, hasilnya diambil dengan `$hasil->denda`.

### 2. View

Buat folder `resources/views/pengembalian`.

**`resources/views/pengembalian/index.blade.php`**

```blade
@extends('layout')

@section('content')
<h3>Data Pengembalian</h3>
<a href="{{ route('pengembalian.create') }}" class="btn btn-primary mb-3">+ Tambah Pengembalian</a>

<table class="table table-bordered bg-white">
    <tr>
        <th>No</th>
        <th>Peminjam</th>
        <th>Alat</th>
        <th>Batas Kembali</th>
        <th>Tgl Dikembalikan</th>
        <th>Denda</th>
        <th>Aksi</th>
    </tr>
    @foreach ($pengembalian as $k)
    <tr>
        <td>{{ $pengembalian->firstItem() + $loop->index }}</td>
        <td>{{ $k->peminjaman->user->name }}</td>
        <td>{{ $k->peminjaman->alat->nama_alat }}</td>
        <td>{{ $k->peminjaman->batas_kembali }}</td>
        <td>{{ $k->tanggal_dikembalikan }}</td>
        <td>Rp {{ number_format($k->denda, 0, ',', '.') }}</td>
        <td>
            <a href="{{ route('pengembalian.edit', $k->id) }}" class="btn btn-warning btn-sm">Edit</a>
            <form action="{{ route('pengembalian.destroy', $k->id) }}" method="POST" class="d-inline">
                @csrf
                @method('DELETE')
                <button class="btn btn-danger btn-sm" onclick="return confirm('Yakin hapus?')">Hapus</button>
            </form>
        </td>
    </tr>
    @endforeach
</table>

{{ $pengembalian->links('pagination::bootstrap-5') }}
@endsection
```

`number_format($k->denda, 0, ',', '.')` → mengubah `15000` jadi `15.000`.

**`resources/views/pengembalian/create.blade.php`**

```blade
@extends('layout')

@section('content')
<h3>Tambah Pengembalian</h3>

<form action="{{ route('pengembalian.store') }}" method="POST" class="card card-body">
    @csrf
    <div class="mb-3">
        <label class="form-label">Peminjaman (yang sedang dipinjam)</label>
        <select name="peminjaman_id" class="form-select">
            @foreach ($peminjaman as $p)
                <option value="{{ $p->id }}">
                    #{{ $p->id }} - {{ $p->user->name }} - {{ $p->alat->nama_alat }} (batas: {{ $p->batas_kembali }})
                </option>
            @endforeach
        </select>
    </div>
    <div class="mb-3">
        <label class="form-label">Tanggal Dikembalikan</label>
        <input type="date" name="tanggal_dikembalikan" class="form-control" value="{{ date('Y-m-d') }}">
    </div>
    <p class="text-muted">Denda dihitung otomatis: Rp 5.000 per hari keterlambatan.</p>
    <div>
        <button class="btn btn-primary">Simpan</button>
        <a href="{{ route('pengembalian.index') }}" class="btn btn-secondary">Kembali</a>
    </div>
</form>
@endsection
```

**`resources/views/pengembalian/edit.blade.php`**

```blade
@extends('layout')

@section('content')
<h3>Edit Pengembalian</h3>

<form action="{{ route('pengembalian.update', $pengembalian->id) }}" method="POST" class="card card-body">
    @csrf
    @method('PUT')
    <p>
        Peminjam: <b>{{ $pengembalian->peminjaman->user->name }}</b><br>
        Alat: <b>{{ $pengembalian->peminjaman->alat->nama_alat }}</b><br>
        Batas Kembali: <b>{{ $pengembalian->peminjaman->batas_kembali }}</b>
    </p>
    <div class="mb-3">
        <label class="form-label">Tanggal Dikembalikan</label>
        <input type="date" name="tanggal_dikembalikan" class="form-control" value="{{ $pengembalian->tanggal_dikembalikan }}">
    </div>
    <p class="text-muted">Denda akan dihitung ulang otomatis.</p>
    <div>
        <button class="btn btn-primary">Update</button>
        <a href="{{ route('pengembalian.index') }}" class="btn btn-secondary">Kembali</a>
    </div>
</form>
@endsection
```

---

## C. Log Aktivitas

Sejak BAB 06 setiap aksi sudah dicatat dengan `LogAktivitas::catat(...)`. Sekarang tinggal menampilkannya.

```bash
php artisan make:controller LogController
```

```php
<?php

namespace App\Http\Controllers;

use App\Models\LogAktivitas;

class LogController extends Controller
{
    public function index()
    {
        $logs = LogAktivitas::with('user')->latest()->paginate(20);
        return view('log.index', compact('logs'));
    }
}
```

Buat folder `resources/views/log`, file **`resources/views/log/index.blade.php`**:

```blade
@extends('layout')

@section('content')
<h3>Log Aktivitas</h3>

<table class="table table-bordered bg-white">
    <tr>
        <th>Waktu</th>
        <th>User</th>
        <th>Aktivitas</th>
    </tr>
    @foreach ($logs as $log)
    <tr>
        <td>{{ $log->created_at }}</td>
        <td>{{ $log->user->name ?? '-' }}</td>
        <td>{{ $log->aktivitas }}</td>
    </tr>
    @endforeach
</table>

{{ $logs->links('pagination::bootstrap-5') }}
@endsection
```

`$log->user->name ?? '-'` → kalau user-nya sudah dihapus (`user_id` = NULL), tampilkan `-` supaya tidak error.

---

## D. Coba alur lengkap sebagai admin

1. **Peminjaman** → Tambah: peminjam *Budi Siswa*, alat *Bola Basket* (stok 10), jumlah 2, tanggal pinjam `2026-11-01`, batas kembali `2026-11-10`.
2. Edit peminjaman tadi → status **Disetujui** → buka menu **Alat**: stok Bola Basket jadi **8** (trigger `kurangi_stok` bekerja).
3. **Pengembalian** → Tambah: pilih peminjaman tadi, tanggal dikembalikan `2026-11-14`.
   - Denda = telat 4 hari × 5.000 = **Rp 20.000** (function `hitung_denda`).
   - Stok Bola Basket kembali **10** (trigger `tambah_stok`).
   - Status peminjaman jadi **dikembalikan** (procedure).
4. Edit pengembalian, ubah tanggal ke `2026-11-12` → denda jadi **Rp 10.000**.
5. Buka **Log Aktivitas** → semua langkah di atas tercatat.

---

## ✅ Checklist Bab 09
- [ ] CRUD peminjaman jalan, stok berkurang saat status disetujui
- [ ] Tambah pengembalian → denda benar, stok kembali
- [ ] Edit pengembalian → denda dihitung ulang
- [ ] Log aktivitas tampil

Lanjut ke **[BAB 10 — Fitur Petugas](/modul-laravel-peminjaman-alat/bab-10-petugas/)**.
{% endraw %}

---

[← BAB 08 — Admin: CRUD User](/modul-laravel-peminjaman-alat/bab-08-admin-user/) · [Daftar isi](/modul-laravel-peminjaman-alat/) · [BAB 10 — Fitur Petugas →](/modul-laravel-peminjaman-alat/bab-10-petugas/)
