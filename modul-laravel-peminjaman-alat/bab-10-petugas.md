---
layout: page
title: "BAB 10 — Fitur Petugas | Modul Laravel 12 CRUD Peminjaman Alat"
description: "Fitur petugas: menyetujui peminjaman dengan DB::beginTransaction, commit dan rollBack, memantau pengembalian yang terlambat, dan mencetak laporan."
permalink: /modul-laravel-peminjaman-alat/bab-10-petugas/
lang: id
---

*Bagian dari seri [Cara Membuat Projek Laravel 12 CRUD Peminjaman Alat](/2026/11/16/cara-membuat-projek-laravel-12-crud-peminjaman-alat/). [Daftar isi](/modul-laravel-peminjaman-alat/) · [Download kode lengkap](/downloads/modulUKK-peminjaman-alat-laravel12.zip)*

{% raw %}
Petugas punya 3 fitur:
1. **Menyetujui / menolak peminjaman** → pakai `DB::beginTransaction()`, `commit()`, `rollBack()`
2. **Memantau pengembalian** → alat yang sedang dipinjam (tandai yang terlambat) + riwayat pengembalian
3. **Mencetak laporan** → filter tanggal + tombol cetak

Semua fitur petugas ditaruh di **satu controller** supaya simpel.

---

## 1. Controller

```bash
php artisan make:controller PetugasController
```

Isi `app/Http/Controllers/PetugasController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Models\LogAktivitas;
use App\Models\Peminjaman;
use App\Models\Pengembalian;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class PetugasController extends Controller
{
    // daftar pengajuan yang menunggu persetujuan
    public function peminjaman()
    {
        $peminjaman = Peminjaman::with('user', 'alat')->where('status', 'menunggu')->oldest()->paginate(10);
        return view('petugas.peminjaman', compact('peminjaman'));
    }

    // menyetujui peminjaman (pakai COMMIT & ROLLBACK)
    public function setujui(Peminjaman $peminjaman)
    {
        DB::beginTransaction();

        try {
            if ($peminjaman->status != 'menunggu') {
                throw new \Exception('Peminjaman ini sudah diproses');
            }

            if ($peminjaman->alat->stok < $peminjaman->jumlah) {
                throw new \Exception('Stok alat tidak cukup');
            }

            // stok alat otomatis berkurang oleh TRIGGER kurangi_stok
            $peminjaman->update([
                'status' => 'disetujui',
                'petugas_id' => auth()->id(),
            ]);
            LogAktivitas::catat('Menyetujui peminjaman #' . $peminjaman->id);

            DB::commit();
            return back()->with('success', 'Peminjaman disetujui');
        } catch (\Exception $e) {
            DB::rollBack();
            return back()->with('error', 'Gagal: ' . $e->getMessage());
        }
    }

    // menolak peminjaman
    public function tolak(Peminjaman $peminjaman)
    {
        $peminjaman->update([
            'status' => 'ditolak',
            'petugas_id' => auth()->id(),
        ]);
        LogAktivitas::catat('Menolak peminjaman #' . $peminjaman->id);

        return back()->with('success', 'Peminjaman ditolak');
    }

    // memantau alat yang sedang dipinjam & yang sudah dikembalikan
    public function pengembalian()
    {
        $dipinjam = Peminjaman::with('user', 'alat')->where('status', 'disetujui')->orderBy('batas_kembali')->get();
        $kembali = Pengembalian::with('peminjaman.user', 'peminjaman.alat')->latest()->paginate(10);

        return view('petugas.pengembalian', compact('dipinjam', 'kembali'));
    }

    // laporan peminjaman berdasarkan tanggal
    public function laporan(Request $request)
    {
        $dari = $request->dari ?? date('Y-m-01');
        $sampai = $request->sampai ?? date('Y-m-d');

        $data = Peminjaman::with('user', 'alat', 'pengembalian')
            ->whereBetween('tanggal_pinjam', [$dari, $sampai])
            ->orderBy('tanggal_pinjam')
            ->get();

        return view('petugas.laporan', compact('data', 'dari', 'sampai'));
    }
}
```

### Penjelasan fungsi `setujui()` (bagian paling penting)

```
DB::beginTransaction();      ← mulai transaksi
try {
    cek status masih "menunggu"?   kalau tidak → lempar error
    cek stok cukup?                kalau tidak → lempar error
    update status = disetujui      ← TRIGGER kurangi_stok jalan otomatis
    catat log
    DB::commit();            ← semua berhasil → simpan permanen
} catch (\Exception $e) {
    DB::rollBack();          ← ada error → batalkan semua
}
```

