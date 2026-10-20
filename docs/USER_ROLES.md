# Hak Akses & Peran Pengguna (User Roles & Permissions)
## Smart Application Education Platform (SAE SMP)

- **Versi Dokumen**: 2.0.0
- **Tanggal Pembaruan**: 2026-10-20
- **Status**: Disetujui (Approved)
- **Dokumen Terkait**:
  - [PRD.md](../PRD.md)
  - [DATABASE.md](DATABASE.md)
  - [ARCHITECTURE.md](ARCHITECTURE.md)
  - [USER_FLOW.md](USER_FLOW.md)
  - [SECURITY.md](SECURITY.md)

---

## 1. Ikhtisar Model Otorisasi (Authorization Model)

SAE SMP menerapkan sistem otorisasi **Role-Based Access Control (RBAC)** berlapis menggunakan paket standar industri **Spatie Laravel-Permission**, dengan penambahan isolasi multi-tenant (`team_id` / `school_id`) dan hak akses kontekstual (*Contextual / Scoped Permissions*).

### 1.1 Hierarki & Kategori Peran

```text
[Super Administrator (Platform SaaS)]
  │
  └─── [Administrator Sekolah / Operator Dapodik]
         │
         ├─── [Kepala Sekolah]
         ├─── [Wakil Kepala Sekolah (Kurikulum / Kesiswaan)]
         ├─── [Guru Piket / Petugas Presensi Gerbang]
         │
         ├─── [Wali Kelas]
         │      │
         │      ├─── [Guru Mata Pelajaran]
         │      │
         │      └─── [Pengurus Kelas (Sekretaris / Ketua Kelas)] (Siswa Berperan Khusus)
         │
         ├─── [Konselor / Guru BK]
         ├─── [Staf Tata Usaha / Keuangan]
         │
         ├─── [Siswa (Reguler)]
         └─── [Orang Tua / Wali Murid (PWA Portal)]
```

---

## 2. Definisi Peran (Role Definitions)

### 2.1 Peran Tingkat Platform (Global SaaS Level)
1. **Super Administrator (`super_admin`)**:
   - Pengelola server dan infrastruktur multi-sekolah.
   - Hak istimewa: Membuat tenant sekolah baru, mengonfigurasi batas kuota siswa, memantau log audit global, dan memperbarui konfigurasi sistem.
   - Tidak terlibat dalam transaksi harian akademik sekolah.

### 2.2 Peran Tingkat Sekolah (Tenant School Level)
2. **Administrator Sekolah (`school_admin`)**:
   - Penanggung jawab operasional teknis sistem di sekolah.
   - Mengelola master data sekolah, akun staf pengajar, pengaturan tahun ajaran, jam operasional presensi, dan perizinan pengguna lokal.
3. **Operator Dapodik (`dapodik_operator`)**:
   - Petugas resmi yang memegang kredensial Web Service Dapodik lokal sekolah.
   - Mengonfigurasi endpoint sinkronisasi, menjalankan proses penarikan data berkala, menyelesaikan konflik identitas, dan memantau log sinkronisasi.
4. **Kepala Sekolah (`principal`)**:
   - Pejabat eksekutif dengan akses pemantauan tingkat tinggi (*Executive Dashboard*).
   - Memantau tren kehadiran siswa dan guru, capaian akademik sekolah, ringkasan pelanggaran BK, serta memberikan pengesahan akhir rapor/ijazah.
5. **Wakil Kepala Sekolah Bidang Kurikulum (`vice_principal_curriculum`)**:
   - Mengelola struktur kurikulum, silabus capaian kompetensi (CP/TP), pembagian beban mengajar guru, jadwal pelajaran, dan kalender KBM.
6. **Wakil Kepala Sekolah Bidang Kesiswaan (`vice_principal_student_affairs`)**:
   - Mengelola tata tertib sekolah, poin pelanggaran & prestasi kesiswaan, kepengurusan OSIS/ekstrakurikuler, dan kebijakan presensi gerbang.
