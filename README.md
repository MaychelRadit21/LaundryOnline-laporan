# Laporan Ujian Tengah Semester

## Pengembangan Aplikasi Berbasis Platform (PBAD) 2026/2027

### Sistem Informasi Rinso Laundry

| Informasi | Keterangan |
|---|---|
| Kelompok | [Isi nama kelompok] |
| Anggota dan NIM | [Isi nama dan NIM setiap anggota] |
| Domain | Laundry: pelanggan, layanan, dan pesanan |
| Repository | [Tautan repository proyek] |
| Tanggal | 9 November 2026 |

> **Catatan penyusunan:** Lengkapi identitas kelompok, sesuaikan uraian pembagian tugas, dan masukkan diagram serta tangkapan layar hasil proyek sebelum laporan dikumpulkan. Jangan mencantumkan hasil pengujian atau bukti yang belum benar-benar dilakukan.

## Daftar Isi

1. [Pendahuluan](#bab-1--pendahuluan)
2. [Analisis Kebutuhan](#bab-2--analisis-kebutuhan)
3. [Arsitektur Sistem dan Teknologi](#bab-3--arsitektur-sistem-dan-teknologi)
4. [Use Case](#bab-4--use-case)
5. [Alur Sistem](#bab-5--alur-sistem)
6. [Perancangan Data](#bab-6--perancangan-data)
7. [Dokumentasi REST API](#bab-7--dokumentasi-rest-api)
8. [Antarmuka Web](#bab-8--antarmuka-web)
9. [Integrasi Web dengan API](#bab-9--integrasi-web-dengan-api)
10. [Pengujian](#bab-10--pengujian)
11. [Pembagian Peran dan Rencana UAS](#bab-11--pembagian-peran-dan-rencana-uas)
12. [Penutup](#bab-12--penutup)

## Bab 1 — Pendahuluan

### 1.1 Latar Belakang

Pada usaha laundry, pencatatan pesanan secara manual dapat menyulitkan pemilik dan kasir. Nota bisa hilang, perhitungan harga dapat keliru, dan pelanggan tidak selalu mengetahui perkembangan cucian. Catatan yang tersebar juga membuat pemilik sulit melihat jumlah pesanan yang sedang dikerjakan maupun pendapatan usaha.

Rinso Laundry dirancang untuk membantu pencatatan pelanggan, layanan, dan pesanan melalui satu REST API dan satu basis data. Pada tahap UTS, fokus pengerjaan adalah Web Client berbasis React dan TypeScript. Web berkomunikasi dengan API Node.js untuk membaca dan mengubah data. Klien mobile dan desktop direncanakan untuk tahap UAS.

### 1.2 Rumusan Masalah

1. Bagaimana menyimpan data pelanggan, layanan, dan pesanan secara terpusat?
2. Bagaimana menghitung biaya pesanan dan memperbarui status cucian secara konsisten?
3. Bagaimana menampilkan informasi operasional yang dibutuhkan pemilik dan kasir?
4. Bagaimana menyediakan API yang dapat digunakan oleh klien web, mobile, dan desktop?

### 1.3 Tujuan

- Membuat REST API untuk mengelola pelanggan, layanan, dan pesanan.
- Membuat Web Client React TypeScript yang terhubung dengan API.
- Mendokumentasikan kebutuhan, rancangan sistem, API, dan pengujian.

### 1.4 Ruang Lingkup

| Termasuk dalam UTS | Rencana untuk UAS | Di luar cakupan UTS |
|---|---|---|
| API server, basis data SQLite, Web Client (dashboard, pesanan, pelanggan, layanan), dokumentasi, dan pengujian API | Klien mobile React Native TypeScript dan desktop C# WinForms yang menggunakan API yang sama | Autentikasi dan otorisasi, pembayaran daring, serta notifikasi WhatsApp/SMS |

## Bab 2 — Analisis Kebutuhan

### 2.1 Aktor

| Aktor | Kegiatan |
|---|---|
| Admin atau kasir | Mengelola pelanggan dan layanan, membuat pesanan, memperbarui status, dan mencatat pembayaran. |
| Pemilik | Memantau ringkasan usaha dan mengelola tarif layanan. |

### 2.2 Kebutuhan Fungsional

| Kode | Kebutuhan | Aktor |
|---|---|---|
| F-01 | Menambah, melihat, mengubah, dan menghapus pelanggan; daftar mendukung pencarian dan pagination. | Admin |
| F-02 | Mengelola layanan, termasuk tarif, satuan (kg/pcs), estimasi hari, dan status aktif. | Admin, pemilik |
| F-03 | Membuat pesanan dengan kode dan total harga yang dihitung oleh server. | Admin |
| F-04 | Memperbarui status pesanan mengikuti alur yang ditentukan atau membatalkan pesanan. | Admin |
| F-05 | Mencatat apakah pesanan sudah dibayar. | Admin |
| F-06 | Mencari dan memfilter pesanan berdasarkan kata kunci, status, dan pembayaran. | Admin |
| F-07 | Menampilkan ringkasan pesanan, pendapatan, grafik tujuh hari, dan layanan terlaris. | Pemilik, admin |

### 2.3 Kebutuhan Nonfungsional

- **Terpusat:** semua klien menggunakan basis data yang sama melalui API.
- **Konsisten:** respons API menggunakan JSON dengan bentuk umum `success`, `data`, `meta`, dan `error`.
- **Validasi:** server memvalidasi masukan dan mengembalikan rincian kesalahan per kolom.
- **Integritas:** relasi antar data dijaga dengan foreign key; data yang masih digunakan tidak boleh dihapus.
- **Kemudahan penggunaan:** web menyediakan tampilan responsif dan keadaan loading, error, serta kosong pada daftar.

### 2.4 Aturan Bisnis

| Kode | Aturan |
|---|---|
| B-01 | Nomor telepon pelanggan dan nama layanan harus unik. |
| B-02 | Total harga dihitung dari tarif per satuan dikalikan jumlah. Tarif disimpan pada pesanan agar perubahan harga layanan tidak mengubah pesanan lama. |
| B-03 | Status pesanan hanya bergerak satu tahap dalam alur: Diterima → Dicuci → Disetrika → Selesai → Diambil. Pesanan dapat dibatalkan sebelum mencapai status Selesai. |
| B-04 | Pesanan berstatus Diambil atau Dibatalkan tidak dapat diubah. |
| B-05 | Pesanan hanya dapat dihapus jika berstatus Diterima atau Dibatalkan. |
| B-06 | Pelanggan atau layanan yang sudah digunakan pesanan tidak dapat dihapus. Layanan yang tidak digunakan lagi dapat dinonaktifkan. |
| B-07 | Layanan nonaktif tidak dapat dipilih untuk pesanan baru. |
| B-08 | Kode pesanan mengikuti format `LDR-YYYYMMDD-NNN`, dengan nomor urut harian. |

## Bab 3 — Arsitektur Sistem dan Teknologi

Sistem menggunakan pola klien–server. Web mengirim permintaan HTTP dengan data JSON ke REST API. API membaca dan menulis data pada basis data SQLite. Dalam rencana pengembangan berikutnya, klien mobile dan desktop juga akan menggunakan API yang sama.

**Gambar 3.1 — Arsitektur sistem**

`[Sisipkan diagram arsitektur. Tampilkan Web sebagai klien UTS dan Mobile/Desktop sebagai rencana UAS.]`

### 3.1 Teknologi

| Lapisan | Teknologi |
|---|---|
| Web Client | React 18, TypeScript, Vite, React Router, CSS |
| API Server | Node.js (≥ 22.13), Express 4, CORS |
| Basis data | SQLite melalui `node:sqlite` |
| Pengujian API | `node:test` |
| Manajemen paket | npm workspaces (`server`, `web`) |

### 3.2 Struktur Proyek

```text
server/
  src/index.js        titik masuk server
  src/app.js          konfigurasi Express, router, penanganan error
  src/db.js           skema dan koneksi SQLite
  src/seed.js         data awal
  src/http.js         validator, paging, dan ApiError
  src/routes/         route pelanggan, layanan, pesanan, dan statistik
  test/api.test.js    pengujian integrasi API
web/
  src/lib/            klien API, tipe data, hooks, dan format
  src/pages/          Dashboard, Pesanan, Pelanggan, dan Layanan
  src/components/     komponen antarmuka bersama
docs/                 dokumentasi dan bahan laporan
```

## Bab 4 — Use Case

### 4.1 Diagram Use Case

**Gambar 4.1 — Use case sistem Rinso Laundry**

`[Sisipkan diagram use case beserta aktor Admin/Kasir dan Pemilik.]`

### 4.2 Ringkasan Spesifikasi

| Use case | Aktor | Alur utama | Kondisi alternatif |
|---|---|---|---|
| UC-01 Kelola pelanggan | Admin | Melihat daftar, menambah atau mengubah data, dan menghapus pelanggan setelah konfirmasi. | Telepon tidak valid atau duplikat menghasilkan kesalahan. Pelanggan yang memiliki pesanan tidak dapat dihapus. |
| UC-02 Kelola layanan | Admin, pemilik | Menambah dan mengubah layanan, serta mengaktifkan atau menonaktifkannya. | Nama duplikat ditolak. Layanan yang dipakai pesanan tidak dapat dihapus. |
| UC-03 Buat pesanan | Admin | Memilih pelanggan dan layanan aktif, mengisi jumlah, lalu menyimpan pesanan. | Masukan tidak valid ditolak; server menghitung kode dan total harga. |
| UC-04 Ubah status | Admin | Membuka detail pesanan dan memilih tahap berikutnya. | Transisi yang melompati tahap ditolak. Pembatalan hanya diizinkan sebelum Selesai. |
| UC-05 Catat pembayaran | Admin | Menandai pesanan lunas atau belum dibayar. | Pesanan yang dibatalkan tidak dapat ditandai lunas. |
| UC-06 Cari pesanan | Admin | Menggunakan pencarian, filter, dan pagination pada daftar pesanan. | Jika tidak ada hasil, web menampilkan keadaan kosong. |
| UC-07 Lihat dashboard | Pemilik, admin | Melihat ringkasan statistik, pesanan terbaru, grafik, dan layanan teratas. | Jika API tidak dapat dihubungi, web menampilkan pesan error dan opsi untuk mencoba lagi. |
| UC-08 Ubah atau hapus pesanan | Admin | Mengubah pesanan yang masih dapat diedit atau menghapusnya setelah konfirmasi. | Pesanan Diambil atau Dibatalkan tidak dapat diubah; penghapusan dibatasi sesuai status. |

Endpoint terkait masing-masing use case dirinci pada Bab 7.

## Bab 5 — Alur Sistem

Diagram berikut perlu disesuaikan dengan perilaku implementasi aktual.

- **Gambar 5.1 — Flowchart pembuatan pesanan:** `[Sisipkan flowchart.]`
- **Gambar 5.2 — Flowchart pembaruan status:** `[Sisipkan flowchart.]`
- **Gambar 5.3 — Sequence diagram pembuatan pesanan (POST):** `[Sisipkan sequence diagram.]`
- **Gambar 5.4 — Sequence diagram pemuatan dashboard (GET):** `[Sisipkan sequence diagram.]`

## Bab 6 — Perancangan Data

### 6.1 ERD

**Gambar 6.1 — ERD sistem laundry**

`[Sisipkan ERD. Tunjukkan bahwa satu pelanggan dan satu layanan dapat terkait dengan banyak pesanan.]`

### 6.2 Kamus Data

#### Tabel `pelanggan`

| Kolom | Tipe | Kendala | Keterangan |
|---|---|---|---|
| `id` | INTEGER | Primary key, auto-increment | Identitas pelanggan |
| `nama` | TEXT | NOT NULL | Nama lengkap, 2–80 karakter |
| `telepon` | TEXT | NOT NULL, UNIQUE | Nomor telepon Indonesia |
| `email` | TEXT | Opsional | Alamat email |
| `alamat` | TEXT | Opsional | Maksimal 200 karakter |
| `created_at` | TEXT | Default waktu pembuatan | Waktu pendaftaran |

#### Tabel `layanan`

| Kolom | Tipe | Kendala | Keterangan |
|---|---|---|---|
| `id` | INTEGER | Primary key, auto-increment | Identitas layanan |
| `nama` | TEXT | NOT NULL, UNIQUE | Nama layanan |
| `deskripsi` | TEXT | Opsional | Keterangan layanan |
| `harga` | INTEGER | NOT NULL, ≥ 0 | Tarif per satuan dalam rupiah |
| `satuan` | TEXT | `kg` atau `pcs` | Satuan tarif |
| `estimasi_hari` | INTEGER | NOT NULL, ≥ 0 | Estimasi pengerjaan |
| `aktif` | INTEGER | Default 1 | Penanda layanan dapat dipilih |

#### Tabel `pesanan`

| Kolom | Tipe | Kendala | Keterangan |
|---|---|---|---|
| `id` | INTEGER | Primary key, auto-increment | Identitas pesanan |
| `kode` | TEXT | NOT NULL, UNIQUE | Kode pesanan harian |
| `pelanggan_id` | INTEGER | Foreign key ke `pelanggan(id)`, RESTRICT | Pelanggan pemilik pesanan |
| `layanan_id` | INTEGER | Foreign key ke `layanan(id)`, RESTRICT | Layanan yang dipilih |
| `jumlah` | REAL | Lebih dari 0 | Berat atau jumlah barang |
| `harga_satuan` | INTEGER | NOT NULL | Tarif saat pesanan dibuat |
| `total_harga` | INTEGER | NOT NULL | Total harga setelah pembulatan |
| `status` | TEXT | Salah satu dari enam status | Status pengerjaan |
| `dibayar` | INTEGER | Default 0 | 1 jika lunas |
| `catatan` | TEXT | Opsional | Catatan, maksimal 300 karakter |
| `tanggal_masuk`, `tanggal_estimasi` | TEXT | NOT NULL | Tanggal dalam format `YYYY-MM-DD` |
| `created_at`, `updated_at` | TEXT | Waktu pembuatan/perubahan | Jejak waktu |

### 6.3 State Diagram

**Gambar 6.2 — State diagram status pesanan**

`[Sisipkan diagram status: Diterima → Dicuci → Disetrika → Selesai → Diambil, serta jalur pembatalan yang sesuai aturan.]`

## Bab 7 — Dokumentasi REST API

**Base URL:** `http://localhost:3001/api`

**Format:** JSON

**Autentikasi:** belum diterapkan pada tahap UTS.

### 7.1 Format Respons

Contoh respons berhasil:

```json
{
  "success": true,
  "data": {},
  "meta": {
    "page": 1,
    "limit": 10,
    "total": 25,
    "totalPages": 3
  }
}
```

Contoh respons gagal:

```json
{
  "success": false,
  "error": {
    "message": "Validasi gagal",
    "details": {
      "nama": "Nama wajib diisi"
    }
  }
}
```

| Kode | Keterangan |
|---|---|
| 200 / 201 | Permintaan berhasil / data berhasil dibuat |
| 400 | Permintaan tidak valid, misalnya ID bukan angka atau JSON rusak |
| 404 | Data atau endpoint tidak ditemukan |
| 409 | Konflik, misalnya data duplikat atau aturan bisnis dilanggar |
| 422 | Validasi gagal; rincian tersedia pada `error.details` |
| 500 | Kesalahan server |

### 7.2 Endpoint umum dan statistik

| Method | Endpoint | Keterangan |
|---|---|---|
| GET | `/health` | Pemeriksaan kesehatan server |
| GET | `/stats` | Statistik pelanggan, layanan, pesanan, pendapatan, pembayaran, status, layanan teratas, dan data harian |

### 7.3 Endpoint pelanggan

| Method | Endpoint | Keterangan |
|---|---|---|
| GET | `/pelanggan?search=&page=&limit=` | Daftar pelanggan dengan pencarian dan pagination |
| GET | `/pelanggan/:id` | Detail pelanggan |
| POST | `/pelanggan` | Membuat pelanggan (`nama`, `telepon` wajib) |
| PUT | `/pelanggan/:id` | Memperbarui pelanggan |
| DELETE | `/pelanggan/:id` | Menghapus pelanggan jika tidak memiliki pesanan |

### 7.4 Endpoint layanan

| Method | Endpoint | Keterangan |
|---|---|---|
| GET | `/layanan?search=&aktif=1&page=&limit=` | Daftar layanan |
| GET | `/layanan/:id` | Detail layanan |
| POST | `/layanan` | Membuat layanan (`nama`, `harga`, `satuan`, `estimasi_hari` wajib) |
| PUT | `/layanan/:id` | Memperbarui layanan |
| DELETE | `/layanan/:id` | Menghapus layanan yang belum dipakai pesanan |

### 7.5 Endpoint pesanan

| Method | Endpoint | Keterangan |
|---|---|---|
| GET | `/pesanan?search=&status=&dibayar=&pelanggan_id=&page=&limit=` | Daftar pesanan dengan filter dan pagination |
| GET | `/pesanan/:id` | Detail pesanan beserta status berikutnya |
| POST | `/pesanan` | Membuat pesanan; kode dan total dihitung server |
| PUT | `/pesanan/:id` | Memperbarui pesanan yang masih dapat diubah |
| PATCH | `/pesanan/:id/status` | Memperbarui status pesanan |
| PATCH | `/pesanan/:id/bayar` | Memperbarui status pembayaran |
| DELETE | `/pesanan/:id` | Menghapus pesanan jika status mengizinkan |

Tanda `*` pada request body menandakan kolom wajib. Untuk operasi tulis, kegagalan validasi mengembalikan 422, sedangkan pelanggaran aturan bisnis mengembalikan 409.

### 7.6 Contoh pembuatan pesanan

Permintaan `POST /api/pesanan`:

```json
{
  "pelanggan_id": 1,
  "layanan_id": 2,
  "jumlah": 3,
  "catatan": "Pisahkan baju putih"
}
```

Contoh respons `201 Created`:

```json
{
  "success": true,
  "data": {
    "id": 10,
    "kode": "LDR-20261109-003",
    "pelanggan_nama": "Budi Santoso",
    "layanan_nama": "Cuci Kilat",
    "jumlah": 3,
    "harga_satuan": 10000,
    "total_harga": 30000,
    "status": "Diterima",
    "dibayar": 0,
    "tanggal_masuk": "2026-11-09",
    "tanggal_estimasi": "2026-11-10"
  }
}
```

## Bab 8 — Antarmuka Web

### 8.1 Halaman

| Halaman | Rute | Fitur |
|---|---|---|
| Dashboard | `/` | Kartu ringkasan, grafik tujuh hari, distribusi status, pesanan terbaru, dan layanan teratas |
| Pesanan | `/pesanan` | Pencarian, filter, pagination, form pesanan, detail, dan perubahan status |
| Pelanggan | `/pelanggan` | Daftar, pencarian, pagination, tambah/ubah, dan hapus dengan konfirmasi |
| Layanan | `/layanan` | Daftar layanan, tarif dan estimasi, tambah/ubah, aktivasi, dan penghapusan |

### 8.2 Prinsip antarmuka

- Navigasi menampilkan indikator koneksi API.
- Tema mengikuti pengaturan terang atau gelap perangkat.
- Daftar menyediakan tampilan loading, error dengan opsi coba lagi, dan keadaan kosong.
- Kesalahan validasi ditampilkan pada kolom terkait; hasil aksi disampaikan melalui notifikasi.
- Warna badge membantu membedakan status pesanan.

### 8.3 Tangkapan layar

- **Gambar 8.1 — Dashboard:** `[Masukkan tangkapan layar dari aplikasi.]`
- **Gambar 8.2 — Pesanan:** `[Masukkan tangkapan layar yang menunjukkan pencarian, filter, status, dan pagination.]`
- **Gambar 8.3 — Pelanggan:** `[Masukkan tangkapan layar dari aplikasi.]`
- **Gambar 8.4 — Layanan:** `[Masukkan tangkapan layar dari aplikasi.]`

## Bab 9 — Integrasi Web dengan API

### 9.1 Mekanisme

Modul `web/src/lib/api.ts` menjadi tempat pemanggilan API dengan `fetch` dan tipe TypeScript. Hook `useAsync` mengelola data, loading, dan error untuk permintaan GET. Form web mengirim operasi POST, PUT, PATCH, atau DELETE; setelah penyimpanan berhasil, daftar dimuat kembali. Pencarian menggunakan jeda (debounce) 300 ms.

Dalam pengembangan, Vite meneruskan permintaan `/api` ke `http://127.0.0.1:3001`. Pada mode produksi, Express menyajikan hasil build web dan API.

### 9.2 Pemetaan fitur dan endpoint

| Aksi web | Method | Endpoint |
|---|---|---|
| Memuat dashboard | GET | `/stats`, `/pesanan?limit=5` |
| Melihat dan mencari pesanan | GET | `/pesanan?search=&status=&dibayar=&page=` |
| Melihat detail pesanan | GET | `/pesanan/:id` |
| Membuat pesanan | POST | `/pesanan` |
| Mengubah pesanan | PUT | `/pesanan/:id` |
| Mengubah status atau pembayaran | PATCH | `/pesanan/:id/status`, `/pesanan/:id/bayar` |
| Menghapus pesanan | DELETE | `/pesanan/:id` |
| Mengelola pelanggan | GET, POST, PUT, DELETE | `/pelanggan`, `/pelanggan/:id` |
| Mengelola layanan | GET, POST, PUT, DELETE | `/layanan`, `/layanan/:id` |

Untuk dokumentasi laporan, tunjukkan satu contoh alur yang benar-benar dijalankan: data dibuat dari web, tersimpan di SQLite, lalu muncul kembali setelah diambil dari API.

## Bab 10 — Pengujian

### 10.1 Pengujian otomatis API

Perintah `npm test` digunakan untuk menjalankan pengujian integrasi API. Rencana cakupan yang diberikan untuk proyek meliputi:

| No. | Skenario |
|---|---|
| 1 | Health check |
| 2 | CRUD pelanggan, validasi, duplikasi telepon, dan pencarian |
| 3 | Penolakan penghapusan pelanggan yang memiliki pesanan |
| 4 | Pembuatan, perubahan, penonaktifan, dan penghapusan layanan; penolakan layanan nonaktif untuk pesanan baru |
| 5 | Perhitungan harga, validasi transisi status, dan pembatasan perubahan/penghapusan pesanan |
| 6 | Validasi referensi pesanan, perubahan jumlah, filter status, dan penghapusan |
| 7 | Statistik, endpoint yang tidak ditemukan, ID tidak valid, dan JSON rusak |

**Hasil pengujian:** `[Isi berdasarkan keluaran terbaru dari npm test, termasuk jumlah tes yang lulus dan gagal.]`

### 10.2 Pengujian manual web

Catat hasil berdasarkan pengujian yang dilakukan pada aplikasi:

| No. | Skenario | Hasil |
|---|---|---|
| 1 | Dashboard menampilkan statistik dan status koneksi API. | [Isi hasil] |
| 2 | Pesanan baru dapat dibuat dan muncul pada daftar. | [Isi hasil] |
| 3 | Form kosong menampilkan kesalahan pada kolom yang sesuai. | [Isi hasil] |
| 4 | Status pesanan dapat diperbarui dan pembayaran dapat dicatat. | [Isi hasil] |
| 5 | Pencarian, filter, dan pagination berfungsi. | [Isi hasil] |
| 6 | Saat API tidak tersedia, web menampilkan error dan opsi coba lagi. | [Isi hasil] |

### 10.3 Build

Perintah `npm run build` ditujukan untuk pemeriksaan tipe TypeScript dan build produksi Vite.

**Hasil build:** `[Isi berdasarkan keluaran terbaru dari npm run build.]`

## Bab 11 — Pembagian Peran dan Rencana UAS

### 11.1 Pembagian peran

| Nama | NIM | Tanggung jawab |
|---|---|---|
| [Nama anggota 1] | [NIM] | [Tugas yang benar-benar dikerjakan] |
| [Nama anggota 2] | [NIM] | [Tugas yang benar-benar dikerjakan] |
| [Nama anggota 3] | [NIM] | [Tugas yang benar-benar dikerjakan] |

Sesuaikan tabel dengan jumlah anggota kelompok (3–6 orang) dan kontribusi masing-masing.

### 11.2 Manajemen Git

Cantumkan repository yang digunakan kelompok dan jelaskan cara kerja kolaborasi sesuai riwayat commit yang tersedia. Pastikan tautan repository dan kontribusi anggota di bagian ini sesuai keadaan sebenarnya.

### 11.3 Rencana UAS

- Membuat klien mobile dengan React Native TypeScript dan klien desktop dengan C# WinForms.
- Menghubungkan kedua klien ke API dan basis data yang sama.
- Mempertimbangkan autentikasi JWT, laporan berdasarkan periode, dan notifikasi pesanan selesai.
- Melengkapi dokumentasi integrasi dan menyiapkan demo aplikasi.

## Bab 12 — Penutup

### 12.1 Kesimpulan

Rinso Laundry dirancang untuk memusatkan pengelolaan pelanggan, layanan, dan pesanan melalui REST API serta menyediakan Web Client untuk kebutuhan operasional. Pada tahap UTS, pekerjaan berfokus pada hubungan antara web dan API. Kesimpulan akhir mengenai fitur yang telah berjalan perlu disesuaikan dengan hasil implementasi dan pengujian kelompok.

### 12.2 Saran

- Terapkan autentikasi dan otorisasi sebelum aplikasi digunakan pada jaringan publik.
- Pertimbangkan basis data server jika kebutuhan akses bersama dan skala penggunaan meningkat.
- Tambahkan laporan dan fitur ekspor sesuai kebutuhan usaha.

### 12.3 Cara Menjalankan

```sh
npm install
npm run dev   # API di port 3001 dan web di http://127.0.0.1:5173
npm test      # pengujian API
npm start     # mode produksi; buka http://localhost:3001
```
