# Product Requirements Document (PRD)
## Smart Application Education Platform (SAE SMP)
### Sistem Informasi Administrasi & Akademik SMP Multi-Tenant Berbasis Laravel

- **Nama Platform**: Smart Application Education Platform (SAE SMP)
- **Versi Dokumen**: 1.1.0
- **Tanggal Revisi**: 2026-10-20
- **Status**: Draft Aktif — Sedang Dikembangkan
- **Tingkat Pendidikan**: Sekolah Menengah Pertama (SMP)
- **Model Lisensi / Distribusi**: Multi-Tenant Web Platform (SaaS-Ready Architecture)
- **Target Pengguna**: Banyak Satuan Pendidikan SMP (Negeri & Swasta)
- **Teknologi Utama**: Laravel (Backend API & Web), MySQL / MariaDB, Tailwind CSS, Alpine.js / Livewire, PWA (Progressive Web App)
- **Dokumen Terkait**:
  - [docs/DATABASE.md](docs/DATABASE.md)
  - [docs/USER_ROLES.md](docs/USER_ROLES.md)
  - [docs/USER_FLOW.md](docs/USER_FLOW.md)
  - [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)
  - [docs/SECURITY.md](docs/SECURITY.md)
  - [docs/ROADMAP.md](docs/ROADMAP.md)

---

## 1. Ringkasan Eksekutif & Latar Belakang

### 1.1 Visi Produk
Smart Application Education Platform (SAE SMP) adalah platform administrasi dan manajemen pendidikan terintegrasi yang dirancang untuk mendukung operasional multi-sekolah (multi-tenant) pada jenjang Sekolah Menengah Pertama (SMP). Platform ini menyediakan ekosistem digital komprehensif yang mengisolasi data tiap sekolah secara aman, memvalidasi presensi siswa secara presisi dengan verifikasi QR Code dan checkpoint gerbang serta jam pelajaran terpisah, mendesentralisasi pencatatan melalui delegasi pengurus kelas, memberikan transparansi penuh kepada orang tua melalui aplikasi PWA dengan pusat riwayat notifikasi internal, serta menyinkronkan data secara resmi dengan sistem Dapodik Kemendikbudristek.

### 1.2 Masalah Pokok yang Diselesaikan
1. **Ketiadaan Platform Multi-Sekolah Terstandar**: Banyak SMP mengoperasikan sistem terpisah dengan kualitas keamanan dan pemeliharaan yang tidak merata. SAE SMP menyediakan satu basis kode yang efisien dan dapat melayani banyak sekolah dengan isolasi data yang ketat.
2. **Kelemahan Presensi Konvensional**: Presensi manual rawan titip absen, kehilangan berkas, dan memakan waktu KBM hingga 10-15 menit per sesi. Presensi gerbang harian sering kali disamakan dengan presensi kelas, padahal siswa bisa saja masuk gerbang namun membolos di mata pelajaran tertentu.
3. **Penghapusan Riwayat Transaksi Presensi**: Koreksi kehadiran yang dilakukan wali kelas atau guru piket sering kali menimpa (*overwrite*) data mentah, menghilangkan transparansi bila terjadi sengketa (*dispute*) kehadiran atau kecelakaan siswa di luar sekolah.
4. **Beban Administratif Guru Kelas**: Guru mapel dan wali kelas sering terbebani pekerjaan klerikal pencatatan jurnal kelas yang sebenarnya dapat didraf terlebih dahulu oleh pengurus kelas (sekretaris/ketua kelas) untuk kemudian diverifikasi guru.
5. **Keterasingan Orang Tua**: Notifikasi WhatsApp eksternal sering kali terhapus atau tertimbun obrolan pribadi. Orang tua memerlukan portal PWA mandiri dengan kotak masuk (*inbox*) riwayat notifikasi internal yang tidak dapat dimanipulasi atau hilang.
6. **Redundansi Input Data Dapodik**: Operator sekolah sering harus melakukan entri ganda antara aplikasi lokal sekolah dan aplikasi Dapodik Kemendikbudristek.

---

## 2. Tujuan Strategis & Kriteria Keberhasilan (Goals & Metrics)

