---
name: testing
description: "Use when: writing, reviewing, or running PHPUnit or Pest automated tests, testing multi-tenant isolation, QR code validation, attendance logic, or Dapodik sync mocks in SAE SMP."
---

# Testing Skill — SAE SMP

Panduan standar pengujian otomatis (Automated Testing) menggunakan **PHPUnit / Pest PHP** untuk menjamin keandalan, keamanan multi-tenant, dan akurasi bisnis platform SAE SMP.

---

## 1. Prinsip & Sasaran Pengujian

1. **Jaminan Nol Kebocoran Tenant (Zero Tenant Leaks)**:
   - Setiap fitur yang berinteraksi dengan database wajib memiliki minimal satu pengujian isolasi tenant (*Cross-Tenant Access Test*).
2. **Pengujian Bisnis Kritis**:
   - Algoritma verifikasi token QR HMAC-SHA256.
   - Perhitungan keterlambatan dan status presensi gerbang.
   - Pendelegasian status draf jurnal KBM oleh sekretaris kelas.
   - Sinkronisasi idempoten penarikan data Dapodik.
3. **Pemberian Mock pada Layanan Eksternal**:
   - Dilarang memanggil server Dapodik sungguhan atau gateway WhatsApp dalam pengujian CI/CD. Gunakan `Http::fake()` atau mock class.

---

## 2. Pengujian Wajib Isolasi Multi-Tenant

Setiap Model atau Action baru harus diuji menggunakan pola berikut:

```php
declare(strict_types=1);

namespace Tests\Feature\Tenancy;

use App\Models\School;
use App\Models\Student;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class TenantIsolationTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_cannot_view_students_from_another_school(): void
    {
        // 1. Arrange: Buat dua sekolah berbeda
        $schoolA = School::factory()->create(['subdomain' => 'smp1']);
        $schoolB = School::factory()->create(['subdomain' => 'smp2']);

        $studentA = Student::factory()->create(['school_id' => $schoolA->id, 'full_name' => 'Budi Santoso']);
        $studentB = Student::factory()->create(['school_id' => $schoolB->id, 'full_name' => 'Siti Rahma']);

        $teacherA = User::factory()->create();
        $schoolA->users()->attach($teacherA->id, ['role' => 'teacher', 'is_active' => true]);

        // 2. Act: Login sebagai guru di Sekolah A
        $response = $this->actingAs($teacherA)
            ->withHeaders(['Host' => 'smp1.sae-smp.test'])
            ->getJson('/api/v1/students');

        // 3. Assert: Respon hanya berisi siswa dari Sekolah A
        $response->assertOk()
            ->assertJsonFragment(['full_name' => 'Budi Santoso'])
            ->assertJsonMissing(['full_name' => 'Siti Rahma']);
    }
}
```

---

## 3. Pengujian Keamanan & Pemindaian QR Code

Menguji ketahanan token terhadap pemalsuan (*tampering*) dan pencegahan scan berulang (*debounce*):

```php
declare(strict_types=1);

namespace Tests\Feature\Attendance;

use App\Models\School;
use App\Models\Student;
use App\Services\QrSecurityService;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class QrAttendanceTest extends TestCase
{
    use RefreshDatabase;

    public function test_rejects_counterfeit_qr_token(): void
    {
        $school = School::factory()->create(['qr_encryption_salt' => 'test-salt-secret']);
        $this->app->instance('current_school', $school);

        $fakePayload = base64_encode('fake-school-id:fake-student-id:2026-10-20:invalid-hmac-signature');

        $response = $this->postJson('/api/v1/attendance/gate-scan', [
            'qr_payload' => $fakePayload,
            'checkpoint_id' => 1,
        ]);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['qr_code']);
    }

    public function test_debounce_rapid_consecutive_scans(): void
    {
        $school = School::factory()->create(['entry_tolerance_time' => '07:00:00']);
        $this->app->instance('current_school', $school);

        $student = Student::factory()->create(['school_id' => $school->id]);
        $qrService = app(QrSecurityService::class);
        $validToken = $qrService->generateStudentToken($student);

        // Scan pertama: Sukses
        $res1 = $this->postJson('/api/v1/attendance/gate-scan', [
            'qr_payload' => $validToken,
            'checkpoint_id' => 1,
        ]);
        $res1->assertOk();

        // Scan kedua dalam interval 1 menit: Diabaikan tanpa membuat duplikasi baris
        $this->travel(1)->minutes();
        $res2 = $this->postJson('/api/v1/attendance/gate-scan', [
            'qr_payload' => $validToken,
            'checkpoint_id' => 1,
        ]);
        $res2->assertOk();

        $this->assertDatabaseCount('daily_attendances', 1);
    }
}
```

---

## 4. Pengujian Pendelegasian Pengurus Kelas & Jurnal KBM

```php
declare(strict_types=1);

namespace Tests\Feature\Journal;

use App\Models\Classroom;
use App\Models\Student;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class ClassOfficerJournalTest extends TestCase
{
    use RefreshDatabase;

    public function test_regular_student_cannot_draft_class_journal(): void
    {
        $studentUser = User::factory()->create();
        $student = Student::factory()->create(['user_id' => $studentUser->id]);
        $classroom = Classroom::factory()->create();

        // Siswa biasa (bukan pengurus kelas)
        $response = $this->actingAs($studentUser)
            ->postJson("/api/v1/classrooms/{$classroom->id}/journals/draft", [
                'teaching_material' => 'Membahas Bilangan Bulat',
            ]);

        $response->assertForbidden();
    }

    public function test_secretary_can_draft_but_cannot_finalize_journal(): void
    {
        $studentUser = User::factory()->create();
        $student = Student::factory()->create(['user_id' => $studentUser->id]);
        $classroom = Classroom::factory()->create();

        // Tunjuk siswa sebagai sekretaris
        $classroom->officers()->create([
            'school_id' => $classroom->school_id,
            'student_id' => $student->id,
            'role' => 'sekretaris_1',
            'is_active' => true,
        ]);

        // Berhasil membuat draf (is_draft = true)
        $resDraft = $this->actingAs($studentUser)
            ->postJson("/api/v1/classrooms/{$classroom->id}/journals/draft", [
                'teaching_material' => 'Membahas Bilangan Bulat',
            ]);
        $resDraft->assertCreated();
        $this->assertDatabaseHas('class_journals', [
            'teaching_material' => 'Membahas Bilangan Bulat',
            'is_draft' => true,
        ]);

        // Gagal mengesahkan jurnal resmi (harus dilakukan guru mapel)
        $journalId = $resDraft->json('data.id');
        $resFinal = $this->actingAs($studentUser)
            ->postJson("/api/v1/journals/{$journalId}/verify", []);
        $resFinal->assertForbidden();
    }
}
```

---

## 5. Menjalankan Suite Pengujian

```bash
# Menjalankan seluruh test suite
php artisan test

# Menjalankan pengujian isolasi tenant secara spesifik
php artisan test --filter=TenantIsolationTest

# Menguji cakupan kode (Code Coverage)
php artisan test --coverage --min=80
```
