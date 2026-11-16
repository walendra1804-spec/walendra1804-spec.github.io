---
layout: page
title: "BAB 06 — Login, Logout, Hak Akses (Role), dan Layout | Modul Laravel 12 CRUD Peminjaman Alat"
description: "Membuat login dan logout pakai username, middleware hak akses per role (admin, petugas, peminjam) dan layout Bootstrap 5 di Laravel 12."
permalink: /modul-laravel-peminjaman-alat/bab-06-login-role-layout/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
Bab ini membuat:
1. Halaman login & proses logout
2. **Middleware** `CekRole` → mengunci halaman berdasarkan role
3. Semua **route** aplikasi
4. **Layout** (navbar + menu sesuai role) dan halaman dashboard

---

## 1. Controller login

```bash
php artisan make:controller AuthController
```

Isi `app/Http/Controllers/AuthController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Models\LogAktivitas;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class AuthController extends Controller
{
    // menampilkan halaman login
    public function index()
    {
        return view('auth.login');
    }

    // memproses login
    public function login(Request $request)
    {
        $data = $request->validate([
            'username' => 'required',
            'password' => 'required',
        ]);

        if (Auth::attempt($data)) {
            $request->session()->regenerate();
            LogAktivitas::catat('Login ke sistem');
            return redirect('/dashboard');
        }

        return back()->withInput()->with('error', 'Username atau password salah!');
    }

    // memproses logout
    public function logout(Request $request)
    {
        LogAktivitas::catat('Logout dari sistem');
        Auth::logout();
        $request->session()->invalidate();
        $request->session()->regenerateToken();
        return redirect('/login');
    }
}
```

Penjelasan:
- `$request->validate([...])` → kalau username/password kosong, otomatis kembali ke form dengan pesan error.
- `Auth::attempt($data)` → Laravel mengecek ke tabel `users`: apakah ada username itu dan password-nya cocok (hash). Hasilnya `true`/`false`.
- `session()->regenerate()` → keamanan: membuat ID sesi baru setelah login.
- `back()->withInput()->with('error', ...)` → kembali ke halaman sebelumnya, isian username tetap ada, dan bawa pesan error.

---

## 2. Middleware cek role

**Middleware** = "satpam" yang memeriksa request sebelum masuk ke controller.

```bash
php artisan make:middleware CekRole
```

Isi `app/Http/Middleware/CekRole.php`:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class CekRole
{
    // contoh pemakaian di route: ->middleware('role:admin')
    public function handle(Request $request, Closure $next, ...$roles)
    {
        if (!in_array(auth()->user()->role, $roles)) {
            abort(403, 'Anda tidak punya akses ke halaman ini.');
        }

        return $next($request);
    }
}
```

- `...$roles` → bisa menerima satu atau lebih role, misal `role:admin` atau `role:admin,petugas`.
- `in_array(...)` → cek apakah role user ada di daftar yang diizinkan.
- `abort(403)` → tampilkan halaman **403 Forbidden** (dilarang masuk).

Daftarkan middleware dengan nama pendek `role`. Buka `bootstrap/app.php`, ganti isinya:

```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware): void {
        // daftarkan middleware CekRole dengan nama 'role'
        $middleware->alias([
            'role' => \App\Http\Middleware\CekRole::class,
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions): void {
        //
    })->create();
```

---

## 3. Route

Ganti **seluruh isi** `routes/web.php`. Di sini langsung kita tulis semua route aplikasi, supaya nanti tidak bolak-balik.

```php
<?php

use App\Http\Controllers\AlatController;
use App\Http\Controllers\AuthController;
use App\Http\Controllers\DashboardController;
use App\Http\Controllers\KategoriController;
use App\Http\Controllers\LogController;
use App\Http\Controllers\PeminjamanController;
use App\Http\Controllers\PeminjamController;
use App\Http\Controllers\PengembalianController;
use App\Http\Controllers\PetugasController;
use App\Http\Controllers\UserController;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    return redirect('/login');
});

