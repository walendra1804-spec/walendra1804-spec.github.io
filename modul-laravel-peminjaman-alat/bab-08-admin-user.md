---
layout: page
title: "BAB 08 — Admin: CRUD User | Modul Laravel 12 CRUD Peminjaman Alat"
description: "CRUD user dengan role di Laravel 12: username unik, password di-hash, password boleh dikosongkan saat edit, dan larangan menghapus akun sendiri."
permalink: /modul-laravel-peminjaman-alat/bab-08-admin-user/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
Sama seperti CRUD sebelumnya, dengan 3 tambahan:
1. Username harus **unik**.
2. Saat edit, password **boleh dikosongkan** (artinya tidak diganti).
3. Admin **tidak boleh menghapus akunnya sendiri**.

---

## 1. Controller

```bash
php artisan make:controller UserController
```

Isi `app/Http/Controllers/UserController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Models\LogAktivitas;
use App\Models\User;
use Illuminate\Http\Request;

class UserController extends Controller
{
    public function index()
    {
        $users = User::latest()->paginate(10);
        return view('user.index', compact('users'));
    }

    public function create()
    {
        return view('user.create');
    }

    public function store(Request $request)
    {
        $data = $request->validate([
            'name' => 'required',
            'username' => 'required|unique:users',
            'password' => 'required|min:6',
            'role' => 'required',
        ]);

        User::create($data);
        LogAktivitas::catat('Menambah user ' . $data['username']);

        return redirect()->route('user.index')->with('success', 'User berhasil ditambah');
    }

    public function edit(User $user)
    {
        return view('user.edit', compact('user'));
    }

    public function update(Request $request, User $user)
    {
        $data = $request->validate([
            'name' => 'required',
            'username' => 'required|unique:users,username,' . $user->id,
            'password' => 'nullable|min:6',
            'role' => 'required',
        ]);

        // kalau password dikosongkan, password lama tidak diubah
        if ($request->password == null) {
            unset($data['password']);
        }

        $user->update($data);
        LogAktivitas::catat('Mengubah user ' . $user->username);

        return redirect()->route('user.index')->with('success', 'User berhasil diubah');
    }

    public function destroy(User $user)
    {
        // admin tidak boleh menghapus akunnya sendiri
        if ($user->id == auth()->id()) {
            return back()->with('error', 'Tidak bisa menghapus akun sendiri');
        }

        $user->delete();
        LogAktivitas::catat('Menghapus user ' . $user->username);

        return redirect()->route('user.index')->with('success', 'User berhasil dihapus');
    }
}
```

Penjelasan:
- `'username' => 'required|unique:users'` → cek ke tabel `users`, username tidak boleh sudah ada.
- `'unique:users,username,' . $user->id` → saat edit, username boleh sama dengan **miliknya sendiri** (id ini dikecualikan).
- `unset($data['password'])` → buang password dari data, jadi kolom password tidak ikut di-update.
- Password **tidak** perlu di-`Hash::make()` karena di model `User` sudah ada cast `'password' => 'hashed'`.

---

## 2. View

Buat folder `resources/views/user`.

**`resources/views/user/index.blade.php`**

```blade
@extends('layout')

@section('content')
<h3>Data User</h3>
<a href="{{ route('user.create') }}" class="btn btn-primary mb-3">+ Tambah User</a>

<table class="table table-bordered bg-white">
    <tr>
        <th>No</th>
        <th>Nama</th>
        <th>Username</th>
        <th>Role</th>
        <th>Aksi</th>
    </tr>
    @foreach ($users as $u)
    <tr>
        <td>{{ $users->firstItem() + $loop->index }}</td>
        <td>{{ $u->name }}</td>
        <td>{{ $u->username }}</td>
        <td>{{ $u->role }}</td>
        <td>
            <a href="{{ route('user.edit', $u->id) }}" class="btn btn-warning btn-sm">Edit</a>
            <form action="{{ route('user.destroy', $u->id) }}" method="POST" class="d-inline">
                @csrf
                @method('DELETE')
                <button class="btn btn-danger btn-sm" onclick="return confirm('Yakin hapus?')">Hapus</button>
            </form>
        </td>
    </tr>
    @endforeach
</table>

{{ $users->links('pagination::bootstrap-5') }}
@endsection
```