- `throw new \Exception('...')` → sengaja membuat error supaya langsung lompat ke `catch`.
- `$e->getMessage()` → mengambil pesan error-nya untuk ditampilkan.
- Pengecekan status mencegah **stok berkurang dua kali** kalau tombol Setujui diklik dua kali.

### Fungsi `laporan()`
- `$request->dari ?? date('Y-m-01')` → kalau filter belum diisi, pakai tanggal 1 bulan ini.
- `whereBetween('tanggal_pinjam', [$dari, $sampai])` → ambil data di antara dua tanggal.

---

## 2. View

Buat folder `resources/views/petugas`.

### `resources/views/petugas/peminjaman.blade.php`

```blade
@extends('layout')

@section('content')
<h3>Persetujuan Peminjaman</h3>
<p>Daftar pengajuan peminjaman yang menunggu persetujuan.</p>

<table class="table table-bordered bg-white">
    <tr>
        <th>No</th>
        <th>Peminjam</th>
        <th>Alat</th>
        <th>Jumlah</th>
        <th>Stok Alat</th>
        <th>Tgl Pinjam</th>
        <th>Batas Kembali</th>
        <th>Aksi</th>
    </tr>
    @forelse ($peminjaman as $p)
    <tr>
        <td>{{ $peminjaman->firstItem() + $loop->index }}</td>
        <td>{{ $p->user->name }}</td>
        <td>{{ $p->alat->nama_alat }}</td>
        <td>{{ $p->jumlah }}</td>
        <td>{{ $p->alat->stok }}</td>
        <td>{{ $p->tanggal_pinjam }}</td>
        <td>{{ $p->batas_kembali }}</td>
        <td>
            <form action="{{ route('petugas.setujui', $p->id) }}" method="POST" class="d-inline">
                @csrf
                <button class="btn btn-success btn-sm">Setujui</button>
            </form>
            <form action="{{ route('petugas.tolak', $p->id) }}" method="POST" class="d-inline">
                @csrf
                <button class="btn btn-danger btn-sm" onclick="return confirm('Tolak peminjaman ini?')">Tolak</button>
            </form>
        </td>
    </tr>
    @empty
    <tr>
        <td colspan="8" class="text-center">Tidak ada pengajuan</td>
    </tr>
    @endforelse
</table>

{{ $peminjaman->links('pagination::bootstrap-5') }}
@endsection
```

`@forelse ... @empty ... @endforelse` → seperti `@foreach`, tapi punya bagian `@empty` yang tampil kalau datanya kosong.

### `resources/views/petugas/pengembalian.blade.php`

```blade
@extends('layout')

@section('content')
<h3>Pantau Pengembalian</h3>

<h5 class="mt-4">Alat yang Sedang Dipinjam</h5>
<table class="table table-bordered bg-white">
    <tr>
        <th>Peminjam</th>
        <th>Alat</th>
        <th>Jumlah</th>
        <th>Batas Kembali</th>
        <th>Keterangan</th>
    </tr>
    @forelse ($dipinjam as $p)
    <tr>
        <td>{{ $p->user->name }}</td>
        <td>{{ $p->alat->nama_alat }}</td>
        <td>{{ $p->jumlah }}</td>
        <td>{{ $p->batas_kembali }}</td>
        <td>
            @if ($p->batas_kembali < date('Y-m-d'))
                <span class="badge bg-danger">Terlambat</span>
            @else
                <span class="badge bg-success">Masih dalam waktu</span>
            @endif
        </td>
    </tr>
    @empty
    <tr>
        <td colspan="5" class="text-center">Tidak ada alat yang sedang dipinjam</td>
    </tr>
    @endforelse
</table>

<h5 class="mt-4">Riwayat Pengembalian</h5>
<table class="table table-bordered bg-white">
    <tr>
        <th>Peminjam</th>
        <th>Alat</th>
        <th>Batas Kembali</th>
        <th>Tgl Dikembalikan</th>
        <th>Denda</th>
    </tr>
    @foreach ($kembali as $k)
    <tr>
        <td>{{ $k->peminjaman->user->name }}</td>
        <td>{{ $k->peminjaman->alat->nama_alat }}</td>
        <td>{{ $k->peminjaman->batas_kembali }}</td>
        <td>{{ $k->tanggal_dikembalikan }}</td>
        <td>Rp {{ number_format($k->denda, 0, ',', '.') }}</td>
    </tr>
    @endforeach
</table>

{{ $kembali->links('pagination::bootstrap-5') }}
@endsection
```

