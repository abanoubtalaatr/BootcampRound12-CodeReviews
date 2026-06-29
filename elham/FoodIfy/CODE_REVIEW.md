# FoodIfy — Code Review & Architecture Guide

> **الهدف:** Controllers رفيعة تمامًا — بدون validation وبدون business logic.  
> كل المنطق في Actions / Services / Repositories مع تطبيق SOLID في كل الطبقات.

**تاريخ المراجعة:** 29 يونيو 2026  
**النطاق:** `app/` — Auth (مُنفَّذ) + باقي الـ modules (scaffold فقط)

---

## 1. ملخص تنفيذي

| المنطقة | الحالة | التقييم |
|---------|--------|---------|
| Auth Module | مُنفَّذ جزئيًا | 🟡 جيد كبداية، يحتاج refactor |
| Form Requests (Auth) | موجود | ✅ |
| Actions (Auth) | موجود | 🟡 يحتاج DTOs وتحسينات |
| Repositories | `UserRepository` فقط | 🔴 بدون Interface |
| Services | `OtpService` فقط | 🟡 بدون Interface |
| باقي Controllers | scaffold فارغ | 🔴 غير جاهز للتنفيذ |
| Models (Meal, Order, …) | فارغة | 🔴 |
| SOLID Compliance | جزئي | 🟡 |

**الخلاصة:** مسار Auth صحيح في الفكرة (Request → Action → Response)، لكن الـ `AuthController` ما زال فيه **conditional logic** و**token handling** يجب نقلهم. باقي الـ controllers لو اتنفّذت بالشكل الحالي (`Request $request` + logic داخل الـ method) هتكسر المعمارية المطلوبة.

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

### قواعد الـ Controller (إلزامية)

```php
// ✅ Controller مثالي — لا if/else، لا validation، لا DB، لا Hash
public function store(StoreMealRequest $request, StoreMealAction $action): JsonResponse
{
    return $this->respond(
        $action->execute($request->toDto())
    );
}

// ❌ ممنوع في Controller
$request->validate([...]);           // validation
if (!$user) { ... }                  // business logic
User::create($data);                 // database access
Hash::make($password);               // transformation logic
$result['reason'] === 'inactive'    // decision mapping
```

### هيكل المجلدات المقترح

```
app/
├── Actions/
│   ├── Auth/
│   ├── Meal/
│   ├── Order/
│   └── Cart/
├── Contracts/                    ← Interfaces (DIP)
│   ├── Repositories/
│   └── Services/
├── DTOs/                         ← Data Transfer Objects
│   ├── Auth/
│   └── Meal/
├── Enums/
├── Exceptions/                   ← Domain exceptions
│   └── Auth/
├── Http/
│   ├── Controllers/Api/          ← PascalCase: Api وليس api
│   ├── Middleware/
│   ├── Requests/
│   │   ├── Auth/
│   │   ├── Meal/
│   │   └── Order/
│   └── Resources/
├── Models/
├── Repositories/
│   └── Eloquent/
├── Services/
└── Support/
    └── Responders/               ← يحوّل Action Result → JsonResponse
```

---

## 3. تحليل SOLID

### S — Single Responsibility Principle

| الملف | المشكلة | الحل |
|-------|---------|------|
| `AuthController` | يقرر الرسائل، status codes، ويحذف tokens | `AuthResponder` + `LogoutAction` |
| `OtpService` | يرسل SMS + يخزّن OTP + يتحقق + يدير reset state | تقسيم: `OtpGenerator`, `OtpStorage`, `SmsGateway` |
| `LoginAction` | login + token revocation + token creation | `TokenService` منفصل |

### O — Open/Closed Principle

| المشكلة | الحل |
|---------|------|
| `OtpService` مربوط بـ Vonage مباشرة | `SmsGatewayInterface` + `VonageSmsGateway` |
| `UserRepository` concrete class | `UserRepositoryInterface` — تقدر تضيف Redis cache layer بدون تعديل Actions |

### L — Liskov Substitution Principle

- أي `Repository` يطبّق الـ Interface لازم يكون قابل للاستبدال في الـ tests بـ `InMemoryUserRepository`.
- الـ Actions تعتمد على الـ Interface مش الـ implementation.

### I — Interface Segregation Principle

