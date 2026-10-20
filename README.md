# Smart Application Education Platform (SAE SMP)

[![PHP Version](https://img.shields.io/badge/PHP-8.2%2B-blue.svg)](https://php.net)
[![Laravel Framework](https://img.shields.io/badge/Laravel-11%2F12-red.svg)](https://laravel.com)
[![Architecture](https://img.shields.io/badge/Architecture-Multi--Tenant%20Modular%20Monolith-green.svg)](#arsitektur)
[![Status](https://img.shields.io/badge/Status-Design%20%26%20Documentation%20Phase-yellow.svg)](#dokumentasi)

**Smart Application Education Platform (SAE SMP)** adalah sistem informasi manajemen dan administrasi sekolah menengah pertama berbasis web yang dirancang khusus untuk mendukung **banyak sekolah (multi-tenant)** dengan isolasi data yang aman, presensi gerbang berbasis kartu pelajar QR Code, presensi per mata pelajaran independen, pendelegasian pengurus kelas, portal PWA orang tua, serta sinkronisasi satu arah resmi dengan aplikasi Dapodik lokal Kemendikbudristek.

---

## 📑 Daftar Isi

- [Fitur Utama](#-fitur-utama)
- [Arsitektur & Tumpukan Teknologi](#-arsitektur--tumpukan-teknologi)
- [Struktur Direktori Repositori](#-struktur-direktori-repositori)
- [Dokumentasi Lengkap](#-dokumentasi-lengkap)
- [Pedoman Pengembang & Agen](#-pedoman-pengembang--agen)
- [Instalasi & Persiapan Lingkungan](#-instalasi--persiapan-lingkungan)
- [Kontribusi & Lisensi](#-kontribusi--lisensi)

---

## 🚀 Fitur Utama

1. **Multi-Tenant Aman Berbasis `school_id`**:
   - Mendukung ratusan sekolah dalam satu instalasi dengan isolasi kueri otomatis melalui Laravel Global Scope (`SchoolScope`).
   - Dukungan subdomain sekolah (`smpn1cipanas.sae-smp.id`) dan domain khusus (*custom domain*).
2. **Presensi Siswa Dual-Layer**:
   - **Presensi Gerbang (Harian)**: Pemindaian kartu pelajar QR Code ber-tanda tangan HMAC-SHA256 untuk mendeteksi kedatangan dan kepulangan secara real-time.
   - **Presensi Mata Pelajaran (KBM)**: Pencatatan kehadiran independen di setiap jam pelajaran dengan deteksi anomali (misal: hadir di gerbang tetapi tidak masuk kelas).
3. **Log Imutabel & Koreksi Manual Akuntabel**:
   - Perubahan status kehadiran manual oleh wali kelas atau guru piket disimpan secara permanen (*append-only logs*) tanpa menghapus riwayat pindaian awal.
4. **Pendelegasian Tugas Pengurus Kelas (Sekretaris)**:
   - Pengurus kelas (siswa) dapat membuat draf jurnal KBM harian dan mencatat siswa yang absen, yang kemudian wajib diverifikasi dan disahkan oleh guru mata pelajaran pengampu.
5. **Portal Orang Tua (PWA) & Pusat Notifikasi In-App**:
   - Aplikasi PWA ringan bagi wali murid untuk memantau kehadiran anak secara real-time, mengajukan surat izin sakit online, serta menerima notifikasi yang tersimpan permanen di basis data.
6. **Sinkronisasi Resmi Satu Arah Dapodik (One-Way Pull)**:
   - Pengambilan data master sekolah, PTK (guru/staf), siswa, dan rombongan belajar langsung dari REST API Web Service Dapodik lokal sekolah secara idempoten tanpa mengubah basis data Dapodik.
7. **Asesmen Kurikulum Merdeka & E-Rapor**:
   - Pencatatan nilai formatif, sumatif materi, sumatif akhir, penyusunan deskripsi capaian tujuan pembelajaran (TP) otomatis, dan generasi cetak rapor PDF.
8. **Pengarsipan Dokumen Digital**:
   - Manajemen buku induk dan berkas siswa terenkripsi dengan pengujian integritas berkas menggunakan hash SHA-256.

---

## 🛠 Arsitektur & Tumpukan Teknologi

- **Back-End**: PHP 8.2+, Laravel 11 / 12 (Action-Domain Layered Architecture).
- **Front-End**: Blade Templating, Livewire 3, Alpine.js, Tailwind CSS.
- **PWA**: Service Worker, Web App Manifest, Web Push API.
- **Database**: MySQL 8.0+ / MariaDB 10.11+ (InnoDB, UTF-8 MB4, Shared DB with Tenant Discriminator).
- **Caching & Queue**: Redis (Session, Cache, Queue Worker untuk Notifikasi & Sinkronisasi).
- **Otorisasi**: Spatie Laravel-Permission dengan Multi-Tenant Scoping.
- **Keamanan**: HMAC-SHA256 QR Signature, Rate Limiting, CSP Headers, Audit Trail.

---

## 📂 Struktur Direktori Repositori

```text
SMP-EDUCATION-SYSTEM/
├── README.md               # Ringkasan proyek, instalasi & arsitektur (file ini)
├── AGENTS.md               # Pedoman instruksi dan aturan kerja agen AI (Copilot/Claude)
├── PRD.md                  # Product Requirements Document resmi (kebutuhan fungsional)
├── docs/                   # Dokumentasi teknis mendalam
│   ├── ARCHITECTURE.md     # Arsitektur sistem, modular monolith & integrasi
│   ├── DATABASE.md         # Rancangan 47 tabel, Mermaid ERD & relasi entitas
│   ├── USER_ROLES.md       # Matriks peran, izin Spatie & aturan pendelegasian
│   ├── USER_FLOW.md        # Diagram alur interaksi pengguna & sequence diagram
│   ├── SECURITY.md         # Kebijakan isolasi data, enkripsi & kepatuhan UU PDP
│   └── ROADMAP.md          # 5 fase pengembangan, jadwal & mitigasi risiko
└── skills/                 # Pengetahuan spesialisasi & aturan rekayasa perangkat lunak
    ├── laravel-development/# Pola Action-Domain, Form Request, Service & Global Scope
    │   └── SKILL.md
    ├── database-design/    # Prinsip normalisasi, skema multi-tenant & indeks komposit
    │   └── SKILL.md
    └── testing/            # Standar pengujian isolasi tenant, kueri & keamanan
        └── SKILL.md
```

---

## 📚 Dokumentasi Lengkap

Dokumentasi proyek disusun secara komprehensif pada direktori `docs/`:

1. [PRD.md](PRD.md): Dokumen Kebutuhan Produk (Functional & Non-Functional Requirements).
2. [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md): Arsitektur Sistem, Desain Modular & Pipeline Middleware.
3. [docs/DATABASE.md](docs/DATABASE.md): Arsitektur Basis Data, Skema 47 Tabel & Diagram Relasi ERD.
4. [docs/USER_ROLES.md](docs/USER_ROLES.md): Matriks Hak Akses RBAC & Peran Pengguna.
5. [docs/USER_FLOW.md](docs/USER_FLOW.md): Diagram Alur Pengguna & Interaksi Sistem.
6. [docs/SECURITY.md](docs/SECURITY.md): Keamanan Multi-Tenant, Enkripsi QR & Kepatuhan Privasi.
7. [docs/ROADMAP.md](docs/ROADMAP.md): Peta Jalan Pengembangan 5 Fase & Jadwal Peluncuran.

---

## 🤖 Pedoman Pengembang & Agen

Bagi pengembang dan asisten AI, harap mematuhi tata tertib teknis berikut sebelum menyentuh kode program:
- Baca [PRD.md](PRD.md) dan [docs/DATABASE.md](docs/DATABASE.md) sebelum memodifikasi logika bisnis.
- Terapkan pola **Action-Domain** (jangan menaruh logika bisnis pada Controller atau Model).
- Pastikan seluruh tabel operasional menyertakan `school_id` dan Model mengimplementasikan `BelongsToSchool`.
- Dilarang membuat file migrasi atau kode aplikasi sebelum seluruh arsitektur dan dokumentasi disetujui.

---

## 💻 Instalasi & Persiapan Lingkungan

*(Catatan: Tahap instalasi aplikasi akan dibuka saat Fase 1 kode dimulai)*

### Prasyarat Sistem
- PHP >= 8.2 (ekstensi: `pdo_mysql`, `redis`, `gd`, `bcmath`, `intl`, `zip`)
- Composer >= 2.6
- Node.js >= 20.x & NPM
- MySQL Server >= 8.0 atau MariaDB >= 10.11
- Redis Server (disarankan)

### Langkah Awal
```bash
# Kloning repositori
git clone <repository-url> sae-smp
cd sae-smp

# Salin konfigurasi lingkungan
cp .env.example .env

# Pasang dependensi PHP & Javascript
composer install
npm install
```

---

## 📄 Lisensi

Hak Cipta © 2026 Smart Application Education Platform (SAE SMP). Seluruh hak cipta dilindungi undang-undang.
