# FoodIfy — Code Review & Architecture Guide

> **الهدف:** Controllers رفيعة تمامًا — بدون validation وبدون business logic.  
> كل المنطق في Actions / Services / Repositories مع تطبيق SOLID في كل الطبقات.

**تاريخ المراجعة:** 4 يوليو 2026   (تحديث — Feature Completeness)
**الطالب:** يوسف (Yossef / Yousef Mehrez)  
**المشروع:** FoodIfy Backend API  
**النطاق:** `app/` — Auth + Catalog + Cart + Orders + Addresses + Payments + Favorites + Reviews + Notifications

---

## 1. ملخص تنفيذي

| المنطقة | الحالة | التقييم |
|---------|--------|---------|
| Auth Module | مُنفَّذ | 🔴 OTP hardcoded + exposed + reset broken |
| Form Requests | 4 files موجودة | 🔴 scaffold فقط — `authorize: false`, rules فارغة |
| API Resources | 4 files موجودة | 🔴 unused — controllers ترجع raw models |
| Actions / Repositories | غير موجود | 🔴 |
| `OtpService` | موجود | 🔴 hardcoded `123456` |
| Catalog (Category/Meal) | مُنفَّذ | 🟡 public category write ops |
| Cart / Orders | مُنفَّذ | 🟡 authorization gaps |
| Addresses / Favorites | مُنفَّذ | ✅ scoped correctly |
| Payments | مُنفَّذ | 🔴 all users' data exposed |
| Reviews / Notifications | مُنفَّذ | ✅ |
| README | project-specific | ✅ (أفضل من معظم المشاريع) |
| Feature Tests | scaffold فقط | 🔴 |
| SOLID Compliance | ضعيف | 🔴 fat controllers |

**الخلاصة:** مشروع **واسع النطاق** — 14 controller، 14 model، domain model متماسك (cart → order → payment → notification). README موجود ويوثّق features. لكن Form Requests و API Resources **اتعملوا scaffold ومش مستخدمين**، والـ authorization فيها **ثغرات حرجة** (payments leak، order status update بدون ownership check، categories CRUD public). OTP service hardcoded + exposed في API response.

---

## 2. المعمارية المستهدفة vs الحالية

```
المستهدف:  FormRequest → Controller → Action → Service → RepositoryInterface → Model
الحالي:    Request ──────→ Controller ──────────→ OtpService ────────────────→ Model
                          (validation + logic)      (hardcoded OTP)
```

**Scaffold موجود لكن غير مفعّل:**

```
app/Http/Requests/     ← 4 files, authorize() = false, rules() = []
app/Http/Resources/    ← 4 files, parent::toArray() only, unused
app/Http/Controllers/API/CheckoutController.php  ← empty, not routed
app/Http/Controllers/API/HomeController.php      ← empty, not routed
```

---

## 3. تحليل SOLID

### S — Single Responsibility Principle

| الملف | المشكلة |
|-------|---------|
| `AuthController` | register + login + logout + OTP + forgot + reset — 228 lines |
| `OrderController` | create + track + status update + list + show + notifications |
| `MealController` | CRUD + search + category filter + file handling |
| `ProfileController` | show + update + image upload/delete |

### O — Open/Closed Principle

- OTP logic hardcoded in `OtpService` — adding Vonage requires editing service
- No payment gateway abstraction — Payment records created as `pending`, never updated

### L — Liskov Substitution Principle

- N/A — no interfaces

### I — Interface Segregation Principle

- N/A — no interfaces

### D — Dependency Inversion Principle

| الوضع الحالي | المطلوب |
|--------------|---------|
| `OtpService` concrete only | `SmsGatewayInterface` |
| Eloquent in controllers | Repository interfaces |
| `AppServiceProvider` فارغ | DI bindings |
| Vonage in composer + config | Unused — misleading |

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

| Method | المشكلة | السطر |
|--------|---------|-------|
| `register` | `Hash::make()` + `'password' => 'hashed'` cast = **double hashing** | 22 |
| `resendOtp` | **`'otp' => $user->otp` في response** — OTP leak | 99–102 |
| `resetPassword` | **لا يتحقق إن OTP verified** — phone + password فقط | 203–227 |
| `verifyResetOtp` | يverify OTP لكن **لا session/token** يربط بـ reset step | 172–201 |
| `logout` | **Dead code** — return مكرر unreachable | 75–80 |
| `profile` | **Unused** — routes تستخدم ProfileController | 105–110 |
| كل methods | Inline `$request->validate()` — Form Requests موجودة لكن unused | — |
| `login` | يرجع full `$user` — ممكن يحتوي OTP fields | 58–62 |