### 2.1 Tujuan Produk
1. **Multi-Tenant Data Isolation**: Menjamin 100% isolasi data antar-sekolah pada tingkat basis data dan query level (Row-Level Tenancy Scoping).
2. **Dual-Layer Attendance Precision**: Memisahkan presensi gerbang (Check-In & Check-Out harian) dan presensi KBM per mata pelajaran, dilengkapi mekanisme kartu pelajar berbasis QR Code.
3. **Auditability & Immutable Logs**: Setiap perubahan status presensi manual oleh wali kelas/piket terekam sebagai transaksi log tanpa menghapus data pindaian awal (*append-only logs*).
4. **Delegated Student Empowerment**: Mengakomodasi peran Pengurus Kelas (Sekretaris & Ketua Kelas) untuk menginput draf kehadiran dan jurnal KBM kelas dengan hak akses terbatas.
5. **Parent Engagement via PWA**: Menyediakan portal orang tua berbasis PWA dengan riwayat notifikasi persisten di dalam sistem.
6. **Official One-Way Dapodik Sync**: Memfasilitasi integrasi satu arah untuk menarik data master resmi dari server Dapodik lokal / Web Service Dapodik ke SAE SMP.

### 2.2 Metrik Kunci Keberhasilan (Success KPIs)
- Tingkat kebocoran data antar-tenant: `0%` (Zero cross-tenant data leakage).
- Waktu pemindaian presensi QR Code per siswa: `< 1 detik` di checkpoint gerbang.
- Pengurangan beban entri jurnal kelas guru: minimal `60%` melalui bantuan draf sekretaris kelas.
- Kecepatan pemutakhiran data semester melalui sinkronisasi Dapodik: `< 15 menit` per sekolah.
- Ketersediaan jejak audit data (*audit trail coverage*): `100%` untuk perubahan nilai, presensi, mutasi, dan pembayaran.

---

## 3. Arsitektur Multi-Tenant & Model Isolasi Data

### 3.1 Pendekatan Tenancy
Platform menggunakan pendekatan **Shared Database, Shared Schema with Discriminator Column (`school_id`)** yang dikombinasikan dengan:
- **Global Query Scope Laravel**: Setiap model Eloquent yang berhubungan dengan data operasional sekolah secara otomatis menerapkan `SchoolScope` yang memfilter baris berdasarkan `school_id` sekolah aktif.
- **Tenant Identification Middleware**: Identifikasi sekolah dilakukan melalui subdomain (misal: `smpn1.sae-smp.id`), custom domain (misal: `sim.smpn1contoh.sch.id`), atau header token API.
- **Tenant Context Manager**: Singletons pada container Laravel yang menampung model `School` aktif selama siklus hidup HTTP Request atau Queue Job.
- **Cross-Tenant Prevention**: Kebijakan Foreign Key di tingkat database yang mengunci integritas relasi antar-tabel dalam tenant yang sama (`school_id` composite index & constraints).

### 3.2 Isolasi Pengguna & Akses Antar-Sekolah
- Pengguna (seperti Guru atau Siswa) terikat pada satu atau lebih sekolah melalui tabel pivot `school_user`, sehingga guru yang mengajar di lebih dari satu SMP dalam konsorsium dapat berpindah konteks (*switch tenant*) secara aman tanpa kebocoran hak akses.
- Super Admin Platform memiliki izin khusus untuk mengawasi status layanan seluruh sekolah tanpa mencampurkan data transaksional akademik antar-sekolah.

---

## 4. Persona Pengguna & Struktur Hak Akses (RBAC)

