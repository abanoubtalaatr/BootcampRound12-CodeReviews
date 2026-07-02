# FoodIfy — Code Review & Architecture Guide

> **الهدف:** Controllers رفيعة تمامًا — بدون validation وبدون business logic.  
> كل المنطق في Actions / Services / Repositories مع تطبيق SOLID في كل الطبقات.

**تاريخ المراجعة:** 2 يوليو 2026  
**الطالب:** Mohamed Esmail  
**المستودع:** [aborafe/foodify](https://github.com/aborafe/foodify)  
**النطاق:** `app/` — Auth + Business APIs (catalog, cart, checkout, orders, notifications)

---

## 1. ملخص تنفيذي

| المنطقة | الحالة | التقييم |
|---------|--------|---------|
| Auth Module | مُنفَّذ بالكامل | ✅ جيد — Service layer + interfaces |
| Form Requests (Auth) | 5 requests | ✅ |
| Form Requests (Business) | غير موجود | 🔴 |
| Services | Auth + OTP + SMS | 🟡 جزئي — business domains بدون services |
| Service Interfaces | 3 interfaces | ✅ DIP في auth stack |
| Actions | غير موجود | 🔴 |
| Repositories | غير موجود | 🔴 |
| DTOs | غير موجود | 🔴 |
| API Resources | غير موجود | 🔴 |
| `ApiResponse` Trait | غير موجود | 🔴 |
| Models | 11 models كاملة | ✅ ممتاز |
| Feature Tests | Auth + Business | ✅ ممتاز |
| Business Controllers | 10 controllers | 🟡 fat — validation + logic + DB |
| SOLID Compliance | جزئي | 🟡 |

**الخلاصة:** FoodIfy هو **أكمل مشروع functionally** في الـ bootcamp — auth كامل مع Vonage OTP، catalog، cart، checkout، orders، notifications، و feature tests. Auth stack يتبع Service + Interface pattern بشكل جيد. لكن **باقي الـ domains** (cart, checkout, orders) تستخدم **fat controllers** مع inline validation و Eloquent مباشرة. لا Actions ولا Repositories ولا ApiResponse trait.

---

## 2. المعمارية المستهدفة vs الحالية

```
المستهدف:  FormRequest → Controller → Action → Service → RepositoryInterface → Model
الحالي (Auth):  FormRequest → Controller → AuthService → OtpService → Eloquent مباشرة
الحالي (Business):  Request (inline validate) → Controller → Eloquent مباشرة
```

**Auth هو المرجع** — الأقرب للمعمارية المستهدفة. باقي الـ modules تحتاج نفس المعالجة.

---

## 3. تحليل SOLID

### S — Single Responsibility Principle

| الملف | المشكلة |
|-------|---------|
| `CheckoutController` | validation + totals + order creation + payment + cart clear + order number generation |
| `CartController` | validation + meal lookup + quantity merge + price snapshot + subtotal calc |
| `AuthController` | service calls + response shaping + OTP formatting + token deletion |
| `AuthService` | ✅ مسؤولية واحدة — auth flows |
| `OtpService` | ✅ OTP generation + verification + SMS |

### O — Open/Closed Principle

- Auth stack قابل للتوسيع عبر interfaces (SMS gateway swap) ✅
- Business controllers — أي تغيير يتطلب تعديل controller مباشرة ❌

### L — Liskov Substitution Principle

- `SmsServiceInterface`, `OtpServiceInterface`, `AuthServiceInterface` — قابلة للاستبدال ✅
- Tests تستخدم mock لـ `SmsServiceInterface` بشكل صحيح ✅

### I — Interface Segregation Principle

- Auth contracts مركّزة ومحددة ✅
- لا interfaces لباقي domains ❌

### D — Dependency Inversion Principle

| الوضع الحالي | المطلوب |
|--------------|---------|
| Auth: يعتمد على interfaces ✅ | — |
| Business controllers: Eloquent مباشرة ❌ | Repository interfaces |
| `AppServiceProvider`: 3 bindings فقط | bindings لكل repository |

---

## 4. مراجعة ملف بملف

### 4.1 `AuthController` — 🟡 الأقرب للمعمارية

```17:37:app/Http/Controllers/AuthController.php
class AuthController extends Controller
{
    public function __construct(private readonly AuthServiceInterface $authService) {}

    public function register(RegisterRequest $request): JsonResponse
    {
        try {
            [$user, $otp] = $this->authService->register($request->validated());
        } catch (RuntimeException $exception) { ... }
        return response()->json([...]);
    }
}
```

**إيجابيات:**
- DI عبر `AuthServiceInterface` ✅
- Form Requests لكل endpoint ✅
- Business logic في Service ✅

**مخالفات:**

| Method | المشكلة | السطر |
|--------|---------|-------|
| `register`, `forgotPassword` | try/catch + response mapping في controller | 23–36, 74–86 |
| `verifyRegisterOtp` | `ValidationException` mapping | 45–48 |
| `login` | response shaping يدوي | 62–67 |
| `logout` | `$request->user()->currentAccessToken()->delete()` — bypasses service | 116–118 |
| `otpResponse()` | private helper في controller — ينقل لـ ApiResponse/Resource | 125–131 |

**المستهدف:**

```php
public function register(RegisterRequest $request, RegisterAction $action): JsonResponse
{
    return $this->registeredResponse($action->execute($request->toDto()));
}
```

---

### 4.2 Form Requests (Auth) — ✅ ممتاز

| Request | الحالة |
|---------|--------|
| `RegisterRequest` | ✅ phone regex, unique, password confirmed |
| `LoginRequest` | ✅ |
| `ForgotPasswordRequest` | ✅ `exists:users,phone` |
| `OtpVerificationRequest` | ✅ 6-digit code |
| `ResetPasswordRequest` | ✅ |

**ناقص:** `toDto()` method في كل Request.

**مفقود لـ business domains:**

| Domain | Requests مطلوبة |
|--------|----------------|
| Cart | `StoreCartItemRequest`, `UpdateCartItemRequest` |
| Checkout | `CheckoutRequest` |
| Profile | `UpdateProfileRequest` |
| Payment | `StorePaymentMethodRequest` |
| Favorite | `StoreFavoriteRequest` |

---

### 4.3 Services — 🟡 جزئي (Auth فقط)

#### `AuthService` — ✅ جيد

```17:74:app/Services/AuthService.php
    public function register(array $data): array { ... }
    public function login(string $phone, string $password): array { ... }
    public function resetPassword(string $phone, string $code, string $password): void { ... }
```

- Transactions ✅
- Password reset flow كامل ✅
- Active + verified checks في login ✅

**مشاكل:**
- يقبل `array $data` بدل DTO
- يرمي `ValidationException` — HTTP concern في service layer
- `User::query()` مباشرة — بدون repository

#### `OtpService` — ✅ ممتاز

```19:47:app/Services/OtpService.php
    public function generate(string $phone, string $type): Otp
    {
        // invalidate old OTPs, create new, send SMS
        // rollback on SMS failure ✅
    }
```

- `random_int(100000, 999999)` — 6 digits, secure ✅
- SMS failure rollback ✅
- Separate `otps` table (مش في users) ✅
- OTP لا يُرجع في API response (tests تتحقق) ✅

**مشكلة بسيطة:** duplicate constants (`REGISTER`, `FORGOT_PASSWORD`) في Service و Interface.

#### `VonageSmsService` — ✅

- SMS behind `SmsServiceInterface` — DIP ✅

**مفقود:** `CheckoutService`, `CartService`, `OrderService`.

---

### 4.4 Contracts / Interfaces — 🟡 Auth فقط

```
app/Contracts/
├── AuthServiceInterface.php    ✅
├── OtpServiceInterface.php     ✅
└── SmsServiceInterface.php     ✅
```

**مفقود:** `UserRepositoryInterface`, `OrderRepositoryInterface`, `CartItemRepositoryInterface`, etc.

---

### 4.5 `CheckoutController` — 🔴 الأسوأ

```13:92:app/Http/Controllers/CheckoutController.php
    public function __invoke(Request $request): JsonResponse
    {
        $data = $request->validate([...]);          // ❌ inline validation
        // payment method ownership check              // ❌ business logic
        // empty cart check                            // ❌ business logic
        $order = DB::transaction(function () {       // ❌ full checkout in controller
            // subtotal, delivery fee, total calc
            // order creation, order items, payment
            // cart clear
        });
        return response()->json([...]);              // ❌ manual response
    }
```

| المخالفة | التفاصيل |
|----------|----------|
| Inline validation | سطر 15–20 |
| Full checkout transaction | سطر 39–77 |
| `generateOrderNumber()` | سطر 85–92 — domain logic في controller |
| Client-controlled `delivery_fee` | سطر 41 — **security risk** — يجب server-side calc |
| Raw model response | يرجع Order model بدون Resource |

**المستهدف:**

```php
public function __invoke(CheckoutRequest $request, PlaceOrderAction $action): JsonResponse
{
    return $this->createdResponse($action->execute($request->toDto()));
}
```

---

### 4.6 `CartController` — 🔴 Fat

| Method | المشاكل |
|--------|---------|
| `index` | subtotal calculation في controller (سطر 23) |
| `store` | inline validation (31–34), meal lookup + availability (36–39), quantity merge (43–50) |
| `update` | inline validation (62–64), ownership `abort_unless` |
| `destroy`, `clear` | ownership check مكرر |

---

### 4.7 Controllers أخرى

| Controller | التقييم | المشاكل |
|------------|---------|---------|
| `OrderController` | 🟡 | cancel eligibility rule في controller (35–39) |
| `FavoriteController` | 🔴 | inline validation + meal availability check |
| `PaymentMethodController` | 🔴 | inline validation + "unset other defaults" logic |
| `ProfileController` | 🔴 | inline validation + update |
| `HomeController` | 🟡 | 3 catalog queries — مقبول للقراءة لكن بدون repository |
| `CategoryController` | 🟡 | direct Eloquent queries |
| `MealController` | ✅ | simple read — الأقرب للـ thin |
| `NotificationController` | 🟡 | simple CRUD — direct Eloquent |

**نمط مكرر:** `abort_unless($model->user_id === $request->user()->id, 404)` — يجب Policies.

---

### 4.8 Models — ✅ ممتاز

11 models مع relationships, casts, و `#[Fillable]` attributes:

| Model | الحالة |
|-------|--------|
| `User` | ✅ Sanctum, hashed password, relations |
| `Meal` | ✅ nutrition fields, availability |
| `Category` | ✅ |
| `Order` | ✅ status, payment fields |
| `OrderItem` | ✅ price snapshot |
| `CartItem` | ✅ unit_price snapshot |
| `Favorite` | ✅ |
| `Payment` | ✅ |
| `PaymentMethod` | ✅ |
| `Notification` | ✅ |
| `Otp` | ✅ separate table — أفضل من تخزين OTP في users |

**ناقص:**
- Model scopes (`scopeActive`, `scopeAvailable`)
- Policies للـ authorization
- Factories (عدا UserFactory)

---

### 4.9 `ApiResponse` Trait — 🔴 غير موجود

كل controller يبني JSON يدويًا:

```php
return response()->json([
    'message' => '...',
    'user' => $user,
], 201);
```

**لا envelope موحّد** — كل controller له شكل response مختلف قليلًا.

**المطلوب:** `app/Traits/ApiResponse.php` مع methods:
- `success()`, `created()`, `error()`, `paginated()`
- Domain-specific: `loginResponse()`, `checkoutResponse()`

---

### 4.10 Actions / Repositories / DTOs — 🔴 غير موجود

| المجلد | الحالة |
|--------|--------|
| `app/Actions/` | ❌ |
| `app/Repositories/` | ❌ |
| `app/DTOs/` | ❌ |
| `app/Http/Resources/` | ❌ |

Auth يستخدم Service بدل Action — مقبول لكن باقي domains تحتاج نفس الطبقة.

---

### 4.11 `AppServiceProvider` — 🟡

```18:23:app/Providers/AppServiceProvider.php
    public function register(): void
    {
        $this->app->bind(SmsServiceInterface::class, VonageSmsService::class);
        $this->app->bind(OtpServiceInterface::class, OtpService::class);
        $this->app->bind(AuthServiceInterface::class, AuthService::class);
    }
```

**إيجابية:** DIP bindings صحيحة للـ auth stack ✅

**ناقص:** repository bindings لما تُنشأ.

---

### 4.12 Routes — ✅ جيد

```16:59:routes/api.php
Route::prefix('auth')->group(function (): void { ... });
Route::get('home', HomeController::class);
Route::middleware('auth:sanctum')->group(function (): void { ... });
```

**إيجابيات:**
- Auth routes مجمّعة ✅
- Sanctum على protected routes ✅
- REST-ish naming ✅
- Invokable controllers للـ single-action endpoints ✅

**ناقص:**
- API versioning (`v1`)
- Rate limiting على OTP routes
- Policies بدل `abort_unless` في controllers

---

### 4.13 Middleware — 🟡

- `auth:sanctum` فقط
- JSON exception rendering لـ `api/*` ✅
- لا custom middleware (phone verified, rate limit, ownership)

---

### 4.14 Tests — ✅ ممتاز

| Test | الحالة |
|------|--------|
| `AuthApiTest.php` | ✅ register, verify, login, forgot password |
| `BusinessApiTest.php` | ✅ catalog, cart, checkout, orders |

**إيجابيات:**
- Pest PHP ✅
- `SmsServiceInterface` mocked بشكل صحيح ✅
- يتحقق إن OTP code **مش** في response ✅
- `RefreshDatabase` ✅

**ناقص:** unit tests للـ services، repository tests.

---

## 5. مشاكل أمنية

| # | المشكلة | الخطورة | الملف |
|---|---------|---------|-------|
| 1 | Client-controlled `delivery_fee` | 🔴 | `CheckoutController:41` |
| 2 | No rate limiting على OTP | 🟡 | `routes/api.php` |
| 3 | `abort_unless` بدل Policies | 🟡 | multiple controllers |
| 4 | Raw model في responses | 🟡 | all controllers — sensitive fields محتملة |

---

## 6. ما هو جيد ✅

1. **أكمل API functionally** — auth + catalog + cart + checkout + orders + notifications.
2. **Auth Service layer** — interfaces + DI + transactions — أفضل auth implementation في الـ bootcamp.
3. **OTP architecture** — separate table, SMS rollback, 6-digit secure random, no code leak in API.
4. **Feature tests** — Auth + Business coverage مع Pest.
5. **Models** — 11 models نظيفة مع relationships و casts.
6. **Migrations** — domain schema كامل.
7. **Route structure** — منطقي ومنظّم.
8. **Vonage SMS** — behind interface, swappable.
9. **Order item price snapshot** — `meal_name`, `unit_price` محفوظة وقت الطلب.
10. **Sanctum token abilities** — password reset flow آمن.

---

## 7. خطة Refactor

### المرحلة 1 — Quick Wins (P0) 🔴

- [ ] إنشاء `ApiResponse` trait — توحيد كل responses
- [ ] إنشاء `CheckoutRequest` — نقل validation من controller
- [ ] نقل `delivery_fee` calculation لـ server-side (لا تقبل من client)
- [ ] إضافة `throttle` على OTP routes
- [ ] إنشاء Form Requests لـ cart, profile, payment methods, favorites

### المرحلة 2 — Extract Business Logic (P1) 🟡

- [ ] `PlaceOrderAction` / `CheckoutService` — استخراج من `CheckoutController`
- [ ] `CartService` — add/update/clear logic
- [ ] `CancelOrderAction` — cancel eligibility rules
- [ ] Repository interfaces + implementations
- [ ] Bind في `AppServiceProvider`
- [ ] Policies بدل `abort_unless`

### المرحلة 3 — Architecture Alignment (P2) 🟢

- [ ] DTOs لكل service/action input
- [ ] API Resources (`UserResource`, `OrderResource`, `MealResource`)
- [ ] نقل `logout` + `otpResponse` من `AuthController`
- [ ] Domain exceptions بدل `ValidationException` في services
- [ ] Model scopes (`active`, `available`)
- [ ] Unit tests للـ services

### المرحلة 4 — Polish (P3) 🟢

- [ ] README — استبدال Laravel boilerplate بتوثيق FoodIfy
- [ ] API versioning (`v1`)
- [ ] Align `APP_NAME`, `composer.json` name
- [ ] Factories لكل models

---

## 8. Controller المستهدف (مثال: Checkout)

```php
<?php

namespace App\Http\Controllers;

use App\Actions\Order\PlaceOrderAction;
use App\Http\Requests\CheckoutRequest;
use App\Traits\ApiResponse;
use Illuminate\Http\JsonResponse;

final class CheckoutController extends Controller
{
    use ApiResponse;

    public function __invoke(CheckoutRequest $request, PlaceOrderAction $action): JsonResponse
    {
        return $this->createdResponse(
            $action->execute($request->user(), $request->toDto())
        );
    }
}
```

---

## 9. File-by-File Scorecard

| File | Thin Controller | Form Request | Service/Action | Repository | ApiResponse | Verdict |
|------|:-:|:-:|:-:|:-:|:-:|---------|
| `AuthController` | 🟡 | ✅ | ✅ Service | ❌ | ❌ | 🟡 Partial |
| `CheckoutController` | ❌ | ❌ | ❌ | ❌ | ❌ | 🔴 Fail |
| `CartController` | ❌ | ❌ | ❌ | ❌ | ❌ | 🔴 Fail |
| `OrderController` | 🟡 | ❌ | ❌ | ❌ | ❌ | 🟡 Partial |
| `FavoriteController` | ❌ | ❌ | ❌ | ❌ | ❌ | 🔴 Fail |
| `PaymentMethodController` | ❌ | ❌ | ❌ | ❌ | ❌ | 🔴 Fail |
| `ProfileController` | ❌ | ❌ | ❌ | ❌ | ❌ | 🔴 Fail |
| `MealController` | ✅ | — | — | ❌ | ❌ | 🟡 OK read |
| `AuthService` | — | — | ✅ | ❌ | — | 🟡 Good |
| `OtpService` | — | — | ✅ | ❌ | — | ✅ Good |
| Models | — | — | — | — | — | ✅ Good |
| Tests | — | — | — | — | — | ✅ Good |
| `AppServiceProvider` | — | — | 🟡 partial | ❌ | — | 🟡 Partial |

---

## 10. مقارنة مع مشاريع Bootcamp

| المعيار | Elham | Abdella | Osama | Mohamed Esmail |
|---------|:-:|:-:|:-:|:-:|
| Auth مُنفَّذ | جزئي | prototype | ❌ | ✅ كامل |
| Business APIs | scaffold | ❌ | ❌ | ✅ كامل |
| Service Interfaces | جزئي | ❌ | ❌ | ✅ auth stack |
| Repositories | ❌ | ❌ | ❌ | ❌ |
| Actions | موجود | ❌ | ❌ | ❌ |
| Feature Tests | ❌ | ❌ | ❌ | ✅ |
| Models | scaffold | User | 8 models | 11 models |
| ApiResponse trait | موجود | ApiTrait | ❌ | ❌ |
| OTP architecture | OtpService | in users | ❌ | ✅ best |

**نقطة قوة Mohamed Esmail:** أكمل مشروع functionally + أفضل auth/OTP architecture + tests.  
**نقطة ضعف:** business controllers fat — يحتاج نفس Service/Action treatment اللي اتعمل في auth.

---

## 11. ملخص الأولويات (Action Items)

### 🔴 Critical

1. **`CheckoutController`** — استخراج كل business logic لـ `PlaceOrderAction`/`CheckoutService`
2. **`delivery_fee`** — server-side calculation فقط
3. **`ApiResponse` trait** — توحيد JSON responses
4. **Form Requests** لكل business endpoint

### 🟡 High

5. **Repository interfaces** + DI bindings
6. **Policies** بدل `abort_unless` المكرر
7. **CartService** — استخراج logic من `CartController`
8. **API Resources** — لا ترجع raw models
9. **Rate limiting** على OTP routes

### 🟢 Medium

10. DTOs لكل service/action
11. Domain exceptions بدل `ValidationException` في services
12. Model scopes
13. README documentation
14. Unit tests للـ services

---

## 12. المراجع

- [Laravel Form Requests](https://laravel.com/docs/validation#form-request-validation)
- [Laravel Sanctum](https://laravel.com/docs/sanctum)
- مراجعة Elham: [elham/FoodIfy/CODE_REVIEW.md](../../elham/FoodIfy/CODE_REVIEW.md)
- مراجعة Abdella: [abdella/FoodIfy/CODE_REVIEW.md](../../abdella/FoodIfy/CODE_REVIEW.md)
- مراجعة Osama: [osama/OnlineRestaurant/CODE_REVIEW.md](../../osama/OnlineRestaurant/CODE_REVIEW.md)

---

*هذا الملف مرجع حي — حدّثه مع كل refactor جديد.*
