# Profitability Command Segment 3

Dashboard profitabilitas Segmen 3 berbasis HTML dan JavaScript, dengan autentikasi Firebase dan sinkronisasi otomatis dari Google Sheets ke Cloud Firestore.

## Repository dan halaman web

- Repository: https://github.com/govsegment3/dashboard-profitability
- URL GitHub Pages setelah diaktifkan: https://govsegment3.github.io/dashboard-profitability/
- Halaman dashboard: https://govsegment3.github.io/dashboard-profitability/dashboard_profitability_govseg3_2026.html

Status deployment dapat diperiksa melalui tab **Actions** dan **Settings → Pages** pada repository.

## Alur data

1. Pengguna memperbarui spreadsheet sumber.
2. Apps Script membaca data dan membentuk dataset dashboard.
3. Dataset dikirim ke Cloud Firestore jika isinya berubah.
4. Listener Firestore pada web menerima pembaruan.
5. Dashboard memperbarui tampilan tanpa refresh halaman.

### Mekanisme sinkronisasi

- **Edit manual:** trigger langsung mencoba menjalankan sinkronisasi.
- **Pemeriksaan cadangan:** berjalan setiap satu menit untuk memeriksa perubahan, termasuk hasil rumus atau impor.
- **Deteksi perubahan:** hash dihitung dari isi dataset tanpa menyertakan waktu `generatedAt`, sehingga data yang sama tidak dikirim berulang.
- **Pencegahan proses bersamaan:** pengiriman menggunakan script lock.

Pada pengujian lokal, edit manual tampil di dashboard sekitar lima detik. Durasi ini bukan jaminan; waktu aktual bergantung pada antrean Apps Script, ukuran data, jaringan, dan waktu pembaruan rumus atau impor.

Pembaruan dari rumus atau impor baru dapat dibaca setelah hasilnya tersedia di spreadsheet.

## Struktur project

| File | Fungsi |
|---|---|
| `index.html` | Mengalihkan pengunjung ke halaman dashboard |
| `dashboard_profitability_govseg3_2026.html` | Halaman login, visualisasi, dan listener pembaruan Firestore |
| `firebase-config.js` | Konfigurasi aplikasi web Firebase |
| `firestore.rules` | Aturan akses profil pengguna, dataset, dan audit |
| `apps-script/Code.gs` | Pembacaan spreadsheet, pembentukan dataset, dan sinkronisasi |
| `apps-script/appsscript.json` | Manifest Apps Script |
| `firebase.json` | Konfigurasi Firebase CLI |
| `.firebaserc` | Referensi project untuk Firebase CLI |
| `README.md` | Dokumentasi pemasangan dan pengoperasian |

## Persiapan Firebase

Gunakan project Firebase yang dikelola untuk deployment ini. Seluruh konfigurasi web, Apps Script, dan Firebase CLI harus menunjuk ke project tujuan yang sama.

### 1. Aktifkan autentikasi

Pada Firebase Console:

1. Buka **Authentication → Sign-in method**.
2. Aktifkan **Email/Password**.
3. Pada **Authentication → Settings → Authorized domains**, tambahkan hostname deployment:
   `govsegment3.github.io`.

Untuk pengujian lokal, tambahkan hostname lokal jika diperlukan oleh metode autentikasi yang digunakan.

### 2. Siapkan Cloud Firestore

1. Buka **Firestore Database**.
2. Buat database jika belum tersedia.
3. Gunakan **Standard edition** dan database **`(default)`**.
4. Pilih lokasi yang sesuai saat pembuatan.
5. Gunakan **Production mode** jika diminta.
6. Salin isi `firestore.rules` ke tab **Rules**, lalu klik **Publish**.

Jika database sudah tersedia, gunakan database tersebut tanpa membuat duplikat.

Mengunggah `firestore.rules` ke GitHub tidak otomatis memasangnya ke Firebase.

### 3. Siapkan akun pengguna

Project menggunakan akun manusia untuk dashboard dan akun khusus untuk sinkronisasi.

| Jenis akun | Fungsi |
|---|---|
| Director | Membuka dashboard dan mengelola pengguna |
| Sync | Digunakan Apps Script untuk mengirim dataset dan audit |

Akun dibuat pada **Authentication → Users**.

Setiap akun memerlukan profil pada collection `users`, dengan **Document ID sama persis dengan UID akun Authentication**.

| Field | Tipe | Keterangan |
|---|---|---|
| `name` | string | Nama tampilan |
| `email` | string | Email akun terkait |
| `role` | string | Peran yang didukung rules |
| `active` | boolean | `true` untuk akun aktif |

Untuk akun sinkronisasi, gunakan `role` bernilai `Sync`. Untuk administrator awal, gunakan `Director`.

Jangan memakai Auto-ID untuk dokumen profil dan jangan menyimpan `active` sebagai teks `"true"`.

### 4. Konfigurasi aplikasi web

