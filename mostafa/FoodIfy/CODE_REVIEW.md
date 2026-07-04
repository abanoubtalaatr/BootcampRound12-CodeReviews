# FoodIfy — Code Review & Architecture Guide

> **الهدف:** Controllers رفيعة تمامًا — بدون validation وبدون business logic.  
> كل المنطق في Actions / Services / Repositories مع تطبيق SOLID في كل الطبقات.

**تاريخ المراجعة:** 4 يوليو 2026  
**الطالب:** مصطفى (Mostafa)  
**المشروع:** FoodIfy — Healthy Meal Delivery API  
**النطاق:** `app/` — Auth + Catalog + Cart + Orders (Stripe) + Favorites + Ratings + Profile + Notifications

---

## 1. ملخص تنفيذي

| المنطقة | الحالة | التقييم |
|---------|--------|---------|
| Auth Module | مُنفَّذ بالكامل | 🟡 يعمل لكن فيه bugs حرجة |
| OTP Flow | مُنفَّذ (بدون SMS حقيقي) | 🔴 OTP يُرجَع في الـ response |
| Form Requests | غير موجود | 🔴 validation داخل Controllers |
| Actions / Services | غير موجود | 🔴 كل المنطق في Controllers |
| Repositories | غير موجود | 🔴 Eloquent مباشرة |
| API Resources | غير موجود | 🟡 raw models في الـ response |
| Catalog (Category/Meal) | مُنفَّذ | ✅ |
| Cart | مُنفَّذ | ✅ |
| Orders + Stripe | مُنفَّذ | 🟡 schema mismatch + `env()` |
| Favorites / Ratings | مُنفَّذ | 🟡 `comment` column ناقص |
| Profile / Notifications | مُنفَّذ | ✅ |
| Models | مُنفَّذة جزئيًا | 🟡 fillable/schema gaps |
| Feature Tests | scaffold فقط | 🔴 |
| README | Laravel default | 🔴 |
| SOLID Compliance | ضعيف | 🔴 fat controllers |

**الخلاصة:** المشروع **مُنفَّذ functionally** أكثر من معظم مشاريع FoodIfy — Auth، Cart، Checkout (Cash + Stripe)، Favorites، Ratings، Notifications كلها شغالة. لكن المعمارية **Fat Controller / Thin Model** بالكامل: validation + business logic + Stripe + DB transactions + notifications كلها داخل الـ controllers. فيه **3 bugs حرجة** لازم تتصلّح فورًا: double password hashing، `payment_status` column ناقص، `comment` column ناقص في ratings.

---

## 2. المعمارية المستهدفة vs الحالية

```
المستهدف:  FormRequest → Controller → Action → Service → RepositoryInterface → Model
الحالي:    Request ──────→ Controller ──────────────────────────────────────→ Model
                          (validation + logic + Stripe + transactions + notify)
```

### قواعد الـ Controller (إلزامية)

```php
// ✅ Controller مثالي — لا if/else، لا validation، لا DB، لا Hash
public function create(CreateOrderRequest $request, CreateOrderAction $action): JsonResponse
{
    return $this->createdResponse($action->execute($request->toDto(), $request->user()));
}

// ❌ الوضع الحالي في OrderController
$request->validate([...]);                    // validation
Stripe::setApiKey(env('STRIPE_SECRET'));      // external integration
$charge = Charge::create([...]);              // payment logic
DB::beginTransaction();                       // transaction management
Order::create([...]);                         // database access
$user->notify(new OrderStatusNotification(...)); // side effects
```

### هيكل المجلدات المقترح