7. **Guru Piket / Petugas Gerbang (`picket_officer`)**:
   - Bertanggung jawab memonitor alur masuk/pulang di gerbang sekolah.
   - Mengoperasikan Kiosk scanner, menangani siswa tanpa kartu QR, dan mencatat alasan keterlambatan gerbang secara real-time.
8. **Wali Kelas (`homeroom_teacher`)**:
   - Pengampu utama satu rombongan belajar (rombel).
   - Memiliki wewenang:
     - Mengubah status presensi harian siswa rombelnya dengan alasan sah (misal: memasukkan izin/sakit dengan lampiran surat).
     - Menyetujui/menolak pengajuan permohonan izin dari orang tua.
     - Memantau catatan jurnal kelas dan menginput catatan perkembangan karakter pada rapor siswa.
9. **Guru Mata Pelajaran (`subject_teacher`)**:
   - Pengampu mata pelajaran pada sesi kelas tertentu.
   - Memiliki wewenang:
     - Memulai sesi pembelajaran (*class session*).
     - Memverifikasi draf jurnal KBM yang diajukan oleh sekretaris kelas.
     - Melakukan presensi per mata pelajaran (*subject attendance*).
     - Menginput nilai asesmen formatif, sumatif, dan deskripsi capaian kompetensi.
10. **Guru Bimbingan Konseling (`counseling_teacher` / BK)**:
    - Menangani kasus kedisiplinan dan pembinaan siswa.
    - Mencatat insiden bimbingan konseling privat, poin pelanggaran, konseling dengan orang tua, serta program pengembangan diri.
11. **Staf Tata Usaha (`administrative_staff`)**:
    - Mengelola pengarsipan dokumen digital siswa, surat mutasi, penerbitan surat keterangan aktif sekolah, dan data inventaris kelas.

### 2.3 Peran Tingkat Siswa & Pengurus Kelas
12. **Pengurus Kelas / Sekretaris Rombel (`class_officer` - Kontekstual)**:
    - Diberikan secara dinamis kepada siswa aktif yang ditunjuk sebagai sekretaris rombel (berdasarkan tabel `class_officers`).
    - Hak akses dibatasi hanya untuk rombelnya sendiri pada tahun ajaran aktif:
      - Menyiapkan draf jurnal KBM harian (mencatat materi dan guru pengajar).
      - Menandai draf siswa yang tidak hadir pada sesi mata pelajaran.
      - *Larangan*: Tidak dapat mengesahkan jurnal KBM, tidak dapat mengubah nilai, dan tidak dapat memanipulasi presensi gerbang.
13. **Siswa Reguler (`student`)**:
    - Mengakses portal siswa untuk:
      - Menampilkan kartu pelajar digital dengan QR code terenkripsi dinamis.
      - Melihat jadwal pelajaran, agenda tugas, dan pengumuman sekolah.
      - Memeriksa rekap presensi pribadi dan histori capaian nilai.

### 2.4 Peran Wali Murid
14. **Orang Tua / Wali Murid (`parent`)**:
    - Mengakses portal PWA (Progressive Web App) wali murid:
      - Menerima notifikasi langsung saat anak tiba di sekolah atau pulang melewati gerbang.
      - Mengajukan permohonan izin/sakit anak disertai unggahan foto surat izin/dokter.
      - Memantau rekap absensi per mata pelajaran (mendeteksi dini siswa membolos).
      - Melihat laporan rapor digital dan agenda kegiatan sekolah anak yang terhubung.

---

## 3. Matriks Hak Akses (Permissions Matrix)

Keterangan:
- **C** = Create (Membuat)
- **R** = Read (Melihat)
- **U** = Update (Mengubah)
- **D** = Delete (Menghapus)
- **V** = Verify / Approve (Verifikasi & Pengesahan)
- **own** = Terbatas pada data miliknya sendiri atau rombel asuhannya
- **-** = Tidak memiliki akses