| Peran Pengguna | Lingkup Akses | Deskripsi & Tanggung Jawab Utama |
| :--- | :--- | :--- |
| **Super Admin Platform** | Global / Multi-Tenant | Mengelola lisensi sekolah, provisioning tenant baru, pemantauan sistem, dan log infrastruktur. |
| **Admin Sekolah** | Tenant Tunggal | Administrator lokal sekolah: kelola master data, integrasi Dapodik, konfigurasi jam sekolah & QR. |
| **Kepala Sekolah** | Tenant Tunggal | Supervisi dashboard analitik, monitoring disiplin & akademik, pengesahan rapor dan kenaikan kelas. |
| **Waka Kurikulum** | Tenant Tunggal | Manajemen tahun ajaran, plotting mata pelajaran, penugasan guru, jadwal KBM, kalender akademik. |
| **Waka Kesiswaan** | Tenant Tunggal | Administrasi siswa, rombel, rekapitulasi poin pelanggaran, prestasi, dan pembinaan karakter. |
| **Guru Mata Pelajaran**| Rombel & Mapel Terkait | Presensi KBM per mapel, input Tujuan Pembelajaran (TP), asesmen formatif/sumatif, verifikasi jurnal. |
| **Wali Kelas** | Rombel Binaan | Rekap presensi harian kelas, koreksi presensi manual, input catatan rapor, approval jurnal kelas. |
| **Guru BK** | Tenant Tunggal | Penanganan kasus kedisiplinan, pencatatan sesi konseling rahasia, penerbitan surat panggilan. |
| **Staf TU / Keuangan** | Tenant Tunggal | Pengelolaan pos tagihan biaya/SPP, penerimaan kasir, cetak kuitansi, monitoring penunggak. |
| **Petugas Gerbang / Piket**| Checkpoint Presensi | Membantu pemindaian QR gerbang, input manual presensi siswa yang terlambat atau lupa membawa kartu. |
| **Pengurus Kelas** (Sekretaris) | Rombel Siswa | Akun siswa dengan hak akses terbatas: input draf presensi jam pelajaran dan draf agenda jurnal kelas. |
| **Siswa** | Data Diri Pribadi | Akses jadwal KBM, riwayat kehadiran, materi/tugas, nilai rapor, dan kartu pelajar digital ber-QR. |
| **Orang Tua / Wali** | Data Anak Terhubung | PWA: Pantau presensi gerbang & mapel real-time, riwayat notifikasi, nilai, catatan BK, dan status SPP. |

---

## 5. Spesifikasi Kebutuhan Fungsional (Functional Requirements)

### 5.1 Modul 1: Manajemen Multi-Tenant & Konfigurasi Sekolah
- **FR-TEN-001**: Sistem harus menyediakan antarmuka manajemen tenant untuk membuat sekolah baru, mengatur identitas (NPSN, Nama, Alamat, Logo, Kop Surat), domain/subdomain, dan kuota pengguna.
- **FR-TEN-002**: Setiap sekolah memiliki konfigurasi independen (`school_configs`):
  - Jam masuk, batas toleransi keterlambatan, dan jam pulang (hari reguler vs hari Jumat).
  - Skema pembobotan nilai (Kurikulum Merdeka / K13).
  - Kebijakan ambang batas poin pelanggaran SP 1, SP 2, dan SP 3.
  - Parameter gateway WhatsApp / Email notifikasi.

### 5.2 Modul 2: Integrasi Dapodik Satu Arah (One-Way Dapodik Sync)
- **FR-DAP-001**: Sistem harus menyediakan konektor sinkronisasi satu arah (*read-only sync*) dari pangkalan data Dapodik Kemendikbudristek menggunakan mekanisme resmi:
  - Sinkronisasi via **Web Service Resmi Dapodik** (REST API lokal pada server Dapodik sekolah dengan IP & Token terotorisasi).
  - Impor berkas snapshot resmi hasil backup/ekspor Dapodik.
- **FR-DAP-002**: Sinkronisasi mencakup entitas: Profil Sekolah, Tenaga Pendidik & Kependidikan (GTK), Peserta Didik (PD), Rombongan Belajar (Rombel), dan Pembelajaran (Mapping Guru - Mapel).
- **FR-DAP-003**: Sistem wajib menyimpan ID referensi Dapodik (`dapodik_id` / GUID) pada setiap entitas terkait agar pembaruan data berkala tidak menimbulkan data duplikat.
- **FR-DAP-004**: Sistem menyediakan *conflict resolution preview* sebelum data diterapkan ke database SAE SMP, sehingga perubahan lokal tidak tertimpa tanpa konfirmasi admin sekolah.
- **FR-DAP-005**: Setiap riwayat penarikan data wajib tercatat pada `dapodik_sync_logs` (waktu, jenis entitas, jumlah data masuk, status berhasil/gagal, user pelaksana).