```
app/
├── Actions/
│   ├── Auth/
│   ├── Cart/
│   ├── Order/
│   └── Rating/
├── Contracts/
│   ├── Repositories/
│   └── Services/
├── DTOs/
├── Enums/
│   ├── OrderStatus.php
│   └── PaymentMethod.php
├── Exceptions/
│   ├── Auth/
│   └── Order/
├── Http/
│   ├── Controllers/Api/
│   ├── Requests/
│   │   ├── Auth/
│   │   ├── Cart/
│   │   └── Order/
│   └── Resources/
├── Models/
├── Repositories/
├── Services/
│   ├── OtpService.php
│   ├── PaymentService.php
│   └── OrderPricingService.php
└── Traits/
    └── ApiResponse.php
```

---

## 3. تحليل SOLID

### S — Single Responsibility Principle

| الملف | المشكلة |
|-------|---------|
| `AuthController` (267 سطر) | register + OTP send/verify + login + logout + reset password + validation + token management |
| `OrderController` | validation + pricing + Stripe charge + DB transaction + cart clearing + notification |
| `CartController` | add/remove/get + total calculation |
| `RatingController` | validation + upsert logic |

### O — Open/Closed Principle

| المشكلة | الحل |
|---------|------|
| `payment_method` hardcoded `cash\|card` في validation | `PaymentMethod` enum + `PaymentGatewayInterface` |
| إضافة PayPal/Fawry = تعديل `OrderController` | `StripeGateway`, `CashGateway` implementations |
| delivery fee `30` hardcoded | `config('foodify.delivery_fee')` أو `OrderPricingService` |

### L — Liskov Substitution Principle

- غير قابل للاختبار حاليًا — Controllers تعتمد على concrete classes (`Stripe\Charge`, Eloquent models) بدون interfaces.

### I — Interface Segregation Principle

- لا interfaces موجودة — `AppServiceProvider::register()` فارغ.

### D — Dependency Inversion Principle

| الوضع الحالي | المطلوب |
|--------------|---------|
| `Stripe::setApiKey(env('STRIPE_SECRET'))` في Controller | `PaymentGatewayInterface` + `config('services.stripe.secret')` |
| `User::create()`, `Order::create()` مباشرة | `UserRepositoryInterface`, `OrderRepositoryInterface` |
| `OtpCode::updateOrCreate()` في Controller | `OtpServiceInterface` |
| `AppServiceProvider` فارغ | DI bindings لكل interface |

---

## 4. مراجعة ملف بملف

### 4.1 `AuthController` — 🟡 مُنفَّذ لكن يحتاج refactor + fixes

**الإيجابيات:**
- Egyptian phone regex validation ✅
- Sanctum token auth ✅
- OTP expiry check (10 دقائق) ✅
- Login يمنع unverified users (`phone_verified_at === null`) ✅
- Response envelope موحّد `{ status, message, data?, errors? }` ✅

**المخالفات:**

| Method | المشكلة | السطر |
|--------|---------|-------|
| `register` | `Hash::make()` + `'password' => 'hashed'` cast = **double hashing** | 50 |
| `register` | Token يُصدر **قبل** OTP verification — تناقض مع رسالة "activate via OTP" | 56–65 |
| `sendOtp` | `rand(1000, 9999)` — weak randomness | 87 |
| `sendOtp` | **`test_code` في response** — خطر أمني في production | 106 |
| `verifyOtp` | indentation غير متسق (methods خارج class formatting) | 70–262 |
| `resetPassword` | `Hash::make()` + hashed cast = **double hashing** | 252 |
| كل methods | `Validator::make()` inline — بدون Form Request | — |
| كل methods | 422 response block **مكرر** 6 مرات | — |

**Double Hashing — 🔴 Critical Bug:**

```php
// User.php — cast يعمل hash تلقائي
'password' => 'hashed',

// AuthController.php — hash يدوي = double hash
'password' => Hash::make($request->password),  // ❌ سطر 50

// النتيجة: Hash::check() في login (سطر 177) **هيفشل** بعد التسجيل
```

**الإصلاح:**

```php
// ✅ سيب الـ cast يتولى الـ hashing
'password' => $request->password,
```

**Auth Flow المطلوب:**

