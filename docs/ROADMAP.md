# Rencana Pengembangan & Peta Jalan (Development Roadmap)
## Smart Application Education Platform (SAE SMP)

- **Versi Dokumen**: 2.0.0
- **Tanggal Pembaruan**: 2026-10-20
- **Status**: Disetujui (Approved)
- **Dokumen Terkait**:
  - [PRD.md](../PRD.md)
  - [DATABASE.md](DATABASE.md)
  - [ARCHITECTURE.md](ARCHITECTURE.md)
  - [USER_ROLES.md](USER_ROLES.md)
  - [SECURITY.md](SECURITY.md)

---

## 1. Ikhtisar Peta Jalan (Roadmap Overview)

Peta jalan pengembangan SAE SMP dirancang secara modular dan berkesinambungan melalui 5 fase utama, memastikan fondasi arsitektur multi-tenant, keamanan data, presensi dual-layer, dan integrasi Dapodik dibangun dengan kokoh sebelum implementasi fitur asesmen lanjutan.

```text
[Fase 1: Fondasi Arsitektur, Multi-Tenancy & Dokumentasi] (Minggu 1 - 2)
   │
   ▼
[Fase 2: Sinkronisasi Dapodik & Master Data Akademik] (Minggu 3 - 4)
   │
   ▼
[Fase 3: Presensi Dual-Layer, Kiosk Gerbang & Log Imutabel] (Minggu 5 - 6)
   │
   ▼
[Fase 4: Administrasi KBM, Jurnal Kelas & Portal Orang Tua PWA] (Minggu 7 - 8)
   │
   ▼
[Fase 5: Asesmen Kurikulum Merdeka, Arsip Dokumen & Go-Live] (Minggu 9 - 10)
```

---

## 2. Rincian Fase Pengembangan

### Fase 1: Fondasi Arsitektur, Multi-Tenancy & Dokumentasi (Minggu 1 - 2)
**Tujuan**: Menyiapkan kerangka kerja Laravel, skema multi-tenant, sistem autentikasi, serta dokumentasi arsitektur teknis yang lengkap.
- **Deliverables**:
  - Dokumentasi teknis lengkap (`PRD.md`, `DATABASE.md`, `ARCHITECTURE.md`, `USER_ROLES.md`, `USER_FLOW.md`, `SECURITY.md`, `ROADMAP.md`).
  - Proyek Laravel bersih dengan konfigurasi PHP 8.2+, Tailwind CSS, Alpine.js, dan Livewire.
  - Skema basis data migrasi 47 tabel sesuai entitas di `DATABASE.md`.
  - Middleware `IdentifyTenantByDomainOrHeader` dan Trait `BelongsToSchool` dengan `SchoolScope`.
  - Pengujian unit dasar isolasi multi-tenant (`MultiTenantIsolationTest`).
- **Kriteria Keberhasilan**: Query data pada sekolah A tidak dapat menampilkan baris data sekolah B dalam kondisi pengujian apapun.

---

### Fase 2: Sinkronisasi Dapodik & Master Data Akademik (Minggu 3 - 4)
**Tujuan**: Menyediakan alur penarikan data satu arah dari server lokal Dapodik ke SAE SMP guna mempercepat *onboarding* sekolah tanpa entri manual.
- **Deliverables**:
  - Konfigurasi koneksi Dapodik Web Service lokal pada tabel `dapodik_configs`.
  - Service `DapodikSyncService` dan job antrean `DapodikSyncJob`.
  - Modul pemetaan entitas (`dapodik_entity_mappings`) untuk Sekolah, PTK/Guru, Siswa, dan Rombongan Belajar.
  - Dasbor status sinkronisasi, riwayat log per transaksi, dan pelaporan konflik data.
  - Modul pengelolaan manual tahun ajaran, semester, dan struktur kurikulum.
- **Kriteria Keberhasilan**: Operator berhasil menarik 100% data siswa, guru, dan rombel dari Dapodik uji coba dalam waktu kurang dari 5 menit tanpa galat duplikasi.

---