1. Buka **Project settings → General → Your apps**.
2. Pilih aplikasi Web yang tersedia, atau daftarkan aplikasi Web baru.
3. Ambil konfigurasi melalui **SDK setup and configuration → Config**.
4. Salin nilai konfigurasi ke `firebase-config.js`.
5. Pertahankan bentuk `window.FIREBASE_CONFIG` yang digunakan dashboard.

Salin konfigurasi lengkap dari project tujuan. Jangan hanya mengganti `projectId` sementara nilai lain masih berasal dari project berbeda.

Konfigurasi Web App Firebase digunakan di browser dan bukan pengganti kontrol akses. Perlindungan data tetap bergantung pada autentikasi dan Firestore Rules.

## Pemasangan Apps Script

### 1. Sesuaikan kode sumber

Pada `apps-script/Code.gs`:

- Pastikan `CFG.PROJECT_ID` menunjuk ke Firebase tujuan.
- Pastikan `CFG.DATASET_ID` sesuai dengan dokumen yang didengarkan dashboard.
- Pertahankan mekanisme pengiriman langsung saat edit dan pengecekan berkala.

### 2. Pasang pada spreadsheet sumber

1. Buka spreadsheet sumber.
2. Pilih **Extensions → Apps Script**.
3. Ganti seluruh isi `Code.gs` dengan `apps-script/Code.gs` dari repository.
4. Jangan membungkus kode dengan fungsi bawaan `myFunction()`.
5. Untuk manifest, buka **Project Settings** dan aktifkan tampilan `appsscript.json`.
6. Salin isi `apps-script/appsscript.json` ke manifest tersebut.
7. Simpan perubahan.
8. Kembali ke spreadsheet dan refresh.

Menu **Profitability Sync** akan muncul pada menu spreadsheet.

Apps Script ini tidak memerlukan deployment sebagai Web App.

### 3. Atur koneksi Firebase

Pilih **Profitability Sync → 1. Atur koneksi Firebase**, lalu masukkan:

- Web API Key Firebase tujuan.
- Email akun Sync.
- Password Firebase akun Sync.

Konfigurasi disimpan di **Script Properties**, bukan pada file repository.

Mengganti `Code.gs` atau `firebase-config.js` tidak otomatis mengganti koneksi yang sudah tersimpan di Script Properties.

Editor project Apps Script dapat mengakses konfigurasi tersebut. Batasi akses editor dan gunakan akun Sync khusus.

### 4. Uji pengiriman awal

1. Pilih **Profitability Sync → Sinkronkan sekarang**.
2. Pastikan muncul pesan pengiriman berhasil.
3. Periksa dokumen `datasets/segment3-ytd2026` di Firestore.
4. Pastikan `updatedAt` berubah dan `updatedBy` sesuai akun Sync.
5. Periksa log sinkronisasi jika terjadi kegagalan.

Login Firebase berhasil belum membuktikan penulisan dataset berhasil.

### 5. Aktifkan sinkronisasi otomatis

1. Pilih **Profitability Sync → 2. Aktifkan sinkron otomatis**.
2. Buka **Cek status** dan pastikan sinkron otomatis aktif.
3. Periksa trigger pada Apps Script jika diperlukan.

Trigger berjalan menggunakan akun Google pemasangnya. Akun tersebut harus tetap memiliki akses dan otorisasi yang diperlukan.

Hindari trigger duplikat dari beberapa akun. Trigger milik akun lain perlu diperiksa oleh pemiliknya.

## Spreadsheet yang dibaca

| Sheet | Penggunaan |
|---|---|
| `Summary` | Revenue dan direct cost SAP/OPS per pelanggan |
| `Breakdown Pareto` | Rincian kategori biaya dan total kategori |
| `Rekap SAP S3` | Tren bulanan, layanan, depresiasi, dan catatan |
| `Log Sinkronisasi` | Riwayat pengiriman yang ditulis oleh script |

Script membaca data sumber dan menulis log pada `Log Sinkronisasi`. Script tidak menulis ulang data pada tiga sheet sumber tersebut.

Log lama dapat dipangkas untuk membatasi jumlah baris.

Pertahankan nama sheet, susunan kolom, dan format header yang digunakan parser. Perubahan struktur sumber perlu diikuti pemeriksaan kode pembentuk dataset.

Kartu total revenue dihitung dari baris pelanggan pada tabel `Summary`; baris `Grand Total` tidak digunakan sebagai satu-satunya sumber total.

## Menjalankan web secara lokal

1. Buka folder project di VS Code.
2. Pasang ekstensi **Live Server** jika belum tersedia.
3. Klik kanan `dashboard_profitability_govseg3_2026.html`.
4. Pilih **Open with Live Server**.
5. Login menggunakan akun dashboard yang aktif.

Alamat `127.0.0.1` atau `localhost` hanya mengarah ke komputer yang membukanya. Untuk pengujian dari perangkat lain melalui internet, gunakan URL deployment.

## Deployment GitHub Pages

1. Pastikan file web sudah diekstrak dan berada pada folder sumber deployment.
2. Letakkan `index.html`, HTML dashboard, dan `firebase-config.js` pada folder yang sesuai.
3. Commit dan push perubahan ke repository ini.
4. Buka **Settings → Pages**.
5. Pilih sumber deployment yang digunakan, misalnya branch `main` dan folder `/ (root)`.
6. Periksa proses deployment pada tab **Actions**.
7. Buka URL Pages setelah deployment berhasil.

