---
layout: page
title: "BAB 07 — Admin: CRUD Kategori & Alat | Modul Laravel 12 CRUD Peminjaman Alat"
description: "CRUD kategori dan alat di Laravel 12: Route::resource, controller, validasi, pagination dan view Bootstrap."
permalink: /modul-laravel-peminjaman-alat/bab-07-admin-kategori-alat/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
CRUD = **C**reate (tambah), **R**ead (lihat), **U**pdate (ubah), **D**elete (hapus).

Setiap CRUD di aplikasi ini **polanya sama persis**. Kalau kamu paham CRUD kategori, CRUD lain tinggal ganti nama.

| Fungsi controller | Tugas | View |
|---|---|---|
| `index()` | tampilkan daftar data | `index.blade.php` |
| `create()` | tampilkan form tambah | `create.blade.php` |
| `store()` | simpan data dari form tambah | – |
| `edit()` | tampilkan form edit | `edit.blade.php` |
| `update()` | simpan perubahan dari form edit | – |
| `destroy()` | hapus data | – |

---

## A. CRUD Kategori

### 1. Controller

```bash
php artisan make:controller KategoriController
```

Isi `app/Http/Controllers/KategoriController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Models\Kategori;
use App\Models\LogAktivitas;
use Illuminate\Http\Request;

class KategoriController extends Controller
{
    public function index()
    {
        $kategori = Kategori::latest()->paginate(10);
        return view('kategori.index', compact('kategori'));
    }

    public function create()
    {
        return view('kategori.create');
    }

    public function store(Request $request)
    {
        $data = $request->validate([
            'nama_kategori' => 'required',
        ]);

        Kategori::create($data);
        LogAktivitas::catat('Menambah kategori ' . $data['nama_kategori']);

        return redirect()->route('kategori.index')->with('success', 'Kategori berhasil ditambah');
    }

    public function edit(Kategori $kategori)
    {
        return view('kategori.edit', compact('kategori'));
    }

    public function update(Request $request, Kategori $kategori)
    {
        $data = $request->validate([
            'nama_kategori' => 'required',
        ]);

        $kategori->update($data);
        LogAktivitas::catat('Mengubah kategori ' . $kategori->nama_kategori);

        return redirect()->route('kategori.index')->with('success', 'Kategori berhasil diubah');
    }

    public function destroy(Kategori $kategori)
    {
        $kategori->delete();
        LogAktivitas::catat('Menghapus kategori ' . $kategori->nama_kategori);

        return redirect()->route('kategori.index')->with('success', 'Kategori berhasil dihapus');
    }
}
```

Penjelasan:
- `latest()` → urutkan dari yang terbaru (`ORDER BY created_at DESC`).
- `paginate(10)` → ambil 10 data per halaman (`LIMIT 10`). Ini memenuhi soal poin 8c **"gunakan limit"**.
- `edit(Kategori $kategori)` → **Route Model Binding**: Laravel otomatis mencari kategori berdasarkan id di URL. Kalau tidak ada → 404. Kita tidak perlu menulis `Kategori::find($id)`.
- `$request->validate()` mengembalikan data yang sudah lolos validasi → langsung dipakai di `create($data)`.

### 2. View

Buat folder `resources/views/kategori`, lalu 3 file:

**`resources/views/kategori/index.blade.php`**

```blade
@extends('layout')

@section('content')
<h3>Data Kategori</h3>
<a href="{{ route('kategori.create') }}" class="btn btn-primary mb-3">+ Tambah Kategori</a>

<table class="table table-bordered bg-white">
    <tr>
        <th>No</th>
        <th>Nama Kategori</th>
        <th>Aksi</th>
    </tr>
    @foreach ($kategori as $k)
    <tr>
        <td>{{ $kategori->firstItem() + $loop->index }}</td>
        <td>{{ $k->nama_kategori }}</td>
        <td>
            <a href="{{ route('kategori.edit', $k->id) }}" class="btn btn-warning btn-sm">Edit</a>
            <form action="{{ route('kategori.destroy', $k->id) }}" method="POST" class="d-inline">
                @csrf
                @method('DELETE')
                <button class="btn btn-danger btn-sm" onclick="return confirm('Yakin hapus?')">Hapus</button>
            </form>
        </td>
    </tr>
    @endforeach
</table>

{{ $kategori->links('pagination::bootstrap-5') }}
@endsection
```

- `$kategori->firstItem() + $loop->index` → nomor urut yang benar di setiap halaman (halaman 2 mulai dari 11).
- Form hapus pakai `@method('DELETE')` karena HTML form hanya mengenal GET & POST.
- `onclick="return confirm(...)"` → muncul konfirmasi sebelum menghapus.
- `links('pagination::bootstrap-5')` → tombol halaman 1, 2, 3 dengan gaya Bootstrap.

