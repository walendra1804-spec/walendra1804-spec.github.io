---
layout: page
title: "BAB 11 — Fitur Peminjam | Modul Laravel 12 CRUD Peminjaman Alat"
description: "Fitur peminjam: mencari alat, mengajukan peminjaman dengan validasi stok, dan mengembalikan alat dengan perhitungan denda otomatis."
permalink: /modul-laravel-peminjaman-alat/bab-11-peminjam/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
Peminjam punya 3 fitur:
1. **Melihat daftar alat** (bisa dicari)
2. **Mengajukan peminjaman**
3. **Mengembalikan alat** → memanggil stored procedure `kembalikan_alat`

---

## 1. Controller

```bash
php artisan make:controller PeminjamController
```

> Hati-hati: **Peminjam**Controller (fitur role peminjam) beda dengan **Peminjaman**Controller (CRUD admin).

Isi `app/Http/Controllers/PeminjamController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Models\Alat;
use App\Models\LogAktivitas;
use App\Models\Peminjaman;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class PeminjamController extends Controller
{
    // melihat daftar alat (bisa dicari)
    public function alat(Request $request)
    {
        $cari = $request->cari;

        $alat = Alat::with('kategori')
            ->where('nama_alat', 'like', '%' . $cari . '%')
            ->paginate(10);

        return view('peminjam.alat', compact('alat', 'cari'));
    }

    // form pengajuan peminjaman
    public function formPinjam(Alat $alat)
    {
        return view('peminjam.pinjam', compact('alat'));
    }

    // menyimpan pengajuan peminjaman
    public function pinjam(Request $request, Alat $alat)
    {
        $request->validate([
            'jumlah' => 'required|integer|min:1|max:' . $alat->stok,
            'batas_kembali' => 'required|date|after_or_equal:today',
        ]);

        Peminjaman::create([
            'user_id' => auth()->id(),
            'alat_id' => $alat->id,
            'jumlah' => $request->jumlah,
            'tanggal_pinjam' => date('Y-m-d'),
            'batas_kembali' => $request->batas_kembali,
            'status' => 'menunggu',
        ]);
        LogAktivitas::catat('Mengajukan peminjaman ' . $alat->nama_alat);

        return redirect()->route('peminjam.riwayat')->with('success', 'Pengajuan berhasil, tunggu persetujuan petugas');
    }

    // riwayat peminjaman milik peminjam yang login
    public function riwayat()
    {
        $peminjaman = Peminjaman::with('alat', 'pengembalian')
            ->where('user_id', auth()->id())
            ->latest()
            ->paginate(10);

        return view('peminjam.riwayat', compact('peminjaman'));
    }

    // mengembalikan alat
    public function kembalikan(Peminjaman $peminjaman)
    {
        // hanya boleh mengembalikan pinjaman milik sendiri yang sedang dipinjam
        if ($peminjaman->user_id != auth()->id() || $peminjaman->status != 'disetujui') {
            return back()->with('error', 'Peminjaman ini tidak bisa dikembalikan');
        }

        // panggil STORED PROCEDURE: simpan pengembalian + hitung denda + ubah status
        DB::statement('CALL kembalikan_alat(?, ?)', [$peminjaman->id, date('Y-m-d')]);
        LogAktivitas::catat('Mengembalikan alat ' . $peminjaman->alat->nama_alat);

        $denda = $peminjaman->pengembalian()->first()->denda;

        return back()->with('success', 'Alat berhasil dikembalikan. Denda: Rp ' . number_format($denda, 0, ',', '.'));
    }
}
```

Penjelasan:
- `where('nama_alat', 'like', '%' . $cari . '%')` → pencarian. Kalau `$cari` kosong, hasilnya `'%%'` = semua alat.
- `'jumlah' => '...|max:' . $alat->stok` → jumlah tidak boleh melebihi stok saat ini.
- `'batas_kembali' => '...|after_or_equal:today'` → tanggal kembali tidak boleh di masa lalu.
- `where('user_id', auth()->id())` → peminjam hanya melihat **miliknya sendiri**.
- Di `kembalikan()`, kita cek dulu pemiliknya & statusnya. Tanpa ini, peminjam lain bisa mengembalikan pinjaman orang lain dengan mengubah id di URL.
- `$peminjaman->pengembalian()->first()` → pakai tanda `()` supaya data diambil **baru** dari database (karena data pengembalian baru saja dibuat oleh procedure).

---

## 2. View

Buat folder `resources/views/peminjam`.

### `resources/views/peminjam/alat.blade.php`

```blade
@extends('layout')

@section('content')
<h3>Daftar Alat</h3>

<form action="{{ route('peminjam.alat') }}" method="GET" class="row g-2 mb-3">
    <div class="col-auto">
        <input type="text" name="cari" class="form-control" placeholder="Cari nama alat..." value="{{ $cari }}">
    </div>
    <div class="col-auto">
        <button class="btn btn-primary">Cari</button>
    </div>
</form>

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
            @if ($a->stok > 0)
                <a href="{{ route('peminjam.form', $a->id) }}" class="btn btn-primary btn-sm">Pinjam</a>
            @else
                <span class="badge bg-secondary">Stok habis</span>
            @endif
        </td>
    </tr>
    @endforeach
</table>

{{ $alat->links('pagination::bootstrap-5') }}
@endsection
```