// hanya untuk yang BELUM login
Route::middleware('guest')->group(function () {
    Route::get('/login', [AuthController::class, 'index'])->name('login');
    Route::post('/login', [AuthController::class, 'login']);
});

// hanya untuk yang SUDAH login
Route::middleware('auth')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout'])->name('logout');
    Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard');

    // ===== ADMIN =====
    Route::middleware('role:admin')->prefix('admin')->group(function () {
        Route::resource('user', UserController::class)->except('show');
        Route::resource('kategori', KategoriController::class)->except('show');
        Route::resource('alat', AlatController::class)->except('show');
        Route::resource('peminjaman', PeminjamanController::class)->except('show');
        Route::resource('pengembalian', PengembalianController::class)->except('show');
        Route::get('log', [LogController::class, 'index'])->name('log.index');
    });

    // ===== PETUGAS =====
    Route::middleware('role:petugas')->prefix('petugas')->group(function () {
        Route::get('peminjaman', [PetugasController::class, 'peminjaman'])->name('petugas.peminjaman');
        Route::post('peminjaman/{peminjaman}/setujui', [PetugasController::class, 'setujui'])->name('petugas.setujui');
        Route::post('peminjaman/{peminjaman}/tolak', [PetugasController::class, 'tolak'])->name('petugas.tolak');
        Route::get('pengembalian', [PetugasController::class, 'pengembalian'])->name('petugas.pengembalian');
        Route::get('laporan', [PetugasController::class, 'laporan'])->name('petugas.laporan');
    });

    // ===== PEMINJAM =====
    Route::middleware('role:peminjam')->prefix('peminjam')->group(function () {
        Route::get('alat', [PeminjamController::class, 'alat'])->name('peminjam.alat');
        Route::get('pinjam/{alat}', [PeminjamController::class, 'formPinjam'])->name('peminjam.form');
        Route::post('pinjam/{alat}', [PeminjamController::class, 'pinjam'])->name('peminjam.pinjam');
        Route::get('riwayat', [PeminjamController::class, 'riwayat'])->name('peminjam.riwayat');
        Route::post('kembalikan/{peminjaman}', [PeminjamController::class, 'kembalikan'])->name('peminjam.kembalikan');
    });
});
```

Penjelasan:
- `middleware('guest')` → hanya untuk yang **belum** login (kalau sudah login, dilempar ke dashboard).
- `middleware('auth')` → hanya untuk yang **sudah** login (kalau belum, dilempar ke route bernama `login`).
- `middleware('role:admin')` → middleware buatan kita tadi.
- `prefix('admin')` → semua URL di dalamnya diawali `/admin/...`.
- `Route::resource(...)` → **satu baris membuat 7 route CRUD** sekaligus (index, create, store, edit, update, destroy, show). `->except('show')` karena kita tidak butuh halaman detail.

Contoh hasil `Route::resource('alat', AlatController::class)`:

| Method | URL | Nama route | Fungsi controller |
|---|---|---|---|
| GET | /admin/alat | alat.index | index() |
| GET | /admin/alat/create | alat.create | create() |
| POST | /admin/alat | alat.store | store() |
| GET | /admin/alat/{alat}/edit | alat.edit | edit() |
| PUT | /admin/alat/{alat} | alat.update | update() |
| DELETE | /admin/alat/{alat} | alat.destroy | destroy() |

> Controller selain `AuthController` & `DashboardController` belum ada. Itu tidak apa-apa — kita buat di bab-bab berikutnya. **Jangan klik menu selain Dashboard dulu**, nanti muncul error *"Target class does not exist"*.

---

## 4. Controller dashboard

```bash
php artisan make:controller DashboardController
```

```php
<?php

namespace App\Http\Controllers;

use App\Models\Alat;
use App\Models\Peminjaman;
use App\Models\User;