**`resources/views/kategori/create.blade.php`**

```blade
@extends('layout')

@section('content')
<h3>Tambah Kategori</h3>

<form action="{{ route('kategori.store') }}" method="POST" class="card card-body">
    @csrf
    <div class="mb-3">
        <label class="form-label">Nama Kategori</label>
        <input type="text" name="nama_kategori" class="form-control" value="{{ old('nama_kategori') }}">
    </div>
    <div>
        <button class="btn btn-primary">Simpan</button>
        <a href="{{ route('kategori.index') }}" class="btn btn-secondary">Kembali</a>
    </div>
</form>
@endsection
```

**`resources/views/kategori/edit.blade.php`**

```blade
@extends('layout')

@section('content')
<h3>Edit Kategori</h3>

<form action="{{ route('kategori.update', $kategori->id) }}" method="POST" class="card card-body">
    @csrf
    @method('PUT')
    <div class="mb-3">
        <label class="form-label">Nama Kategori</label>
        <input type="text" name="nama_kategori" class="form-control" value="{{ old('nama_kategori', $kategori->nama_kategori) }}">
    </div>
    <div>
        <button class="btn btn-primary">Update</button>
        <a href="{{ route('kategori.index') }}" class="btn btn-secondary">Kembali</a>
    </div>
</form>
@endsection
```

- `@method('PUT')` → untuk route `update`.
- `old('nama_kategori', $kategori->nama_kategori)` → tampilkan isian lama kalau validasi gagal, kalau tidak, tampilkan data dari database.

### 3. Coba
Login admin → menu **Kategori** → tambah, edit, hapus. Coba simpan form kosong → muncul pesan error merah.

---

## B. CRUD Alat

Polanya sama dengan kategori. Bedanya: alat punya **kategori** (dropdown) dan **stok**.

### 1. Controller

```bash
php artisan make:controller AlatController
```

```php
<?php

namespace App\Http\Controllers;

use App\Models\Alat;
use App\Models\Kategori;
use App\Models\LogAktivitas;
use Illuminate\Http\Request;

class AlatController extends Controller
{
    public function index()
    {
        // with('kategori') = ambil data kategori sekaligus (query lebih efisien)
        $alat = Alat::with('kategori')->latest()->paginate(10);
        return view('alat.index', compact('alat'));
    }

    public function create()
    {
        $kategori = Kategori::all();
        return view('alat.create', compact('kategori'));
    }

    public function store(Request $request)
    {
        $data = $request->validate([
            'kategori_id' => 'required',
            'nama_alat' => 'required',
            'stok' => 'required|integer|min:0',
            'keterangan' => 'nullable',
        ]);

        Alat::create($data);
        LogAktivitas::catat('Menambah alat ' . $data['nama_alat']);

        return redirect()->route('alat.index')->with('success', 'Alat berhasil ditambah');
    }

    public function edit(Alat $alat)
    {
        $kategori = Kategori::all();
        return view('alat.edit', compact('alat', 'kategori'));
    }

    public function update(Request $request, Alat $alat)
    {
        $data = $request->validate([
            'kategori_id' => 'required',
            'nama_alat' => 'required',
            'stok' => 'required|integer|min:0',
            'keterangan' => 'nullable',
        ]);

        $alat->update($data);
        LogAktivitas::catat('Mengubah alat ' . $alat->nama_alat);

        return redirect()->route('alat.index')->with('success', 'Alat berhasil diubah');
    }

    public function destroy(Alat $alat)
    {
        $alat->delete();
        LogAktivitas::catat('Menghapus alat ' . $alat->nama_alat);

        return redirect()->route('alat.index')->with('success', 'Alat berhasil dihapus');
    }
}
```

- `Alat::with('kategori')` → **eager loading**. Tanpa `with`, setiap baris tabel akan menjalankan 1 query tambahan untuk mengambil kategori (10 alat = 11 query). Dengan `with` → cukup **2 query**. Ini memenuhi soal poin 8a **"query yang efisien"**.
- `'stok' => 'required|integer|min:0'` → stok wajib angka dan tidak boleh minus.

### 2. View

Buat folder `resources/views/alat`.

**`resources/views/alat/index.blade.php`**