```php
// ❌ Interface ضخم
interface UserRepositoryInterface {
    public function create(...);
    public function findByPhone(...);
    public function updatePassword(...);
    public function getOrders(...);      // مش مسؤولية User repo
}

// ✅ Interfaces صغيرة ومركّزة
interface CreatesUsers { public function create(array $data): User; }
interface FindsUsersByPhone { public function findByPhone(string $phone): ?User; }
```

> **عمليًا:** interface واحد لكل Repository كافي في مشروع بحجم FoodIfy، لكن لا تخلط مسؤوليات (مثلاً Order logic داخل UserRepository).

### D — Dependency Inversion Principle

| الوضع الحالي | المطلوب |
|--------------|---------|
| Actions تعتمد على `UserRepository` (concrete) | تعتمد على `UserRepositoryInterface` |
| `OtpService` ينشئ `Vonage\Client` داخليًا | يُحقَن عبر `SmsGatewayInterface` |
| `AppServiceProvider::register()` فارغ | يربط كل Interface → Implementation |

```php
// AppServiceProvider.php
$this->app->bind(UserRepositoryInterface::class, UserRepository::class);
$this->app->bind(SmsGatewayInterface::class, VonageSmsGateway::class);
```

---

## 4. مراجعة ملف بملف

### 4.1 `AuthController` — 🟡 يحتاج Refactor

**الإيجابيات:**
- يستخدم `FormRequest` للـ validation ✅
- يحقن `Action` classes ✅
- يستخدم `UserResource` ✅

**المخالفات (Logic داخل Controller):**

| Method | المشكلة | السطر |
|--------|---------|-------|
| `verifyRegister` | `if (!$result['verified'])` + mapping `reason` → message | 41–47 |
| `login` | `if (!$result['success'])` + status code decision | 61–68 |
| `verifyResetOtp` | `if (!$action->execute(...))` | 88–90 |
| `resetPassword` | `if (!$action->execute(...))` | 98–100 |
| `logout` | `$request->user()->currentAccessToken()->delete()` — DB logic | 108 |
| `me` | مقبول تقريبًا، لكن الأفضل `GetAuthenticatedUserAction` | 114–118 |

**الشكل المستهدف:**

```php
public function login(LoginRequest $request, LoginAction $action): JsonResponse
{
    return $this->authResponder->login($action->execute($request->toDto()));
}

public function logout(LogoutAction $action): JsonResponse
{
    return $this->authResponder->logout($action->execute(auth()->user()));
}
```

**ملاحظة:** الـ `if/else` واختيار الـ HTTP status code ينتقلوا لـ:
- **Option A:** `AuthResponder` (طبقة presentation)
- **Option B:** Domain Exceptions (`InvalidCredentialsException` → Handler يحوّلها لـ 401)

---

### 4.2 Form Requests (Auth) — ✅ جيد

| Request | الحالة | ملاحظات |
|---------|--------|---------|
| `RegisterRequest` | ✅ | أضف `messages()` و `attributes()` للـ API errors |
| `LoginRequest` | ✅ | — |
| `ForgetPasswordRequest` | ✅ | `exists:users,phone` — صح |
| `VerifyOtpRequest` | ✅ | — |
| `ResetPasswordRequest` | ✅ | — |

**تحسين مقترح — DTO method في كل Request:**

```php
// RegisterRequest.php
public function toDto(): RegisterData
{
    return new RegisterData(
        fullName: $this->validated('full_name'),
        phone: $this->validated('phone'),
        email: $this->validated('email'),
        password: $this->validated('password'),
        address: $this->validated('address'),
        birthDate: $this->validated('birth_date'),
    );
}
```

> الـ Action يستقبل `RegisterData` مش `array` — type safety + IDE support.

---

### 4.3 Actions (Auth)

#### `LoginAction` — 🟡

**مشاكل:**
- يرجع `array` بدل DTO/Result object
- `$user->tokens()->delete()` — token logic يجب يكون في `TokenService`
- الـ Controller يفهم `reason` keys — coupling

**المستهدف:**

```php
// app/DTOs/Auth/LoginResult.php
readonly class LoginResult
{
    public function __construct(
        public bool $success,
        public ?User $user = null,
        public ?string $token = null,
        public ?LoginFailureReason $reason = null,
    ) {}
}

// LoginAction.php
public function execute(LoginData $data): LoginResult
```