**Double Hashing:**

```php
// User.php
'password' => 'hashed',

// AuthController.php — سطر 22, 219
'password' => Hash::make($request->password),  // ❌
```

**Password Reset Chain Broken:**

```
forgot-password → verify-reset-otp → reset-password
                      ↓                    ↓
                 validates OTP      NO check that OTP was verified
                                    anyone with phone can reset
```

**Fix reset flow:**

```php
// Option A: reset-password requires OTP in same request
// Option B: verify-reset-otp sets short-lived reset token in cache
// Option C: merge verify + reset into single endpoint
```

---

### 4.2 `OtpService` — 🔴 Critical

```php
// OtpService.php — سطر 13
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
// Send via Vonage/SMS — never return in response
```

---

### 4.3 `OrderController` — 🟡 Authorization Gaps

**الإيجابيات:**
- DB transaction ✅
- Cart → order → payment → clear cart → notification ✅
- User-scoped index/show/track ✅
- Price snapshot on order items ✅
- Eager loading ✅

**المشاكل:**

| # | المشكلة | السطر | الخطورة |
|---|---------|-------|---------|
| 1 | `updateStatus` — **no user_id check** — أي user يغيّر status أي order | 132–136 | 🔴 |
| 2 | `address_id` validated `exists` but **not ownership** | 21 | 🔴 |
| 3 | Exception message exposed in 500 response | 99 | 🟡 |
| 4 | Payment always `pending` — never updated to `paid` | 64–70 | 🟡 |
| 5 | Inline validation | 20–23 | 🟡 |

```php
// ❌ updateStatus — any authenticated user can cancel/deliver ANY order
$order = Order::findOrFail($id);
$order->update(['status' => $request->status]);

// ✅ Fix: admin middleware OR ownership check
$order = Order::where('id', $id)->where('user_id', $request->user()->id)->firstOrFail();
// OR: Route::put(...)->middleware('admin')
```

---

### 4.4 `PaymentController` — 🔴 Data Leak

```php
// PaymentController.php — سطر 11–15
Payment::with('order')->get()  // ❌ ALL users' payments

// Fix:
Payment::whereHas('order', fn($q) => $q->where('user_id', auth()->id()))->get()
```

Same issue in `show($id)` — no user scoping.

---

### 4.5 `CategoryController` — 🔴 Public Write Ops

```php
// routes/api.php — سطر 64
Route::apiResource('categories', CategoryController::class);  // NO auth middleware
```

**النتيجة:** أي شخص يقدر POST/PUT/DELETE categories بدون token.

**مشاكل إضافية:**

| # | Issue |
|---|-------|
| Validation typo | `'nullable\|\|image'` — double pipe (L22) |
| Image upload commented out on create | stores raw string |
| `update` uses `$request->all()` — no validation | mass assignment risk |

---

### 4.6 `MealController` — 🟡

**الإيجابيات:**
- Search, category filter, CRUD ✅
- Image handling on update/destroy ✅
- Relations loaded ✅

**المشاكل:**

| # | Issue |
|---|-------|
| All meal routes require auth (including GET index) | catalog not public |
| Image upload commented out on `store` | incomplete create |
| `MealRequest` exists but unused | inline validation |
| `MealResource` exists but unused | raw model returned |

---

### 4.7 `CartController` — 🟡

**الإيجابيات:**
- firstOrCreate cart pattern ✅
- Price snapshot on add ✅
- User-scoped operations ✅

**المشاكل:**

| # | Issue |
|---|-------|
| `destroy` — **no JSON response returned** | client gets empty response |
| No unique `(cart_id, meal_id)` in migration | duplicate rows possible |
| Inline validation | no Form Request |

---

### 4.8 Controllers الأخرى

#### `AddressController` — ✅ (mostly)

- CRUD scoped to user ✅
- **`update` has no validation** — uses `$request->only(...)`

#### `FavoriteController` — ✅

- firstOrCreate on add ✅
- User-scoped ✅

#### `ReviewController` — ✅

- Duplicate review check ✅
- User-scoped update/delete ✅
- Route `{meal}` works with `$mealId` param (implicit binding by ID)

#### `NotificationController` — ✅

- Clean, user-scoped ✅

#### `ProfileController` — 🟡

- Image upload via Storage ✅
- `birth_date`, `address`, `image` not in `$fillable` — works via direct assignment
- Returns raw user — `UserResource` unused

#### `PaymentMethodController` — ✅

- Read-only ✅

#### Empty Stubs

- `CheckoutController` — empty, not routed
- `HomeController` — empty, not routed

---

### 4.9 Form Requests — 🔴 Scaffold Only