### 5.3 Modul 3: Presensi Harian Berbasis QR Code & Gate System
- **FR-ATT-001**: Setiap siswa memiliki Kartu Pelajar yang dilengkapi **QR Code Unik**. QR Code memuat token acak terenkripsi/ber-salt yang terhubung dengan `student_id` dan `school_id` guna mencegah pemalsuan QR manual.
- **FR-ATT-002**: Sistem mendukung pemindaian mandiri (*kiosk mode*) pada tablet/kamera gerbang sekolah serta pemindaian oleh Petugas Gerbang/Piket:
  - **Check-In (Masuk)**: Mencatat waktu kedatangan siswa. Jika waktu scan > batas masuk, otomatis tercatat status `Terlambat` dengan durasi menit keterlambatan.
  - **Check-Out (Pulang)**: Mencatat waktu kepulangan siswa di akhir jam sekolah.
- **FR-ATT-003**: Sistem memvalidasi agar tidak terjadi pemindaian ganda (*double scan prevention*) dalam rentang waktu yang tidak wajar (misal: scan masuk 2 kali dalam 5 menit).
- **FR-ATT-004**: Sistem mencatat checkpoint lokasi pemindaian (misal: Gerbang Utama, Gerbang Barat, Pos Piket).
- **FR-ATT-005**: Siswa yang tidak melakukan scan masuk hingga batas akhir KBM jam pertama otomatis ditandai `Alpa` secara sistem melalui scheduled task, kecuali telah diinput `Sakit` atau `Izin`.

### 5.4 Modul 4: Presensi Per Mata Pelajaran (KBM)
- **FR-SUB-001**: Presensi per mata pelajaran berjalan secara independen dan terpisah dari presensi gerbang harian.
- **FR-SUB-002**: Presensi mapel dilakukan untuk setiap sesi KBM (`class_sessions`) sesuai jadwal pelajaran aktif.
- **FR-SUB-003**: Status presensi mapel mencakup: Hadir (H), Sakit (S), Izin (I), Alpa/Bolos (A), Dispensasi (D).
- **FR-SUB-004**: Sistem mampu mendeteksi anomali: Siswa berstatus `Hadir` di presensi gerbang harian, namun `Alpa` pada presensi mata pelajaran tertentu (indikasi membolos jam pelajaran), dan otomatis mengirimkan alert ke Wali Kelas dan Guru BK.

### 5.5 Modul 5: Pencatatan Manual & Immutabilitas Transaksi Presensi
- **FR-MAN-001**: Wali Kelas, Guru Piket, dan Admin berwenang dapat melakukan pencatatan atau penyesuaian (*adjustment*) presensi manual (misal: siswa lupa membawa kartu pelajar, izin keluarga darurat, sakit di UKS di tengah hari).
- **FR-MAN-002**: **Prinsip Immutabilitas**: Koreksi status manual TIDAK BOLEH menghapus rekaman awal. Sistem wajib menyimpan entri penyesuaian pada tabel log terpisah (`daily_attendance_logs` / `subject_attendance_logs`) yang mencatat:
  - Status lama vs Status baru.
  - Alasan perubahan (*reason*).
  - Lampiran bukti (surat dokter / pesan izin orang tua).
  - Waktu perubahan dan ID pengguna petugas yang melakukan perubahan.
- **FR-MAN-003**: Laporan kehadiran dapat menampilkan audit perbandingan antara *data asli pindaian mesin* dengan *data akhir hasil verifikasi manual*.

### 5.6 Modul 6: Administrasi Kelas & Akses Pengurus Kelas (Sekretaris / Ketua)
- **FR-CLS-001**: Wali Kelas dapat menunjuk siswa tertentu sebagai **Pengurus Kelas** (`class_officers`) dengan peran khusus: Ketua Kelas dan Sekretaris Kelas.
- **FR-CLS-002**: Pengurus Kelas memperoleh akses login khusus siswa dengan hak akses terbatas:
  - Menginput **Draf Presensi Mapel**: Sekretaris menandai teman sekelas yang tidak hadir saat guru mapel memulai pembelajaran.
  - Menginput **Draf Jurnal Kelas**: Mencatat materi yang diajarkan, catatan kejadian kelas, dan guru yang mengajar.
- **FR-CLS-003**: **Verifikasi Guru**: Draf presensi dan jurnal yang dibuat oleh sekretaris kelas berstatus `draft` dan TIDAK LANGSUNG menjadi data sah sampai diverifikasi dan disetujui (`verified`) oleh Guru Mata Pelajaran bersangkutan atau Wali Kelas.
- **FR-CLS-004**: Pengurus kelas TIDAK MEMILIKI hak untuk mengubah nilai, melihat data konseling BK, atau mengubah data siswa lain.