#### `VerifyRegisterOtpAction` — 🟡

**مشاكل:**
- `Hash::make($data['password'])` **مكرر** — الـ `User` model عنده `'password' => 'hashed'` cast، فـ `Hash::make` هنا = double hashing
- `$user->createToken()` — ينقل لـ `TokenService`

```php
// ❌ الحالي — double hash
'password' => Hash::make($data['password']),

// ✅ الصح
'password' => $data['password'],  // الـ cast يتولى الـ hashing
```

#### `RegisterAction` — ✅

- مسؤولية واحدة: cache pending data + send OTP

#### `ForgetPasswordAction` — 🟡

- لا يتحقق إن الـ phone موجود (الـ Request يعمل `exists` في forget flow بس مش هنا)
- لا يفرق بين user موجود ومش موجود (security: لا تكشف إن الـ phone مش مسجّل)

#### `ResetPasswordAction` — ✅

- منطق واضح، لكن يرجع `bool` — الأفضل exception أو Result DTO

#### `VerifyResetOtpAction` — ✅

- thin wrapper — مقبول

#### Actions ناقصة:

| Action | الغرض |
|--------|-------|
| `LogoutAction` | revoke current token |
| `GetAuthenticatedUserAction` | fetch + eager load relations |

---

### 4.4 `UserRepository` — 🟡

**إيجابيات:** يفصل DB access عن Actions ✅

**مشاكل:**
- لا Interface (DIP violation)
- `User::where()` مباشرة — انقل لـ Eloquent implementation
- لا methods للـ: `existsByPhone`, `markPhoneVerified`

```php
// app/Contracts/Repositories/UserRepositoryInterface.php
interface UserRepositoryInterface
{
    public function create(array $data): User;
    public function findByPhone(string $phone): ?User;
    public function updatePassword(string $phone, string $password): void;
    public function existsByPhone(string $phone): bool;
}
```

---

### 4.5 `OtpService` — 🟡

**مشاكل:**

| # | المشكلة | الخطورة |
|---|---------|---------|
| 1 | `rand(1000, 9999)` — weak randomness | 🔴 أمني |
| 2 | Vonage client في `__construct` — hard to test | 🟡 |
| 3 | SMS + Cache + Verification في class واحد | 🟡 SRP |
| 4 | OTP expires in 1 minute — قصير جدًا لـ UX | 🟡 |

**المستهدف:**

```php
// استخدم
$otp = str_pad((string) random_int(0, 9999), 4, '0', STR_PAD_LEFT);

// وفصّل
interface SmsGatewayInterface {
    public function send(string $to, string $message): void;
}
```

---

### 4.6 Controllers الأخرى — 🔴 غير جاهزة

الملفات التالية scaffold من `php artisan make:controller --resource`:

- `CartController`, `CartItemController`
- `OrderController`, `OrderItemController`
- `MealController`, `CategoryController`
- `FavoriteController`, `PaymentMethodController`
- `DeliveryRiderController`

**مشاكل التصميم الحالي:**

| # | المشكلة |
|---|---------|
| 1 | `Request $request` — الـ validation هتتحط في Controller لما يتنفّذ |
| 2 | `create()` و `edit()` — غير مطلوبة في API (للـ Blade فقط) |
| 3 | لا Actions ولا Services ولا Repositories |
| 4 | Namespace `api` lowercase — المفروض `Api` (PSR-4) |

**قاعدة الـ Controllers الفاضية (إلزامية):**

> **لا تضع أي تعليقات على الـ controller الفاضي** — لا PHPDoc، لا `//` داخل الـ methods، ولا تعليقات على الـ imports.  
> الـ scaffold يفضل يفضل **نظيف وفاضي** لحد ما يتنفّذ الـ endpoint فعليًا.  
> أول ما الـ method يتملّى، وقتها بس تضيف الكود الحقيقي (FormRequest + Action + return).

```php
// ✅ Controller فاضي — بدون تعليقات
class MealController extends Controller
{
    public function index()
    {
    }

    public function store(StoreMealRequest $request, StoreMealAction $action): JsonResponse
    {
        return $this->responder->created($action->execute($request->toDto()));
    }
}

// ❌ ممنوع على controller فاضي
/**
 * Display a listing of the resource.
 */
public function index()
{
    // TODO: implement later
}
```