`$p->batas_kembali < date('Y-m-d')` → membandingkan tanggal dalam format `YYYY-MM-DD` bisa langsung dengan `<`.

### `resources/views/petugas/laporan.blade.php`

```blade
@extends('layout')

@section('content')
<h3>Laporan Peminjaman Alat</h3>

<form action="{{ route('petugas.laporan') }}" method="GET" class="row g-2 mb-3 no-print">
    <div class="col-auto">
        <input type="date" name="dari" class="form-control" value="{{ $dari }}">
    </div>
    <div class="col-auto">
        <input type="date" name="sampai" class="form-control" value="{{ $sampai }}">
    </div>
    <div class="col-auto">
        <button class="btn btn-primary">Tampilkan</button>
        <button type="button" class="btn btn-success" onclick="window.print()">Cetak</button>
    </div>
</form>

<p>Periode: <b>{{ $dari }}</b> s/d <b>{{ $sampai }}</b></p>

<table class="table table-bordered bg-white">
    <tr>
        <th>No</th>
        <th>Peminjam</th>
        <th>Alat</th>
        <th>Jumlah</th>
        <th>Tgl Pinjam</th>
        <th>Batas Kembali</th>
        <th>Tgl Dikembalikan</th>
        <th>Status</th>
        <th>Denda</th>
    </tr>
    @php $totalDenda = 0; @endphp
    @forelse ($data as $p)
    <tr>
        <td>{{ $loop->iteration }}</td>
        <td>{{ $p->user->name }}</td>
        <td>{{ $p->alat->nama_alat }}</td>
        <td>{{ $p->jumlah }}</td>
        <td>{{ $p->tanggal_pinjam }}</td>
        <td>{{ $p->batas_kembali }}</td>
        <td>{{ $p->pengembalian->tanggal_dikembalikan ?? '-' }}</td>
        <td>{{ $p->status }}</td>
        <td>Rp {{ number_format($p->pengembalian->denda ?? 0, 0, ',', '.') }}</td>
    </tr>
    @php $totalDenda += $p->pengembalian->denda ?? 0; @endphp
    @empty
    <tr>
        <td colspan="9" class="text-center">Tidak ada data</td>
    </tr>
    @endforelse
    <tr>
        <th colspan="8" class="text-end">Total Denda</th>
        <th>Rp {{ number_format($totalDenda, 0, ',', '.') }}</th>
    </tr>
</table>
@endsection
```

- `window.print()` → membuka dialog cetak browser (bisa juga **Save as PDF**).
- Form filter & navbar punya class `no-print` → tidak ikut tercetak.
- `@php ... @endphp` → menulis kode PHP biasa di Blade, di sini untuk menjumlah total denda.
- `$p->pengembalian->denda ?? 0` → kalau belum dikembalikan (pengembalian = null), anggap 0.

---

## 3. Coba

Siapkan data dulu: login **peminjam** belum bisa (BAB 11 belum dibuat), jadi pakai **admin** → menu Peminjaman → tambah 2 peminjaman (status menunggu).

Lalu login **petugas**:
1. **Persetujuan** → klik **Setujui** pada data pertama → pesan sukses, stok alat berkurang.
2. Klik **Tolak** pada data kedua → status ditolak.
3. Coba buat peminjaman dengan jumlah lebih besar dari stok (lewat admin), lalu Setujui → muncul **"Gagal: Stok alat tidak cukup"** (ROLLBACK).
4. **Pantau Pengembalian** → data yang disetujui tampil di "Sedang Dipinjam". Kalau batas kembalinya sudah lewat, ada label merah **Terlambat**.
5. **Laporan** → pilih rentang tanggal → **Tampilkan** → **Cetak**.

---

## ✅ Checklist Bab 10
- [ ] Setujui → stok berkurang, Tolak → status ditolak
- [ ] Stok tidak cukup → gagal (rollback)
- [ ] Pantau pengembalian tampil, ada label terlambat
- [ ] Laporan bisa difilter dan dicetak (navbar tidak ikut tercetak)

Lanjut ke **[BAB 11 — Fitur Peminjam](/modul-laravel-peminjaman-alat/bab-11-peminjam/)**.
{% endraw %}

---

[← BAB 09 — Admin: CRUD Peminjaman, Pengembalian, dan Log Aktivitas](/modul-laravel-peminjaman-alat/bab-09-admin-peminjaman-pengembalian-log/) · [Daftar isi](/modul-laravel-peminjaman-alat/) · [BAB 11 — Fitur Peminjam →](/modul-laravel-peminjaman-alat/bab-11-peminjam/)
