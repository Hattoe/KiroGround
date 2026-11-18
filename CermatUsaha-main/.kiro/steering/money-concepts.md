# Cara CermatUsaha menghitung uang (penjelasan untuk tim)

Dokumen ini menjelaskan, dengan bahasa sederhana, kenapa angka di beberapa layar
terlihat berbeda padahal semuanya **benar**. Tujuannya supaya semua anggota tim
(dan pembaca kode) paham logika uang di aplikasi, tanpa perlu latar belakang akuntansi.

Kalau Anda sedang mengubah kode yang menghitung uang (`src/data/selectors.js`,
`src/data/reports.js`, `src/data/aiContext.js`), baca ini dulu.

---

## Dua cara menghitung "uang masuk" — dan keduanya betul

CermatUsaha membedakan dua hal yang sering dikira sama:

1. **Penjualan (omzet / pendapatan)** — total nilai barang yang terjual, entah
   sudah dibayar penuh atau belum.
2. **Uang masuk (kas)** — uang yang benar-benar diterima dan masuk ke laci/kas.

Contoh paling gampang: Anda menjual nasi kotak senilai **Rp 750.000**, tapi
pembeli baru membayar **Rp 400.000** (sisanya diutang).

- **Penjualan** bertambah **Rp 750.000** (barangnya memang terjual segitu).
- **Uang masuk** hanya bertambah **Rp 400.000** (itu yang benar-benar diterima).

Jadi kalau di satu layar tertulis "penjualan Rp 1,4 juta" dan di layar lain
"uang masuk Rp 1,2 juta", itu **bukan bug**. Yang pertama menghitung nilai
jualan; yang kedua menghitung uang yang sudah di tangan.

### Di data contoh (Warung Bu Sri)

| Angka | Nilai | Artinya |
| --- | --- | --- |
| Penjualan bulan ini | Rp 1.402.000 | Total semua jualan (dibayar + belum) |
| Uang masuk (kas) | Rp 1.202.000 | Uang yang benar-benar diterima |
| Selisih | Rp 350.000 | Masih diutang pembeli (belum dibayar) |

Angka "uang masuk" juga termasuk pemasukan selain jualan (misalnya terima
cicilan Rp 150.000), jadi perhitungannya:
uang jualan yang dibayar (Rp 1.052.000) + terima uang lain (Rp 150.000) = **Rp 1.202.000**.

**Di dalam kode:** nilai penjualan pakai field `total`; uang masuk pakai field
`dibayar` (lihat `arusKas()` di `src/data/selectors.js`). Kalau `dibayar` tidak
diisi, dianggap sama dengan `total` (`t.dibayar ?? t.total`).

---

## "Untung" bukan "uang di kas"

Dua hal ini juga sering tertukar:

- **Laba bersih (untung/rugi)** = penjualan − modal barang yang terjual − biaya.
  Ini jawaban "usaha saya untung atau rugi bulan ini?".
- **Uang di kas** = sisa uang yang benar-benar ada sekarang.

Usaha bisa **rugi di atas kertas tapi kasnya masih tebal** (karena uang kas itu
sisa dari bulan-bulan sebelumnya), atau sebaliknya. Di data contoh: rugi
**Rp 326.000** bulan ini, tapi uang di kas masih **Rp 6.852.000**.

---

## "Modal barang terjual" (HPP) dikunci saat jualan dicatat

Saat sebuah barang **terjual**, aplikasi menyimpan harga belinya **saat itu juga**
ke dalam catatan penjualan. Jadi kalau nanti harga beli barang itu diubah
(misalnya pemasok naikkan harga), laporan laba rugi **bulan-bulan lalu tidak ikut
berubah**. Laporan masa lalu harus tetap, tidak boleh "bergeser" sendiri.

- Penjualan baru menyimpan `hargaBeli` di tiap barisnya.
- Penjualan lama (dari sebelum fitur ini / data contoh) tidak punya itu, jadi
  dipakai harga beli barang yang sekarang sebagai cadangan.
- Lihat `labaRugi()` dan `penjualanItem()` di `src/data/reports.js`.

---

## Belanja stok bukan "biaya"

Saat belanja stok (beli barang dagangan), itu **tidak** langsung dihitung sebagai
biaya/kerugian. Uangnya hanya berubah bentuk: dari kas jadi barang. Barang itu
baru dihitung sebagai "modal barang terjual" (HPP) **ketika barangnya laku**.
Biaya operasional yang dihitung adalah hal seperti gaji, listrik, sewa —
pengeluaran yang memang habis, bukan jadi stok.

---

## Ringkasan cepat

- **Penjualan** = nilai barang terjual (pakai `total`).
- **Uang masuk** = uang yang benar-benar diterima (pakai `dibayar`).
- **Laba** = penjualan − modal terjual − biaya. Beda dengan **uang di kas**.
- **HPP** dikunci saat jualan dicatat, biar laporan lama tidak berubah.
- **Belanja stok** bukan biaya; jadi biaya saat barangnya terjual.

---

## Lampiran: ringkasan perubahan dari versi awal

Catatan untuk tim — apa saja yang sudah diubah dari versi CermatUsaha yang pertama,
dan apa yang sengaja **tidak** diubah. Total: 5 berkas, lewat 2 pull request.

### PR #1 — perbaikan kode (sudah digabung ke `main`)
Branch `fix/hpp-snapshot-and-cleanup`.

1. **`src/components/itemForms.js`** — penjualan kini menyimpan `hargaBeli`
   (modal barang **saat dijual**) di tiap barisnya. Belanja tidak diubah.
2. **`src/data/reports.js`** — HPP di `labaRugi()` dan `penjualanItem()` dibaca dari
   `hargaBeli` yang tersimpan itu (mundur ke harga beli barang sekarang untuk
   transaksi lama). Ini memperbaiki bug: dulu mengubah harga beli ikut mengubah
   laba rugi bulan-bulan lalu.
3. **`src/data/selectors.js`** — menghapus fungsi `seriArusKas()` yang tidak terpakai
   (sudah digantikan `seriArus()`).

### PR #2 — dokumentasi (berkas ini)
Branch `docs/money-concepts`.

4. **`README.md`** — bagian singkat "Cara menghitung uang".
5. **`.kiro/steering/money-concepts.md`** — berkas ini (penjelasan konsep uang +
   ringkasan perubahan).

### Yang sengaja TIDAK diubah
- Backend AI (`server/ai.js`, `api/ai.js`, `vite.config.js`, `.env.example`) —
  rencana pindah dari Gemini ke Kiro CLI sudah diteliti lalu diputuskan **tidak**
  dijalankan. Semua kode AI tetap seperti aslinya.
- Angka di layar chat, Beranda, Riwayat, dan Laporan sudah dicek benar, jadi tidak
  ada yang diperbaiki di sana.
- Ide tambahan (ubah/hapus transaksi, dll.) baru usulan, belum dikerjakan.