| File | authorize() | rules() | Used? |
|------|-------------|---------|-------|
| `RegisterRequest` | `false` | `[]` | ❌ |
| `LoginRequest` | `false` | `[]` | ❌ |
| `MealRequest` | `false` | `[]` | ❌ |
| `CheckoutRequest` | `false` | `[]` | ❌ |

**Action:** Either implement and wire them, or delete to avoid confusion.

---

### 4.10 API Resources — 🔴 Unused

| Resource | Status |
|----------|--------|
| `UserResource` | `parent::toArray()` only — unused |
| `MealResource` | same |
| `CategoryResource` | same |
| `OrderResource` | same |

Controllers return raw Eloquent models everywhere.

---

### 4.11 Models — 🟡

| Model | الحالة |
|-------|--------|
| `User` | 🟡 missing `birth_date, address, image` in fillable; hidden references nonexistent `mobile_verify_code` |
| `Category`, `Meal`, `MealImage` | ✅ |
| `Cart`, `CartItem` | ✅ one cart per user |
| `Favorite` | ✅ unique constraint |
| `Address` | ✅ structured fields |
| `Order`, `OrderItem` | ✅ status enum, price snapshot |
| `Payment`, `PaymentMethod` | ✅ |
| `Notification` | ✅ custom model (not Laravel Notifications) |
| `Review` | 🟡 no unique(user_id, meal_id) in DB — app-level only |

---

### 4.12 `routes/api.php` — 🟡

**الإيجابيات:**
- RESTful structure ✅
- Sanctum on protected routes ✅

**المشاكل:**

| # | Issue |
|---|-------|
| Categories CRUD **public** | security |
| Two separate `auth:sanctum` groups | merge |
| Meals listing requires auth | unusual |
| `resend-otp` outside auth group but public | OK but no rate limit |
| Inconsistent indentation | style |

---

### 4.13 Database / Migrations — 🟡

| Migration | Notes |
|-----------|-------|
| `users` + phone/otp + profile fields | ✅ |
| `categories`, `meals`, `meal_images` | ✅ |
| `carts` (unique user_id) | ✅ |
| `cart_items` | 🟡 no unique(cart_id, meal_id) |
| `favorites` | ✅ unique constraint |
| `addresses`, `orders`, `order_items` | ✅ |
| `payments`, `notifications`, `reviews` | ✅ |
| **`create_meals_table.php.php`** | 🔴 filename typo — double `.php` |

**Schema concerns:**
- OTP plain text in users table
- Payment never transitions pending → paid
- No indexes beyond FK defaults on hot columns

---

### 4.14 Seeders — 🟡

| Seeder | Content |
|--------|---------|
| `PaymentMethodSeeder` | Cash, Visa |
| `DatabaseSeeder` | PaymentMethodSeeder only |

**Missing:** categories, meals, users seeders — demo data sparse vs MohamedAdel.

---

## 5. Security Review

| Issue | Location | Risk | Priority |
|-------|----------|------|----------|
| **Hardcoded OTP `123456`** | OtpService:13 | Trivial account takeover | 🔴 P0 |
| **OTP exposed in API** | AuthController:101 | Leak in response | 🔴 P0 |
| **Password reset without OTP verify** | resetPassword:203–227 | Anyone resets with phone | 🔴 P0 |
| **Double password hashing** | register:22, reset:219 | Login broken | 🔴 P0 |
| **All payments visible** | PaymentController:14 | Data leak | 🔴 P0 |
| **Order status by any user** | OrderController:132 | Privilege escalation | 🔴 P0 |
| **Public category CRUD** | routes/api.php:64 | Unauthorized writes | 🔴 P1 |
| **Address not ownership-checked** | OrderController:21 | Order to others' address | 🔴 P1 |
| **No rate limiting** | auth/OTP routes | Brute force | 🔴 P1 |
| **Exception messages in 500** | OrderController:99 | Info disclosure | 🟡 P2 |
| **Full user in login response** | AuthController:61 | Field leakage | 🟡 P2 |
| **Vonage unused** | composer.json | Misleading readiness | 🟢 P3 |

---

## 6. خطة Refactor — الأولويات

### المرحلة 0 — Critical Fixes (فورًا)

- [ ] Replace hardcoded OTP with `random_int()` — never return in response
- [ ] Fix password reset — require OTP verification token/session before reset
- [ ] Fix double hashing — plain password, let cast handle it
- [ ] Scope `PaymentController` to authenticated user's orders
- [ ] Add ownership/admin check on `OrderController@updateStatus`
- [ ] Move `categories` apiResource inside `auth:sanctum` (or admin middleware)
- [ ] Verify `address_id` belongs to user on order create
- [ ] Remove dead code in `logout` (lines 78–80)

