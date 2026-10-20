# Keamanan & Perlindungan Data (Security & Data Protection)
## Smart Application Education Platform (SAE SMP)

- **Versi Dokumen**: 2.0.0
- **Tanggal Pembaruan**: 2026-10-20
- **Status**: Disetujui (Approved)
- **Dokumen Terkait**:
  - [PRD.md](../PRD.md)
  - [DATABASE.md](DATABASE.md)
  - [ARCHITECTURE.md](ARCHITECTURE.md)
  - [USER_ROLES.md](USER_ROLES.md)
  - [USER_FLOW.md](USER_FLOW.md)

---

## 1. Arsitektur Isolasi Multi-Tenant (Multi-Tenant Data Isolation)

Dalam arsitektur *Shared Database, Shared Schema*, risiko kebocoran data antar-sekolah (*tenant data leak*) merupakan ancaman tingkat tertinggi (*Severity Critical*). SAE SMP menerapkan sistem pertahanan berlapis (*Defense-in-Depth*):

### 1.1 Isolasi pada Lapisan Routing & Middleware
1. **Resolusi Tenant Dini**: Setiap request HTTP wajib melewati middleware `IdentifyTenantByDomainOrHeader`. Jika subdomain atau header tidak terdaftar pada tabel `schools`, request langsung diputus dengan respon `404 Not Found` atau `400 Bad Request`.
2. **Konteks Aplikasi Singleton**: Tenant yang tervalidasi didaftarkan sebagai singleton `app('current_school')` di container Laravel.
3. **Pencocokan Afiliasi Pengguna**: Middleware `EnforceSchoolAffiliation` memastikan bahwa akun pengguna yang sedang login benar-benar terafiliasi dengan `app('current_school')` melalui tabel `school_user`.

### 1.2 Isolasi pada Lapisan ORM (Laravel Global Scopes)
1. **Trait `BelongsToSchool`**:
   - Seluruh model data operasional menerapkan trait ini.
   - Global Scope secara otomatis menambahkan filter `WHERE school_id = ?` ke seluruh operasi pembacaan, pengubahan, dan penghapusan.
2. **Larangan Keras `withoutGlobalScope(SchoolScope::class)`**:
   - Penghapusan scope hanya diizinkan untuk tugas Command CLI internal Super Admin (misal: agregasi sistem berkala) dan dilarang keras dipanggil di dalam Web Controller publik.
3. **Indeks Komposit Unik**: Integritas data dijamin pada level mesin database melalui `UNIQUE(school_id, ...)`.

---

## 2. Keamanan Kartu Pelajar & Pemindaian QR Code (HMAC-SHA256 Tokenization)

Untuk mencegah pemalsuan kartu pelajar (*forgery*), rekayasa foto, atau serangan replay (*replay attack*), QR code kartu pelajar tidak menyimpan informasi polos (seperti NISN atau ID mentah).

### 2.1 Struktur Payload QR Code
Format token QR terenkripsi:
```text
SAE-QR.{school_uuid}.{student_uuid}.{issued_timestamp}.{hmac_signature}
```

1. **Pembuatan Tanda Tangan Digital (Signature Generation)**:
   ```php
   $payload = "{$schoolUuid}:{$studentUuid}:{$issuedTimestamp}";
   $secretKey = config('app.qr_secret_salt') . $school->qr_encryption_salt;
   $hmac = hash_hmac('sha256', $payload, $secretKey);
   $qrContent = base64_url_encode("{$payload}:{$hmac}");
   ```
2. **Mekanisme Pencegahan Duplikasi & Replay**:
   - **Debounce Window**: Server menolak pindaian dari siswa yang sama dalam jeda waktu kurang dari 5 menit untuk mencegah double-tap/scanning loop.
   - **Regenerasi QR**: Jika kartu fisik siswa hilang, admin sekolah dapat menerbitkan ulang kartu baru. Tombol *Re-issue Card* akan memperbarui `issued_timestamp`, sehingga kartu fisik lama otomatis tidak valid (*revoked*).

---

## 3. Mitigasi Risiko OWASP Top 10

