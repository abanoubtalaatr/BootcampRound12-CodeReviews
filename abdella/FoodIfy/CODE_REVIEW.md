# Foodify — Code Review & Architecture Guide

> **الهدف:** Controllers رفيعة تمامًا — بدون validation وبدون business logic.  
> كل المنطق في Actions / Services / Repositories مع تطبيق SOLID في كل الطبقات.

**تاريخ المراجعة:** 2 يوليو 2026   (تحديث — Feature Completeness)
**الطالب:** Abdella (Abdallah Hefzy)  
**المستودع:** [Abdallah-Hefzy/Foodify](https://github.com/Abdallah-Hefzy/Foodify)  
**النطاق:** `app/` — Auth module فقط (لا يوجد domain modules بعد)

---

## 1. ملخص تنفيذي

| المنطقة | الحالة | التقييم |
|---------|--------|---------|
| Auth Module | مُنفَّذ (functional prototype) | 🟡 يعمل لكن يحتاج refactor كامل |
| Form Requests | جزئي (2 من 7 flows) | 🟡 Login + Register فقط |
| Actions | غير موجود | 🔴 |
| Repositories | غير موجود | 🔴 |
| Services | غير موجود | 🔴 |
| DTOs | غير موجود | 🔴 |
| API Resources | غير موجود | 🔴 |
| `ApiResponse` Trait | `ApiTrait` بديل غير مطابق | 🟡 |
| Middleware مخصص | غير موجود | 🔴 |
| Feature Tests | غير موجود | 🔴 |
| SOLID Compliance | ضعيف | 🔴 |

**الخلاصة:** المشروع يقدّم auth flow كامل (register, login, logout, phone verification, forgot password) باستخدام Laravel Sanctum — وهذا إنجاز جيد كبداية. لكن **كل المنطق موجود داخل Controllers** مباشرة: validation، DB queries، OTP generation، token handling، و conditional responses. لا توجد طبقة Actions ولا Repositories ولا Services. المعمارية المستهدفة **غير مطبّقة**.

---

## 2. المعمارية المستهدفة (Target Architecture)

```
HTTP Request
    │
    ▼
Form Request          ← Validation + Authorization فقط
    │
    ▼
Controller            ← يستدعي Action/Service ويرجع Response فقط (zero logic)
    │
    ▼
Action / Service      ← Business Logic (use case واحد)
    │
    ▼
Repository            ← Database access فقط (عبر Interface)
    │
    ▼
Model / Eloquent
```

### الوضع الحالي vs المستهدف

```
المستهدف:  FormRequest → Controller → Action → Service → RepositoryInterface → Model
الحالي:    FormRequest → Controller ──────────────────────────────→ Model (Eloquent مباشرة)
           (جزئي)        (+ inline validate, OTP, hashing, tokens, if/else)
```

---

## 3. تحليل SOLID

### S — Single Responsibility Principle

| الملف | المشكلة |
|-------|---------|
| `LoginController` | credential check + token creation + response mapping |
| `RegisterController` | user creation + password hashing + token issuance |
| `ForgotPasswordController` | validation + OTP send/verify + token abilities + password reset |
| `PhoneVerificationController` | OTP generation + verification + persistence |

كل controller يملك **عدة مسؤوليات** — يجب تقسيمها إلى Actions منفصلة.

### O — Open/Closed Principle

- OTP logic مكرر في `PhoneVerificationController` و `ForgotPasswordController` — أي تغيير (مثلاً SMS gateway) يتطلب تعديل controllers متعددة.
- لا interfaces — لا يمكن إضافة cache layer أو mock في tests بدون تعديل الكود.

### L — Liskov Substitution Principle

- غير قابل للتطبيق حاليًا — لا توجد abstractions.

### I — Interface Segregation Principle

- غير قابل للتطبيق حاليًا — لا توجد interfaces.

### D — Dependency Inversion Principle

| الوضع الحالي | المطلوب |
|--------------|---------|
| Controllers تعتمد على `User` model مباشرة | تعتمد على `UserRepositoryInterface` |
| OTP generation داخل controllers | `OtpService` مُحقَن عبر interface |
| `AppServiceProvider::register()` فارغ | يربط كل Interface → Implementation |

---

## 4. مراجعة ملف بملف

### 4.1 Controllers — 🔴 يحتاج Refactor كامل

المشروع يستخدم **4 controllers منفصلة** بدل `AuthController` موحّد — التقسيم مقبول، لكن كل controller **سمين (fat)**.

#### `LoginController` — 🔴

```17:30:app/Http/Controllers/Auth/LoginController.php
    public function login(LoginRequest $request)
    {
        $user = User::where('phone', $request->phone)->first();

        if (! $user || ! Hash::check($request->password, $user->password)) {
            return $this->error([], 'username or password is wrong', 401);
        }
        $token = $user->createToken('foodify_auth_token')->plainTextToken;
        return $this->data([
            'user' => $user,
            'token' => $token,
        ], 'You have successfully logged in');
    }
```

| المخالفة | التفاصيل |
|----------|----------|
| DB access | `User::where()` مباشرة في controller |
| Business logic | credential check + `Hash::check()` |
| Token handling | `createToken()` في controller |
| Response mapping | `if/else` لاختيار error vs success |
| Raw model | يرجع `$user` بدون `UserResource` |
| رسالة خاطئة | `"username"` بينما الـ auth بالـ phone |

`logout()` (سطر 34–38): `$request->user()->currentAccessToken()->delete()` — DB logic في controller.

**المستهدف:**

```php
public function login(LoginRequest $request, LoginAction $action): JsonResponse
{
    return $this->loginResponse($action->execute($request->toDto()));
}

public function logout(LogoutAction $action): JsonResponse
{
    $action->execute(auth()->user());
    return $this->loggedOutResponse();
}
```

---

#### `RegisterController` — 🔴

```15:29:app/Http/Controllers/Auth/RegisterController.php
    public function register(RegisgterRequest $request)
    {
        $user = User::create([...]);
        $token = $user->createToken('foodify_auth_token')->plainTextToken;
        return $this->data(['user' => $user, 'token' => $token], ...);
    }
```

| المخالفة | التفاصيل |
|----------|----------|
| DB access | `User::create()` مباشرة |
| Double hashing risk | `Hash::make()` (سطر 21) + `'password' => 'hashed'` cast في Model (سطر 36) |
| Token handling | في controller |
| Typo في Request | `RegisgterRequest` بدل `RegisterRequest` |

---

#### `PhoneVerificationController` — 🔴 + 🐛 Bug حرج

```15:50:app/Http/Controllers/Auth/PhoneVerificationController.php
    public function sendVerificationOTP() { ... }  // لا Form Request

    public function verifyPhone(Request $request)
    {
        $request->validate([...]);           // validation في controller
        if ($request->otp_code != $user->otp_code) { ... }
        if (Carbon::now(...)->greaterThan(...)) {
            return $this->error(...);
        // ❌ BUG: قوس } مفقود — الكود التالي داخل if block بالخطأ
        $user->otp_code = null;
        ...
    }
```

**🐛 Bug حرج (P0):** السطر 42–44 — `if` block الـ expiry **ناقص `}`** بعد `return`. الكود من سطر 45–49 لن يُنفَّذ أبدًا في حالة النجاح، والملف قد يسبب **PHP parse error**.

| المخالفة | التفاصيل |
|----------|----------|
| Syntax bug | قوس `}` مفقود بعد expiry check |
| Inline validation | `$request->validate()` في controller |
| OTP logic | generation + verification + persistence |
| Security | `dev_otp` يُرجع في response (سطر 23) |
| No Form Request | `sendVerificationOTP()` بدون validation |

---

#### `ForgotPasswordController` — 🔴

| Method | المشاكل |
|--------|---------|
| `sendOTP` | inline validation (21–23), user lookup, OTP generation, `dev_otp` leak, user enumeration (404 "Wrong phone number") |
| `verifyOTP` | inline validation (43–46), duplicated OTP logic, token creation |
| `resetPassword` | inline validation (75–80), ability check في controller (84–86), `Hash::make` + hashed cast |

**إيجابية:** استخدام Sanctum token abilities (`password-reset`) — pattern أمني جيد، لكن يجب نقله لـ `ResetPasswordAction` + middleware.

---

### 4.2 Form Requests — 🟡 جزئي

| Request | الحالة | ملاحظات |
|---------|--------|---------|
| `LoginRequest` | ✅ | قواعد جيدة — لكن في `app/Http/Requests/` مش `Auth/` |
| `RegisgterRequest` | 🟡 | typo في الاسم + نفس مشكلة المسار |
| `SendOtpRequest` | 🔴 مفقود | validation في controller |
| `VerifyOtpRequest` | 🔴 مفقود | validation في controller |
| `VerifyPhoneRequest` | 🔴 مفقود | validation في controller |
| `ResetPasswordRequest` | 🔴 مفقود | validation في controller |

**تحسينات مطلوبة:**
- نقل كل Requests لـ `app/Http/Requests/Auth/`
- إصلاح typo: `RegisgterRequest` → `RegisterRequest`
- إضافة `toDto()` method في كل Request
- استخراج phone regex لـ custom rule مشتركة (مكرر 4 مرات)

---

### 4.3 Actions — 🔴 غير موجود

المجلد `app/Actions/Auth/` **غير موجود**. كل business logic في controllers.

**Actions مطلوبة:**

| Action | الغرض |
|--------|-------|
| `LoginAction` | credential check + token creation |
| `RegisterAction` | user creation + token |
| `LogoutAction` | revoke current token |
| `SendVerificationOtpAction` | generate + store OTP |
| `VerifyPhoneAction` | verify OTP + mark phone verified |
| `SendResetOtpAction` | send OTP for password reset |
| `VerifyResetOtpAction` | verify OTP + issue reset token |
| `ResetPasswordAction` | update password + revoke token |

---

### 4.4 Repositories — 🔴 غير موجود

لا يوجد `UserRepository` ولا `UserRepositoryInterface`.

**المطلوب:**

```php
// app/Contracts/Repositories/UserRepositoryInterface.php
interface UserRepositoryInterface
{
    public function create(array $data): User;
    public function findByPhone(string $phone): ?User;
    public function updateOtp(User $user, string $otp, Carbon $expiresAt): void;
    public function clearOtp(User $user): void;
    public function markPhoneVerified(User $user): void;
    public function updatePassword(User $user, string $password): void;
}
```

---

### 4.5 Services — 🔴 غير موجود

OTP logic مكرر بين `PhoneVerificationController` و `ForgotPasswordController`:

```php
$otp_code = random_int(1000, 9999);
$user->otp_code = $otp_code;
$user->otp_expires_at = Carbon::now('Africa/Cairo')->addMinutes(5);
$user->save();
```

**المطلوب:** `OtpService` + `TokenService` + `SmsGatewayInterface` (للمستقبل).

**ملاحظة إيجابية:** استخدام `random_int()` بدل `rand()` — صحيح أمنيًا ✅

---

### 4.6 `ApiTrait` — 🟡 يحتاج إعادة تسمية وتوحيد

**الموجود:** `app/traits/ApiTrait.php`  
**المطلوب:** `app/Traits/ApiResponse.php`

| المشكلة | التفاصيل |
|---------|----------|
| Namespace | `App\traits` → `App\Traits` (PSR-4) |
| Folder | `traits` lowercase → `Traits` |
| الاسم | `ApiTrait` → `ApiResponse` |
| Response shape | `{ message, error, data }` — مختلف عن المعيار `{ status, message, data, errors }` |
| Default error code | `404` (سطر 21) — غير مناسب لـ auth errors |

**إيجابية:** envelope موحّد للـ JSON responses ✅

**المطلوب:** إضافة response methods مثل `loginResponse()`, `otpSentResponse()`, `authenticatedResponse()` — الـ controller ينادي method جاهزة بدل `if/else`.

---

### 4.7 `User` Model — 🟡

```14:24:app/Models/User.php
#[Fillable(['name', 'email', 'password'])]
#[Hidden(['password', 'remember_token'])]
class User extends Authenticatable
{
    protected $fillable = [
        'name','phone','password','email'
    ];
```

| المشكلة | التفاصيل |
|---------|----------|
| Duplicate fillable | attribute `#[Fillable]` + `$fillable` property — `phone` ناقص من attribute |
| Missing hidden | `otp_code`, `otp_expires_at` غير مخفية — تسريب محتمل |
| Missing casts | `otp_expires_at`, `phone_verified_at` بدون `datetime` cast |
| Double hashing | `'password' => 'hashed'` cast + `Hash::make()` في controllers |

---

### 4.8 Middleware — 🔴

**الموجود:** `auth:sanctum` فقط على routes.

**المفقود:**
- `throttle` على OTP endpoints (brute-force risk على 4-digit OTP)
- `EnsurePhoneVerified` middleware
- Token ability middleware لـ password-reset (حاليًا inline في controller)
- توحيد error response format

---

### 4.9 `routes/api.php` — 🟡

```10:22:routes/api.php
Route::post('register', [RegisterController::class, 'register']);
Route::post('login', [LoginController::class, 'login']);
...
```

**إيجابيات:** Sanctum على protected routes ✅

**تحسينات:**
- `Route::prefix('v1')` للـ versioning
- `Route::prefix('auth')->group(...)` لتجميع auth routes
- `throttle:6,1` على OTP routes
- named routes للـ testing

---

### 4.10 `AppServiceProvider` — 🔴

```12:23:app/Providers/AppServiceProvider.php
    public function register(): void
    {
        //
    }
```

فارغ — لا DI bindings. يجب ربط interfaces عند إنشائها.

---

### 4.11 Database / Migrations — 🟡

```17:17:database/migrations/0001_01_01_000000_create_users_table.php
            $table->string('phone');
```

| المشكلة | التفاصيل |
|---------|----------|
| No unique index | `phone` بدون `unique()` — race condition مع validation |
| Plain OTP storage | `otp_code` مخزّن plain text — يجب hashing |
| Schema OK | `otp_code`, `otp_expires_at`, `phone_verified_at` موجودة ✅ |

---

### 4.12 Tests — 🔴

فقط scaffold tests (`ExampleTest.php`). لا feature tests لأي auth endpoint.

---

## 5. مشاكل أمنية (Security)

| # | المشكلة | الخطورة | الملف |
|---|---------|---------|-------|
| 1 | `dev_otp` في API response | 🔴 | `PhoneVerificationController:23`, `ForgotPasswordController:36` |
| 2 | OTP plain text في DB | 🟡 | migration |
| 3 | User enumeration | 🟡 | `ForgotPasswordController:28-29` — 404 يكشف إن الـ phone مش مسجّل |
| 4 | No rate limiting على OTP | 🔴 | `routes/api.php` |
| 5 | OTP fields not hidden | 🟡 | `User.php` |
| 6 | Syntax bug يمنع verify-phone | 🔴 | `PhoneVerificationController:42-44` |

---

## 6. ما هو جيد ✅

1. **Sanctum integration** — `HasApiTokens`, bearer token flow, logout يحذف current token.
2. **Password-reset token abilities** — `createToken(..., ['password-reset'])` + `tokenCan()` — pattern أمني ممتاز.
3. **Partial Form Request usage** — `LoginRequest` و `RegisgterRequest` بقواعد validation قوية (phone regex, strong password).
4. **Consistent API envelope** — `ApiTrait` يوفر `{ message, error, data }` موحّد.
5. **JSON exception handling** — `bootstrap/app.php` يُعدّ JSON errors لـ `api/*`.
6. **Secure OTP generation** — `random_int()` بدل `rand()`.
7. **Auth feature completeness** — register, login, logout, phone verification, forgot password, reset password كلها موجودة.
8. **Migration covers auth fields** — phone, OTP, verification timestamps.

---

## 7. خطة Refactor — Auth Module

### المرحلة 1 — Bugs & Security (فوري) 🔴

- [ ] إصلاح syntax bug في `PhoneVerificationController` (قوس `}` مفقود)
- [ ] إزالة/guard `dev_otp` في responses (`app()->environment('local')` فقط)
- [ ] إضافة `unique()` index على `phone` في migration
- [ ] إضافة `$hidden` لـ `otp_code`, `otp_expires_at`
- [ ] إضافة `throttle` middleware على OTP routes
- [ ] إصلاح typo `RegisgterRequest` → `RegisterRequest`

### المرحلة 2 — Architecture Skeleton 🟡

- [ ] إنشاء `app/Traits/ApiResponse.php` (migrate من `ApiTrait`)
- [ ] إنشاء `UserRepositoryInterface` + `UserRepository`
- [ ] إنشاء `app/Actions/Auth/` — Login, Register, Logout, OTP actions
- [ ] إنشاء DTOs — `LoginData`, `RegisterData`, etc.
- [ ] إنشاء Form Requests لكل flow في `app/Http/Requests/Auth/`
- [ ] Bind interfaces في `AppServiceProvider`
- [ ] إنشاء `UserResource` للـ API output
- [ ] Slim controllers: validate → DTO → action → response

### المرحلة 3 — Services & Quality 🟢

- [ ] `OtpService` — extract duplicated OTP logic
- [ ] `TokenService` — centralize token creation/revocation
- [ ] إزالة `Hash::make()` من controllers (اعتمد على `hashed` cast)
- [ ] Feature tests لكل auth endpoint
- [ ] Update `UserFactory` بـ `phone` field
- [ ] Route groups + named routes + API versioning

---

## 8. Controller المستهدف (مثال)

```php
<?php

namespace App\Http\Controllers\Api\Auth;

use App\Actions\Auth\LoginAction;
use App\Actions\Auth\LogoutAction;
use App\Actions\Auth\RegisterAction;
use App\Http\Controllers\Controller;
use App\Http\Requests\Auth\LoginRequest;
use App\Http\Requests\Auth\RegisterRequest;
use App\Traits\ApiResponse;
use Illuminate\Http\JsonResponse;

final class LoginController extends Controller
{
    use ApiResponse;

    public function login(LoginRequest $request, LoginAction $action): JsonResponse
    {
        return $this->loginResponse($action->execute($request->toDto()));
    }

    public function logout(LogoutAction $action): JsonResponse
    {
        $action->execute(auth()->user());

        return $this->loggedOutResponse();
    }
}
```

---

## 9. File-by-File Scorecard

| File | Thin Controller | Form Request | Action/Service | Repository | ApiResponse | Verdict |
|------|:-:|:-:|:-:|:-:|:-:|---------|
| `LoginController` | ❌ | جزئي | ❌ | ❌ | ApiTrait | 🔴 Fail |
| `RegisterController` | ❌ | جزئي | ❌ | ❌ | ApiTrait | 🔴 Fail |
| `PhoneVerificationController` | ❌ | ❌ | ❌ | ❌ | ApiTrait | 🔴 Fail + Bug |
| `ForgotPasswordController` | ❌ | ❌ | ❌ | ❌ | ApiTrait | 🔴 Fail |
| `LoginRequest` | — | ✅ | — | — | — | 🟡 Partial |
| `RegisgterRequest` | — | ✅ | — | — | — | 🟡 Typo |
| `User.php` | — | — | — | — | — | 🟡 Needs cleanup |
| `AppServiceProvider` | — | — | — | ❌ | — | 🔴 Empty |
| `ApiTrait.php` | — | — | — | — | Wrong name | 🟡 Partial |
| `routes/api.php` | — | — | — | — | — | 🟡 Basic |
| Tests | — | — | — | — | — | 🔴 Missing |

---

## 10. ملخص الأولويات (Action Items)

### 🔴 Critical (اعملها فورًا)

1. إصلاح **syntax bug** في `PhoneVerificationController` (قوس `}` مفقود)
2. إزالة **`dev_otp`** من production responses
3. إضافة **`throttle`** على OTP endpoints
4. إضافة **`unique()`** على `phone` column
5. إخفاء **`otp_code`** و `otp_expires_at` من User model

### 🟡 High (قبل بناء modules جديدة)

6. إنشاء **Actions layer** ونقل كل business logic من controllers
7. إنشاء **Repository Interface** + implementation + DI binding
8. إنشاء **Form Requests** لكل OTP/password flow
9. إعادة تسمية `ApiTrait` → **`ApiResponse`** + توحيد response format
10. إنشاء **`UserResource`** — لا ترجع raw model
11. إصلاح typo **`RegisgterRequest`** → `RegisterRequest`
12. إزالة **`Hash::make()`** من controllers (اعتمد على `hashed` cast)

### 🟢 Medium (أثناء التطوير)

13. DTOs لكل Action
14. `OtpService` + `TokenService`
15. Feature + Unit tests
16. Route versioning (`v1`) + named routes
17. Exception-based error handling بدل `if/else` في controllers
18. `SmsGatewayInterface` عند ربط SMS provider حقيقي

---

## 13. تقرير Feature Completeness — النواقص في الـ Application

> **مرجع المتطلبات:** Authentication, Profile, Cart, My Orders, Notifications, Favorites, Meals/Categories, Reset Password, Category Details, Meal Details, Settings, Payments/Checkout.

### 13.1 Feature Matrix

| # | Feature | الحالة | Route / Implementation | النواقص |
|---|---------|--------|------------------------|---------|
| 1 | **Authentication** | 🟡 **85%** | register, login, logout, verify-phone | **syntax bug** in PhoneVerificationController |
| 2 | **Reset Password** | 🟡 **80%** | send-otp, verify-otp, reset-password | dev_otp exposed |
| 3 | **Profile** | 🔴 **0%** | — | غير موجود |
| 4–13 | **Catalog → Orders** | 🔴 **0%** | — | **14 module غير مبني** |
| 14 | **Notifications** | 🔴 **0%** | — | Notifiable only |
| 15 | **Settings** | 🔴 **0%** | — | غير موجود |
| 16 | **Admin** | 🔴 **0%** | — | غير موجود |

**Overall Feature Completeness: ~10%** — Auth scaffold فقط.

### 13.2 Critical Bugs

| المشكلة | الملف |
|---------|-------|
| Missing `}` — verify-phone crashes | `PhoneVerificationController.php:42` |
| `dev_otp` in responses | ForgotPasswordController |
| Factory/seeder missing `phone` | tests fail |
| No unique on `users.phone` | migration |

### 13.3 Route Map

```
POST /api/register, /login, /logout              ✅
POST /api/send-otp, /verify-otp, /reset-password ✅
Everything else (catalog, cart, orders...)       ❌
```

### 13.4 Feature Completeness Scorecard

| Category | Score |
|----------|-------|
| Auth + Reset | ~83% |
| All other modules | 0% |
| **Overall** | **~10%** |

---

## 12. المراجع

- [Laravel Form Requests](https://laravel.com/docs/validation#form-request-validation)
- [Laravel Sanctum](https://laravel.com/docs/sanctum)
- [SOLID Principles in PHP](https://www.php.net/manual/en/language.oop5.basic.php)
- مراجعة Elham (نفس Bootcamp): [elham/FoodIfy/CODE_REVIEW.md](../elham/FoodIfy/CODE_REVIEW.md)

---

*هذا الملف مرجع حي — حدّثه مع كل refactor جديد.*
