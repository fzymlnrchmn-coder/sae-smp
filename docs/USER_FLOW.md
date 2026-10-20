# Alur Pengguna & Interaksi Sistem (User Flows & System Interaction)
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

## 1. Alur Penyiapan Sekolah Baru (School Onboarding & Setup)

Alur pendaftaran dan inisialisasi tenant sekolah baru oleh Super Admin dan Admin Sekolah:

```mermaid
flowchart TD
    Start([Super Admin]) --> CreateTenant[Input Data Sekolah & Buat Subdomain / Domain]
    CreateTenant --> SeedTenant[Sistem Inisialisasi: Role Standar, Tipe Dokumen, Status Presensi]
    SeedTenant --> CreateAdminAccount[Generate Akun Administrator Sekolah]
    CreateAdminAccount --> SendCredential[Kirim Kredensial Akses ke Sekolah]
    SendCredential --> SchoolLogin([Admin Sekolah Login ke Portal])
    SchoolLogin --> SetupAcademic[Konfigurasi Tahun Pelajaran & Semester Aktif]
    SetupAcademic --> SetupSchedule[Atur Jam Kerja Sekolah & Batas Keterlambatan Gerbang]
    SetupSchedule --> NextStep([Siap Lakukan Sinkronisasi Dapodik])
```

---

## 2. Alur Sinkronisasi Satu Arah Dapodik (One-Way Dapodik Sync)

Proses penarikan data resmi dari server lokal Dapodik ke SAE SMP:

```mermaid
sequenceDiagram
    autonumber
    actor Op as Operator Dapodik
    participant App as SAE SMP (Web Admin)
    participant SyncJob as Background Sync Engine
    participant Dapodik as Server Lokal Dapodik (REST Web Service)
    participant DB as Basis Data SAE SMP

    Op->>App: Buka Modul Sinkronisasi Dapodik
    Op->>App: Input Web Service URL, Port & NPSN Auth Token
    App->>Dapodik: Tes Koneksi & Handshake (GET /api/v1/sekolah)
    Dapodik-->>App: Respon Status 200 OK (Data Sekolah Divalidasi)
    Op->>App: Klik "Mulai Tarik Data Master"
    App->>SyncJob: Jadwalkan DapodikSyncJob (Queue Worker)
    
    activate SyncJob
    SyncJob->>DB: Catat dapodik_sync_logs (status: in_progress)
    SyncJob->>Dapodik: GET /api/v1/ptk (Data Guru & Tendik)
    Dapodik-->>SyncJob: Response JSON PTK
    SyncJob->>DB: Upsert Master Guru & dapodik_entity_mappings
    
    SyncJob->>Dapodik: GET /api/v1/peserta_didik (Data Siswa)
    Dapodik-->>SyncJob: Response JSON Siswa
    SyncJob->>DB: Upsert Master Siswa & Mapping NISN / GUID
    
    SyncJob->>Dapodik: GET /api/v1/rombongan_belajar (Data Rombel & Anggota)
    Dapodik-->>SyncJob: Response JSON Rombel
    SyncJob->>DB: Upsert Kelas, Wali Kelas & Hubungan Siswa-Kelas
    
    SyncJob->>DB: Update dapodik_sync_logs (status: completed, total_records)
    deactivate SyncJob

    SyncJob-->>App: Notifikasi Selesai via WebSocket
    App-->>Op: Tampilkan Laporan Ringkasan Sinkronisasi
```

---

## 3. Alur Presensi Gerbang (QR Code Scan) & Notifikasi PWA Orang Tua

Alur saat siswa memindai kartu pelajar di gerbang sekolah pada pagi dan sore hari:

```mermaid
sequenceDiagram
    autonumber
    actor Siswa as Siswa (Kartu QR)
    participant Kiosk as Terminal Kiosk / Guru Piket
    participant Server as SAE SMP Server API
    participant DB as Basis Data (MySQL)
    participant Queue as Queue Worker Notifikasi
    actor Ortu as Wali Murid (PWA App)

    Siswa->>Kiosk: Tunjukkan Kartu QR Pelajar ke Pemindai
    Kiosk->>Server: POST /api/v1/attendance/gate-scan {qr_payload, checkpoint_id}
    Server->>Server: Dekripsi & Verifikasi Tanda Tangan HMAC-SHA256
    alt QR Code Tidak Valid atau Kedaluwarsa
        Server-->>Kiosk: 422 Unprocessable (QR Invalid / Terblokir)
        Kiosk-->>Siswa: Bunyi Alarm Merah: "Kartu Tidak Valid"
    else QR Valid & Aktif
        Server->>DB: Cek Duplikasi Scan (Debounce 5 Menit)
        alt Sudah Scan dalam 5 Menit Terakhir
            Server-->>Kiosk: 200 OK (Abaikan: Scan Berulang)
        else Scan Pertama Hari Ini
            Server->>Server: Komparasi Waktu dengan Jam Toleransi Keterlambatan
            Server->>DB: Simpan daily_attendances (status: Hadir / Terlambat)
            Server->>DB: Buat Catatan daily_attendance_logs
            Server-->>Kiosk: 200 OK (Tampilkan Foto, Nama, Status: Tepat Waktu / Telat X Menit)
            Kiosk-->>Siswa: Bunyi Nada Hijau: "Selamat Pagi, [Nama]!"
            
            Server->>Queue: Dispatch StudentGateAttendanceEvent
            activate Queue
            Queue->>DB: Simpan Riwayat di tabel notifications (User ID Ortu)
            Queue->>Ortu: Kirim Web Push Notification (PWA) & Opsi Pesan WhatsApp
            deactivate Queue
            Ortu-->>Ortu: Muncul Notifikasi: "Anak Anda telah tiba di sekolah pukul 06.45 WIB"
        end
    end
```

---

## 4. Alur Koreksi Presensi Harian Manual & Audit Trail

Alur ketika orang tua membawa surat dokter atau wali kelas mengoreksi ketidakhadiran siswa:

```mermaid
sequenceDiagram
    autonumber
    actor WaliKelas as Wali Kelas / Staf Piket
    participant UI as Dashboard Guru (SAE SMP)
    participant Action as AdjustDailyAttendanceAction
    participant DB as Basis Data

    WaliKelas->>UI: Buka Rekap Presensi Rombel
    WaliKelas->>UI: Pilih Siswa (Status Awal: Alpa)
    WaliKelas->>UI: Ubah Status ke "Sakit", Input Alasan ("Surat Dokter Terlampir"), Unggah Bukti
    UI->>Action: POST /attendance/daily/adjust {attendance_id, new_status, reason, attachment}
    Action->>Action: Validasi Otorisasi (Apakah Wali Kelas Sah dari Rombel Ini?)
    
    Action->>DB: UPDATE daily_attendances SET final_status = 'S', is_manually_adjusted = true
    Action->>DB: INSERT INTO daily_attendance_logs (previous_status, new_status, change_reason, attachment_path, changed_by_user_id)
    Action->>DB: INSERT INTO audit_logs (event: 'daily_attendance.adjusted', severity: 'medium')
    Action-->>UI: 200 Success (Status Berhasil Diperbarui)
    UI-->>WaliKelas: Tampilkan Ikon "Terkoreksi Manual (Ada Bukti Riwayat)"
```

---

## 5. Alur Presensi Mata Pelajaran & Pendelegasian Pengurus Kelas

Alur kolaborasi antara Sekretaris Kelas (Siswa) dan Guru Mata Pelajaran dalam pencatatan KBM:

```mermaid
sequenceDiagram
    autonumber
    actor Sekre as Sekretaris Kelas (Siswa)
    actor Guru as Guru Mata Pelajaran
    participant App as Aplikasi SAE SMP
    participant DB as Basis Data

    Note over Sekre, Guru: Jam Pelajaran Dimulai di Ruang Kelas
    Sekre->>App: Buka Menu Jurnal KBM Rombel
    Sekre->>App: Input Draf Materi Pembelajaran, Jam Ke-, dan Tandai Siswa Tidak Hadir
    Sekre->>App: Simpan Draf Jurnal (is_draft = true)
    App->>DB: INSERT class_journals (is_draft = true, created_by_student_id)
    App->>DB: INSERT subject_attendances (status: Draf)
    App-->>Sekre: Draf Tersimpan, Menunggu Pengesahan Guru

    Note over Guru: Guru Memeriksa Ruang Kelas
    Guru->>App: Buka Sesi KBM Terkait
    Guru->>App: Review Materi & Cek Daftar Hadir Siswa yang Ditandai
    alt Ada Kekeliruan
        Guru->>App: Perbaiki Status Siswa atau Materi
    end
    Guru->>App: Klik "Sahkan & Verifikasi Jurnal KBM"
    App->>DB: UPDATE class_journals SET is_draft = false
    App->>DB: INSERT class_journal_verifications (teacher_id, signature, verified_at)
    App->>DB: UPDATE subject_attendances SET is_final = true
    App-->>Guru: Jurnal KBM Sah dan Terarsip Resmi
```

---

## 6. Alur Pengajuan Izin/Sakit Orang Tua via PWA

Alur saat orang tua mengirim surat izin dari rumah menggunakan aplikasi PWA:

```mermaid
sequenceDiagram
    autonumber
    actor Ortu as Orang Tua Murid (PWA)
    participant PWA as Aplikasi PWA Wali Murid
    participant Server as Server SAE SMP
    actor WaliKelas as Wali Kelas
    participant DB as Basis Data

    Ortu->>PWA: Buka Menu "Ajukan Izin / Sakit"
    Ortu->>PWA: Pilih Rentang Tanggal, Alasan, dan Foto Surat Dokter
    Ortu->>PWA: Klik Kirim Permohonan
    PWA->>Server: POST /api/v1/parent/leave-requests
    Server->>DB: Simpan leave_requests (status: pending)
    Server->>WaliKelas: Notifikasi Masuk ke Dashboard Wali Kelas
    
    WaliKelas->>Server: Periksa Foto Surat & Alasan
    alt Permohonan Disetujui
        WaliKelas->>Server: Klik "Setujui Izin"
        Server->>DB: UPDATE leave_requests SET status = 'approved', approved_by_user_id
        Server->>DB: UPDATE daily_attendances (Set final_status = 'I' / 'S')
        Server->>DB: INSERT daily_attendance_logs (source: 'parent_leave_request')
        Server->>DB: Simpan notifikasi persetujuan di tabel notifications
        Server-->>Ortu: Notifikasi PWA: "Permohonan Izin Telah Disetujui Wali Kelas"
    else Permohonan Ditolak
        WaliKelas->>Server: Klik "Tolak" & Beri Catatan Alasan
        Server->>DB: UPDATE leave_requests SET status = 'rejected'
        Server-->>Ortu: Notifikasi PWA: "Permohonan Izin Ditolak: [Alasan]"
    end
```

---

## 7. Alur Pelaporan Rapor & Pengarsipan Digital Siswa

```mermaid
flowchart TD
    A([Akhir Semester]) --> B[Guru Mapel Input Nilai Formatif & Sumatif]
    B --> C[Sistem Hitung Rerata & Generate Deskripsi Capaian TP]
    C --> D[Wali Kelas Input Catatan Sikap, Ekstrakurikuler & Kehadiran]
    D --> E[Wali Kelas Kunci Nilai Rombel]
    E --> F[Kepala Sekolah Review & Berikan Pengesahan Digital]
    F --> G[Sistem Generate Dokumen PDF Rapor Siswa]
    G --> H[Simpan Arsip di student_documents dengan Hash SHA-256]
    H --> I[Orang Tua & Siswa Akses Rapor Digital di Portal PWA]
```
