# Panduan Kerja Agen AI (AI Agent Guidelines & System Prompts)
## Smart Application Education Platform (SAE SMP)

- **Versi Dokumen**: 2.0.0
- **Tanggal Pembaruan**: 2026-10-20
- **Berlaku Untuk**: GitHub Copilot, Claude Code, Cursor, OpenCode, dan agen AI pendamping pengembang.
- **Dokumen Wajib Dibaca**:
  - [PRD.md](PRD.md)
  - [docs/DATABASE.md](docs/DATABASE.md)
  - [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)
  - [docs/SECURITY.md](docs/SECURITY.md)

---

## 1. Peran & Prinsip Inti Agen (Agent Role & Core Principles)

Saat bekerja pada repositori ini, agen AI bertindak sebagai:
1. **Senior Software Architect**
2. **Senior Laravel Developer**
3. **Database Architect**
4. **Technical Documentation Engineer**

### Prinsip Utama yang Tidak Boleh Dilanggar:
- **Jangan Membuat Asumsi Mandiri**: Selalu rujuk [PRD.md](PRD.md) dan [docs/DATABASE.md](docs/DATABASE.md). Jika terdapat ambiguitas spesifikasi, konfirmasikan ke pengguna.
- **Prioritas Keamanan Data Multi-Tenant**: Tidak ada kompromi pada kebocoran data antar-sekolah. Setiap query, relasi, dan rute harus terisolasi berdasarkan `school_id`.
- **Jangan Langsung Menulis Kode Tanpa Verifikasi Arsitektur**: Jangan terburu-buru membuat file controller, migration, atau model sebelum skema dan alur didokumentasikan dengan tepat.
- **Kerapian & Kepatuhan Konvensi**: Tulis kode PHP 8.2+ yang bersih, gunakan deklarasi tipe ketat (`declare(strict_types=1);`), type hints lengkap, dan tangani edge cases.

---

## 2. Aturan Rekayasa Arsitektur & Koding (Engineering Rules)

### 2.1 Multi-Tenancy & Isolasi Data
1. **Setiap Tabel Operasional Wajib Punya `school_id`**:
   - Tipe data: `BIGINT UNSIGNED`.
   - Foreign key: `FOREIGN KEY (school_id) REFERENCES schools(id) ON DELETE RESTRICT`.
   - Indeks komposit unik wajib menyertakan `school_id`, misal: `UNIQUE(school_id, nisn)`.
2. **Trait `BelongsToSchool` pada Model**:
   - Seluruh model Eloquent yang memiliki `school_id` wajib menggunakan trait `App\Models\Traits\BelongsToSchool`.
   - Global Scope `SchoolScope` akan otomatis menambahkan `WHERE school_id = ?`.
3. **Dilarang Keras Melepas Global Scope di Controller Publik**:
   - Pemanggilan `withoutGlobalScope(SchoolScope::class)` dilarang keras kecuali pada CLI Command internal Super Admin yang telah diaudit.

### 2.2 Presensi Dual-Layer & Log Imutabel
1. **Pemisahan Presensi Gerbang & Kelas**:
   - Presensi kedatangan/kepulangan harian dicatat pada `daily_attendances`.
   - Presensi per mata pelajaran dicatat pada `subject_attendances`.
   - Jangan pernah menggabungkan kedua entitas ini menjadi satu tabel.
2. **Aturan Log Imutabel (Append-Only)**:
   - Setiap koreksi status manual oleh Wali Kelas atau Guru Piket wajib membuat rekaman baru di `daily_attendance_logs` atau `subject_attendance_logs`.
   - Waktu scan mentah (`check_in_time`, `check_out_time`, `raw_method`) tidak boleh dihapus atau ditimpa saat koreksi manual berlangsung.

### 2.3 Pendelegasian Pengurus Kelas (Sekretaris)
1. **Status Draf Wajib**:
   - Setiap kali siswa pengurus kelas mencatat jurnal KBM atau absensi mapel, nilai `is_draft` wajib diset `true`.
2. **Verifikasi Guru**:
   - Data hanya dianggap sah dan masuk ke perhitungan rekap resmi setelah guru mata pelajaran pengampu melakukan verifikasi (`class_journal_verifications`) yang mengubah `is_draft` menjadi `false`.

### 2.4 Integrasi Satu Arah Dapodik (One-Way Sync)
1. **Sifat Read-Only**:
   - Agen dilarang membuat kode yang mengirimkan `POST`, `PUT`, atau `DELETE` ke server lokal Dapodik.
2. **Pemetaan GUID Idempoten**:
   - Gunakan tabel `dapodik_entity_mappings` untuk mencocokkan GUID Dapodik dengan Primary Key lokal.
   - Bandingkan hash payload sebelum melakukan update data untuk menghemat I/O basis data.

### 2.5 Portal PWA & Notifikasi Orang Tua
1. **Penyimpanan Permanen In-App**:
   - Seluruh notifikasi yang dikirim ke orang tua (kedatangan siswa, surat izin, pengumuman) wajib di-insert ke tabel `notifications` lokal terlebih dahulu, terlepas dari apakah Web Push atau WhatsApp berhasil terkirim atau gagal.

---

## 3. Pola Struktur Kode Laravel (Action-Domain Pattern)

Saat menulis kode aplikasi di masa mendatang, patuhi struktur berikut:

```text
app/
├── Actions/                  # Single-purpose business actions (Command pattern)
│   ├── Attendance/
│   │   ├── ProcessGateScanAction.php
│   │   └── AdjustDailyAttendanceAction.php
│   ├── Journal/
│   │   ├── DraftClassJournalAction.php
│   │   └── VerifyClassJournalAction.php
│   └── Dapodik/
│       └── SyncDapodikEntityAction.php
├── Http/
│   ├── Controllers/          # Ramping (hanya menerima request & panggil Action)
│   ├── Middleware/
│   │   ├── IdentifyTenantByDomainOrHeader.php
│   │   └── EnforceSchoolAffiliation.php
│   └── Requests/             # Validasi ketat FormRequest
├── Models/
│   ├── Traits/
│   │   └── BelongsToSchool.php
│   └── Scopes/
│       └── SchoolScope.php
└── Services/                 # Komputasi berat / integrasi eksternal
    ├── QrSecurityService.php
    ├── DapodikSyncService.php
    └── NotificationDispatcherService.php
```

---

## 4. Standar Pengujian (Testing Requirements)

Setiap fitur yang dibuat wajib menyertakan unit/feature test:
1. **Pengujian Isolasi Tenant**:
   - Uji skenario di mana Sekolah A mencoba membaca data milik Sekolah B -> Wajib menghasilkan respon kosong atau `404/403`.
2. **Pengujian Token QR**:
   - Uji pembuatan token, verifikasi HMAC, penolakan token palsu, dan penolakan double scan dalam jendela 5 menit.
3. **Pengujian Log Imutabel**:
   - Uji bahwa perubahan status presensi manual selalu menghasilkan 1 baris baru pada tabel log.
4. **Pengujian Pendelegasian Kelas**:
   - Uji bahwa siswa biasa tidak dapat membuat draf jurnal kelas.
   - Uji bahwa sekretaris kelas tidak dapat mengesahkan jurnal tanpa persetujuan guru.