### المرحلة 1 — Wire Existing Scaffold

- [ ] Implement Form Requests (Register, Login, Meal, Checkout/Order)
- [ ] Use API Resources in all controllers
- [ ] Delete or implement `CheckoutController`, `HomeController`
- [ ] Fix `CartController@destroy` — return JSON response
- [ ] Fix Category validation typo `nullable||image`

### المرحلة 2 — Service Layer + Auth

- [ ] `SmsGatewayInterface` + Vonage implementation
- [ ] Rate limiting on OTP/auth routes
- [ ] `UserResource` hides OTP fields
- [ ] Policies: `OrderPolicy`, `PaymentPolicy`, `AddressPolicy`
- [ ] Rename migration file `create_meals_table.php.php`

### المرحلة 3 — Architecture

- [ ] Actions: `PlaceOrderAction`, `RegisterAction`, etc.
- [ ] Repository interfaces
- [ ] `AppServiceProvider` bindings
- [ ] Payment status workflow (pending → paid)
- [ ] Unique constraint on cart_items(cart_id, meal_id)
- [ ] Unique constraint on reviews(user_id, meal_id)

### المرحلة 4 — Testing + Docs

- [ ] Feature tests: auth, order, payment scoping, category auth
- [ ] Fix `UserFactory` for phone-based model
- [ ] Complete README endpoint table
- [ ] Seeders for categories/meals
- [ ] Pagination on list endpoints

---

## 7. Exception-Based Error Handling

```php
// app/Exceptions/Auth/UnverifiedResetException.php
class UnverifiedResetException extends Exception {}

// resetPassword — after fixing flow
if (!$this->resetTokenService->isValid($request->phone, $request->reset_token)) {
    throw new UnverifiedResetException();
}
```

---

## 8. Testing Strategy

| الطبقة | نوع الاختبار | Priority |
|--------|-------------|----------|
| PaymentController | Feature — must NOT see other users' payments | 🔴 |
| OrderController@updateStatus | Feature — must NOT update others' orders | 🔴 |
| Auth reset flow | Feature — must require OTP verification | 🔴 |
| Category CRUD | Feature — must require auth for writes | 🔴 |
| Cart / Order | Feature — happy path | 🟡 |

```php
public function test_user_cannot_see_other_users_payments(): void
{
    $userA = User::factory()->create();
    $userB = User::factory()->create();
    // create payment for userB's order

    $this->actingAs($userA)
        ->getJson('/api/payments')
        ->assertJsonMissing(['order_id' => $orderB->id]);
}
```

---

## 9. Code Style & Conventions

| القاعدة | الوضع الحالي | المطلوب |
|---------|-------------|---------|
| Namespace | `App\Http\Controllers\API` | ✅ PascalCase (consistent) |
| Validation | Inline in controllers | Form Requests (already scaffolded) |
| Response format | Inconsistent `{ message }` vs raw arrays | ApiResponse trait |
| Password hashing | Double hash in register/reset | Cast only |
| OTP | Hardcoded + exposed | random + SMS only |
| Dead code | logout, profile method | Remove |
| File naming | `create_meals_table.php.php` | Fix typo |
| Comments | Arabic mixed | PHPDoc English |

---

## 10. ملخص الأولويات (Action Items)

### 🔴 Critical (اعملها فورًا)

1. Fix **hardcoded OTP** + remove from API response
2. Fix **password reset** — require verified OTP session
3. Fix **double password hashing**
4. Scope **PaymentController** to current user
5. Fix **OrderController@updateStatus** authorization
6. Protect **categories** write routes with auth/admin
7. Verify **address ownership** on checkout

### 🟡 High (قبل demo)

8. Wire Form Requests + API Resources (already exist!)
9. Rate limiting on auth/OTP
10. Fix CartController destroy response
11. Implement Vonage SMS or remove dependency
12. Payment status workflow
13. Policies for Order, Payment, Address

### 🟢 Medium (أثناء التطوير)

14. Actions + Repository layer
15. Delete/implement CheckoutController, HomeController
16. Feature tests for security scenarios
17. Seeders for demo data
18. Pagination
19. Public meal browse routes (optional)
20. Complete README endpoints

---

## 11. ما تم تنفيذه vs الناقص

### ✅ مُنفَّذ (Functional)

- Register → OTP verify → login/logout
- Forgot/reset password (broken security)
- Profile with image upload
- Categories CRUD (unprotected writes)
- Meals CRUD, search, filter by category
- Cart add/update/clear
- Orders create, list, show, track
- Addresses CRUD
- Favorites
- Payment methods (read)
- Payments (unscoped leak)
- Custom notifications
- Reviews with average rating
- Project README