```blade
@extends('layout')

@section('content')
<h3>Data Alat</h3>
<a href="{{ route('alat.create') }}" class="btn btn-primary mb-3">+ Tambah Alat</a>

<table class="table table-bordered bg-white">
    <tr>
        <th>No</th>
        <th>Nama Alat</th>
        <th>Kategori</th>
        <th>Stok</th>
        <th>Keterangan</th>
        <th>Aksi</th>
    </tr>
    @foreach ($alat as $a)
    <tr>
        <td>{{ $alat->firstItem() + $loop->index }}</td>
        <td>{{ $a->nama_alat }}</td>
        <td>{{ $a->kategori->nama_kategori }}</td>
        <td>{{ $a->stok }}</td>
        <td>{{ $a->keterangan }}</td>
        <td>
            <a href="{{ route('alat.edit', $a->id) }}" class="btn btn-warning btn-sm">Edit</a>
            <form action="{{ route('alat.destroy', $a->id) }}" method="POST" class="d-inline">
                @csrf
                @method('DELETE')
                <button class="btn btn-danger btn-sm" onclick="return confirm('Yakin hapus?')">Hapus</button>
            </form>
        </td>
    </tr>
    @endforeach
</table>

{{ $alat->links('pagination::bootstrap-5') }}
@endsection
```

**`resources/views/alat/create.blade.php`**

```blade
@extends('layout')

@section('content')
<h3>Tambah Alat</h3>

<form action="{{ route('alat.store') }}" method="POST" class="card card-body">
    @csrf
    <div class="mb-3">
        <label class="form-label">Kategori</label>
        <select name="kategori_id" class="form-select">
            <option value="">-- pilih kategori --</option>
            @foreach ($kategori as $k)
                <option value="{{ $k->id }}">{{ $k->nama_kategori }}</option>
            @endforeach
        </select>
    </div>
    <div class="mb-3">
        <label class="form-label">Nama Alat</label>
        <input type="text" name="nama_alat" class="form-control" value="{{ old('nama_alat') }}">
    </div>
    <div class="mb-3">
        <label class="form-label">Stok</label>
        <input type="number" name="stok" class="form-control" value="{{ old('stok') }}">
    </div>
    <div class="mb-3">
        <label class="form-label">Keterangan</label>
        <input type="text" name="keterangan" class="form-control" value="{{ old('keterangan') }}">
    </div>
    <div>
        <button class="btn btn-primary">Simpan</button>
        <a href="{{ route('alat.index') }}" class="btn btn-secondary">Kembali</a>
    </div>
</form>
@endsection
```

**`resources/views/alat/edit.blade.php`**

```blade
@extends('layout')

@section('content')
<h3>Edit Alat</h3>

<form action="{{ route('alat.update', $alat->id) }}" method="POST" class="card card-body">
    @csrf
    @method('PUT')
    <div class="mb-3">
        <label class="form-label">Kategori</label>
        <select name="kategori_id" class="form-select">
            @foreach ($kategori as $k)
                <option value="{{ $k->id }}" @selected($k->id == $alat->kategori_id)>{{ $k->nama_kategori }}</option>
            @endforeach
        </select>
    </div>
    <div class="mb-3">
        <label class="form-label">Nama Alat</label>
        <input type="text" name="nama_alat" class="form-control" value="{{ old('nama_alat', $alat->nama_alat) }}">
    </div>
    <div class="mb-3">
        <label class="form-label">Stok</label>
        <input type="number" name="stok" class="form-control" value="{{ old('stok', $alat->stok) }}">
    </div>
    <div class="mb-3">
        <label class="form-label">Keterangan</label>
        <input type="text" name="keterangan" class="form-control" value="{{ old('keterangan', $alat->keterangan) }}">
    </div>
    <div>
        <button class="btn btn-primary">Update</button>
        <a href="{{ route('alat.index') }}" class="btn btn-secondary">Kembali</a>
    </div>
</form>
@endsection
```

`@selected(kondisi)` → menambahkan atribut `selected` pada option kalau kondisinya benar. Jadi kategori lama otomatis terpilih.

### 3. Coba
Menu **Alat** → tambah alat baru (misal *Kamera*, stok 2), edit stoknya, hapus.

---

## ✅ Checklist Bab 07
- [ ] CRUD kategori jalan
- [ ] CRUD alat jalan, kolom kategori tampil di tabel
- [ ] Validasi form kosong muncul
- [ ] Setiap tambah/ubah/hapus tercatat (nanti dilihat di Log Aktivitas, BAB 09)

Lanjut ke **[BAB 08 — CRUD User](/modul-laravel-peminjaman-alat/bab-08-admin-user/)**.
{% endraw %}

---

[← BAB 06 — Login, Logout, Hak Akses (Role), dan Layout](/modul-laravel-peminjaman-alat/bab-06-login-role-layout/) · [Daftar isi](/modul-laravel-peminjaman-alat/) · [BAB 08 — Admin: CRUD User →](/modul-laravel-peminjaman-alat/bab-08-admin-user/)