class DashboardController extends Controller
{
    public function index()
    {
        $jumlahAlat = Alat::count();
        $jumlahUser = User::count();
        $menunggu = Peminjaman::where('status', 'menunggu')->count();
        $dipinjam = Peminjaman::where('status', 'disetujui')->count();

        return view('dashboard', compact('jumlahAlat', 'jumlahUser', 'menunggu', 'dipinjam'));
    }
}
```

`compact('jumlahAlat', ...)` = cara singkat mengirim variabel ke view. Sama dengan `['jumlahAlat' => $jumlahAlat, ...]`.

---

## 5. View

Buat folder `resources/views/auth`. Lalu buat file-file berikut.

### `resources/views/layout.blade.php` (kerangka semua halaman)

```blade
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Peminjaman Alat</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
    <style>
        /* saat dicetak, sembunyikan navbar & tombol */
        @media print {
            .no-print { display: none !important; }
        }
    </style>
</head>
<body class="bg-light">

<nav class="navbar navbar-expand-lg navbar-dark bg-primary mb-4 no-print">
    <div class="container">
        <a class="navbar-brand" href="/dashboard">Peminjaman Alat</a>
        <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#menu">
            <span class="navbar-toggler-icon"></span>
        </button>

        <div class="collapse navbar-collapse" id="menu">
            <ul class="navbar-nav me-auto">
                <li class="nav-item"><a class="nav-link" href="/dashboard">Dashboard</a></li>

                @if (auth()->user()->role == 'admin')
                    <li class="nav-item"><a class="nav-link" href="{{ route('user.index') }}">User</a></li>
                    <li class="nav-item"><a class="nav-link" href="{{ route('kategori.index') }}">Kategori</a></li>
                    <li class="nav-item"><a class="nav-link" href="{{ route('alat.index') }}">Alat</a></li>
                    <li class="nav-item"><a class="nav-link" href="{{ route('peminjaman.index') }}">Peminjaman</a></li>
                    <li class="nav-item"><a class="nav-link" href="{{ route('pengembalian.index') }}">Pengembalian</a></li>
                    <li class="nav-item"><a class="nav-link" href="{{ route('log.index') }}">Log Aktivitas</a></li>
                @endif

                @if (auth()->user()->role == 'petugas')
                    <li class="nav-item"><a class="nav-link" href="{{ route('petugas.peminjaman') }}">Persetujuan</a></li>
                    <li class="nav-item"><a class="nav-link" href="{{ route('petugas.pengembalian') }}">Pantau Pengembalian</a></li>
                    <li class="nav-item"><a class="nav-link" href="{{ route('petugas.laporan') }}">Laporan</a></li>
                @endif

                @if (auth()->user()->role == 'peminjam')
                    <li class="nav-item"><a class="nav-link" href="{{ route('peminjam.alat') }}">Daftar Alat</a></li>
                    <li class="nav-item"><a class="nav-link" href="{{ route('peminjam.riwayat') }}">Peminjaman Saya</a></li>
                @endif
            </ul>

            <span class="navbar-text text-white me-3">
                {{ auth()->user()->name }} ({{ auth()->user()->role }})
            </span>
            <form action="{{ route('logout') }}" method="POST">
                @csrf
                <button class="btn btn-light btn-sm">Logout</button>
            </form>
        </div>
    </div>
</nav>

<div class="container">
    {{-- pesan sukses / gagal --}}
    @if (session('success'))
        <div class="alert alert-success no-print">{{ session('success') }}</div>
    @endif
    @if (session('error'))
        <div class="alert alert-danger no-print">{{ session('error') }}</div>
    @endif

    {{-- pesan error validasi --}}
    @if ($errors->any())
        <div class="alert alert-danger no-print">
            <ul class="mb-0">
                @foreach ($errors->all() as $error)
                    <li>{{ $error }}</li>
                @endforeach
            </ul>
        </div>
    @endif

    @yield('content')
</div>

<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

Penjelasan Blade:
- `{{ ... }}` → menampilkan isi variabel (otomatis aman dari XSS).
- `@if ... @endif`, `@foreach ... @endforeach` → percabangan & perulangan.
- `@yield('content')` → "lubang" yang nanti diisi oleh setiap halaman.
- `@csrf` → token keamanan, **wajib** ada di setiap `<form method="POST">`. Kalau lupa → error **419 Page Expired**.
- `session('success')` → pesan yang dikirim dari controller lewat `->with('success', '...')`.
- `$errors` → pesan error validasi.
- `@media print { .no-print ... }` → elemen dengan class `no-print` disembunyikan saat dicetak (dipakai di laporan).