### ❌ Scaffold / Broken / Missing

- Form Requests (4 files — unused)
- API Resources (4 files — unused)
- CheckoutController, HomeController (empty)
- Vonage SMS (config only)
- Actions, Repositories, Policies
- Feature tests
- Payment pending → paid transition
- Rich seeders

---

## 12. مقارنة سريعة مع مشاريع FoodIfy

| Feature | Yossef | MohamedAdel | Mostafa |
|---------|--------|-------------|---------|
| Domain breadth | ✅ widest (14 models) | ✅ | 🟡 |
| README | ✅ project-specific | ❌ default | ❌ default |
| Form Requests | scaffold unused | ✅ Auth | ❌ |
| API Resources | scaffold unused | ❌ | ❌ |
| Actions | ❌ | ✅ | ❌ |
| Payment integration | records only | Paymob Strategy | Stripe inline |
| Authorization | 🔴 critical gaps | 🟡 | 🟡 |
| OTP security | 🔴 worst (hardcoded) | 🟡 | 🔴 exposed |

---

## 13. تقرير Feature Completeness — النواقص في الـ Application

> **مرجع المتطلبات:** Authentication, Profile, Cart, My Orders, Notifications, Favorites, Meals/Categories, Reset Password, Category Details, Meal Details, Settings, Payments/Checkout.

### 13.1 Feature Matrix

| # | Feature | الحالة | Route / Implementation | النواقص |
|---|---------|--------|------------------------|---------|
| 1 | **Authentication** | 🟡 **88%** | register, verify, login, logout, resend-otp | OTP hardcoded |
| 2 | **Reset Password** | 🟡 **65%** | forgot, verify-reset, reset | **reset skips OTP gate** |
| 3 | **Profile** | 🟡 **85%** | GET/PUT /api/profile | fillable gaps |
| 4 | **Categories** | 🟡 **80%** | apiResource (public!) | **anyone can CRUD categories** |
| 5 | **Category Details** | ✅ **90%** | show + `{id}/meals` | — |
| 6 | **Meals** | 🟡 **68%** | index (auth), search (public) | catalog behind auth |
| 7 | **Meal Details** | 🟡 **72%** | show (auth) | — |
| 8 | **Favorites** | ✅ **95%** | full CRUD | — |
| 9 | **Cart** | 🟡 **88%** | full lifecycle | destroy missing return |
| 10 | **Checkout** | 🟡 **55%** | `POST /api/orders` | CheckoutController empty stub |
| 11 | **Payment** | 🟡 **50%** | read payments | **leaks all users' payments**; no gateway |
| 12 | **My Orders** | ✅ **95%** | index + track | — |
| 13 | **Order Details** | ✅ **95%** | show | — |
| 14 | **Notifications** | ✅ **90%** | list, read, delete | — |
| 15 | **Settings** | 🔴 **0%** | — | **غير موجود** |
| 16 | **Admin** | 🔴 **0%** | — | meal/category CRUD unprotected |

**Overall Feature Completeness: ~70%**

### 13.2 Critical Security Issues

| المشكلة | Impact |
|---------|--------|
| Category CRUD public (no auth) | data tampering |
| Any user can update order status | `PUT /orders/{id}/status` |
| PaymentController returns all payments | data leak |
| Reset password without verify-reset-otp | account takeover risk |
| OTP exposed in resend-otp response | security |

### 13.3 Route Map

```
/api/auth/*, /profile, /cart, /favorites, /orders, /notifications   ✅
/api/categories CRUD (public — no admin guard)                      🟡 risky
/api/settings, /admin/*                                             ❌
CheckoutController stub — unused                                    🟡
```

### 13.4 Feature Completeness Scorecard

| Category | Score |
|----------|-------|
| Auth + Orders + Favorites + Notifications | 90% |
| Catalog | 78% |
| Checkout / Payment | 52% |
| Settings + Admin | 0% |
| **Overall** | **~70%** |

---

## 14. المراجع

- [Laravel Form Requests](https://laravel.com/docs/validation#form-request-validation)
- [Laravel API Resources](https://laravel.com/docs/eloquent-resources)
- [Laravel Policies](https://laravel.com/docs/authorization#creating-policies)
- [Vonage SMS Laravel](https://github.com/Vonage/vonage-laravel)
- [Laravel Sanctum](https://laravel.com/docs/sanctum)

---

*هذا الملف مرجع حي — حدّثه مع كل refactor أو module جديد.*
