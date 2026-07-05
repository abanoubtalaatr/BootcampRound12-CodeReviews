# Foodify — Code Review & Architecture Guide

> **الهدف:** Controllers رفيعة تمامًا — بدون validation وبدون business logic.  
> كل المنطق في Actions / Services / Repositories مع تطبيق SOLID في كل الطبقات.

**تاريخ المراجعة:** 5 يوليو 2026  
**الطالب:** محمد عادل (MohamedAdel)  
**المستودع:** [MohamedAdel-Elaraby/Foodify](https://github.com/MohamedAdel-Elaraby/Foodify)  
**النطاق:** `app/` — Auth + Catalog + Cart + Orders + Payment (Cash/Stripe/PayPal) + Favorites + Profile + Notifications + Admin (جزئي)

---

## 1. ملخص تنفيذي

| المنطقة | الحالة | التقييم |
|---------|--------|---------|
| Auth Module | مُنفَّذ | ✅ Actions + Form Requests |
| OTP / Phone Verify | SMS Misr + throttle جزئي | 🔴 `phone_verified_at` لا يُحدَّث |
| Reset Password | OTP flow | 🟡 verify مرتين / consume bug محتمل |
| Form Requests | Auth + Profile | 🟡 Cart/Order/Favorite inline |
| Actions Layer | Auth + Cart + Order + Profile | ✅ |
| Payment Strategy | Cash / Stripe / PayPal | 🟡 enum mismatch + no webhooks |
| Repositories | جزئي في أماكن | 🟡 غير موحّد |
| API Resources | غير موجود | 🟡 raw models |
| `ApiResponseTrait` | موجود | ✅ envelope موحّد |
| Catalog (Meal/Category) | مُنفَّذ | ✅ filter/search + ingredients |
| Cart / Favorites | مُنفَّذ | 🟡 inline validation |
| Orders + Checkout | `PlaceOrderAction` | 🟡 payment_status bug |
| Profile | مُضاف مؤخرًا | ✅ show/update/change-password/orders |
| Notifications | جزئي | 🟡 لا mark-all / لا queue |
| Admin | جزئي | 🟡 CRUD غير مكتمل |
| Settings | غير موجود | 🔴 |
| Feature Tests | scaffold فقط | 🔴 |
| SOLID Compliance | جيد نسبيًا | 🟡 أفضل من معظم Bootcamp |

**الخلاصة:** MohamedAdel بنى **Foodify API من أقوى المشاريع معماريًا في الـ Bootcamp (~68%)** — Actions layer، Form Requests في Auth/Profile، Strategy Pattern للدفع (Cash / Stripe / PayPal)، وRepository pattern في أماكن محددة. Profile module أُضيف مؤخرًا بشكل نظيف. لكن فيه **bugs حرجة** تمنع production: OTP verify لا يضبط `phone_verified_at`، `payment_status` enum غير متطابق (`pending` vs `pending_payment`)، ولا webhooks لـ Stripe/PayPal — فالدفع الإلكتروني يُعلَن ناجحًا قبل التأكيد الفعلي.

**Overall Feature Completeness: ~68%**

---

## 2. المعمارية المستهدفة vs الحالية

```
المستهدف:  FormRequest → Controller → Action → Service → RepositoryInterface → Model

Auth:      FormRequest → Controller → Action → OtpService ──────────────→ Model  (~85%)
Profile:   FormRequest → Controller → Action ───────────────────────────→ Model  (~80%)
Orders:    inline validate → Controller → PlaceOrderAction → PaymentContext → Model  (~70%)
Catalog:   Request     → Controller ─────────────────────────────────────→ Model  (~55%)
Admin:     Request     → Controller ─────────────────────────────────────→ Model  (~40%)
```

### قواعد الـ Controller (إلزامية)

```php
// ✅ AuthController — thin controller (مطبَّق)
public function register(RegisterRequest $request, RegisterAction $action): JsonResponse
{
    $user = $action->execute($request->validated());

    return $this->successResponse(
        ['user' => $user],
        'Account created successfully. OTP sent to your phone.',
        201
    );
}

// ❌ OrderController — validation داخل controller
$request->validate([
    'payment_method' => 'required|in:cash,stripe,paypal',
]);
$result = $action->execute($request->user(), $request->payment_method);
```

### هيكل المجلدات الحالي / المقترح

```
app/
├── Actions/
│   ├── Auth/          ← ✅ Register, Login, VerifyOtp, Forgot, Reset
│   ├── Cart/          ← ✅ CartAction
│   ├── Order/         ← ✅ PlaceOrderAction
│   └── Profile/       ← ✅ UpdateProfile, ChangePassword (مُضاف مؤخرًا)
├── Contracts/
│   └── Repositories/  ← 🟡 جزئي — وسّع لكل domains
├── Http/
│   ├── Controllers/Api/
│   ├── Requests/      ← ✅ Auth + Profile — أضف Cart/Order
│   └── Resources/     ← ❌ مطلوب
├── Repositories/      ← 🟡 موجود في أماكن — وحّد النمط
├── Services/
│   ├── OtpService.php
│   └── Payment/
│       ├── PaymentContext.php
│       ├── Contracts/PaymentStrategy.php
│       └── Strategies/
│           ├── CashPayment.php
│           ├── StripePayment.php
│           └── PayPalPayment.php
└── Traits/
    └── ApiResponseTrait.php  ← ✅
```

---

## 3. تحليل SOLID

### S — Single Responsibility Principle

| الملف | المشكلة |
|-------|---------|
| `PlaceOrderAction` | pricing + order items + payment + cart clear في class واحد |
| `OtpService` | generate + verify + SMS + logging |
| `MealController` | query building + filtering بدون Form Request |
| `FavoriteController` | validation + duplicate check + response formatting |
| `OrderController` | inline validation + exception handling |

### O — Open/Closed Principle

| المشكلة | الحل |
|---------|------|
| `PaymentContext` يستخدم `new StripePayment()` | Strategy registry عبر Container |
| enum `payment_status` في migration ≠ القيم في Action | `PaymentStatus` enum موحّد |
| إضافة gateway جديد = تعديل match + controller validation | `PaymentMethod` enum + mapping layer |

### L — Liskov Substitution Principle

- Strategies تطبّق `PaymentStrategy` — ✅
- لكن instantiation بـ `new` — لا DI حقيقي للـ swap في tests

### I — Interface Segregation Principle

- `PaymentStrategy` — ✅ focused interface
- Repository interfaces موجودة جزئيًا — 🟡 ليست لكل domain

### D — Dependency Inversion Principle

| الوضع الحالي | المطلوب |
|--------------|---------|
| Actions → Eloquent مباشرة في أغلب الأماكن | Repository interfaces في كل domains |
| `PaymentContext` → `new` strategies | Container resolves by payment method |
| `AppServiceProvider` bindings جزئية | bindings كاملة لكل contracts |

---

## 4. مراجعة ملف بملف

### 4.1 Auth Module — ✅ الأقوى في المشروع

#### `AuthController` — ✅ thin

- Form Requests لكل endpoint ✅
- Actions منفصلة ✅
- Register **لا يصدر token** — token بعد verify OTP ✅
- Token rotation في login ✅

#### `VerifyOtpAction` — 🔴 Bug حرج

```php
// المفروض بعد verify ناجح (type=register):
$user->update([
    'is_phone_verified' => true,
    'phone_verified_at' => now(),   // ← غير مضبوط / لا يُحفَظ
]);
```

| # | المشكلة | التفاصيل |
|---|---------|----------|
| 1 | `phone_verified_at` لا يُحدَّث | middleware أو login يعتمد على الحقل — user يبقى "غير موثّق" |
| 2 | `is_phone_verified` قد يُحدَّث بدون timestamp | inconsistency بين boolean و datetime |
| 3 | forgot_password verify يستهلك OTP | reset-password يفشل إذا أُعيد verify |

**Fix:**

```php
$user->forceFill([
    'is_phone_verified' => true,
    'phone_verified_at' => now(),
])->save();
```

#### `ResetPasswordAction` — 🟡

- OTP verify مرة ثانية في reset — conflict مع verify-otp step
- Password cast `'hashed'` — تأكد من عدم `Hash::make()` مزدوج

#### Form Requests (Auth) — ✅

| Request | الحالة |
|---------|--------|
| `RegisterRequest` | ✅ phone unique |
| `LoginRequest` | ✅ |
| `VerifyOtpRequest` | ✅ type: register/forgot_password |
| `ForgotPasswordRequest` | 🟡 user enumeration |
| `ResetPasswordRequest` | ✅ |

---

### 4.2 Payment System — 🟡 Strategy Pattern (جيد لكن incomplete)

**الإيجابيات:**
- Strategy Pattern صح ✅ — `PaymentStrategy` interface
- 3 implementations: `CashPayment`, `StripePayment`, `PayPalPayment` ✅
- `PlaceOrderAction` + DB transaction ✅
- Price snapshot على order items ✅

#### 🔴 Bug — `payment_status` enum mismatch

```php
// Migration — orders table
enum: 'pending_payment', 'paid', 'failed', 'refunded'

// PlaceOrderAction — عند الإنشاء
'payment_status' => 'pending',   // ❌ قيمة غير موجودة في enum

// بعد Stripe/PayPal "success"
$order->update(['payment_status' => 'paid']);  // ✅ لكن مبكرًا بدون webhook
```

**النتيجة:** MySQL/Laravel يرفض insert أو يفشل silently حسب driver — checkout مكسور أو inconsistent.

**Fix:** وحّد everywhere:

```php
enum PaymentStatus: string {
    case PendingPayment = 'pending_payment';
    case Paid = 'paid';
    case Failed = 'failed';
    case Refunded = 'refunded';
}
```

#### 🔴 No webhooks — Stripe / PayPal

| Gateway | الوضع | المشكلة |
|---------|-------|---------|
| Stripe | intention/charge في Strategy | لا `POST /webhooks/stripe` — لا signature verify |
| PayPal | order create | لا IPN/webhook handler |
| Cash | immediate success | ✅ مقبول |

```php
// ❌ الوضع الحالي — paid قبل التأكيد
if (in_array($paymentMethod, ['stripe', 'paypal'])) {
    $order->update(['payment_status' => 'paid']);
}

// ✅ المطلوب
$order->update(['payment_status' => PaymentStatus::PendingPayment]);
// webhook يحدّث لـ paid بعد checkout.session.completed
```

#### 🟡 `OrderController` validation vs strategies

```php
'payment_method' => 'required|in:cash,stripe,paypal',
// تأكد من تطابق مع PaymentContext match — أي قيمة زائدة = runtime exception
```

---

### 4.3 `PlaceOrderAction` — 🟡

**الإيجابيات:**
- Full DB transaction ✅
- Subtotal + delivery fee ✅
- Cart cleared after success ✅

| # | المشكلة |
|---|---------|
| 1 | Delivery fee hardcoded `20.00` — مكرر في CartController |
| 2 | `(new PaymentContext($paymentMethod))` — not DI |
| 3 | `payment_status = 'pending'` ≠ migration enum |
| 4 | Electronic payments marked paid بدون webhook |
| 5 | لا order-placed notification/event |

---

### 4.4 Profile Module — ✅ (مُضاف مؤخرًا)

#### `ProfileController` — ✅ thin

```php
public function update(UpdateProfileRequest $request, UpdateProfileAction $action): JsonResponse
public function changePassword(ChangePasswordRequest $request, ChangePasswordAction $action): JsonResponse
```

| Endpoint | الحالة |
|----------|--------|
| `GET /api/profile` | ✅ |
| `POST /api/profile` | ✅ name + profile_image |
| `POST /api/profile/change-password` | ✅ |
| `GET /api/profile/orders` | ✅ |

**الإيجابيات:**
- Form Requests ✅
- Actions منفصلة ✅
- `profile_image_url` accessor على User ✅
- حذف الصورة القديمة عند upload جديد ✅

**النواقص:**
- لا API Resource — raw user في response
- `profile/orders` يكرر query من `OrderController` — يحتاج `OrderRepository`

---

### 4.5 `CartController` + `CartAction` — 🟡

**الإيجابيات:**
- `CartAction` handles add/update/remove/clear ✅
- Unique constraint `(user_id, meal_id)` ✅
- PATCH quantity endpoint ✅

**المشاكل:**
- Inline validation في controller
- Delivery fee مكرر
- لا Form Requests

---

### 4.6 `FavoriteController` — 🟡

- CRUD كامل ✅
- 🟡 named parameters bug محتمل في `successResponse(message: '...')` — message يذهب لـ `$data`
- inline validation

---

### 4.7 `MealController` — 🟡

**الإيجابيات:**
- categories, index (filter/search), show with ingredients ✅
- Route model binding ✅

**المشاكل:**
- كل routes behind `auth:sanctum` — catalog ليس public
- لا pagination
- query params غير validated

---

### 4.8 Repositories — 🟡 جزئي (نقطة قوة نسبية)

| الموقع | الحالة |
|--------|--------|
| Order queries | 🟡 repository في مكان — Eloquent مباشر في controllers أخرى |
| User/Cart | ❌ Eloquent في Actions |
| Pattern موجود | ✅ interface + implementation في أماكن — **وسّعه** |

**مقارنة:** أفضل من Abdella/Mostafa (صفر repositories) — لكن غير متسق عبر المشروع.

---

### 4.9 Notifications — 🟡 جزئي

| Feature | الحالة |
|---------|--------|
| Order status notification | 🟡 قد يكون موجود جزئيًا |
| `GET /notifications` | 🟡 |
| mark read / unread count | 🟡 غير مكتمل |
| mark-all-read | ❌ |
| Queued notifications | ❌ |

---

### 4.10 Admin — 🟡 جزئي

| Feature | الحالة |
|---------|--------|
| Categories CRUD | 🟡 جزئي |
| Meals CRUD | 🟡 جزئي |
| Orders management | 🟡 عرض فقط أو update محدود |
| Middleware `admin` | 🟡 يحتاج مراجعة |
| Policies | ❌ |

---

### 4.11 Models — ✅

| Model | الحالة |
|-------|--------|
| `User` | ✅ phone-based, hashed cast, profile_image |
| `OtpCode` | ✅ typed with expiry |
| `Order`, `OrderItem` | 🟡 payment_status enum mismatch |
| `CartItem`, `Favorite` | ✅ unique constraints |
| `Category`, `Meal`, `Ingredient` | ✅ pivot |

---

### 4.12 `routes/api.php` — 🟡

```
POST /api/auth/register, verify-otp, login, forgot, reset     ✅
GET  /api/auth/me, POST logout (sanctum)                      ✅
GET/POST /api/profile/*                                       ✅
GET  /api/meals/*, /api/cart/*, /api/favorites/*              ✅
POST/GET /api/orders/*                                        ✅
/api/notifications/*                                          🟡 جزئي
/api/admin/*                                                  🟡 جزئي
POST /webhooks/stripe, /webhooks/paypal                       ❌
/api/settings                                                 ❌
```

---

## 5. مشاكل أمنية

| # | المشكلة | الملف | Severity |
|---|---------|-------|----------|
| 1 | `phone_verified_at` لا يُحدَّث — bypass phone gate | `VerifyOtpAction` | 🔴 |
| 2 | `payment_status` enum invalid value | `PlaceOrderAction` + migration | 🔴 |
| 3 | Stripe/PayPal marked paid بدون webhook verify | `PlaceOrderAction` | 🔴 |
| 4 | لا webhook signature verification | routes | 🔴 |
| 5 | Exception message للـ client | `OrderController` | 🟡 |
| 6 | OTP logged plaintext في dev | `OtpService` | 🟡 |
| 7 | لا rate limiting على OTP routes | `routes/api.php` | 🟡 |
| 8 | User enumeration | `ForgotPasswordRequest` | 🟡 |
| 9 | Raw models — field leakage | all controllers | 🟡 |
| 10 | Admin routes — تحقق من authorization | Admin controllers | 🟡 |

---

## 6. ما هو جيد ✅

1. **Actions layer** — Auth, Cart, Order, Profile منفصلة عن controllers.
2. **Payment Strategy Pattern** — Cash / Stripe / PayPal قابلة للتوسيع.
3. **Register flow صحيح** — لا token قبل phone verify (نظريًا).
4. **Form Requests** — Auth + Profile مع `failedValidation` موحّد.
5. **`ApiResponseTrait`** — JSON envelope consistent.
6. **Profile module حديث** — update + change-password + orders list.
7. **Repository pattern جزئي** — أفضل من أغلب مشاريع Bootcamp.
8. **Rich seeders** — 18 meals, 28 ingredients, 6 categories.
9. **Cart lifecycle كامل** — add/update/remove/clear + delivery fee في totals.
10. **Order ownership check** — show يتحقق من `user_id`.
11. **SMS Misr integration** — OTP delivery حقيقي.
12. **DB transactions** في checkout — rollback عند فشل الدفع.

---

## 7. خطة Refactor

### Sprint 0 — Critical Fixes (P0)

1. Fix `VerifyOtpAction` — ضبط `phone_verified_at` + `is_phone_verified` معًا
2. Align `payment_status`: `pending_payment` everywhere (migration = Action = Strategy)
3. Stripe/PayPal: `pending_payment` حتى webhook — لا `paid` مبكر
4. Add webhook routes + signature verification
5. Fix forgot-password OTP double-consume flow

### Sprint 1 — Architecture (P1)

6. Form Requests: Cart, Order, Favorite, Meal filters
7. API Resources: `UserResource`, `MealResource`, `OrderResource`
8. `config/foodify.php` — delivery fee مركزي
9. DI لـ payment strategies (not `new` in PaymentContext)
10. وسّع Repository interfaces لكل domains

### Sprint 2 — Features (P2)

11. Notifications كاملة — mark-all-read, queue
12. Admin CRUD مكتمل + policies
13. Settings module
14. Rate limiting على auth/OTP
15. Feature tests: register→verify→login, checkout mock Stripe

---

## 8. Controller المستهدف (مثال: Place Order)

```php
public function store(PlaceOrderRequest $request, PlaceOrderAction $action): JsonResponse
{
    $result = $action->execute(
        user: $request->user(),
        paymentMethod: $request->enum('payment_method', PaymentMethod::class)
    );

    return $this->successResponse(
        [
            'order'   => new OrderResource($result['order']),
            'payment' => $result['payment'],
        ],
        'Order placed successfully.',
        201
    );
}
```

---

## 9. File-by-File Scorecard

| File | Thin | Form Request | Service/Action | Repository | ApiResponse | Verdict |
|------|:----:|:------------:|:--------------:|:----------:|:-----------:|---------|
| `AuthController` | ✅ | ✅ | ✅ Actions | ❌ | ✅ | ✅ |
| `VerifyOtpAction` | — | — | ✅ | ❌ | — | 🔴 bug |
| `ResetPasswordAction` | — | — | ✅ | ❌ | — | 🟡 |
| `ProfileController` | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ |
| `MealController` | 🟡 | ❌ | ❌ | ❌ | ✅ | 🟡 |
| `CartController` | 🟡 | ❌ | ✅ CartAction | ❌ | ✅ | 🟡 |
| `FavoriteController` | ❌ | ❌ | ❌ | ❌ | ✅ | 🟡 |
| `OrderController` | 🟡 | ❌ | ✅ PlaceOrder | 🟡 partial | ✅ | 🟡 |
| `PlaceOrderAction` | — | — | ✅ Payment | 🟡 | — | 🟡 |
| `PaymentContext` | — | — | ✅ Strategy | — | — | 🟡 |
| `StripePayment` | — | — | ✅ | — | — | 🟡 no webhook |
| `PayPalPayment` | — | — | ✅ | — | — | 🟡 no webhook |
| `CashPayment` | — | — | ✅ | — | — | ✅ |
| Admin controllers | 🟡 | ❌ | ❌ | ❌ | ✅ | 🟡 partial |
| Notifications | 🟡 | — | 🟡 partial | ❌ | ✅ | 🟡 partial |
| Tests | — | — | — | — | — | 🔴 |

---

## 10. مقارنة مع مشاريع Bootcamp

| المعيار | MohamedAdel | Abdella | Ali | Elham |
|---------|:-----------:|:-------:|:---:|:-----:|
| Feature completeness | ~68% | ~70% | ~76% | ~68% |
| Actions layer | ✅ | ❌ | 🟡 partial | ✅ Auth |
| Payment Strategy | ✅ Cash/Stripe/PayPal | ✅ Cashier | ✅ | ✅ Paymob |
| Repositories | 🟡 partial | ❌ | 🟡 | ✅ |
| Form Requests | 🟡 Auth+Profile | 🟡 3 files | ✅ many | ✅ many |
| Phone verify | 🔴 bug | ✅ middleware | 🟡 | ✅ |
| Webhooks | ❌ | N/A Cashier | 🟡 | 🟡 |
| Profile | ✅ recent | 🟡 | ✅ | ✅ |
| Admin | 🟡 partial | ❌ | 🟡 | ✅ API |
| Feature tests | ❌ | ❌ | ✅ | 🟡 |

**نقطة قوة MohamedAdel:** Actions + Strategy Pattern + Repository جزئي — أقرب للمعمارية المستهدفة.  
**نقطة ضعف:** bugs حرجة في OTP verify و payment_status + غياب webhooks.

---

## 11. ملخص الأولويات (Action Items)

### 🔴 Critical

1. Fix `VerifyOtpAction` — تحديث `phone_verified_at` عند verify ناجح
2. Align `payment_status`: استخدم `pending_payment` لا `pending`
3. Stripe/PayPal webhooks + signature verification
4. لا تضبط `paid` قبل webhook confirmation
5. Fix forgot-password OTP flow — verify مرة واحدة فقط

### 🟡 High

6. Form Requests لـ Cart, Order, Favorite
7. API Resources لكل response
8. Rate limiting على auth/OTP routes
9. `config/foodify.php` لـ delivery fee
10. إكمال Notifications (mark-all-read)
11. إكمال Admin CRUD + policies
12. DI لـ payment strategies

### 🟢 Medium

13. Repository interfaces في كل domains
14. Feature tests (auth + checkout mock)
15. Settings module
16. Public meal routes (optional)
17. Queued SMS/notifications
18. Pagination على list endpoints
19. Foodify README (setup, env vars, endpoints)

---

## 13. تقرير Feature Completeness — النواقص في الـ Application

> **مرجع المتطلبات:** Authentication, Profile, Cart, My Orders, Notifications, Favorites, Meals/Categories, Reset Password, Category Details, Meal Details, Settings, Payments/Checkout, Admin.

### 13.1 Feature Matrix

| # | Feature | الحالة | Route / Implementation | النواقص |
|---|---------|--------|------------------------|---------|
| 1 | **Authentication** | 🟡 **80%** | register, verify-otp, login, logout | `phone_verified_at` bug |
| 2 | **Reset Password** | 🟡 **72%** | forgot + verify + reset | OTP double consume |
| 3 | **Profile** | ✅ **90%** | show, update, change-password, orders | no Resource |
| 4 | **Categories** | ✅ **85%** | `GET /api/meals/categories` | — |
| 5 | **Category Details** | 🟡 **55%** | filter via `category_id` | no dedicated show |
| 6 | **Meals** | ✅ **88%** | index + show | auth required |
| 7 | **Meal Details** | ✅ **92%** | show + ingredients | — |
| 8 | **Favorites** | ✅ **92%** | index, add, remove | inline validation |
| 9 | **Cart** | ✅ **93%** | full lifecycle + PATCH qty | — |
| 10 | **Checkout** | 🟡 **60%** | `POST /api/orders` | payment_status bug |
| 11 | **Payment** | 🟡 **50%** | Cash/Stripe/PayPal strategies | no webhooks, enum mismatch |
| 12 | **My Orders** | ✅ **88%** | index + show | — |
| 13 | **Order Details** | ✅ **90%** | show + ownership | — |
| 14 | **Notifications** | 🟡 **35%** | جزئي | mark-all, queue |
| 15 | **Settings** | 🔴 **0%** | — | — |
| 16 | **Admin** | 🟡 **40%** | جزئي | CRUD غير مكتمل |

**Overall Feature Completeness: ~68%**

### 13.2 Route Map

```
POST /api/auth/register, verify-otp, login, forgot, reset        ✅
GET  /api/auth/me, POST logout                                   ✅

GET/POST /api/profile/*, change-password, orders               ✅
GET  /api/meals/categories, /meals, /meals/{id}                ✅
/api/cart/*, /api/favorites/*                                    ✅
POST/GET /api/orders/*                                           🟡 payment bugs

/api/notifications/*                                             🟡 جزئي
/api/admin/*                                                     🟡 جزئي
POST /webhooks/stripe, /webhooks/paypal                          ❌
/api/settings                                                    ❌
```

### 13.3 Database Tables

| Table | موجود | API Wired |
|-------|-------|-----------|
| `users` | ✅ | Auth + Profile |
| `otp_codes` | ✅ | OTP flows |
| `categories`, `meals`, `ingredients` | ✅ | Catalog |
| `cart_items`, `favorites` | ✅ | Cart + Favorites |
| `orders`, `order_items` | ✅ | Orders (enum bug) |
| `notifications` | 🟡 | جزئي |
| `payment_transactions` | ❌ | — |
| `settings` | ❌ | — |

### 13.4 Feature Completeness Scorecard

| Category | Score |
|----------|-------|
| Auth + Reset + Phone Verify | 76% |
| Profile + Catalog | 88% |
| Cart + Favorites | 92% |
| Checkout + Payment + Orders | 58% |
| Notifications | 35% |
| Admin + Settings | 20% |
| **Overall** | **~68%** |

---

## 14. المراجع

- [Laravel Sanctum](https://laravel.com/docs/sanctum)
- [Laravel Form Requests](https://laravel.com/docs/validation#form-request-validation)
- [Strategy Pattern](https://refactoring.guru/design-patterns/strategy)
- [Stripe Webhooks](https://stripe.com/docs/webhooks)
- [PayPal Webhooks](https://developer.paypal.com/api/rest/webhooks/)
- مراجعة Abdella: [abdella/FoodIfy/CODE_REVIEW.md](../../abdella/FoodIfy/CODE_REVIEW.md)
- مراجعة Ali: [ali/FoodIfy/CODE_REVIEW.md](../../ali/FoodIfy/CODE_REVIEW.md)

---

*هذا الملف مرجع حي — حدّثه مع كل refactor جديد.*