**قبل التنفيذ — أنشئ لكل module:**

```
Meal/
├── StoreMealRequest.php
├── UpdateMealRequest.php
├── StoreMealAction.php
├── UpdateMealAction.php
├── DeleteMealAction.php
├── ListMealsAction.php
├── MealRepositoryInterface.php
├── MealRepository.php
├── MealResource.php
└── MealController.php          ← thin فقط
```

**Controller مثالي لـ Meal:**

```php
class MealController extends Controller
{
    use ApiResponse;

    public function __construct(private readonly MealResponder $responder) {}

    public function index(ListMealsRequest $request, ListMealsAction $action): JsonResponse
    {
        return $this->responder->paginated($action->execute($request->toDto()));
    }

    public function store(StoreMealRequest $request, StoreMealAction $action): JsonResponse
    {
        return $this->responder->created($action->execute($request->toDto()));
    }

    public function show(Meal $meal, ShowMealAction $action): JsonResponse
    {
        return $this->responder->single($action->execute($meal));
    }

    public function update(UpdateMealRequest $request, Meal $meal, UpdateMealAction $action): JsonResponse
    {
        return $this->responder->single($action->execute($meal, $request->toDto()));
    }

    public function destroy(Meal $meal, DeleteMealAction $action): JsonResponse
    {
        return $this->responder->deleted($action->execute($meal));
    }
}
```

---

### 4.7 Models — 🔴

| Model | الحالة | المطلوب |
|-------|--------|---------|
| `User` | ✅ جيد | أضف return types للـ relationships |
| `Meal` | فارغ | fillable, casts, relationships, scopes |
| `Order` | فارغ | + migration فيها columns معلّقة |
| `Category` | فارغ | — |
| `Cart`, `CartItem` | فارغ | — |
| `Favorite` | فارغ | — |
| `PaymentMethod` | فارغ | — |
| `DeliveryRider` | فارغ | — |
| `OrderItem` | فارغ | — |
| `PaymentTransaction` | فارغ | — |

**مثال `Meal` model:**

```php
class Meal extends Model
{
    protected $fillable = [
        'category_id', 'name', 'description', 'price',
        'calories', 'carbs', 'protein', 'fat', 'fiber', 'sugar', 'sodium',
        'address', 'location', 'working_hours', 'is_available',
    ];

    protected $casts = [
        'price'          => 'decimal:2',
        'working_hours'  => 'array',
        'is_available'   => 'boolean',
    ];

    public function category(): BelongsTo
    {
        return $this->belongsTo(Category::class);
    }

    public function scopeAvailable($query)
    {
        return $query->where('is_available', true);
    }
}
```

> **قاعدة:** الـ Model فيه relationships + scopes + casts فقط. لا business logic (حساب total order مثلاً → `OrderService`).

---

### 4.8 Middleware — 🟡

#### `EnsureIsActive`

```php
// ❌ response format مختلف عن ApiResponse trait
return response()->json(['message' => '...'], 403);

// ✅ وحّد الشكل
return response()->json([
    'status'  => false,
    'message' => 'Your account has been deactivated.',
    'errors'  => null,
], 403);
```

> **أفضل:** `InactiveAccountException` + `bootstrap/app.php` exception handler.

#### `EnsureIsAdmin`

- يستخدم `abort_if` — مقبول
- `EnsureIsClient` middleware **مفقود** (مذكور في routes لكن غير موجود)

---

### 4.9 `ApiResponse` Trait — ✅

- جيد كـ base
- **تحسين:** اعمل `Responder` classes بدل ما كل controller يختار message/status يدوي

---

### 4.10 `routes/api.php` — 🟡

- Auth routes شغالة ✅
- باقي routes معلّقة في comment block
- فيه **duplicate commented code** — احذفه
- لما تفعّل routes: استخدم `apiResource` بدون `create` و `edit`:

```php
Route::apiResource('meals', MealController::class)->except(['create', 'edit']);
```

---

### 4.11 `AppServiceProvider` — 🔴

```php
public function register(): void
{
    $this->app->bind(UserRepositoryInterface::class, UserRepository::class);
    $this->app->bind(SmsGatewayInterface::class, VonageSmsGateway::class);
    // ... باقي bindings
}
```

---

### 4.12 Database / Migrations — 🟡

