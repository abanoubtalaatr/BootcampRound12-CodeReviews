# FoodIfy — Code Review & Architecture Guide

> **الهدف:** Controllers رفيعة تمامًا — بدون validation وبدون business logic.  
> كل المنطق في Actions / Services / Repositories مع تطبيق SOLID في كل الطبقات.

**تاريخ المراجعة:** 5 يوليو 2026  
**الطالب:** مصطفى (Mostafa Elshazly)  
**المستودع:** [MostafaElshazly/Foodify](https://github.com/MostafaElshazly/Foodify)  
**النطاق:** `app/` — Auth + Catalog + Cart + Orders (Cash + Stripe) + Favorites + Ratings + Profile + Notifications + Admin (جزئي)

---

## 1. ملخص تنفيذي

| المنطقة | الحالة | التقييم |
|---------|--------|---------|
| Auth Module | مُنفَّذ بالكامل | 🔴 bugs حرجة (double hash + OTP leak) |
| OTP Flow | مُنفَّذ (بدون SMS حقيقي) | 🔴 OTP يُرجَع في الـ response |
| Form Requests | غير موجود | 🔴 validation داخل Controllers |
| Actions / Services | غير موجود | 🔴 كل المنطق في Controllers |
| Repositories | غير موجود | 🔴 Eloquent مباشرة |
| API Resources | غير موجود | 🟡 raw models في الـ response |
| `ApiResponse` Trait | غير موجود | 🟡 response block مكرر ~8 مرات |
| Catalog (Category/Meal) | مُنفَّذ | ✅ |
| Cart | مُنفَّذ | ✅ add/view/remove يعمل |
| Orders + Stripe | مُنفَّذ | 🔴 Stripe خارج transaction + `payment_status` bug |
| Checkout (Cash + Card) | مُنفَّذ | 🟡 functionally يعمل |
| Favorites / Ratings | مُنفَّذ | 🟡 `comment` column ناقص في ratings |
| Profile | مُنفَّذ | ✅ get/update — double hash في password |
| Notifications | مُنفَّذ | ✅ list + mark read |
| Admin | جزئي | 🟡 routes/controllers موجودة جزئيًا |
| Feature Tests | scaffold فقط | 🔴 |
| SOLID Compliance | ضعيف | 🔴 fat controllers everywhere |

**الخلاصة:** Mostafa بنى **FoodIfy MVP functionally كامل نسبيًا (~68%)** — Auth، Catalog، Cart، Checkout (Cash + Stripe)، Orders، Favorites، Ratings، Profile، Notifications كلها شغالة. ده من **أكثر المشاريع اكتمالًا feature-wise** في الـ bootcamp. لكن المعمارية **Fat Controller / Thin Model** بالكامل: validation + business logic + Stripe + DB transactions + notifications كلها داخل الـ controllers. فيه **4 bugs حرجة** لازم تتصلّح فورًا: **double password hashing**، **OTP leaked في response**، **Stripe charge خارج DB transaction**، **`payment_status` column/schema mismatch**.

**Overall Feature Completeness: ~68%**

---

## 2. المعمارية المستهدفة vs الحالية

```
المستهدف:  FormRequest → Controller → Action → Service → RepositoryInterface → Model

Auth:      Request ──────→ AuthController ──────────────────────────────→ Model  (~20%)
Business:  Request ──────→ *Controller ─────────────────────────────────→ Model  (0%)
                          (inline validate + Stripe + DB + notify)
```

### قواعد الـ Controller (إلزامية)

```php
// ✅ Controller مثالي — لا if/else، لا validation، لا DB، لا Hash
public function store(PlaceOrderRequest $request, PlaceOrderAction $action): JsonResponse
{
    return $this->data(
        ['order' => new OrderResource($action->execute($request->user(), $request->toDto()))],
        'Order placed successfully',
        201
    );
}

// ❌ الوضع الحالي في OrderController
$request->validate([...]);                    // validation
Stripe::setApiKey(env('STRIPE_SECRET'));      // external integration
$charge = Charge::create([...]);              // payment logic — خارج transaction
DB::beginTransaction();                       // transaction management
Order::create(['payment_status' => 'paid']);  // column مش في migration
$user->notify(new OrderStatusNotification(...));
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
├── Http/
│   ├── Controllers/Api/
│   ├── Middleware/
│   ├── Requests/          ← Form Request لكل endpoint
│   └── Resources/         ← UserResource, MealResource, OrderResource
├── Repositories/
├── Services/
│   ├── OtpService.php
│   ├── PaymentService.php
│   └── OrderPricingService.php
└── Traits/
    └── ApiResponse.php    ← مفقود 🔴
```

---

## 3. تحليل SOLID

### S — Single Responsibility Principle

| الملف | المشكلة |
|-------|---------|
| `AuthController` (~267 سطر) | register + OTP send/verify + login + logout + reset password + validation + token management |
| `OrderController` | validation + pricing + Stripe charge + DB transaction + cart clearing + notification |
| `CartController` | add/remove/get + total calculation |
| `RatingController` | validation + upsert logic |
| `ProfileController` | get/update + password change + file logic |

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

### 4.1 `AuthController` — 🔴 Critical Bugs

**الإيجابيات:**
- Egyptian phone regex validation ✅
- Sanctum token auth ✅
- OTP expiry check (10 دقائق) ✅
- Login يمنع unverified users (`phone_verified_at === null`) ✅
- Response envelope موحّد `{ status, message, data?, errors? }` ✅

**المخالفات:**

| Method | المشكلة | Severity |
|--------|---------|----------|
| `register` | `Hash::make()` + `'password' => 'hashed'` cast = **double hashing** | 🔴 |
| `register` | Token يُصدر **قبل** OTP verification — تناقض مع رسالة "activate via OTP" | 🔴 |
| `sendOtp` | `rand(1000, 9999)` — weak randomness | 🟡 |
| `sendOtp` | **`test_code` في response** — OTP leak في production | 🔴 |
| `resetPassword` | `Hash::make()` + hashed cast = **double hashing** | 🔴 |
| كل methods | `Validator::make()` inline — بدون Form Request | 🟡 |
| كل methods | 422 response block **مكرر** 6 مرات | 🟡 |

**Double Hashing — 🔴 Critical Bug:**

```php
// User.php — cast يعمل hash تلقائي
'password' => 'hashed',

// AuthController.php — hash يدوي = double hash
'password' => Hash::make($request->password),  // ❌

// النتيجة: Hash::check() في login **هيفشل** بعد التسجيل أو reset
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

---

### 4.2 `OrderController` — 🔴 Critical Fat

**الإيجابيات:**
- DB transaction للـ order creation ✅
- Eager loading في `getMyOrders` (`items.meal`) ✅
- Cart validation قبل الطلب ✅
- Database notification بعد commit ✅
- Cash + Stripe card payment paths ✅

**المشاكل:**

| # | المشكلة | Severity |
|---|---------|----------|
| 1 | **`payment_status` في `Order::create()` لكن مش في migration ولا `$fillable`** | 🔴 silent data loss |
| 2 | **Stripe charge قبل `DB::beginTransaction()`** — لو DB fail، الفلوس اتسحبت | 🔴 |
| 3 | `env('STRIPE_SECRET')` في runtime | 🔴 breaks config cache |
| 4 | Stripe exception message في response | 🟡 information disclosure |
| 5 | Legacy `Charge::create()` + token — deprecated | 🟡 |
| 6 | Delivery fee hardcoded `30` | 🟡 |
| 7 | Success message typo: "application" بدل "order" | 🟢 |

**Schema Fix:**

```php
// migration: add to orders table
$table->string('payment_status')->default('unpaid');

// Order.php $fillable
'payment_status',
```

**Payment Flow المستهدف:**

```php
return DB::transaction(function () use ($data, $user) {
    // 1. validate cart + calculate pricing
    // 2. create order (pending)
    // 3. charge Stripe INSIDE transaction (or refund on rollback)
    // 4. update payment_status
    // 5. create order items + clear cart
    // 6. dispatch notification
});
```

---

### 4.3 `CartController` — ✅ جيد functionally

**الإيجابيات:**
- Upsert/increment logic صح ✅
- Decrement or delete on remove ✅
- Eager load meals ✅
- Cart index + totals يعمل ✅

**المشاكل:**

| # | المشكلة |
|---|---------|
| 1 | Total calculation في controller — ينقل لـ `CartService` |
| 2 | `number_format()` يرجع string — type inconsistency |
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
// RatingController — يحفظ comment
'comment' => $request->comment

// ratings migration — لا يوجد comment column
$table->unsignedTinyInteger('rating');  // فقط
```

**Fix:** أضف migration + `'comment'` في `Rating::$fillable`.

#### `ProfileController` — 🟡

- GET/PUT profile يعمل ✅
- `Hash::make()` + hashed cast = **double hashing** في update password
- يرجع raw `$user` model — استخدم `UserResource`

#### `NotificationController` — ✅

- Scoped lookup (`where('id', $id)`) ✅
- `markAsRead()` ✅

#### Admin Controllers — 🟡 جزئي

- routes/controllers موجودة جزئيًا لإدارة المحتوى أو الطلبات
- **ناقص:** auth/role middleware، CRUD كامل، order status workflow
- **تحسين:** فصل admin routes تحت `/api/admin` + Policy/Guard

---

### 4.5 Models — 🟡

| Model | الحالة | المطلوب |
|-------|--------|---------|
| `User` | ✅ جيد | return types للـ relationships؛ شيل `Hash::make` من controllers |
| `Category` | ✅ | — |
| `Meal` | ✅ | `ingredients` cast ✅؛ أضف scopes (`available`) |
| `CartItem` | ✅ | unique index `(user_id, meal_id)` |
| `Order` | 🔴 | أضف `payment_status` في fillable + migration |
| `OrderItem` | 🔴 | `$fillable` فيه `payment_method` — **column مش موجود** |
| `Rating` | 🔴 | `comment` ناقص من fillable + migration |
| `OtpCode` | ✅ | أضف unique index على `phone_number` |

---

### 4.6 `routes/api.php` — 🟡

**الإيجابيات:**
- Routes واضحة ومنظمة ✅
- Sanctum middleware على protected routes ✅
- Full customer journey wired ✅

**المشاكل:**

| # | المشكلة |
|---|---------|
| 1 | 6 separate `auth:sanctum` groups — ادمجهم في group واحد |
| 2 | Scaffold route `GET /user` — احذفه |
| 3 | No rate limiting على auth/OTP routes |
| 4 | No API versioning (`/api/v1/...`) |

---

### 4.7 Database / Migrations — 🔴

| Issue | Details |
|-------|---------|
| **`orders.payment_status` missing** | Controller يحفظه — mass assignment silently drops it |
| **`ratings.comment` missing** | Controller يحفظ comment — SQL error at runtime |
| **`order_items.payment_method` phantom** | في `$fillable` بس مش في migration |
| **`otp_codes.phone_number` not unique** | `updateOrCreate` يفترض row واحد per phone |
| **`cart_items` no unique (user_id, meal_id)** | duplicate rows ممكنة |
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
| `DatabaseSeeder` | test user + factory users ✅ |
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
| Custom Middleware | ❌ (no phone.verified gate) |
| `ApiResponse` Trait | ❌ |
| Policies | ❌ |
| Feature Tests | ❌ — `ExampleTest` فقط |
| `AppServiceProvider` bindings | ❌ فارغ |

---

## 5. مشاكل أمنية

| # | المشكلة | الملف | Severity |
|---|---------|-------|----------|
| 1 | **Double password hashing** — login broken after register/reset | `AuthController`, `ProfileController` | 🔴 P0 |
| 2 | **OTP exposed in response** (`test_code`) | `AuthController::sendOtp` | 🔴 P0 |
| 3 | **Token issued on register before OTP verify** | `AuthController::register` | 🔴 P0 |
| 4 | **Stripe charged outside DB transaction** — money lost if DB fails | `OrderController` | 🔴 P0 |
| 5 | Weak OTP: `rand()` instead of `random_int()` | `AuthController::sendOtp` | 🔴 P1 |
| 6 | No rate limiting on auth/OTP routes | `routes/api.php` | 🔴 P1 |
| 7 | `env('STRIPE_SECRET')` in controller | `OrderController` | 🟡 P1 |
| 8 | Stripe error message leaked to client | `OrderController` | 🟡 P2 |
| 9 | Raw user model in responses | Auth, Profile | 🟡 P2 |
| 10 | No `STRIPE_SECRET` in `.env.example` | `.env.example` | 🟡 P2 |

---

## 6. ما هو جيد ✅

1. **Full MVP customer journey** — register → OTP → login → browse → cart → checkout (cash/card) → orders → notifications.
2. **Cart module solid** — add, view, remove, quantity logic يعمل.
3. **Checkout end-to-end** — cash path + Stripe card charge functional.
4. **Profile module** — GET/PUT profile يعمل (بعد fix double hash).
5. **Orders history** — `getMyOrders` مع eager loading `items.meal`.
6. **Favorites toggle** — pivot pattern نظيف.
7. **Ratings upsert** — rate + update existing rating (بعد fix comment column).
8. **Notifications** — list + mark-as-read scoped by user.
9. **Sanctum auth** — token-based API protection.
10. **Login blocks unverified users** — `phone_verified_at` check ✅.
11. **Unified JSON envelope** — `{ status, message, data?, errors? }` consistent.
12. **Database seeders** — categories + meals + test users for demo.
13. **Admin partial** — بداية admin layer (أكثر من مشاريع bootcamp كتير).

---

## 7. خطة Refactor

### Sprint 0 — Critical Fixes (P0 — فورًا)

1. إصلاح **double password hashing** — شيل `Hash::make()` من AuthController, ProfileController, resetPassword
2. شيل **`test_code`** من OTP response (أو restrict لـ `APP_DEBUG=true` فقط)
3. **لا token قبل OTP verification** — issue token بعد verify فقط
4. Fix **Stripe-before-transaction** — charge داخل transaction أو compensating refund
5. Migration لـ **`orders.payment_status`** + update `$fillable`
6. Migration لـ **`ratings.comment`** + update `$fillable`
7. شيل **`payment_method`** من `OrderItem::$fillable`

### Sprint 1 — Architecture Foundation (P1)

8. `ApiResponse` trait — stop duplicating response blocks
9. Form Requests: `RegisterRequest`, `LoginRequest`, `SendOtpRequest`, `CreateOrderRequest`, `RateMealRequest`, `UpdateProfileRequest`
10. API Resources: `UserResource`, `MealResource`, `OrderResource`, `CategoryResource`
11. `config/services.php` لـ Stripe — not `env()` in controller
12. Rate limiting على auth/OTP routes
13. `random_int()` بدل `rand()` للـ OTP

### Sprint 2 — Service Layer (P2)

14. `OtpService` — generation, storage, verification, send (SMS gateway ready)
15. `PaymentGatewayInterface` + `StripePaymentGateway` + `CashGateway`
16. `OrderPricingService` — subtotal + delivery fee from config
17. `CartService` — add/remove/get with totals
18. `PlaceOrderAction` — extract OrderController store logic

### Sprint 3 — Quality + Admin (P3)

19. Feature tests: auth flow, cart, checkout (cash + mocked Stripe)
20. Complete Admin module — auth middleware + meal/category CRUD + order status update
21. Settings API (`GET/PATCH /api/settings`)
22. Repository interfaces + DI bindings
23. README documentation (setup, endpoints, env vars)
24. Pagination على list endpoints

---

## 8. Controller المستهدف (مثال: Place Order)

```php
public function store(CreateOrderRequest $request, CreateOrderAction $action): JsonResponse
{
    $order = $action->execute(
        user: $request->user(),
        data: $request->toDto()
    );

    return $this->data(
        ['order' => new OrderResource($order->load('items.meal'))],
        'Order placed successfully!',
        201
    );
}
```

```php
// CreateOrderAction — Stripe داخل transaction
public function execute(User $user, CreateOrderData $data): Order
{
    return DB::transaction(function () use ($user, $data) {
        $pricing = $this->pricingService->calculate($user);
        $order   = $this->orderRepository->createPending($user, $pricing, $data);

        if ($data->paymentMethod === PaymentMethod::Card) {
            $this->paymentGateway->charge($order, $data->stripeToken);
            $order->update(['payment_status' => 'paid']);
        }

        $this->orderRepository->createItemsFromCart($order, $user);
        $this->cartService->clear($user);
        $user->notify(new OrderStatusNotification($order));

        return $order;
    });
}
```

---

## 9. File-by-File Scorecard

| File | Thin | Form Request | Service/Action | Repository | ApiResponse | Verdict |
|------|:----:|:------------:|:--------------:|:----------:|:-----------:|---------|
| `AuthController` | ❌ | ❌ | ❌ | ❌ | 🟡 manual | 🔴 |
| `OrderController` | ❌ | ❌ | ❌ | ❌ | 🟡 manual | 🔴 |
| `CartController` | ❌ | ❌ | ❌ | ❌ | 🟡 manual | 🟡 |
| `CategoryController` | ✅ | — | ❌ | ❌ | 🟡 manual | 🟡 OK |
| `MealController` | 🟡 | — | ❌ | ❌ | 🟡 manual | 🟡 |
| `FavoriteController` | 🟡 | ❌ | ❌ | ❌ | 🟡 manual | 🟡 |
| `RatingController` | ❌ | ❌ | ❌ | ❌ | 🟡 manual | 🔴 schema |
| `ProfileController` | ❌ | ❌ | ❌ | ❌ | 🟡 manual | 🟡 |
| `NotificationController` | 🟡 | — | ❌ | ❌ | 🟡 manual | 🟡 |
| Admin Controllers | 🟡 | ❌ | ❌ | ❌ | 🟡 manual | 🟡 partial |
| Models | — | — | — | — | — | 🟡 gaps |
| Tests | — | — | — | — | — | 🔴 |
| `AppServiceProvider` | — | — | ❌ | ❌ | — | 🔴 |

---

## 10. مقارنة مع مشاريع Bootcamp

| المعيار | Mostafa | Abdella | Mohamed Esmail | Ali |
|---------|:-------:|:-------:|:--------------:|:---:|
| Feature completeness | ~68% | ~70% | ~83% | ~68% |
| Full MVP (cart/orders/checkout) | ✅ | ✅ | ✅ | 🟡 wiring gaps |
| Stripe / Payment | 🟡 inline Charge | ✅ Cashier | 🟡 pending | ✅ Strategy |
| Actions layer | ❌ | ❌ | ❌ | 🟡 partial |
| Form Requests | ❌ | 🟡 3 files | 🟡 partial | 🟡 partial |
| Auth security | 🔴 OTP leak + double hash | ✅ Sanctum + OTP hash | ✅ | ✅ |
| Payment transaction safety | 🔴 charge outside TX | 🟡 inline | 🟡 | ✅ |
| Feature tests | ❌ | ❌ | ✅ | ✅ |
| Admin | 🟡 partial | ❌ | ✅ web | 🟡 users only |
| Fat controllers | 🔴 all | 🔴 all | 🔴 business | 🟡 improving |

**نقطة قوة Mostafa:** **أوسع MVP functionally** — cart + checkout + profile + ratings + notifications كلها wired. Admin partial موجود.  
**نقطة ضعف:** **4 critical bugs** (double hash, OTP leak, Stripe outside TX, payment_status) + zero architecture layers.

---

## 11. ملخص الأولويات (Action Items)

### 🔴 Critical (اعملها فورًا)

1. إصلاح **double password hashing** — AuthController + ProfileController + resetPassword
2. شيل **`test_code`** من OTP response
3. **لا token قبل OTP verification**
4. Fix **Stripe charge قبل DB transaction**
5. Migration لـ **`orders.payment_status`** + **`ratings.comment`**
6. شيل **`payment_method`** من `OrderItem::$fillable`

### 🟡 High (قبل أي features جديدة)

7. `ApiResponse` trait — stop duplicating response blocks
8. Form Requests لكل endpoint
9. `config/services.php` لـ Stripe — not `env()` in controller
10. Rate limiting على auth/OTP
11. `random_int()` بدل `rand()` للـ OTP
12. API Resources بدل raw models
13. Unique index على `(user_id, meal_id)` في cart

### 🟢 Medium (أثناء التطوير)

14. Service layer (Payment, OTP, Cart, Pricing)
15. Actions + Repository interfaces
16. Feature tests للـ auth + checkout
17. Complete Admin module (CRUD + order status + auth guard)
18. Settings API
19. README documentation
20. Pagination + API versioning
21. Queued notifications
22. Consolidate `auth:sanctum` route groups

---

## 12. مقارنة سريعة مع مشاريع FoodIfy

| Feature | Mostafa | Abdella | Elham | Mohamed Esmail |
|---------|:-------:|:-------:|:-----:|:--------------:|
| Cart + Checkout wired | ✅ full | ✅ | ✅ | ✅ |
| Profile API | ✅ | ✅ | ✅ | 🟡 |
| Ratings | ✅ | ❌ | 🟡 | ❌ |
| OTP security | 🔴 leaked in response | ✅ hashed + rate limit | ✅ | ✅ |
| Password hashing | 🔴 double hash bug | ✅ | ✅ | 🟡 |
| Payment safety | 🔴 outside transaction | 🟡 inline | ✅ | 🟡 |
| Admin | 🟡 partial | ❌ | ✅ full | ✅ web |
| Architecture layers | ❌ fat controllers | ❌ fat controllers | ✅ Actions | 🟡 |
| Test coverage | ❌ | ❌ | 🟡 | ✅ |

**الخلاصة:** Mostafa في **النصف العلوي** من الـ bootcamp من ناحية **feature completeness** (MVP شغال)، لكن في **النصف السفلي** من ناحية **security + architecture** — bugs حرجة تمنع production demo بدون fixes.

---

## 13. تقرير Feature Completeness — النواقص في الـ Application

> **مرجع المتطلبات:** Authentication, Profile, Cart, My Orders, Notifications, Favorites, Meals/Categories, Reset Password, Category Details, Meal Details, Settings, Payments/Checkout, Admin.

### 13.1 Feature Matrix

| # | Feature | الحالة | Route / Implementation | النواقص |
|---|---------|--------|------------------------|---------|
| 1 | **Authentication** | 🟡 **75%** | register, login, logout, send/verify OTP | double hash, OTP leak, token before verify |
| 2 | **Reset Password** | 🟡 **70%** | send-otp, verify-otp, reset-password | double hash, inline validation |
| 3 | **Profile** | ✅ **88%** | GET/PUT profile | double hash on password change, no UserResource |
| 4 | **Categories** | ✅ **90%** | `GET /api/categories` | no pagination |
| 5 | **Category Details** | ✅ **90%** | category show + meals | — |
| 6 | **Meals** | ✅ **88%** | list + filter by category | no pagination |
| 7 | **Meal Details** | ✅ **90%** | meal show + ingredients | — |
| 8 | **Favorites** | ✅ **92%** | index, toggle | — |
| 9 | **Cart** | ✅ **90%** | index, add, remove | no unique constraint, inline validation |
| 10 | **Checkout** | 🟡 **78%** | `POST /api/orders` cash + card | Stripe outside TX, payment_status bug |
| 11 | **Payment** | 🟡 **72%** | Stripe Charge API inline | no PaymentMethod CRUD, env() in controller |
| 12 | **My Orders** | ✅ **88%** | `GET /api/orders` | paginated 🟡 optional |
| 13 | **Order Details** | 🟡 **75%** | order show partial | ownership check 🟡 |
| 14 | **Notifications** | ✅ **85%** | index, mark read | no mark-all-read |
| 15 | **Ratings** | 🟡 **70%** | rate meal upsert | comment column missing |
| 16 | **Settings** | 🔴 **0%** | — | — |
| 17 | **Admin** | 🟡 **35%** | partial routes/controllers | no full CRUD, no auth guard |

**Overall Feature Completeness: ~68%**

### 13.2 Route Map

```
POST /api/register, /login, /logout                    ✅ (bugs in auth)
POST /api/send-otp, /verify-otp, /reset-password     ✅ (OTP leak)

GET  /api/categories, /meals, /meals/{id}             ✅
POST /api/favorites/toggle, GET /api/favorites        ✅
GET/POST/DELETE /api/cart/*                           ✅
POST /api/orders, GET /api/orders                     ✅ (payment_status bug)
POST /api/ratings                                     🟡 (comment column)
GET/PUT /api/profile                                  ✅
GET /api/notifications, mark-read                     ✅

/api/settings                                         ❌
/api/admin/* (full CRUD)                              🟡 partial
```

### 13.3 Database Tables

| Table | موجود | API Wired |
|-------|-------|-----------|
| `users` | ✅ | Auth + Profile |
| `categories`, `meals`, `ingredients` | ✅ | Catalog |
| `favorites`, `cart_items` | ✅ | Favorites + Cart |
| `orders`, `order_items` | ✅ | Orders (payment_status gap) |
| `ratings` | 🟡 | Ratings (comment gap) |
| `otp_codes` | ✅ | Auth OTP |
| `notifications` | ✅ | Notifications |
| `settings` | ❌ | — |

### 13.4 Feature Completeness Scorecard

| Category | Score |
|----------|-------|
| Auth + Reset + OTP | 72% |
| Profile + Catalog | 89% |
| Cart + Favorites | 91% |
| Checkout + Payment + Orders | 78% |
| Ratings + Notifications | 78% |
| Settings + Admin | 18% |
| **Overall** | **~68%** |

---

## 14. المراجع

- [Laravel Sanctum](https://laravel.com/docs/sanctum)
- [Laravel Form Requests](https://laravel.com/docs/validation#form-request-validation)
- [Stripe Payment Intents (recommended over Charges API)](https://stripe.com/docs/payments/payment-intents)
- [Laravel Database Transactions](https://laravel.com/docs/database#database-transactions)
- مراجعة Abdella: [abdella/FoodIfy/CODE_REVIEW.md](../../abdella/FoodIfy/CODE_REVIEW.md)
- مراجعة Ali: [ali/FoodIfy/CODE_REVIEW.md](../../ali/FoodIfy/CODE_REVIEW.md)
- مراجعة Mohamed Esmail: [mohamedEsmail/FoodIfy/CODE_REVIEW.md](../../mohamedEsmail/FoodIfy/CODE_REVIEW.md)

---

*هذا الملف مرجع حي — حدّثه مع كل refactor أو module جديد.*