| Modul & Fitur | Super Admin | School Admin | Dapodik Op | Principal | Wali Kelas | Guru Mapel | Guru Piket | Sekretaris Kelas | Orang Tua (PWA) | Siswa |
|---|---|---|---|---|---|---|---|---|---|---|
| **Manajemen Tenant Sekolah** | CRUD | - | - | - | - | - | - | - | - | - |
| **Konfigurasi Dapodik & Sync** | R | R | CRUD | R | - | - | - | - | - | - |
| **Master Data Akademik (TP/Semester)** | CRUD | CRUD | R | R | R | R | - | - | - | R |
| **Master Siswa & Guru** | CRUD | CRUD | R | R | R (own) | R (own) | R | - | - | R (own) |
| **Presensi Gerbang: Scan Kiosk** | - | - | - | - | - | - | C (scan) | - | - | R (card) |
| **Presensi Gerbang: Koreksi Manual** | - | U (audit) | - | - | U (own+audit) | - | U (audit) | - | - | - |
| **Presensi Gerbang: Permohonan Izin** | - | - | - | - | V (own) | - | - | - | C (anak) | - |
| **Presensi Mapel: Input / Edit** | - | - | - | - | R (own) | CRUD (own) | - | C (draft) | - | - |
| **Jurnal KBM: Buat Draf** | - | - | - | - | - | C | - | C (draft) | - | - |
| **Jurnal KBM: Verifikasi & Sahkan** | - | - | - | - | - | V (own) | - | - | - | - |
| **Asesmen & Nilai Raport** | - | R | - | R | R (own) | CRUD (own) | - | - | R (anak) | R (own) |
| **Buku Induk & Arsip Dokumen** | R | CRUD | R | R | R (own) | - | - | - | R (anak) | R (own) |
| **Catatan Bimbingan Konseling** | - | R | - | R | R (own) | - | - | - | - | - |
| **Notifikasi In-App & PWA** | - | C (broadcast) | - | R | C (own) | C (own) | - | - | R (own) | R (own) |
| **Audit Logs & Keamanan** | CRUD | R (sekolah) | - | R (sekolah) | - | - | - | - | - | - |

---

## 4. Kebijakan & Aturan Otorisasi Khusus (Special Authorization Rules)

### 4.1 Isolasi Tenant Sekolah (School Tenant Scoping)
- Setiap pengguna (kecuali `super_admin`) terikat pada satu atau lebih sekolah melalui tabel relasi `school_user`.
- Saat login, sesi sekolah aktif disimpan dalam konteks aplikasi (`app('current_school')`).
- Pengguna tidak diizinkan mengakses data dari sekolah lain (`403 Forbidden`).

### 4.2 Hak Akses Pengurus Kelas (Class Officer Delegation)
- Akses pengurus kelas dievaluasi saat runtime melalui policy:
  ```php
  // Contoh Gate/Policy untuk draf jurnal kelas
  Gate::define('draft-class-journal', function (User $user, Classroom $classroom) {
      if ($user->hasRole('teacher')) return true;

      return $user->student && $classroom->officers()
          ->where('student_id', $user->student->id)
          ->where('role', 'sekretaris_1')
          ->where('is_active', true)
          ->exists();
  });
  ```
- Draf jurnal yang dibuat oleh sekretaris kelas ditandai bendera `is_draft = true`. Data tersebut belum dianggap resmi hingga guru mata pelajaran menandatangani/memverifikasi draf tersebut.

### 4.3 Imutabilitas & Penelusuran Koreksi Presensi
- Koreksi presensi yang dilakukan oleh Wali Kelas atau Petugas Piket wajib menyertakan alasan (`change_reason`) dan dokumen bukti jika ada.
- Tidak ada penghapusan data presensi (`No Hard Delete`). Setiap modifikasi memicu penulisan rekaman baru di tabel `daily_attendance_logs` atau `subject_attendance_logs`.
- Status yang dikunci oleh batas waktu tutup buku semester tidak dapat diubah kembali kecuali dibuka melalui tiket resmi Administrator Sekolah.