```
register → send OTP (no token) → verify OTP → issue token
         ↘ (أو) register → send OTP → login after verify
```

**الشكل المستهدف:**

```php
class AuthController extends Controller
{
    use ApiResponse;

    public function register(RegisterRequest $request, RegisterAction $action): JsonResponse
    {
        $action->execute($request->toDto());

        return $this->otpSentResponse();
    }

    public function login(LoginRequest $request, LoginAction $action): JsonResponse
    {
        return $this->loginResponse($action->execute($request->toDto()));
    }
}
```

---

### 4.2 `OrderController` — 🟡 الأكثر تعقيدًا — يحتاج Service Layer

**الإيجابيات:**
- DB transaction للـ order creation ✅
- Eager loading في `getMyOrders` (`items.meal`) ✅
- Cart validation قبل الطلب ✅
- Database notification بعد commit ✅

**المشاكل:**

| # | المشكلة | السطر | الخطورة |
|---|---------|-------|---------|
| 1 | `payment_status` في `Order::create()` لكن **مش في migration ولا `$fillable`** | 77 | 🔴 silent data loss |
| 2 | `env('STRIPE_SECRET')` في runtime | 45 | 🔴 breaks config cache |
| 3 | Stripe exception message في response | 63 | 🟡 information disclosure |
| 4 | Legacy `Charge::create()` + token — deprecated | 48–53 | 🟡 |
| 5 | Delivery fee hardcoded `30` | 38 | 🟡 |
| 6 | Success message typo: "application" بدل "order" | 104 | 🟢 |
| 7 | Unused import `Validator` | 11 | 🟢 |
| 8 | Stripe **قبل** transaction — لو DB fail، الفلوس اتسحبت | 42–69 | 🔴 |

**Schema Fix:**

```php
// migration: add to orders table
$table->string('payment_status')->default('unpaid');

// Order.php $fillable
'payment_status',
```

**Payment Service المستهدف:**

```php
interface PaymentGatewayInterface
{
    public function charge(int $amountCents, string $currency, string $token): ChargeResult;
}

class CreateOrderAction
{
    public function execute(CreateOrderData $data, User $user): Order
    {
        return DB::transaction(function () use ($data, $user) {
            // pricing → payment → persist → clear cart → notify
        });
    }
}
```

---

### 4.3 `CartController` — ✅ جيد functionally

**الإيجابيات:**
- Upsert/increment logic صح ✅
- Decrement or delete on remove ✅
- Eager load meals ✅

**المشاكل:**

| # | المشكلة |
|---|---------|
| 1 | Total calculation في controller — ينقل لـ `CartService` |
| 2 | `number_format()` يرجع string (سطر 80) — type inconsistency |
| 3 | No unique constraint على `(user_id, meal_id)` — duplicate rows ممكنة |
| 4 | Inline validation |

---

### 4.4 Controllers الأخرى

#### `CategoryController` — ✅

- `Category::all()` — بسيط ومقبول
- **تحسين:** pagination + `CategoryResource`

#### `MealController` — ✅

- Filter by `category_id` ✅
- **مشاكل:** `category_id` query param غير validated؛ no pagination

#### `FavoriteController` — ✅

- `toggle()` via pivot — clean ✅
- Unused import `Meal`

#### `RatingController` — 🔴 Schema Bug

```php
// RatingController.php — يحفظ comment
'comment' => $request->comment  // سطر 41

// ratings migration — لا يوجد comment column
$table->unsignedTinyInteger('rating');  // فقط
```

**Fix:** أضف migration:

```php
$table->text('comment')->nullable();
// + أضف 'comment' في Rating::$fillable
```

#### `ProfileController` — 🟡

- `Hash::make()` + hashed cast = **double hashing** في update password (سطر 55)
- يرجع raw `$user` model — استخدم `UserResource`

#### `NotificationController` — ✅