### 5.7 Modul 7: Penilaian & Rapor Kurikulum Merdeka
- **FR-GRD-001**: Pengelolaan Tujuan Pembelajaran (TP) dan Lingkup Materi per mata pelajaran per semester.
- **FR-GRD-002**: Input asesmen formatif berkala dan asesmen sumatif (Sumatif Lingkup Materi dan Sumatif Akhir Semester - SAS).
- **FR-GRD-003**: Kalkulasi otomatis nilai akhir dan pembuatan kalimat deskripsi capaian tertinggi serta capaian yang perlu bimbingan sesuai kaidah e-Rapor Kemendikbudristek.
- **FR-GRD-004**: Penilaian Projek Penguatan Profil Pelajar Pancasila (P5) Fase D: Tema, Dimensi, Elemen, Sub-Elemen, dan Predikat (BB, MB, BSH, SB).
- **FR-GRD-005**: Penyusunan Rapor Digital, approval bertingkat (Wali Kelas -> Kepala Sekolah), dan ekspor cetak format PDF resmi.

### 5.8 Modul 8: Kesiswaan, Tata Tertib & Bimbingan Konseling (BK)
- **FR-DIS-001**: Master data pelanggaran dan poin pelanggaran berbasis kategori (kerapian, keterlambatan, etika, pelanggaran berat).
- **FR-DIS-002**: Master data prestasi siswa dan poin penghargaan sebagai penyeimbang poin pelanggaran.
- **FR-DIS-003**: Sistem otomatis mengakumulasi poin dan mentrigger pembuatan draf Surat Panggilan Orang Tua (SP 1, SP 2, SP 3).
- **FR-DIS-004**: Pencatatan Sesi Konseling Siswa oleh Guru BK yang terenkripsi dan bersifat rahasia (hanya dapat diakses Guru BK terkait dan Kepala Sekolah).

### 5.9 Modul 9: Keuangan & Pembayaran SPP
- **FR-FIN-001**: Konfigurasi pos tagihan (SPP bulanan, uang gedung/sarpras, iuran kegiatan).
- **FR-FIN-002**: Pembuatan tagihan berkala otomatis per siswa atau per rombel kelas.
- **FR-FIN-003**: Pencatatan transaksi pembayaran di loket TU (Tunai, Transfer Bank) dengan pencetakan kuitansi resmi.
- **FR-FIN-004**: Rekapitulasi laporan kas masuk harian/bulanan dan daftar siswa menunggak tagihan.

### 5.10 Modul 10: Aplikasi PWA Orang Tua & Pusat Riwayat Notifikasi
- **FR-PAR-001**: Aplikasi web responsif berbasis **Progressive Web App (PWA)** untuk orang tua, mendukung *Add to Home Screen* di Android maupun iOS, mode offline terbatas (cache dashboard terakhir), dan push notification browser.creen*, akses cepat, dan Push Notification peramban.
- **FR-PAR-002**: Dashboard Orang Tua menampilkan profil anak, status kehadiran gerbang hari ini, kehadiran per mapel, catatan kedisiplinan, nilai tugas/rapor, dan status tagihan SPP.
- **FR-PAR-003**: **Pusat Notifikasi Riwayat Internal (In-App Notification Center)**:
  - Setiap peristiwa penting (misal: anak masuk gerbang, anak terlambat, anak alpa di mapel matematika, tagihan SPP terbit, surat panggilan BK terbit) otomatis tersimpan ke tabel `notifications` akun orang tua.
  - Notifikasi memiliki status `read` / `unread` dan tersimpan permanen di database sehingga riwayatnya tidak hilang meskipun pesan eksternal WhatsApp terhapus.
- **FR-PAR-004**: Integrasi WhatsApp Gateway sebagai kanal pengiriman sekunder secara otomatis saat peristiwa terjadi.

### 5.11 Modul 11: Audit Trail Menyeluruh (Audit & Compliance)
- **FR-AUD-001**: Seluruh perubahan data penting (perubahan nilai siswa, penyesuaian presensi, pembayaran keuangan, mutasi siswa, perubahan role pengguna) wajib tercatat pada tabel audit log.
- **FR-AUD-002**: Data log mencakup: `school_id`, `user_id`, nama model (`auditable_type`), ID rekaman (`auditable_id`), aksi (`create`, `update`, `delete`), nilai lama (`old_values`), nilai baru (`new_values`), alamat IP, dan User Agent.
- **FR-AUD-003**: Log audit bersifat *read-only* bahkan untuk level Admin Sekolah, dan hanya dapat dilihat untuk kebutuhan kepatuhan hukum dan investigasi integritas data.