### `resources/views/auth/login.blade.php`

```blade
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Login - Peminjaman Alat</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body class="bg-light">

<div class="container">
    <div class="row justify-content-center mt-5">
        <div class="col-md-4">
            <div class="card shadow-sm">
                <div class="card-body">
                    <h4 class="text-center mb-4">Login Peminjaman Alat</h4>

                    @if (session('error'))
                        <div class="alert alert-danger">{{ session('error') }}</div>
                    @endif

                    <form action="/login" method="POST">
                        @csrf
                        <div class="mb-3">
                            <label class="form-label">Username</label>
                            <input type="text" name="username" class="form-control" value="{{ old('username') }}" required>
                        </div>
                        <div class="mb-3">
                            <label class="form-label">Password</label>
                            <input type="password" name="password" class="form-control" required>
                        </div>
                        <button class="btn btn-primary w-100">Login</button>
                    </form>
                </div>
            </div>
        </div>
    </div>
</div>

</body>
</html>
```

`old('username')` → mengisi ulang input dengan nilai sebelumnya kalau login gagal.

### `resources/views/dashboard.blade.php`

```blade
@extends('layout')

@section('content')
<h3>Dashboard</h3>
<p>Selamat datang, <b>{{ auth()->user()->name }}</b>. Anda login sebagai <b>{{ auth()->user()->role }}</b>.</p>

<div class="row">
    <div class="col-md-3 mb-3">
        <div class="card text-bg-primary">
            <div class="card-body">
                <h6>Jumlah Alat</h6>
                <h2>{{ $jumlahAlat }}</h2>
            </div>
        </div>
    </div>
    <div class="col-md-3 mb-3">
        <div class="card text-bg-success">
            <div class="card-body">
                <h6>Jumlah User</h6>
                <h2>{{ $jumlahUser }}</h2>
            </div>
        </div>
    </div>
    <div class="col-md-3 mb-3">
        <div class="card text-bg-warning">
            <div class="card-body">
                <h6>Menunggu Persetujuan</h6>
                <h2>{{ $menunggu }}</h2>
            </div>
        </div>
    </div>
    <div class="col-md-3 mb-3">
        <div class="card text-bg-danger">
            <div class="card-body">
                <h6>Sedang Dipinjam</h6>
                <h2>{{ $dipinjam }}</h2>
            </div>
        </div>
    </div>
</div>
@endsection
```

- `@extends('layout')` → pakai kerangka `layout.blade.php`.
- `@section('content') ... @endsection` → isi yang dimasukkan ke `@yield('content')`.

---

## 6. Coba!

Pastikan `php artisan serve` masih jalan, buka http://localhost:8000.

1. Otomatis diarahkan ke `/login`.
2. Login dengan password salah → muncul **"Username atau password salah!"**.
3. Login `admin` / `admin123` → masuk dashboard, menu admin tampil.
4. Klik **Logout**, coba login `petugas` dan `peminjam` → menu yang tampil berbeda.
5. Login sebagai `peminjam`, lalu ketik manual di address bar: http://localhost:8000/admin/user → harus muncul **403**. Artinya middleware bekerja.

---

## ✅ Checklist Bab 06
- [ ] Login gagal → ada notifikasi
- [ ] Login 3 role berhasil, menu berbeda-beda
- [ ] Peminjam membuka `/admin/user` → 403
- [ ] Logout kembali ke halaman login

Lanjut ke **[BAB 07 — CRUD Kategori & Alat](/modul-laravel-peminjaman-alat/bab-07-admin-kategori-alat/)**.
{% endraw %}

---

[← BAB 05 — Model dan Seeder](/modul-laravel-peminjaman-alat/bab-05-model-dan-seeder/) · [Daftar isi](/modul-laravel-peminjaman-alat/) · [BAB 07 — Admin: CRUD Kategori & Alat →](/modul-laravel-peminjaman-alat/bab-07-admin-kategori-alat/)