| Risiko OWASP | Deskripsi Ancaman | Tindakan Mitigasi SAE SMP |
|---|---|---|
| **A01: Broken Access Control** | Pengguna mengakses data sekolah atau rombel lain | - Isolasi tenant via `SchoolScope`<br>- Laravel Authorization Gate & Policies<br>- Pengecekan kepemilikan rombel pada setiap request |
| **A02: Cryptographic Failures** | Data sensitif (NIK, NISN, Password) bocor | - Password di-hash menggunakan algoritma **Bcrypt / Argon2id**<br>- Enkripsi NIK & kontak keluarga menggunakan AES-256-GCM via `encrypt()` Laravel<br>- HTTPS/TLS 1.3 wajib untuk seluruh kanal komunikasi |
| **A03: Injection** | Serangan SQL Injection, Command Injection, XSS | - 100% database query menggunakan Eloquent ORM atau PDO Parameterized Binding<br>- Blade templating otomatis melakukan HTML entity encoding (`{{ }}`)<br>- Sanitasi berkas unggahan dan pelarangan eksekusi skrip |
| **A04: Insecure Design** | Alur presensi atau koreksi nilai dimanipulasi | - Pemisahan ketat hak input draf (Sekretaris) dan verifikasi resmi (Guru)<br>- Log imutabel yang tidak dapat ditimpa untuk setiap koreksi presensi |
| **A05: Security Misconfiguration** | Debug bar terbuka di produksi, port terekspos | - `APP_DEBUG=false` wajib pada lingkungan production<br>- Header keamanan HTTP ketat: HSTS, X-Frame-Options, X-Content-Type-Options, Content-Security-Policy (CSP) |
| **A06: Vulnerable Components** | Paket dependensi memiliki kerentanan | - Audit dependensi berkala dengan `composer audit` dan `npm audit`<br>- GitHub Dependabot alerts aktif |
| **A07: Identification & Auth Failures** | Pembajakan sesi, serangan Brute Force | - Rate limiting pada endpoint login (`throttle:5,1`)<br>- Session regeneration otomatis setelah login untuk mencegah Session Fixation<br>- Token Sanctum dengan masa kedaluwarsa ketat untuk Kiosk Gerbang |
| **A08: Software & Data Integrity Failures** | Integritas arsip dokumen digital dimanipulasi | - Setiap dokumen (Rapor, Ijazah) disimpan bersama nilai hash **SHA-256**<br>- Verifikasi hash saat berkas diunduh kembali |
| **A09: Security Logging & Monitoring Failures** | Insiden peretasan tidak terdeteksi | - Tabel `audit_logs` dan `security_incidents` mencatat aktivitas mencurigakan secara real-time<br>- Alert otomatis ke administrator jika terjadi anomali scan massal |
| **A10: Server-Side Request Forgery (SSRF)** | Integrasi Dapodik mengeksekusi request berbahaya | - Validasi IP lokal Dapodik terhadap subnet private yang diizinkan<br>- Timeout ketat (10 detik) dan penolakan skema selain HTTP/HTTPS |

---

## 4. Kepatuhan Privasi Data Siswa (UU PDP No. 27 Tahun 2022)

Sebagai platform pendidikan yang mengelola data anak di bawah umur, SAE SMP mematuhi regulasi Pelindungan Data Pribadi:

1. **Prinsip Minimisasi Data**: Sistem hanya menyimpan data yang diwajibkan oleh regulasi Kemendikbudristek dan operasional KBM.
2. **Persetujuan Pemrosesan**: Orang tua menyetujui syarat & ketentuan saat pertama kali mengaktifkan portal PWA.
3. **Pemberian Akses Berbatas**:
   - Siswa hanya dapat melihat data dirinya sendiri.
   - Orang tua hanya dapat melihat data anak-anaknya yang sah secara administratif (`student_parent`).
   - Guru hanya dapat mengakses daftar siswa di rombel atau mata pelajaran yang diampunya.
4. **Penyimpanan Berkas Digital Pribadi**:
   - Berkas kartu keluarga, akta kelahiran, dan surat izin dokter disimpan pada direktori privat (*private storage*).
   - Akses unduhan dikawal melalui Controller ber-otorisasi (*Signed URLs* atau Controller Stream).

---

## 5. Strategi Log Audit Imutabel (Immutable Audit Strategy)

Untuk menjamin akuntabilitas hukum dan integritas data sekolah:

1. **Jejak Audit Khusus Presensi (`daily_attendance_logs` & `subject_attendance_logs`)**:
   - Berisi status lama, status baru, alasan pengubahan, nama pengguna pengubah, alamat IP, dan tautan dokumen bukti.
   - Tabel ini bersifat **Append-Only** (hanya mendukung `INSERT`). Tidak ada fitur `UPDATE` atau `DELETE` pada level aplikasi untuk tabel ini.
2. **Log Audit Global (`audit_logs`)**:
   - Mencatat seluruh peristiwa krusial: pembuatan akun, perubahan nilai rapor, perubahan jadwal pelajaran, dan penarikan data Dapodik.
   - Menyimpan *snapshot* data sebelum dan sesudah perubahan dalam format JSON.
3. **Pencatatan Insiden Keamanan (`security_incidents`)**:
   - Otomatis mencatat percobaan login gagal berturut-turut, pemindaian QR code palsu, atau upaya pengaksesan data lintas tenant (*unauthorized tenant access attempt*).

---

## 6. Prosedur Tanggap Insiden Keamanan (Incident Response Plan)

Jika terjadi anomali atau insiden keamanan:
1. **Deteksi & Isolasi**:
   - Sistem otomatis memblokir token atau sesi pengguna yang melakukan aktivitas berbahaya.
   - Akun atau terminal gerbang yang terinfeksi dapat dinonaktifkan seketika oleh Admin Sekolah.
2. **Investigasi**:
   - Tim teknis menelusuri data pada tabel `audit_logs` dan `security_incidents` berdasarkan rentang waktu dan `school_id`.
3. **Pemulihan & Laporan**:
   - Data dipulihkan dari cadangan harian terenkripsi jika terjadi kerusakan.
   - Administrator sekolah menerbitkan laporan evaluasi keamanan kepada pihak berwenang.