### 5.12 Modul 12: Arsip Dokumen Digital
- **FR-DOC-001**: Penyimpanan dokumen resmi siswa (Akta Kelahiran, Kartu Keluarga, Ijazah SD, Surat Mutasi).
- **FR-DOC-002**: Pengarsipan berkas PDF Rapor Semester yang telah disahkan dan ditandatangani secara digital oleh Kepala Sekolah.

---

## 6. Kebutuhan Non-Fungsional (Non-Functional Requirements)

1. **Keamanan (Security)**:
   - Tenant Data Leakage Prevention: Pengujian otomatis (Pest/PHPUnit) untuk memastikan query lintas tenant ditolak.
   - Password Hashing menggunakan Bcrypt / Argon2id dengan salt aman.
   - QR Code Token terlindungi tanda tangan kriptografis (HMAC SHA-256) untuk mencegah pemalsuan kode QR oleh siswa.
   - Proteksi ketat OWASP Top 10 (CSRF token, XSS sanitize, PDO bind parameter, strict form requests).
2. **Kinerja (Performance)**:
   - Waktu respons API pemindaian QR Code di checkpoint gerbang: `< 500 ms`.
   - Waktu render halaman dashboard: `< 1.2 detik`.
   - Penggunaan background queue (Redis / Database Queue) untuk pengiriman notifikasi massal dan pembuatan dokumen PDF.
3. **Ketersediaan & Keandalan (Availability & Reliability)**:
   - Ketersediaan operasional: 99.8% pada jam KBM sekolah (06.00 - 18.00 WIB).
   - Mekanisme pencadangan data otomatis terisolasi per sekolah setiap hari.
4. **Skalabilitas**:
   - Mendukung minimal 100 sekolah SMP dalam satu klaster server, dengan kapasitas 500 - 1.500 siswa per sekolah.

---

## 7. Batasan & Asumsi Teknis

- Server lokal Dapodik sekolah harus berada dalam jaringan yang dapat diakses oleh gateway sinkronisasi (atau diakses via IP publik / tunnel resmi sekolah yang aman).
- Pemindaian kartu QR Code dapat menggunakan kamera webcam laptop, kamera smartphone petugas, atau alat pemindai barcode/QR USB 2D.
- Pengurus kelas berstatus peserta didik aktif yang terdaftar di rombel kelas terkait pada semester berjalan.

---

## 8. Ringkasan Matriks Fitur per Fase Pengembangan

| Modul | Fase 1 (Core Foundation) | Fase 2 (Attendance & Class) | Fase 3 (Academic & BK) | Fase 4 (Dapodik & PWA) |
| :--- | :---: | :---: | :---: | :---: |
| Multi-Tenant Core & Konfigurasi Sekolah | ✓ | Penyempurnaan | - | - |
| Master Data Pengguna, Guru, Siswa, Rombel | ✓ | - | - | - |
| Presensi QR Gate (Check-In/Out) & Log Imutabel | - | ✓ | - | - |
| Presensi Per Mapel & Sesi KBM | - | ✓ | - | - |
| Administrasi & Akses Pengurus Kelas (Sekretaris) | - | ✓ | - | - |
| Penilaian Kurikulum Merdeka (Formatif, Sumatif, P5) | - | - | ✓ | - |
| Rapor Digital & Approval Bertingkat | - | - | ✓ | - |
| Bimbingan Konseling & Poin Tata Tertib | - | - | ✓ | - |
| Keuangan, Tagihan SPP & Kasir | - | - | ✓ | - |
| Arsip Dokumen Digital Siswa | - | - | ✓ | - |
| PWA Orang Tua & In-App Notification Center | - | - | - | ✓ |
| Integrasi Satu Arah Dapodik | - | - | - | ✓ |
| Notifikasi WhatsApp Gateway (Sekunder) | - | - | - | ✓ |
| Audit Trail Menyeluruh | ✓ | ✓ | ✓ | ✓ |