- `orders` migration: كل الـ columns **معلّقة** — الـ Order model مش هيشتغل
- تأكد إن كل foreign keys مفعّلة قبل بناء Repositories

---

## 5. خطة Refactor — Auth Module (الأولوية)

### المرحلة 1 — Quick Wins

- [ ] إصلاح double hashing في `VerifyRegisterOtpAction`
- [ ] إنشاء `LogoutAction` و `GetAuthenticatedUserAction`
- [ ] نقل token logic لـ `TokenService`
- [ ] حذف الـ commented code من `routes/api.php`

### المرحلة 2 — إزالة Logic من Controller

- [ ] إنشاء `LoginResult`, `VerifyOtpResult` DTOs
- [ ] إنشاء `AuthResponder` (أو استخدام Exceptions)
- [ ] refactor `AuthController` ليكون thin بالكامل

### المرحلة 3 — SOLID Infrastructure

- [ ] `UserRepositoryInterface` + binding
- [ ] `SmsGatewayInterface` + `VonageSmsGateway`
- [ ] `random_int` بدل `rand` في OTP
- [ ] توحيد error response format (Middleware + Exception Handler)

### المرحلة 4 — AuthController النهائي (Target)

```php
<?php

namespace App\Http\Controllers\Api\Auth;

use App\Actions\Auth\ForgetPasswordAction;
use App\Actions\Auth\GetAuthenticatedUserAction;
use App\Actions\Auth\LoginAction;
use App\Actions\Auth\LogoutAction;
use App\Actions\Auth\RegisterAction;
use App\Actions\Auth\ResetPasswordAction;
use App\Actions\Auth\VerifyRegisterOtpAction;
use App\Actions\Auth\VerifyResetOtpAction;
use App\Http\Controllers\Controller;
use App\Http\Requests\Auth\ForgetPasswordRequest;
use App\Http\Requests\Auth\LoginRequest;
use App\Http\Requests\Auth\RegisterRequest;
use App\Http\Requests\Auth\ResetPasswordRequest;
use App\Http\Requests\Auth\VerifyOtpRequest;
use App\Support\Responders\AuthResponder;
use Illuminate\Http\JsonResponse;

class AuthController extends Controller
{
    public function __construct(private readonly AuthResponder $responder) {}

    public function register(RegisterRequest $request, RegisterAction $action): JsonResponse
    {
        $action->execute($request->toDto());
        return $this->responder->otpSent();
    }

    public function verifyRegister(VerifyOtpRequest $request, VerifyRegisterOtpAction $action): JsonResponse
    {
        return $this->responder->authenticated($action->execute($request->toDto()));
    }

    public function login(LoginRequest $request, LoginAction $action): JsonResponse
    {
        return $this->responder->login($action->execute($request->toDto()));
    }

    public function forgetPassword(ForgetPasswordRequest $request, ForgetPasswordAction $action): JsonResponse
    {
        $action->execute($request->validated('phone'));
        return $this->responder->otpSent();
    }

    public function verifyResetOtp(VerifyOtpRequest $request, VerifyResetOtpAction $action): JsonResponse
    {
        return $this->responder->otpVerified($action->execute($request->toDto()));
    }

    public function resetPassword(ResetPasswordRequest $request, ResetPasswordAction $action): JsonResponse
    {
        return $this->responder->passwordReset($action->execute($request->toDto()));
    }

    public function logout(LogoutAction $action): JsonResponse
    {
        $action->execute(auth()->user());
        return $this->responder->loggedOut();
    }

    public function me(GetAuthenticatedUserAction $action): JsonResponse
    {
        return $this->responder->user($action->execute(auth()->user()));
    }
}
```

---

## 6. خطة التنفيذ — باقي الـ Modules

### ترتيب الأولوية

| # | Module | السبب |
|---|--------|-------|
| 1 | Category | بسيط — CRUD أساسي، مفيش business logic معقّد |
| 2 | Meal | يعتمد على Category |
| 3 | Favorite | يعتمد على Meal + User |
| 4 | Cart / CartItem | business logic (quantities, totals) |
| 5 | Order / OrderItem | أعقد — transactions, status workflow |
| 6 | PaymentMethod / PaymentTransaction | تكامل خارجي |
| 7 | DeliveryRider | admin + assignment logic |

### Checklist لكل Endpoint جديد