**`resources/views/user/create.blade.php`**

```blade
@extends('layout')

@section('content')
<h3>Tambah User</h3>

<form action="{{ route('user.store') }}" method="POST" class="card card-body">
    @csrf
    <div class="mb-3">
        <label class="form-label">Nama</label>
        <input type="text" name="name" class="form-control" value="{{ old('name') }}">
    </div>
    <div class="mb-3">
        <label class="form-label">Username</label>
        <input type="text" name="username" class="form-control" value="{{ old('username') }}">
    </div>
    <div class="mb-3">
        <label class="form-label">Password</label>
        <input type="password" name="password" class="form-control">
    </div>
    <div class="mb-3">
        <label class="form-label">Role</label>
        <select name="role" class="form-select">
            <option value="admin">Admin</option>
            <option value="petugas">Petugas</option>
            <option value="peminjam" selected>Peminjam</option>
        </select>
    </div>
    <div>
        <button class="btn btn-primary">Simpan</button>
        <a href="{{ route('user.index') }}" class="btn btn-secondary">Kembali</a>
    </div>
</form>
@endsection
```

**`resources/views/user/edit.blade.php`**

```blade
@extends('layout')

@section('content')
<h3>Edit User</h3>

<form action="{{ route('user.update', $user->id) }}" method="POST" class="card card-body">
    @csrf
    @method('PUT')
    <div class="mb-3">
        <label class="form-label">Nama</label>
        <input type="text" name="name" class="form-control" value="{{ old('name', $user->name) }}">
    </div>
    <div class="mb-3">
        <label class="form-label">Username</label>
        <input type="text" name="username" class="form-control" value="{{ old('username', $user->username) }}">
    </div>
    <div class="mb-3">
        <label class="form-label">Password <small class="text-muted">(kosongkan jika tidak diubah)</small></label>
        <input type="password" name="password" class="form-control">
    </div>
    <div class="mb-3">
        <label class="form-label">Role</label>
        <select name="role" class="form-select">
            <option value="admin" @selected($user->role == 'admin')>Admin</option>
            <option value="petugas" @selected($user->role == 'petugas')>Petugas</option>
            <option value="peminjam" @selected($user->role == 'peminjam')>Peminjam</option>
        </select>
    </div>
    <div>
        <button class="btn btn-primary">Update</button>
        <a href="{{ route('user.index') }}" class="btn btn-secondary">Kembali</a>
    </div>
</form>
@endsection
```

---

## 3. Coba
1. Tambah user baru role **peminjam** (misal `siti` / `siti123`).
2. Tambah lagi dengan username `siti` → harus error *"The username has already been taken."*
3. Edit user, kosongkan password → simpan → user tetap bisa login dengan password lama.
4. Hapus akun `admin` sendiri → muncul *"Tidak bisa menghapus akun sendiri"*.
5. Logout, login dengan `siti` → berhasil.

---

## ✅ Checklist Bab 08
- [ ] CRUD user jalan
- [ ] Username unik
- [ ] Password kosong saat edit tidak mengubah password

Lanjut ke **[BAB 09 — Peminjaman, Pengembalian, Log](/modul-laravel-peminjaman-alat/bab-09-admin-peminjaman-pengembalian-log/)**.
{% endraw %}

---

[← BAB 07 — Admin: CRUD Kategori & Alat](/modul-laravel-peminjaman-alat/bab-07-admin-kategori-alat/) · [Daftar isi](/modul-laravel-peminjaman-alat/) · [BAB 09 — Admin: CRUD Peminjaman, Pengembalian, dan Log Aktivitas →](/modul-laravel-peminjaman-alat/bab-09-admin-peminjaman-pengembalian-log/)