GitHub Pages tidak mengekstrak ZIP untuk menjalankan file di dalamnya.

### Perbedaan pembaruan kode dan data

| Perubahan | Tindakan |
|---|---|
| HTML atau konfigurasi web | Commit, push, dan pastikan deployment selesai |
| `Code.gs` | Perbarui repository serta salin kode ke Apps Script spreadsheet |
| Firestore Rules | Perbarui file dan publish ke Firebase atau deploy melalui CLI |
| Data spreadsheet | Tidak perlu commit atau deployment ulang |

GitHub Pages tidak menjalankan `Code.gs`. Sinkronisasi dijalankan oleh Google Apps Script.

## Pengujian pembaruan otomatis

1. Buka dashboard dengan akun manusia yang aktif.
2. Pastikan menggunakan dataset resmi, bukan upload pribadi.
3. Pilih angka yang bisa dibandingkan dengan spreadsheet.
4. Gunakan pembaruan rutin atau perubahan uji yang diizinkan pengelola.
5. Jangan menimpa rumus atau mengubah data produksi tanpa izin.
6. Biarkan dashboard terbuka tanpa refresh dan tanpa sinkron manual.
7. Periksa log, `updatedAt` Firestore, dan angka terkait pada dashboard.
8. Jika memakai data uji, pulihkan nilai awal dan pastikan pemulihannya ikut tersinkron.

| Hasil | Makna |
|---|---|
| Log `OK` dengan mode `otomatis-edit` | Pengiriman dipicu langsung oleh perubahan |
| Log `OK` dengan mode `otomatis` | Pengiriman dilakukan pemeriksaan berkala |
| Firestore berubah dan web ikut berubah | Alur pembaruan otomatis berhasil |
| Firestore berubah tetapi web tetap | Periksa konfigurasi web, listener, filter, dan mode dataset |
| Log `GAGAL` | Periksa pesan error sebelum menyimpulkan sinkronisasi berhasil |

Pada mode upload pribadi, pembaruan resmi tidak langsung menimpa tampilan. Gunakan pilihan kembali ke dataset resmi pada modul pengelolaan data.

Chip **Live** menunjukkan status listener, bukan bukti bahwa perubahan spreadsheet sudah berhasil dikirim.

## Keamanan dan privasi

Jangan menyimpan atau mengunggah:

- Password akun dashboard atau akun Sync.
- Private key atau kredensial service account.
- Ekspor Script Properties.
- Dataset keuangan, log berisi data internal, atau cadangan spreadsheet.
- ZIP lama yang memuat data atau kredensial.
- Screenshot yang menampilkan password atau informasi internal.

Spreadsheet sumber tetap menggunakan akses **Restricted** sesuai pengelolaan akses organisasi.

Halaman login tidak melindungi data yang sudah tertanam di HTML, ZIP, atau riwayat commit publik. Periksa file dan riwayat repository sebelum publikasi.

Jangan membuka Firestore Rules untuk semua pengguna hanya untuk mengatasi error koneksi.

## Pemecahan masalah

| Gejala | Pemeriksaan |
|---|---|
| `API_KEY_INVALID` | Periksa nilai `FIREBASE_API_KEY` yang tersimpan di Script Properties dan cocokkan dengan Firebase tujuan |
| Email atau password salah | Periksa akun Sync pada Authentication project tujuan dan password Firebase-nya |
| Firestore 403 | Periksa Project ID, UID profil, role, `active`, rules yang dipublikasikan, dan validasi dataset/audit |
| Konfigurasi web tidak ditemukan | Periksa `firebase-config.js`, lokasi file, dan hasil deployment |
| Akun belum aktif | Periksa dokumen `users/{UID}` dan tipe boolean `active` |
| Manual berhasil tetapi otomatis tidak | Periksa trigger, akun pemasang, otorisasi, dan menu Executions |
| Web masih menampilkan data lama | Periksa project Firebase web, filter, mode upload pribadi, dan error browser |
| Tidak ada log baru setiap menit | Normal jika isi dataset tidak berubah; periksa Executions untuk memastikan jadwal berjalan |
| Editor Apps Script terus menampilkan Running | Periksa dialog pada tab spreadsheet yang mungkin menunggu tombol OK |
| Perubahan rumus/impor terlambat | Pastikan hasil sumber sudah diperbarui; perubahan diperiksa melalui jadwal cadangan |

## Pemeliharaan

- Samakan kode dalam repository dengan kode Apps Script yang berjalan.
- Uji ulang setelah perubahan struktur spreadsheet, rules, atau konfigurasi Firebase.
- Periksa kegagalan trigger dan akses akun pemasang secara berkala.
- Gunakan akun dashboard terpisah untuk setiap pengguna sesuai perannya.
- Simpan kredensial di tempat yang dikelola pemilik sistem, bukan dalam dokumentasi repository.