- Scoped lookup (`where('id', $id)`) ✅
- `markAsRead()` ✅

---

### 4.5 Models — 🟡

| Model | الحالة | المطلوب |
|-------|--------|---------|
| `User` | ✅ جيد | return types للـ relationships؛ شيل `Hash::make` من controllers |
| `Category` | ✅ | — |
| `Meal` | ✅ | `ingredients` cast ✅؛ أضف scopes (`available`) |
| `CartItem` | ✅ | unique index `(user_id, meal_id)` |
| `Order` | 🟡 | أضف `payment_status` في fillable + casts |
| `OrderItem` | 🔴 | `$fillable` فيه `payment_method` — **column مش موجود** |
| `Rating` | 🔴 | `comment` ناقص من fillable + migration |
| `OtpCode` | ✅ | أضف unique index على `phone_number` |

**مثال Order model:**

```php
class Order extends Model
{
    protected $fillable = [
        'user_id', 'total_price', 'delivery_fee',
        'payment_method', 'payment_status', 'status',
    ];

    protected $casts = [
        'total_price'   => 'decimal:2',
        'delivery_fee'  => 'decimal:2',
    ];
}
```

---

### 4.6 `routes/api.php` — 🟡

**الإيجابيات:**
- Routes واضحة ومنظمة ✅
- Sanctum middleware على protected routes ✅

**المشاكل:**

| # | المشكلة |
|---|---------|
| 1 | 6 separate `auth:sanctum` groups — ادمجهم في group واحد |
| 2 | Scaffold route `GET /user` (سطر 16–18) — احذفه |
| 3 | No rate limiting على auth/OTP routes |
| 4 | No API versioning (`/api/v1/...`) |
| 5 | Comments عربية في routes file |

**المستهدف:**

```php
Route::middleware('auth:sanctum')->group(function () {
    // favorites, cart, orders, ratings, profile, notifications
});

Route::middleware('throttle:5,1')->group(function () {
    Route::post('/login', ...);
    Route::post('/send-otp', ...);
});
```

---

### 4.7 Database / Migrations — 🔴

| Issue | Details |
|-------|---------|
| **`ratings.comment` missing** | Controller يحفظ comment — column مش موجود → SQL error |
| **`orders.payment_status` missing** | Controller يحفظه — mass assignment silently drops it |
| **`order_items.payment_method` phantom** | في `$fillable` بس مش في migration |
| **`otp_codes.phone_number` not unique** | `updateOrCreate` يفترض row واحد per phone |
| **`cart_items` no unique (user_id, meal_id)** | duplicate rows ممكنة |
| **`password_reset_tokens` unused** | OTP flow replaces it — dead schema |
| **`orders.status` free string** | استخدم enum: `pending`, `preparing`, `delivered`, `cancelled` |

---

### 4.8 Notifications — ✅

**`OrderStatusNotification.php`:**
- Database channel ✅
- Dynamic title/message/order_id ✅
- **تحسين:** implement `ShouldQueue` للـ async delivery
- **تحسين:** انقل الإرسال لـ Event/Listener بدل inline في controller

---

### 4.9 Seeders — 🟡

| Seeder | الحالة |
|--------|--------|
| `CategorySeeder` | 4 categories ✅ |
| `MealSeeder` | 3 meals — Low Carbs و Vegan **فاضيين** |
| `DatabaseSeeder` | test user + 9 factory users ✅ |
| Missing | cart, orders, favorites, ratings seeders |

---

### 4.10 Missing Layers — 🔴

| Layer | Status |
|-------|--------|
| Form Requests | ❌ |
| Actions | ❌ |
| Services | ❌ |
| Repositories | ❌ |
| API Resources | ❌ |
| Custom Middleware | ❌ |
| `ApiResponse` Trait | ❌ — response block مكرر ~8 مرات |
| Policies | ❌ |
| Feature Tests | ❌ — `ExampleTest` فقط |
| `AppServiceProvider` bindings | ❌ فارغ |

