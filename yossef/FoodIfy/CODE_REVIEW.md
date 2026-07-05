# FoodIfy — Code Review & Architecture Guide

> **الهدف:** Controllers رفيعة تمامًا — بدون validation وبدون business logic.  
> كل المنطق في Actions / Services / Repositories مع تطبيق SOLID في كل الطبقات.

**تاريخ المراجعة:** 5 يوليو 2026  
**الطالب:** يوسف (Yossef / Youssef Mehrez)  
**المستودع:** [YossefElshazly/Foodify](https://github.com/YossefElshazly/Foodify)  
**النطاق:** `app/` — Auth + Catalog + Cart + Orders + Stripe + Addresses + Payments + Favorites + Reviews + Profile + Notifications

---

## 1. ملخص تنفيذي

| المنطقة | الحالة | التقييم |
|---------|--------|---------|
| Auth Module | مُنفَّذ | 🔴 OTP hardcoded + exposed + reset chain broken |
| OTP / Phone Verify | `OtpService` موجود | 🔴 `123456` لكل المستخدمين |
| Reset Password | flow موجود structurally | 🔴 لا يربط verify بـ reset |
| Form Requests | 4 files scaffold | 🔴 `authorize: false`, rules فارغة — unused |
| API Resources | 4 files scaffold | 🔴 unused — raw models في كل response |
| Actions / Repositories | غير موجود | 🔴 |
| Catalog (Category/Meal) | مُنفَّذ | 🔴 category write ops **public بدون auth** |
| Cart | مُنفَّذ | 🟡 user-scoped — destroy بدون JSON response |
| Orders + Stripe | مُنفَّذ | 🔴 **IDOR على updateStatus** + address ownership |
| Checkout | داخل OrderController | 🟡 transaction + cart clear + notification |
| Payments | مُنفَّذ | 🔴 **IDOR — كل payments لكل users** |
| Addresses / Favorites / Reviews | مُنفَّذ | ✅ scoped correctly |
| Profile | مُنفَّذ | 🟡 image upload — `UserResource` unused |
| Notifications | مُنفَّذ | ✅ user-scoped |
| Payment Methods | read-only | ✅ |
| Settings | غير موجود | 🔴 |
| Admin | غير موجود | 🔴 |
| README | project-specific | ✅ أفضل من معظم المشاريع |
| Feature Tests | scaffold فقط | 🔴 |
| SOLID Compliance | ضعيف | 🔴 fat controllers everywhere |
| **Security Posture** | ثغرات حرجة | 🔴 **تخفض التقييم الكلي** |

**الخلاصة:** يوسف بنى **FoodIfy API واسع النطاق functionally (~70%)** — Auth، Catalog، Cart، Checkout (Stripe)، Orders، Addresses، Favorites، Reviews، Profile، Notifications كلها موجودة. README يوثّق الـ features بوضوح. لكن **الأمان يُسقط التقييم الفعلي إلى ~58% weighted**: IDOR حرج على payments و order status update، categories CRUD عام بدون authentication، OTP hardcoded ومكشوف في API، وسلسلة reset password مكسورة. Form Requests و API Resources **اتعملوا scaffold ومش مستخدمين**.

| المقياس | النسبة |
|---------|--------|
| **Feature Completeness** | **~70%** |
| **Weighted Score (security-adjusted)** | **~58%** |

> **لماذا الفرق؟** تغطية الـ features جيدة، لكن ثغرات IDOR و public write ops و OTP leaks تجعل الـ API **غير آمن للإنتاج** — يُخصم ~12 نقطة من التقييم الكلي.

---

## 2. المعمارية المستهدفة vs الحالية

```
المستهدف:  FormRequest → Controller → Action → Service → RepositoryInterface → Model

Auth:      Request ──→ AuthController ──→ OtpService (hardcoded) ──→ Model  (~25%)
Business:  Request ──→ Controller ───────────────────────────────────→ Model  (0%)
                          (inline validate + Stripe + DB + notify)
                          (NO ownership checks on payments/orders)
```

### قواعد الـ Controller (إلزامية)

```php
// ✅ Controller مثالي
public function store(PlaceOrderRequest $request, PlaceOrderAction $action): JsonResponse
{
    return response()->json(
        new OrderResource($action->execute($request->user(), $request->toDto())),
        201
    );
}

// ❌ الوضع الحالي في PaymentController
Payment::with('order')->get();  // ALL users' payments — IDOR

// ❌ الوضع الحالي في OrderController@updateStatus
$order = Order::findOrFail($id);
$order->update(['status' => $request->status]);  // ANY user can change ANY order
```

### Scaffold موجود لكن غير مفعّل

```
app/Http/Requests/     ← RegisterRequest, LoginRequest, MealRequest, CheckoutRequest
                         authorize() = false, rules() = [], NOT wired
app/Http/Resources/    ← UserResource, MealResource, CategoryResource, OrderResource
                         parent::toArray() only, NOT used
app/Http/Controllers/API/CheckoutController.php  ← empty, not routed
app/Http/Controllers/API/HomeController.php      ← empty, not routed
```

### هيكل المجلدات المقترح

```
app/
├── Actions/
│   ├── Auth/
│   ├── Cart/
│   ├── Order/
│   └── Payment/
├── Contracts/
│   ├── Repositories/
│   └── Services/
├── DTOs/
├── Http/
│   ├── Controllers/API/
│   ├── Middleware/          ← admin, phone.verified
│   ├── Requests/            ← implement + wire existing scaffold
│   └── Resources/           ← implement + use everywhere
├── Policies/                ← OrderPolicy, PaymentPolicy, AddressPolicy
├── Repositories/
├── Services/
│   ├── OtpService.php       ← موجود — fix hardcoded OTP + add SmsGateway
│   └── StripePaymentService.php
└── Traits/
    └── ApiResponse.php      ← add for consistent envelope
```

---

## 3. تحليل SOLID

### S — Single Responsibility Principle

| الملف | المشكلة |
|-------|---------|
| `AuthController` | register + login + logout + OTP + forgot + reset — ~228 lines |
| `OrderController` | create + track + **updateStatus** + list + show + notifications + Stripe |
| `MealController` | CRUD + search + category filter + file handling |
| `ProfileController` | show + update + image upload/delete |
| `PaymentController` | index + show — **no authorization logic at all** |

### O — Open/Closed Principle

- OTP logic hardcoded في `OtpService` — أي gateway جديد (Vonage/SMS) يتطلب تعديل الـ service مباشرة.
- Stripe charge logic داخل `OrderController` — لا abstraction للـ payment gateway.
- Vonage في `composer.json` + config لكن **غير مستخدم** — misleading.

### L — Liskov Substitution Principle

- N/A — لا interfaces.

### I — Interface Segregation Principle

- N/A — لا interfaces.

### D — Dependency Inversion Principle

| الوضع الحالي | المطلوب |
|--------------|---------|
| `OtpService` concrete only | `SmsGatewayInterface` + Vonage impl |
| Controllers → Eloquent مباشرة | Repository interfaces |
| Stripe inline في OrderController | `PaymentGatewayInterface` |
| `AppServiceProvider` فارغ | DI bindings + Policies |

---

## 4. مراجعة ملف بملف

### 4.1 `AuthController` — 🔴 Security Critical

**الإيجابيات:**
- Register لا يصدر token قبل verify ✅
- Login يمنع unverified users ✅
- OTP expiry check ✅
- `OtpService` injected via DI ✅
- Password reset flow structure موجود ✅

**المشاكل:**

| Method | المشكلة | Severity |
|--------|---------|----------|
| `register` | `Hash::make()` + `'password' => 'hashed'` cast = **double hashing** | 🔴 |
| `resendOtp` | **`'otp' => $user->otp` في response** — OTP leak | 🔴 |
| `resetPassword` | **لا يتحقق إن OTP verified** — phone + password فقط | 🔴 |
| `verifyResetOtp` | يverify OTP لكن **لا session/token** يربط بـ reset step | 🔴 |
| `logout` | **Dead code** — return مكرر unreachable | 🟡 |
| `profile` | **Unused** — routes تستخدم ProfileController | 🟡 |
| كل methods | Inline `$request->validate()` — Form Requests موجودة لكن unused | 🟡 |
| `login` | يرجع full `$user` — ممكن يحتوي OTP fields | 🟡 |

**Double Hashing:**

```php
// User.php
'password' => 'hashed',

// AuthController.php
'password' => Hash::make($request->password),  // ❌ double hash
```

**Password Reset Chain Broken:**

```
forgot-password → verify-reset-otp → reset-password
                      ↓                    ↓
                 validates OTP      NO check that OTP was verified
                                    anyone with phone can reset
```

---

### 4.2 `OtpService` — 🔴 Critical

```php
// OtpService.php
'otp' => '123456',  // ❌ hardcoded — same for ALL users
```

| Issue | Details |
|-------|---------|
| Hardcoded OTP | Every user gets `123456` |
| No SMS | Vonage installed but unused |
| Plain text storage | OTP in `users.otp` column |
| Exposed via API | `resendOtp` returns OTP in JSON |

**Fix:**

```php
$otp = str_pad((string) random_int(0, 999999), 6, '0', STR_PAD_LEFT);
// Send via Vonage/SMS — NEVER return in response
```

---

### 4.3 `OrderController` — 🔴 IDOR + Fat Controller

**الإيجابيات:**
- DB transaction ✅
- Cart → order → Stripe charge → clear cart → notification ✅
- User-scoped index/show/track ✅
- Price snapshot on order items ✅
- Eager loading ✅

**المشاكل:**

| # | المشكلة | Severity |
|---|---------|----------|
| 1 | **`updateStatus` — no user_id check** — أي user يغيّر status أي order | 🔴 **IDOR** |
| 2 | **`address_id` validated `exists` but not ownership** | 🔴 |
| 3 | Stripe charge logic داخل controller | 🟡 |
| 4 | Exception message exposed in 500 response | 🟡 |
| 5 | Payment status always `pending` — never updated to `paid` | 🟡 |
| 6 | Inline validation | 🟡 |

```php
// ❌ updateStatus — privilege escalation
$order = Order::findOrFail($id);
$order->update(['status' => $request->status]);

// ✅ Fix Option A: admin middleware only
Route::put('orders/{id}/status', ...)->middleware('admin');

// ✅ Fix Option B: ownership check (if user can cancel own order)
$order = Order::where('id', $id)
    ->where('user_id', $request->user()->id)
    ->firstOrFail();
$order->update(['status' => $request->only('cancelled')]); // whitelist statuses
```

```php
// ❌ address_id — no ownership
'address_id' => 'required|exists:addresses,id',

// ✅ Fix
'address_id' => [
    'required',
    Rule::exists('addresses', 'id')->where('user_id', $request->user()->id),
],
```

---

### 4.4 `PaymentController` — 🔴 Critical IDOR

```php
// ❌ PaymentController — ALL users' payments visible
public function index()
{
    return Payment::with('order')->get();
}

public function show($id)
{
    return Payment::with('order')->findOrFail($id);  // any payment by ID
}
```

| Attack Scenario | Impact |
|-----------------|--------|
| `GET /api/payments` as User A | يرى payments لـ User B, C, D... |
| `GET /api/payments/{id}` | enumerate payment IDs — full order + amount leak |

**Fix:**

```php
public function index(Request $request)
{
    return Payment::whereHas('order', fn ($q) =>
        $q->where('user_id', $request->user()->id)
    )->with('order')->get();
}

public function show(Request $request, $id)
{
    return Payment::whereHas('order', fn ($q) =>
        $q->where('user_id', $request->user()->id)
    )->with('order')->findOrFail($id);
}
```

**أفضل:** `PaymentPolicy` + `$this->authorize('view', $payment)`.

---

### 4.5 `CategoryController` — 🔴 Public Write Ops

```php
// routes/api.php — NO auth middleware on categories
Route::apiResource('categories', CategoryController::class);
```

**النتيجة:** أي شخص (بدون token) يقدر `POST/PUT/DELETE` categories.

| Attack | Endpoint | Auth Required? |
|--------|----------|----------------|
| Create fake category | `POST /api/categories` | ❌ لا |
| Deface category | `PUT /api/categories/{id}` | ❌ لا |
| Delete all categories | `DELETE /api/categories/{id}` | ❌ لا |

**Fix:**

```php
// Public read only
Route::get('categories', [CategoryController::class, 'index']);
Route::get('categories/{category}', [CategoryController::class, 'show']);

// Admin write
Route::middleware(['auth:sanctum', 'admin'])->group(function () {
    Route::post('categories', ...);
    Route::put('categories/{category}', ...);
    Route::delete('categories/{category}', ...);
});
```

**مشاكل إضافية:**
- Validation typo: `'nullable||image'` — double pipe
- `update` uses `$request->all()` — mass assignment risk
- Image upload commented out on create

---

### 4.6 `MealController` — 🟡

**الإيجابيات:** search, category filter, CRUD, image handling on update/destroy ✅

**المشاكل:**
- All meal routes require auth (including GET index) — catalog not public
- Image upload commented out on `store`
- `MealRequest` + `MealResource` exist but unused

---

### 4.7 `CartController` — 🟡

**الإيجابيات:** firstOrCreate cart, price snapshot, user-scoped ✅

**المشاكل:**
- `destroy` — **no JSON response returned**
- No unique `(cart_id, meal_id)` in migration — duplicate rows possible
- Inline validation

---

### 4.8 Controllers الأخرى

| Controller | Verdict | Notes |
|------------|---------|-------|
| `AddressController` | ✅ mostly | CRUD scoped — `update` بدون validation |
| `FavoriteController` | ✅ | firstOrCreate, user-scoped |
| `ReviewController` | ✅ | duplicate check, user-scoped |
| `NotificationController` | ✅ | clean, user-scoped |
| `ProfileController` | 🟡 | image upload OK — `UserResource` unused |
| `PaymentMethodController` | ✅ | read-only, safe |
| `CheckoutController` | 🔴 stub | empty, not routed |
| `HomeController` | 🔴 stub | empty, not routed |

---

### 4.9 Form Requests — 🔴 Scaffold Only

| File | authorize() | rules() | Used? |
|------|-------------|---------|-------|
| `RegisterRequest` | `false` | `[]` | ❌ |
| `LoginRequest` | `false` | `[]` | ❌ |
| `MealRequest` | `false` | `[]` | ❌ |
| `CheckoutRequest` | `false` | `[]` | ❌ |

---

### 4.10 API Resources — 🔴 Unused

| Resource | Status |
|----------|--------|
| `UserResource` | `parent::toArray()` only — hides nothing, unused |
| `MealResource` | same |
| `CategoryResource` | same |
| `OrderResource` | same |

---

### 4.11 Models — 🟡

| Model | الحالة |
|-------|--------|
| `User` | 🟡 missing `birth_date, address, image` in fillable; hidden references nonexistent `mobile_verify_code` |
| `Category`, `Meal`, `MealImage` | ✅ |
| `Cart`, `CartItem` | ✅ — 🟡 no unique(cart_id, meal_id) |
| `Favorite` | ✅ unique constraint |
| `Address` | ✅ |
| `Order`, `OrderItem` | ✅ status enum, price snapshot |
| `Payment`, `PaymentMethod` | ✅ — payment status never transitions |
| `Notification` | ✅ custom model |
| `Review` | 🟡 no unique(user_id, meal_id) in DB |

---

### 4.12 `routes/api.php` — 🔴 Security Routing

| # | Issue | Severity |
|---|-------|----------|
| Categories CRUD **public** | unauthorized writes | 🔴 |
| Two separate `auth:sanctum` groups | merge for clarity | 🟡 |
| Meals listing requires auth | unusual for catalog | 🟡 |
| `resend-otp` public — **no rate limit** | brute force | 🔴 |
| Order status update inside auth group but **no ownership** | IDOR | 🔴 |

---

### 4.13 Database / Migrations — 🟡

| Migration | Notes |
|-----------|-------|
| `users` + phone/otp + profile fields | ✅ — OTP plain text |
| `categories`, `meals`, `meal_images` | ✅ |
| `carts` (unique user_id) | ✅ |
| `cart_items` | 🟡 no unique(cart_id, meal_id) |
| `favorites` | ✅ unique constraint |
| `addresses`, `orders`, `order_items` | ✅ |
| `payments`, `notifications`, `reviews` | ✅ |
| **`create_meals_table.php.php`** | 🔴 filename typo — double `.php` |

---

## 5. مشاكل أمنية

> **هذا القسم هو الأهم في المراجعة.** الثغرات التالية **يجب إصلاحها قبل أي demo أو deploy**.

### 5.1 ثغرات حرجة (P0 — فورًا)

| # | الثغرة | الملف / Route | النوع | التأثير |
|---|--------|---------------|-------|---------|
| 1 | **Payment IDOR** — كل users يروا كل payments | `PaymentController` | IDOR | تسريب amounts, order data, PII |
| 2 | **Order Status IDOR** — أي user يغيّر status أي order | `OrderController@updateStatus` | Privilege Escalation | cancel/deliver orders للغير |
| 3 | **Public Category CRUD** | `routes/api.php` | Broken Access Control | deface/delete catalog |
| 4 | **Hardcoded OTP `123456`** | `OtpService:13` | Auth Bypass | account takeover trivial |
| 5 | **OTP exposed in API** | `AuthController@resendOtp` | Information Disclosure | OTP in JSON response |
| 6 | **Reset password without OTP verify** | `AuthController@resetPassword` | Auth Bypass | reset أي account بالـ phone |
| 7 | **Double password hashing** | register + reset | Logic Bug | login broken after register |

### 5.2 ثغرات عالية (P1 — قبل demo)

| # | الثغرة | الملف | التأثير |
|---|--------|-------|---------|
| 8 | Address ownership not checked on checkout | `OrderController` | order shipped to others' address |
| 9 | No rate limiting on OTP/auth routes | `routes/api.php` | brute force OTP |
| 10 | Full `$user` model in login response | `AuthController` | OTP/password hash field leak risk |
| 11 | Exception `$e->getMessage()` in 500 | `OrderController` | info disclosure |
| 12 | `$request->all()` on category update | `CategoryController` | mass assignment |

### 5.3 إصلاحات أمنية — Code Snippets

**Payment Policy:**

```php
// app/Policies/PaymentPolicy.php
public function view(User $user, Payment $payment): bool
{
    return $payment->order->user_id === $user->id;
}

// PaymentController
public function show(Payment $payment)
{
    $this->authorize('view', $payment);
    return new PaymentResource($payment->load('order'));
}
```

**Order Status — Admin Only:**

```php
// bootstrap/app.php or RouteServiceProvider
Route::put('orders/{order}/status', [OrderController::class, 'updateStatus'])
    ->middleware(['auth:sanctum', 'admin']);
```

**Category Routes Split:**

```php
Route::get('categories', [CategoryController::class, 'index']);
Route::get('categories/{category}', [CategoryController::class, 'show']);

Route::middleware(['auth:sanctum', 'admin'])->group(function () {
    Route::apiResource('categories', CategoryController::class)->except(['index', 'show']);
});
```

**Reset Password — Require Verified Token:**

```php
// verify-reset-otp stores token in cache
Cache::put("reset:{$phone}", true, now()->addMinutes(10));

// reset-password checks cache
if (!Cache::pull("reset:{$request->phone}")) {
    return response()->json(['message' => 'OTP not verified'], 403);
}
```

### 5.4 Security Scorecard

| Area | Score | Notes |
|------|-------|-------|
| Authentication | 35% | OTP hardcoded + exposed + reset broken |
| Authorization | 25% | IDOR on payments + orders + public writes |
| Input Validation | 45% | inline only — scaffold unused |
| Data Exposure | 40% | raw models, OTP in response |
| Rate Limiting | 10% | none on auth/OTP |
| **Overall Security** | **~31%** | **blocks production use** |

---

## 6. ما هو جيد ✅

1. **Broad feature coverage** — 14 controller، 14 model — أوسع domain model في Bootcamp.
2. **Full customer journey** — register → verify → browse → cart → Stripe checkout → orders → notifications.
3. **Stripe integration** — charge on checkout داخل order flow.
4. **README project-specific** — يوثّق endpoints و features (نادر في Bootcamp).
5. **DB transactions** — order creation wrapped in transaction.
6. **Price snapshot** — order items تحفظ السعر وقت الطلب.
7. **User-scoped resources** — addresses, favorites, reviews, notifications صح.
8. **Review duplicate check** — prevents double review per meal.
9. **Cart firstOrCreate pattern** — one cart per user.
10. **Sanctum** — protected routes behind `auth:sanctum` (except categories write).
11. **Scaffold awareness** — Form Requests + Resources files exist (need wiring).
12. **Custom Notification model** — works for order status updates.

---

## 7. خطة Refactor

### Sprint 0 — Security Fixes (P0 — **قبل أي شيء**)

1. Scope `PaymentController` to authenticated user's orders — or delete endpoints
2. Add admin middleware OR ownership check on `OrderController@updateStatus`
3. Move category write routes behind `auth:sanctum` + `admin`
4. Replace hardcoded OTP with `random_int()` — never return in response
5. Fix password reset chain — cache token after verify-reset-otp
6. Fix double hashing — plain password, let cast handle it
7. Verify `address_id` belongs to user on order create
8. Add rate limiting: `throttle:3,5` on OTP routes

### Sprint 1 — Wire Existing Scaffold (P1)

9. Implement Form Requests (Register, Login, Meal, Checkout/Order)
10. Implement API Resources — hide OTP, internal fields
11. Fix `CartController@destroy` — return JSON response
12. Fix Category validation typo + add validation on update
13. Delete or implement `CheckoutController`, `HomeController`

### Sprint 2 — Architecture (P2)

14. `SmsGatewayInterface` + Vonage implementation (or remove Vonage)
15. `PlaceOrderAction` — extract Stripe + transaction from OrderController
16. Policies: `OrderPolicy`, `PaymentPolicy`, `AddressPolicy`, `CategoryPolicy`
17. `ApiResponse` trait for consistent envelope
18. Payment status workflow: pending → paid after Stripe success
19. Unique constraint on `cart_items(cart_id, meal_id)`

### Sprint 3 — Quality (P3)

20. Feature tests: **security scenarios first** (payment IDOR, order status, category auth)
21. Rename migration `create_meals_table.php.php`
22. Seeders for categories/meals demo data
23. Settings module + Admin module
24. Pagination on list endpoints

---

## 8. Controller المستهدف (مثال: Payment Index — بعد إصلاح IDOR)

```php
// ❌ Before — IDOR
public function index()
{
    return Payment::with('order')->get();
}

// ✅ After — thin + authorized
public function index(Request $request, ListUserPaymentsAction $action): JsonResponse
{
    $payments = $action->execute($request->user());

    return response()->json([
        'data' => PaymentResource::collection($payments),
    ]);
}

// ListUserPaymentsAction
public function execute(User $user): Collection
{
    return Payment::whereHas('order', fn ($q) =>
        $q->where('user_id', $user->id)
    )->with('order')->latest()->get();
}
```

---

## 9. File-by-File Scorecard

| File | Thin | Form Request | Service/Action | Authorization | ApiResponse/Resource | Verdict |
|------|:----:|:------------:|:--------------:|:-------------:|:--------------------:|---------|
| `AuthController` | ❌ | ❌ scaffold | 🟡 OtpService | 🔴 OTP leaks | ❌ raw user | 🔴 |
| `OtpService` | — | — | 🔴 hardcoded | — | — | 🔴 |
| `OrderController` | ❌ | ❌ | ❌ | 🔴 status IDOR | ❌ | 🔴 |
| `PaymentController` | ✅ | — | ❌ | 🔴 **IDOR** | ❌ | 🔴 |
| `CategoryController` | ❌ | ❌ | ❌ | 🔴 public writes | ❌ | 🔴 |
| `MealController` | ❌ | ❌ scaffold | ❌ | 🟡 auth on read | ❌ scaffold | 🟡 |
| `CartController` | ❌ | ❌ | ❌ | ✅ scoped | ❌ | 🟡 |
| `AddressController` | 🟡 | ❌ | ❌ | ✅ scoped | ❌ | 🟡 |
| `FavoriteController` | ✅ | ❌ | ❌ | ✅ | ❌ | 🟡 |
| `ReviewController` | 🟡 | ❌ | ❌ | ✅ | ❌ | 🟡 |
| `ProfileController` | ❌ | ❌ | ❌ | ✅ | ❌ scaffold | 🟡 |
| `NotificationController` | ✅ | — | ❌ | ✅ | ❌ | 🟡 |
| `PaymentMethodController` | ✅ | — | ❌ | ✅ read-only | ❌ | ✅ |
| Form Requests (4) | — | 🔴 empty | — | — | — | 🔴 |
| API Resources (4) | — | — | — | — | 🔴 unused | 🔴 |
| Tests | — | — | — | — | — | 🔴 |

---

## 10. مقارنة مع مشاريع Bootcamp

| المعيار | Yossef | Abdella | Mohamed Esmail | Ali |
|---------|:------:|:-------:|:--------------:|:---:|
| Feature completeness | ~70% | ~70% | ~83% | ~76% |
| **Weighted (security)** | **~58%** | ~68% | ~78% | ~72% |
| Stripe / Payment | ✅ checkout | ✅ Cashier | 🟡 pending | ✅ checkout |
| Domain breadth | ✅ widest | 🟡 | 🟡 | 🟡 |
| README | ✅ | ❌ default | ❌ | 🟡 |
| Form Requests | scaffold unused | 🟡 3 files | 🟡 partial | ✅ many |
| Authorization | 🔴 **critical IDOR** | 🟡 partial | 🟡 | 🟡 |
| OTP security | 🔴 worst | ✅ hashed | 🟡 | 🟡 |
| Actions layer | ❌ | ❌ | ❌ | 🟡 partial |
| Admin | ❌ | ❌ | ✅ web | 🟡 |

**نقطة قوة Yossef:** أوسع تغطية features + README + Stripe checkout + reviews/addresses.  
**نقطة ضعف:** **أخطر authorization gaps في Bootcamp** — payments IDOR + order status IDOR + public category CRUD.

---

## 11. ملخص الأولويات (Action Items)

### 🔴 Critical — Security (اعملها **اليوم**)

1. **Fix Payment IDOR** — scope `PaymentController` to `$request->user()` orders only
2. **Fix Order Status IDOR** — admin middleware OR ownership + status whitelist
3. **Protect Category writes** — `auth:sanctum` + `admin` on POST/PUT/DELETE
4. **Replace hardcoded OTP** — `random_int()` + SMS — never in response
5. **Fix password reset chain** — require verified OTP session/token before reset
6. **Fix double password hashing** — remove `Hash::make()` where cast exists
7. **Verify address ownership** on checkout — `Rule::exists` with `user_id`
8. **Add rate limiting** on OTP/auth routes

### 🟡 High — قبل demo

9. Wire Form Requests + API Resources (already scaffolded!)
10. Hide OTP/sensitive fields via `UserResource`
11. Fix `CartController@destroy` empty response
12. Implement Vonage SMS or remove unused dependency
13. Payment status: pending → paid after Stripe success
14. Policies: `OrderPolicy`, `PaymentPolicy`, `AddressPolicy`
15. Remove dead code: `logout` duplicate return, unused `profile` in AuthController

### 🟢 Medium — أثناء التطوير

16. `PlaceOrderAction` — extract Stripe + transaction
17. Feature tests for **all security scenarios**
18. Settings module + Admin module
19. Public meal browse (GET without auth)
20. Seeders for demo data
21. Fix migration filename typo
22. Pagination on lists
23. Unique constraints: cart_items, reviews

---

## 13. تقرير Feature Completeness — النواقص في الـ Application

> **مرجع المتطلبات:** Authentication, Profile, Cart, My Orders, Notifications, Favorites, Meals/Categories, Reset Password, Category Details, Meal Details, Settings, Payments/Checkout.

### 13.1 Feature Matrix

| # | Feature | الحالة | Route / Implementation | النواقص |
|---|---------|--------|------------------------|---------|
| 1 | **Authentication** | 🟡 **70%** | register, login, logout, verify-otp | OTP hardcoded + exposed |
| 2 | **Reset Password** | 🔴 **45%** | forgot, verify-reset, reset | reset without OTP verify |
| 3 | **Profile** | ✅ **88%** | show, update, image upload/delete | no change-password |
| 4 | **Categories** | 🟡 **75%** | apiResource CRUD | **write ops public** |
| 5 | **Category Details** | ✅ **85%** | show with meals | — |
| 6 | **Meals** | ✅ **85%** | CRUD, search, filter | auth required for GET |
| 7 | **Meal Details** | ✅ **88%** | show with images | — |
| 8 | **Favorites** | ✅ **92%** | index, store, destroy | — |
| 9 | **Cart** | ✅ **88%** | index, add, update, destroy | destroy no JSON |
| 10 | **Checkout** | ✅ **82%** | order create + Stripe | fat controller |
| 11 | **Payment** | 🔴 **55%** | Stripe + payments index/show | **IDOR leak** |
| 12 | **My Orders** | 🟡 **78%** | index, show, track | **status update IDOR** |
| 13 | **Order Details** | ✅ **85%** | show scoped | — |
| 14 | **Notifications** | ✅ **90%** | index, mark read | — |
| 15 | **Addresses** | ✅ **88%** | CRUD scoped | update no validation |
| 16 | **Reviews** | ✅ **85%** | CRUD + avg rating | no DB unique |
| 17 | **Settings** | 🔴 **0%** | — | — |
| 18 | **Admin** | 🔴 **0%** | — | — |

**Overall Feature Completeness: ~70%**

### 13.2 Weighted Score (Security-Adjusted)

| Category | Feature Score | Security Multiplier | Weighted |
|----------|:-------------:|:-------------------:|:--------:|
| Auth + Reset | 58% | ×0.55 (OTP/reset broken) | 32% |
| Profile + Catalog | 82% | ×0.70 (public category writes) | 57% |
| Cart + Favorites | 90% | ×0.95 | 86% |
| Checkout + Payment + Orders | 72% | ×0.45 (IDOR critical) | 32% |
| Notifications + Reviews | 88% | ×0.95 | 84% |
| Settings + Admin | 0% | — | 0% |
| **Overall Weighted** | **~70%** | — | **~58%** |

> **Interpretation:** الـ features موجودة، لكن **Checkout/Payment/Orders و Auth** — اللي هي core business — فيها ثغرات تمنع production use.

### 13.3 Route Map

```
POST /api/register, /login, /logout                    ✅
POST /api/resend-otp, /verify-otp                      ✅ (🔴 OTP leak)
POST /api/forgot-password, /verify-reset-otp, /reset   🟡 (🔴 reset broken)

GET/POST/PUT/DELETE /api/categories                    🔴 writes PUBLIC
GET/POST/PUT/DELETE /api/meals                         ✅ (auth required)
/api/cart/*, /api/addresses/*, /api/favorites/*        ✅
/api/orders/* (create, index, show, track, status)    🟡 (🔴 status IDOR)
/api/payments/*                                        🔴 (IDOR)
/api/notifications/*, /api/reviews/*, /api/profile/*   ✅
/api/payment-methods/*                                 ✅ read-only

/api/settings, /admin/*                                ❌
```

### 13.4 Database Tables

| Table | موجود | API Wired | Security |
|-------|-------|-----------|----------|
| `users` | ✅ | Auth + Profile | 🔴 OTP plain text |
| `categories`, `meals`, `meal_images` | ✅ | Catalog | 🔴 public writes |
| `carts`, `cart_items` | ✅ | Cart | ✅ scoped |
| `favorites` | ✅ | Favorites | ✅ |
| `addresses` | ✅ | Addresses | ✅ scoped |
| `orders`, `order_items` | ✅ | Orders | 🔴 status IDOR |
| `payments` | ✅ | Payments | 🔴 IDOR |
| `notifications`, `reviews` | ✅ | ✅ | ✅ |
| `payment_methods` | ✅ | read-only | ✅ |
| `settings` | ❌ | — | — |

### 13.5 Feature Completeness Scorecard

| Category | Feature Score | Weighted Score |
|----------|:-------------:|:--------------:|
| Auth + Reset + Phone Verify | 58% | 32% |
| Profile + Catalog | 82% | 57% |
| Cart + Favorites + Addresses | 90% | 86% |
| Checkout + Stripe + Orders | 72% | 32% |
| Payments | 55% | 25% |
| Notifications + Reviews | 88% | 84% |
| Settings + Admin | 0% | 0% |
| **Overall** | **~70%** | **~58%** |

---

## 14. المراجع

- [Laravel Sanctum](https://laravel.com/docs/sanctum)
- [Laravel Cashier (Stripe)](https://laravel.com/docs/cashier)
- [Laravel Form Requests](https://laravel.com/docs/validation#form-request-validation)
- [Laravel API Resources](https://laravel.com/docs/eloquent-resources)
- [Laravel Policies](https://laravel.com/docs/authorization#creating-policies)
- [OWASP IDOR Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html)
- مراجعة Abdella: [abdella/FoodIfy/CODE_REVIEW.md](../../abdella/FoodIfy/CODE_REVIEW.md)
- مراجعة Mohamed Esmail: [mohamedEsmail/FoodIfy/CODE_REVIEW.md](../../mohamedEsmail/FoodIfy/CODE_REVIEW.md)

---

*هذا الملف مرجع حي — حدّثه مع كل refactor أو security fix.*