```
□ Form Request (validation + authorize + toDto)
□ DTO class
□ Action class (business logic)
□ Repository Interface + Implementation (لو في DB)
□ API Resource (response transformation)
□ Responder method (أو exception mapping)
□ Controller method (سطرين: execute + respond)
□ Feature Test
□ Route registration
```

---

## 7. Exception-Based Error Handling (بديل الـ if/else)

```php
// app/Exceptions/Auth/InvalidCredentialsException.php
class InvalidCredentialsException extends Exception {}

// LoginAction.php
if (!$user || !Hash::check($password, $user->password)) {
    throw new InvalidCredentialsException();
}

// bootstrap/app.php
$exceptions->render(function (InvalidCredentialsException $e) {
    return response()->json([
        'status'  => false,
        'message' => 'The provided credentials are incorrect.',
        'errors'  => null,
    ], 401);
});
```

**فائدة:** الـ Controller يبقى بدون أي `if` — الـ Action يرمي exception والـ Handler يتولى الـ response.

---

## 8. Testing Strategy

| الطبقة | نوع الاختبار | Mock |
|--------|-------------|------|
| Controller | Feature Test | — |
| Action | Unit Test | Repository Interface |
| Repository | Integration Test | Database |
| Service (OTP/SMS) | Unit Test | SmsGatewayInterface |

```php
// مثال Unit Test لـ LoginAction
public function test_login_fails_with_wrong_password(): void
{
    $user = User::factory()->create(['password' => 'password']);
    
    $repo = Mockery::mock(UserRepositoryInterface::class);
    $repo->shouldReceive('findByPhone')->andReturn($user);
    
    $tokenService = Mockery::mock(TokenServiceInterface::class);
    
    $action = new LoginAction($repo, $tokenService);
    
    $this->expectException(InvalidCredentialsException::class);
    $action->execute(new LoginData(phone: $user->phone, password: 'wrong'));
}
```

---

## 9. Code Style & Conventions

| القاعدة | التفاصيل |
|---------|----------|
| Namespace | `App\Http\Controllers\Api` (PascalCase) |
| Controllers | `final class` — لا تورّث منها |
| Actions | verb + noun: `StoreMealAction`, `LoginAction` |
| Requests | `StoreMealRequest`, مش `MealStoreRequest` |
| DTOs | `readonly class` (PHP 8.2+) |
| Repositories | Interface في `Contracts/`, Implementation في `Repositories/` |
| لا تعليقات عربية في الكود | استخدم PHPDoc إنجليزي أو self-documenting names |
| Controllers فاضية | **بدون أي comments** — لا PHPDoc scaffold ولا `//` ولا TODO لحد التنفيذ |
| API methods | لا `create()` / `edit()` في API controllers |

---

## 10. ملخص الأولويات (Action Items)

### 🔴 Critical (اعملها فورًا)

1. إصلاح **double password hashing** في `VerifyRegisterOtpAction`
2. إزالة **conditional logic** من `AuthController`
3. إنشاء **Repository Interfaces** + Service Provider bindings
4. إكمال **Models** (fillable, relationships, casts)
5. إصلاح **orders migration** (الـ columns المعلّقة)

### 🟡 High (قبل بناء modules جديدة)

6. إنشاء `AuthResponder` أو Exception Handler
7. `TokenService` منفصل
8. `SmsGatewayInterface` + refactor `OtpService`
9. `EnsureIsClient` middleware
10. توحيد API response format في كل Middleware

### 🟢 Medium (أثناء التطوير)

11. DTOs لكل Action
12. Responder classes لكل module
13. Feature + Unit tests
14. إعادة تسمية namespace `api` → `Api`
15. حذف scaffold methods (`create`, `edit`) من API controllers
16. إبقاء الـ controllers الفاضية **بدون تعليقات** لحد ما تتنفّذ

---

## 11. المراجع

- [Laravel Form Requests](https://laravel.com/docs/validation#form-request-validation)
- [Laravel Actions Pattern](https://laravel.com/docs/structure) — community pattern
- [SOLID Principles in PHP](https://www.php.net/manual/en/language.oop5.basic.php)
- Project Rules: `.cursor/rules` — Laravel Code Quality Guidelines

---

*هذا الملف مرجع حي — حدّثه مع كل module جديد يتنفّذ.*
