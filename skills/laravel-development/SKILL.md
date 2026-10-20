---
name: laravel-development
description: "Use when: implementing Laravel backend features, building Actions, FormRequests, Domain Services, Eloquent Models with multi-tenant scopes, or Livewire components in SAE SMP."
---

# Laravel Development Skill — SAE SMP

Panduan implementasi teknis dan standar kode pengembangan backend Laravel 11/12 (PHP 8.2+) untuk proyek Smart Application Education Platform (SAE SMP).

## 1. Aturan Dasar & Standar Penulisan Kode

1. **Deklarasi Tipe Ketat (Strict Types)**:
   - Setiap file PHP wajib diawali `declare(strict_types=1);`.
   - Gunakan type hinting eksplisit pada setiap parameter fungsi dan return type.
2. **Pola Action-Domain (Single Responsibility)**:
   - Hindari Controller gemuk (*Fat Controllers*) dan Model gemuk (*Fat Models*).
   - Logika bisnis utama ditempatkan pada kelas Action di `app/Actions/{Domain}/`.
   - Setiap Action hanya memiliki satu metode publik: `execute()`.
3. **Validasi FormRequest Terpusat**:
   - Seluruh validasi input HTTP ditangani oleh kelas `FormRequest` khusus di `app/Http/Requests/{Domain}/`.
   - Metode `authorize()` pada FormRequest wajib memeriksa izin pengguna sebelum aturan validasi dieksekusi.

---

## 2. Pola Implementasi Multi-Tenancy

Setiap model operasional yang menyimpan data sekolah wajib menerapkan Trait `BelongsToSchool`:

```php
declare(strict_types=1);

namespace App\Models\Traits;

use App\Models\Scopes\SchoolScope;
use App\Models\School;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

trait BelongsToSchool
{
    protected static function bootBelongsToSchool(): void
    {
        static::addGlobalScope(new SchoolScope());

        static::creating(function ($model): void {
            if (! $model->school_id && app()->bound('current_school')) {
                $model->school_id = app('current_school')->id;
            }
        });
    }

    public function school(): BelongsTo
    {
        return $this->belongsTo(School::class);
    }
}
```

Implementasi Global Scope `SchoolScope`:

```php
declare(strict_types=1);

namespace App\Models\Scopes;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Scope;

class SchoolScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        if (app()->bound('current_school')) {
            $builder->where($model->getTable() . '.school_id', app('current_school')->id);
        }
    }
}
```

---

## 3. Pola Pembuatan Action (Action Pattern)

Contoh struktur kelas Action untuk transaksi presensi gerbang siswa:

```php
declare(strict_types=1);

namespace App\Actions\Attendance;

use App\Models\DailyAttendance;
use App\Models\Student;
use App\Services\QrSecurityService;
use App\Events\StudentCheckedInEvent;
use Illuminate\Support\Facades\DB;
use Illuminate\Validation\ValidationException;
use Carbon\Carbon;

class ProcessGateScanAction
{
    public function __construct(
        protected QrSecurityService $qrService
    ) {}

    public function execute(string $qrPayload, int $checkpointId, ?int $officerUserId = null): DailyAttendance
    {
        // 1. Verifikasi tanda tangan kriptografis QR
        $tokenData = $this->qrService->verifyAndDecode($qrPayload);
        if (! $tokenData) {
            throw ValidationException::withMessages(['qr_code' => 'QR Code tidak valid atau telah kedaluwarsa.']);
        }

        return DB::transaction(function () use ($tokenData, $checkpointId, $officerUserId): DailyAttendance {
            $student = Student::findOrFail($tokenData->studentId);
            $today = Carbon::today();
            $now = Carbon::now();

            // 2. Cek apakah sudah pernah scan hari ini
            $attendance = DailyAttendance::firstOrNew([
                'student_id' => $student->id,
                'attendance_date' => $today->toDateString(),
            ]);

            // 3. Debounce scan berulang (< 5 menit)
            if ($attendance->exists && $attendance->check_in_time && $now->diffInMinutes($attendance->check_in_time) < 5) {
                return $attendance;
            }

            // 4. Hitung keterlambatan berdasarkan jam toleransi sekolah
            $schoolConfig = app('current_school');
            $toleranceTime = Carbon::parse($schoolConfig->entry_tolerance_time ?? '07:00:00');
            $isLate = $now->greaterThan($toleranceTime);
            $lateMinutes = $isLate ? (int) $now->diffInMinutes($toleranceTime) : 0;

            if (! $attendance->exists) {
                $attendance->check_in_time = $now->toTimeString();
                $attendance->raw_method = 'qr_kiosk';
                $attendance->final_status = $isLate ? 'T' : 'H';
                $attendance->late_minutes = $lateMinutes;
                $attendance->checkpoint_id = $checkpointId;
                $attendance->save();

                // 5. Catat log imutabel
                $attendance->logs()->create([
                    'school_id' => $attendance->school_id,
                    'previous_status' => null,
                    'new_status' => $attendance->final_status,
                    'change_reason' => 'Pindaian QR gerbang otomatis.',
                    'changed_by_user_id' => $officerUserId,
                ]);

                // 6. Picu event notifikasi asinkron
                event(new StudentCheckedInEvent($attendance));
            } elseif (! $attendance->check_out_time) {
                // Skenario scan pulang
                $attendance->check_out_time = $now->toTimeString();
                $attendance->save();
            }

            return $attendance;
        });
    }
}
```

---

## 4. Pola Controller Ramping (Thin Controller)

```php
declare(strict_types=1);

namespace App\Http\Controllers\Api\Attendance;

use App\Actions\Attendance\ProcessGateScanAction;
use App\Http\Controllers\Controller;
use App\Http\Requests\Attendance\GateScanRequest;
use Illuminate\Http\JsonResponse;

class GateScanController extends Controller
{
    public function __invoke(GateScanRequest $request, ProcessGateScanAction $action): JsonResponse
    {
        $attendance = $action->execute(
            qrPayload: $request->validated('qr_payload'),
            checkpointId: (int) $request->validated('checkpoint_id'),
            officerUserId: $request->user()?->id
        );

        return response()->json([
            'status' => 'success',
            'message' => 'Presensi berhasil dicatat.',
            'data' => [
                'student_name' => $attendance->student->full_name,
                'nisn' => $attendance->student->nisn,
                'status' => $attendance->final_status,
                'time' => $attendance->check_in_time,
                'late_minutes' => $attendance->late_minutes,
            ],
        ]);
    }
}
```

---

## 5. Konvensi Penamaan (Naming Conventions)

- **Models**: PascalCase tunggal (`School`, `DailyAttendance`, `ClassJournal`).
- **Migrations**: snake_case jamak (`create_daily_attendances_table`).
- **Actions**: PascalCase dengan akhiran `Action` (`AdjustDailyAttendanceAction`).
- **Services**: PascalCase dengan akhiran `Service` (`DapodikSyncService`).
- **Events / Listeners**: PascalCase kejadian lampau (`StudentCheckedInEvent`, `SendParentNotificationListener`).
- **Controllers**: PascalCase berakhiran `Controller` (`DailyAttendanceController`).