### Fase 3: Presensi Dual-Layer, Kiosk Gerbang & Log Imutabel (Minggu 5 - 6)
**Tujuan**: Mengimplementasikan pencatatan kedatangan/kepulangan gerbang via kartu QR terenkripsi dan mekanisme koreksi manual yang akuntabel.
- **Deliverables**:
  - Generator token QR kartu pelajar dengan tanda tangan HMAC-SHA256 (`QrSecurityService`).
  - Halaman terminal Kiosk pemindai gerbang responsif dan cepat untuk Guru Piket / Scanner Hardware.
  - Pencatatan otomatis waktu masuk, waktu pulang, dan kalkulasi keterlambatan harian (`daily_attendances`).
  - Mekanisme koreksi presensi manual oleh Wali Kelas dengan penulisan riwayat imutabel di `daily_attendance_logs`.
  - Pengujian performa pemindaian QR (mampu menangani hingga 120 pindaian/menit per terminal).
- **Kriteria Keberhasilan**: Pemindaian QR menghasilkan respons validasi di bawah 200 ms, dan setiap pengubahan status manual tercatat permanen tanpa menghapus data mentah pemindaian.

---

### Fase 4: Administrasi KBM, Jurnal Kelas & Portal Orang Tua PWA (Minggu 7 - 8)
**Tujuan**: Memfasilitasi pencatatan kegiatan belajar mengajar (KBM) di kelas dengan pendelegasian pengurus kelas, serta portal komunikasi wali murid via PWA.
- **Deliverables**:
  - Modul penugasan Pengurus Kelas (`class_officers`) dengan pembatasan hak akses berbasis peran sekretaris kelas.
  - Pengisian draf jurnal KBM harian oleh Sekretaris Rombel (`is_draft = true`).
  - Fitur pengesahan dan verifikasi jurnal KBM oleh Guru Mata Pelajaran.
  - Presensi per mata pelajaran (`subject_attendances`) dengan deteksi anomali kehadiran gerbang vs ruang kelas.
  - Portal PWA Orang Tua dengan Web Push notification dan Pusat Riwayat Notifikasi In-App (`notifications`).
  - Fitur pengajuan permohonan izin/sakit anak oleh orang tua dengan unggahan surat dokter.
- **Kriteria Keberhasilan**: Wali murid menerima notifikasi real-time kedatangan siswa di gerbang, dan guru dapat memverifikasi draf jurnal yang diajukan oleh pengurus kelas.

---

### Fase 5: Asesmen Kurikulum Merdeka, Arsip Dokumen & Go-Live (Minggu 9 - 10)
**Tujuan**: Melengkapi modul penilaian akademik formatif & sumatif, generasi rapor digital, pengarsipan dokumen siswa, dan uji beban sistem.
- **Deliverables**:
  - Manajemen tujuan pembelajaran (TP), pembobotan asesmen formatif, sumatif materi, dan sumatif akhir semester.
  - Mesin pembuat deskripsi capaian kompetensi otomatis sesuai panduan Kurikulum Merdeka.
  - Generator dokumen PDF Rapor Siswa dan Buku Induk digital.
  - Modul pengarsipan berkas siswa dengan verifikasi hash integritas SHA-256 (`student_documents`).
  - Uji penetrasi keamanan (Penetration Testing) dan audit kerentanan OWASP Top 10.
  - *Dry-run* dan pelatihan pengguna (*User Acceptance Testing* - UAT) di sekolah percontohan.
- **Kriteria Keberhasilan**: Rapor berhasil dicetak dalam format resmi, seluruh hasil audit keamanan bersih (*zero critical vulnerabilities*), dan sistem siap diluncurkan secara produksi.

---

## 3. Matriks Risiko & Mitigasi (Risk Matrix)

| Risiko | Dampak | Probabilitas | Tindakan Mitigasi |
|---|---|---|---|
| **Server lokal Dapodik lambat atau tidak stabil** | Tinggi | Sedang | Gunakan antrean background (Queue Jobs) dengan pagination kecil (50 baris per request) dan mekanisme retry otomatis dengan exponential backoff. |
| **Koneksi internet gerbang sekolah terputus** | Tinggi | Sedang | Kiosk scanner dilengkapi mode offline sementara berbasis IndexedDB lokal yang melakukan sinkronisasi otomatis saat koneksi pulih. |
| **Siswa memalsukan QR Code kartu pelajar** | Tinggi | Rendah | Menggunakan enkripsi HMAC-SHA256 ber-salt unik per sekolah dan timestamp penerbitan kartu. |
| **Beban trafik puncak saat jam kedatangan (06.30 - 07.15 WIB)** | Tinggi | Tinggi | Optimasi kueri database dengan komposit indeks, pemanfaatan Redis cache untuk data siswa aktif, dan penundaan pengiriman notifikasi via antrean asinkron. |