---

## 5. Security Review — 🔴

| Issue | Location | Risk | Priority |
|-------|----------|------|----------|
| **Double password hashing** | AuthController:50, ProfileController:55, resetPassword:252 | Login broken after register | 🔴 P0 |
| **OTP exposed in response** | AuthController:106 (`test_code`) | Account takeover | 🔴 P0 |
| **Token before verification** | AuthController:56–65 | Unverified API access | 🔴 P0 |
| **Weak OTP: `rand()`** | AuthController:87 | Predictable codes | 🔴 P1 |
| **No rate limiting** | routes/api.php | Brute force / OTP flooding | 🔴 P1 |
| **`env()` in controller** | OrderController:45 | Config cache break | 🟡 P1 |
| **Stripe error leak** | OrderController:63 | Info disclosure | 🟡 P2 |
| **Payment before DB commit** | OrderController:42–69 | Money charged, order lost | 🔴 P0 |
| **Raw user in responses** | Auth, Profile | Field leakage over time | 🟡 P2 |
| **No `STRIPE_SECRET` in `.env.example`** | `.env.example` | Misconfiguration | 🟡 P2 |

---

## 6. خطة Refactor — الأولويات

### المرحلة 0 — Critical Fixes (فورًا)

- [ ] إصلاح **double password hashing** — شيل `Hash::make()` من AuthController, ProfileController, resetPassword
- [ ] إضافة migration لـ `ratings.comment`
- [ ] إضافة migration لـ `orders.payment_status` + update `$fillable`
- [ ] شيل `payment_method` من `OrderItem::$fillable`
- [ ] شيل `test_code` من OTP response (أو restrict لـ `APP_DEBUG`)
- [ ] **لا تصدر token في register** — بعد verify OTP فقط
- [ ] Fix Stripe-before-transaction: payment **داخل** transaction أو compensating refund

### المرحلة 1 — ApiResponse + Form Requests

- [ ] إنشاء `ApiResponse` trait — توحيد `{ status, message, data?, errors? }`
- [ ] Form Requests: `RegisterRequest`, `LoginRequest`, `SendOtpRequest`, `CreateOrderRequest`, `RateMealRequest`, `UpdateProfileRequest`
- [ ] `UserResource`, `MealResource`, `OrderResource`

### المرحلة 2 — Service Layer

- [ ] `OtpService` — generation (`random_int`), storage, verification
- [ ] `PaymentGatewayInterface` + `StripePaymentGateway`
- [ ] `OrderPricingService` — subtotal + delivery fee
- [ ] `CartService` — add/remove/get with totals
- [ ] `config/services.php` → Stripe key (not `env()`)

### المرحلة 3 — Actions + Repositories

- [ ] `CreateOrderAction`, `RegisterAction`, `LoginAction`, etc.
- [ ] Repository interfaces + Eloquent implementations
- [ ] `AppServiceProvider` bindings
- [ ] Refactor controllers → thin (2 lines per method)

### المرحلة 4 — Testing + Docs

- [ ] Feature tests: auth flow, cart, checkout (cash + mocked Stripe)
- [ ] Rate limiting middleware
- [ ] README خاص بـ FoodIfy (setup, endpoints, env vars)
- [ ] Pagination على list endpoints

---

## 7. Exception-Based Error Handling

```php
// app/Exceptions/Auth/InvalidCredentialsException.php
class InvalidCredentialsException extends Exception {}

// LoginAction
if (!$user || !Hash::check($password, $user->password)) {
    throw new InvalidCredentialsException();
}

// bootstrap/app.php
$exceptions->render(function (InvalidCredentialsException $e) {
    return response()->json([
        'status'  => false,
        'message' => 'Incorrect credentials (wrong phone number or password).',
        'errors'  => null,
    ], 401);
});
```

---

## 8. Testing Strategy

