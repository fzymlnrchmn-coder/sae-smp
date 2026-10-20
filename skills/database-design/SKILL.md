---
name: database-design
description: "Use when: designing, reviewing, or optimizing relational database schemas, indexes, foreign keys, and multi-tenant constraints for SAE SMP."
---

# Database Design Skill — SAE SMP

Panduan arsitektur basis data, pemodelan relasional, standar indeks, dan aturan multi-tenant untuk Smart Application Education Platform (SAE SMP).

---

## 1. Standar Konvensi Skema Relasional

1. **Mesin Penyimpanan & Karakter**:
   - Engine: `InnoDB`.
   - Charset & Collation: `utf8mb4` dengan `utf8mb4_unicode_ci` (mendukung karakter nama daerah, simbol, dan emoji).
2. **Kunci Utama (Primary Key)**:
   - Gunakan `id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY` pada setiap tabel data.
   - Kolom UUID unik (`uuid CHAR(36)`) disediakan jika tabel memerlukan rujukan publik yang tidak dapat ditebak (*non-sequential public identifier*).
3. **Kunci Asing (Foreign Keys)**:
   - Nama kolom: `{singular_table}_id` (contoh: `school_id`, `student_id`, `classroom_id`).
   - Tipe data: `BIGINT UNSIGNED`.
   - Konvensi onDelete:
     - Gunakan `ON DELETE RESTRICT` untuk entitas master penting (mencegah penghapusan sekolah/siswa yang memiliki data riwayat).
     - Gunakan `ON DELETE CASCADE` hanya untuk entitas turunan murni (contoh: `daily_attendance_logs` terhadap `daily_attendances`).

---

## 2. Aturan Wajib Multi-Tenant (`school_id`)

Setiap tabel yang menyimpan data spesifik sekolah **wajib** memiliki kolom `school_id`:

```sql
school_id BIGINT UNSIGNED NOT NULL,
CONSTRAINT fk_table_school_id FOREIGN KEY (school_id) REFERENCES schools(id) ON DELETE RESTRICT
```

### Aturan Indeks Komposit Tenant
Untuk memastikan integritas unik per sekolah (bukan unik global) dan performa kueri yang optimal:
- **Kombinasi Unik**:
  ```sql
  -- BENAR: NISN unik dalam lingkup satu sekolah
  UNIQUE KEY uq_students_school_nisn (school_id, nisn)
  
  -- SALAH: NISN unik global dapat menyebabkan kegagalan pendaftaran siswa mutasi antar sekolah
  UNIQUE KEY uq_students_nisn (nisn)
  ```
- **Urutan Kolom Indeks**: Selalu letakkan `school_id` sebagai kolom pertama pada indeks majemuk:
  ```sql
  INDEX idx_daily_attendances_query (school_id, attendance_date, final_status)
  ```

---

## 3. Pola Log Imutabel (Append-Only Strategy)

Untuk data transaksi presensi dan audit finansial/akademik:
1. **Dilarang Menimpa Waktu Asli**: Kolom `check_in_time`, `check_out_time`, dan `raw_method` pada tabel `daily_attendances` merupakan fakta fisik pemindaian yang tidak boleh diubah nilainya.
2. **Tabel Log Pendamping**:
   - Setiap mutasi status melalui mekanisme manual wajib mencatat baris baru pada tabel log (`daily_attendance_logs`).
   - Skema tabel log:
     ```sql
     CREATE TABLE daily_attendance_logs (
         id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
         school_id BIGINT UNSIGNED NOT NULL,
         daily_attendance_id BIGINT UNSIGNED NOT NULL,
         previous_status VARCHAR(20) NULL,
         new_status VARCHAR(20) NOT NULL,
         change_reason TEXT NOT NULL,
         attachment_path VARCHAR(255) NULL,
         changed_by_user_id BIGINT UNSIGNED NOT NULL,
         created_at TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP,
         CONSTRAINT fk_dal_attendance FOREIGN KEY (daily_attendance_id) REFERENCES daily_attendances(id) ON DELETE CASCADE,
         CONSTRAINT fk_dal_school FOREIGN KEY (school_id) REFERENCES schools(id) ON DELETE RESTRICT
     ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
     ```

---

## 4. Tipe Data Standar Proyek

| Kebutuhan Data | Tipe Data yang Dianjurkan | Alasan Pemilihan |
|---|---|---|
| ID / Relasi | `BIGINT UNSIGNED` | Skalabilitas jutaan baris data presensi tahunan. |
| UUID Identifiers | `CHAR(36)` | Panjang tetap, efisiensi hashing string UUID v4. |
| Status Presensi | `VARCHAR(20)` / `TINYINT` | Fleksibilitas kode status ('H', 'T', 'I', 'S', 'A'). |
| Nilai / Skor Asesmen | `DECIMAL(5,2)` | Akurasi nilai angka tanpa galat floating point (misal: 87.50). |
| Bendera Boolean | `BOOLEAN` (TINYINT(1)) | Standar kompatibilitas Laravel & MySQL. |
| Tanggal | `DATE` | Format standar ISO-8601 `YYYY-MM-DD`. |
| Waktu Presensi | `TIME` | Format `HH:MM:SS` presisi detik. |
| Hash / Token QR | `VARCHAR(128)` | Cukup untuk digest HMAC-SHA256 (64 hex characters). |
| JSON Metadata | `JSON` | Fleksibilitas data dinamis (misal: Dapodik raw payload). |

---

## 5. Checklist Verifikasi Skema Baru

Sebelum menambahkan tabel atau kolom baru:
- [ ] Apakah tabel memerlukan `school_id`? (Ya, untuk semua tabel data sekolah).
- [ ] Apakah sudah ada foreign key constraint dengan tindakan `ON DELETE` yang aman?
- [ ] Apakah kueri pencarian tersering sudah dilindungi indeks komposit dengan prefix `school_id`?
- [ ] Apakah ada potensi data hilang akibat hard delete? (Gunakan soft delete `deleted_at` untuk data master).
- [ ] Apakah penamaan kolom mengikuti konvensi snake_case bahasa Inggris (`student_id`, bukan `id_siswa`)?