Form pencarian pakai `method="GET"` → kata kunci muncul di URL (`?cari=bola`), jadi pagination tetap ingat pencariannya.

### `resources/views/peminjam/pinjam.blade.php`

```blade
@extends('layout')

@section('content')
<h3>Ajukan Peminjaman</h3>

<form action="{{ route('peminjam.pinjam', $alat->id) }}" method="POST" class="card card-body">
    @csrf
    <p>
        Alat: <b>{{ $alat->nama_alat }}</b><br>
        Stok tersedia: <b>{{ $alat->stok }}</b>
    </p>
    <div class="mb-3">
        <label class="form-label">Jumlah</label>
        <input type="number" name="jumlah" class="form-control" value="{{ old('jumlah', 1) }}" min="1" max="{{ $alat->stok }}">
    </div>
    <div class="mb-3">
        <label class="form-label">Tanggal Kembali</label>
        <input type="date" name="batas_kembali" class="form-control" value="{{ old('batas_kembali') }}">
    </div>
    <p class="text-muted">Terlambat mengembalikan dikenakan denda Rp 5.000 per hari.</p>
    <div>
        <button class="btn btn-primary">Ajukan</button>
        <a href="{{ route('peminjam.alat') }}" class="btn btn-secondary">Kembali</a>
    </div>
</form>
@endsection
```

### `resources/views/peminjam/riwayat.blade.php`

```blade
@extends('layout')

@section('content')
<h3>Peminjaman Saya</h3>

<table class="table table-bordered bg-white">
    <tr>
        <th>No</th>
        <th>Alat</th>
        <th>Jumlah</th>
        <th>Tgl Pinjam</th>
        <th>Batas Kembali</th>
        <th>Status</th>
        <th>Denda</th>
        <th>Aksi</th>
    </tr>
    @forelse ($peminjaman as $p)
    <tr>
        <td>{{ $peminjaman->firstItem() + $loop->index }}</td>
        <td>{{ $p->alat->nama_alat }}</td>
        <td>{{ $p->jumlah }}</td>
        <td>{{ $p->tanggal_pinjam }}</td>
        <td>{{ $p->batas_kembali }}</td>
        <td><span class="badge bg-secondary">{{ $p->status }}</span></td>
        <td>Rp {{ number_format($p->pengembalian->denda ?? 0, 0, ',', '.') }}</td>
        <td>
            @if ($p->status == 'disetujui')
                <form action="{{ route('peminjam.kembalikan', $p->id) }}" method="POST">
                    @csrf
                    <button class="btn btn-success btn-sm" onclick="return confirm('Kembalikan alat ini?')">Kembalikan</button>
                </form>
            @endif
        </td>
    </tr>
    @empty
    <tr>
        <td colspan="8" class="text-center">Belum ada peminjaman</td>
    </tr>
    @endforelse
</table>

{{ $peminjaman->links('pagination::bootstrap-5') }}
@endsection
```

Tombol **Kembalikan** hanya muncul kalau status `disetujui`.

---

## 3. Coba alur lengkap (3 role)

1. Login **peminjam** → **Daftar Alat** → cari "proyektor" → **Pinjam** → jumlah 2, tanggal kembali 3 hari lagi → **Ajukan**.
   → Masuk ke **Peminjaman Saya** dengan status `menunggu`.
2. Coba ajukan jumlah melebihi stok → muncul error validasi.
3. Logout, login **petugas** → **Persetujuan** → **Setujui**.
4. Logout, login **peminjam** → **Peminjaman Saya** → status `disetujui` → klik **Kembalikan**.
   → Pesan: *"Alat berhasil dikembalikan. Denda: Rp 0"* (karena belum lewat batas).
5. Login **petugas** → **Pantau Pengembalian** → data muncul di Riwayat Pengembalian.
6. Login **admin** → **Log Aktivitas** → semua langkah tercatat.

> **Mau uji denda?** Login admin → edit peminjaman yang sedang `disetujui` → ubah **Tanggal Pinjam** ke 7 hari lalu dan **Batas Kembali** ke 5 hari lalu (status biarkan *Disetujui*). Lalu peminjam klik Kembalikan → denda 5 × 5.000 = **Rp 25.000**.

---

## ✅ Checklist Bab 11
- [ ] Daftar alat + pencarian jalan
- [ ] Pengajuan tersimpan dengan status menunggu
- [ ] Kembalikan → status dikembalikan, stok bertambah, denda tampil

**Selamat, aplikasinya sudah lengkap!** 🎉

Lanjut ke **[BAB 12 — Pengujian & Debugging](/modul-laravel-peminjaman-alat/bab-12-pengujian-debugging/)**.
{% endraw %}

---

[← BAB 10 — Fitur Petugas](/modul-laravel-peminjaman-alat/bab-10-petugas/) · [Daftar isi](/modul-laravel-peminjaman-alat/) · [BAB 12 — Pengujian & Debugging →](/modul-laravel-peminjaman-alat/bab-12-pengujian-debugging/)