| الطبقة | نوع الاختبار | Mock |
|--------|-------------|------|
| Controller | Feature Test | — |
| Action | Unit Test | Repository Interface |
| PaymentService | Unit Test | `PaymentGatewayInterface` |
| OTP Service | Unit Test | Cache/DB |

```php
// Feature Test — Auth flow
public function test_user_can_register_and_verify_otp(): void
{
    $response = $this->postJson('/api/register', [
        'name' => 'Test User',
        'phone_number' => '01012345678',
        'password' => 'password123',
        'password_confirmation' => 'password123',
        'birth_date' => '1990-01-01',
        'address' => 'Cairo',
    ]);

    $response->assertStatus(201);
    // should NOT return token before OTP verification
    $response->assertJsonMissing(['token']);
}
```

---

## 9. Code Style & Conventions

| القاعدة | الوضع الحالي | المطلوب |
|---------|-------------|---------|
| Validation | Inline `Validator::make()` | Form Requests |
| Response format | Manual `response()->json()` | `ApiResponse` trait |
| Password hashing | Manual + cast (double) | Cast only |
| Payment config | `env('STRIPE_SECRET')` | `config('services.stripe.secret')` |
| Comments in code | عربية mixed | PHPDoc إنجليزي أو self-documenting |
| Indentation | `AuthController` inconsistent | PSR-12 |
| Unused imports | `Meal`, `Validator` | Clean up |

---

## 10. ملخص الأولويات (Action Items)

### 🔴 Critical (اعملها فورًا)

1. إصلاح **double password hashing** — AuthController + ProfileController + resetPassword
2. إضافة **`ratings.comment`** و **`orders.payment_status`** migrations
3. شيل **`test_code`** من OTP response
4. **لا token قبل OTP verification**
5. Fix **Stripe charge قبل DB transaction**
6. شيل **`payment_method`** من `OrderItem::$fillable`

### 🟡 High (قبل أي features جديدة)

7. `ApiResponse` trait — stop duplicating response blocks
8. Form Requests لكل endpoint
9. `config/services.php` لـ Stripe
10. Rate limiting على auth/OTP
11. `random_int()` بدل `rand()` للـ OTP
12. API Resources بدل raw models

### 🟢 Medium (أثناء التطوير)

13. Service layer (Payment, OTP, Cart, Pricing)
14. Actions + Repository interfaces
15. Feature tests للـ auth + checkout
16. README documentation
17. Pagination + API versioning
18. Queued notifications
19. Admin endpoints (order status update, meal CRUD)
20. Consolidate `auth:sanctum` route groups

---

## 11. ما تم تنفيذه vs الناقص

### ✅ مُنفَّذ (Functional)

- Phone registration + OTP send/verify + login/logout + password reset
- Categories listing
- Meals list/filter/show
- Favorites toggle/list
- Cart add/view/remove
- Order create (cash + Stripe card) + order history
- Meal ratings (upsert)
- Profile get/update
- Notifications list/mark read
- Database seeders (categories, meals, users)
- Order placed notification

### ❌ ناقص / Scaffold

- README (Laravel default)
- Form Requests, Actions, Services, Repositories
- API Resources
- Feature tests
- Admin panel / order status workflow
- SMS delivery for OTP
- Image upload
- Pagination
- API documentation (OpenAPI)
- Order status updates beyond `pending`

---

## 12. المراجع

- [Laravel Form Requests](https://laravel.com/docs/validation#form-request-validation)
- [Laravel Sanctum](https://laravel.com/docs/sanctum)
- [Stripe Payment Intents (recommended over Charges API)](https://stripe.com/docs/payments/payment-intents)
- [SOLID Principles in PHP](https://www.php.net/manual/en/language.oop5.basic.php)
- Project Rules: Laravel Code Quality Guidelines

---

*هذا الملف مرجع حي — حدّثه مع كل refactor أو module جديد.*
