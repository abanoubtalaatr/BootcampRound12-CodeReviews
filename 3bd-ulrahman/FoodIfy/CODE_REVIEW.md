# FoodIfy — Code Review & Architecture Guide

> **الهدف:** Controllers رفيعة تمامًا — بدون validation وبدون business logic.  
> كل المنطق في Actions / Services / Repositories مع تطبيق SOLID في كل الطبقات.

**تاريخ المراجعة:** 5 يوليو 2026  
**الطالب:** 3bd-ulrahman (Abdul Rahman)  
**المستودع:** [3bd-ulrahman/foodify](https://github.com/3bd-ulrahman/foodify)  
**النطاق:** `app/` — Auth + Categories + Meals + Cart + Orders (جزئي) + schema-only domain (Favorites, Reviews, Ingredients)

---

## 1. ملخص تنفيذي

| المنطقة | الحالة | التقييم |
|---------|--------|---------|
| Auth Module | مُنفَّذ | 🟡 78% — Actions جيدة + bugs حرجة |
| Reset Password | مُنفَّذ | 🔴 double-hash bug |
| Categories CRUD | مُنفَّذ | 🟡 JSON:API — **public writes** |
| Meals CRUD | مُنفَّذ | 🟡 JSON:API — **public writes** |
| Cart / Cart Items | مُنفَّذ | 🔴 **IDOR critical** |
| Orders | جزئي | 🟡 index/create/update — no show |
| Favorites / Reviews | schema فقط | 🔴 no routes |
| Payments / Notifications / Settings / Admin | غير موجود | 🔴 |
| Form Requests | 14 requests | ✅ |
| Actions | 19 actions | ✅ أقرب للمعمارية المستهدفة |
| Repositories | غير موجود | 🔴 |
| Services | غير موجود | 🔴 |
| DTOs | غير موجود | 🔴 |
| JSON:API Resources | 10 resources | ✅ ممتاز — غير مستخدمة في Auth |
| Response Layer | `ResponseServiceProvider` macros | 🟡 mixed مع JSON:API |
| Models + Factories + Seeders | 10 models | ✅ infrastructure قوية |
| Feature Tests | scaffold فقط | 🔴 |
| SOLID Compliance | جزئي | 🟡 |

**الخلاصة:** المشروع يطبّق **Action pattern** بشكل أفضل من معظم مشاريع الـ bootcamp — Form Request → Controller → Action على Auth و Catalog و Cart و Orders. لكن **Feature Completeness منخفض (~42%)** بسبب modules ناقصة (Favorites, Reviews, Payments, Notifications, Settings, Admin) و **ثغرات أمنية حرجة** في Cart (IDOR) و Catalog (public CRUD) و Auth (OTP hardcoded, password double-hash, phone E164 mismatch). Response format **غير موحّد** بين Auth (macros envelope) و business endpoints (JSON:API).

**Overall Feature Completeness: ~42%**

---

## 2. المعمارية المستهدفة vs الحالية

```
المستهدف:  FormRequest → Controller → Action → Service → RepositoryInterface → Model

Auth:       FormRequest → Controller → Action ──────────────────────────→ Model  (~78%)
Catalog:    FormRequest → Controller → Action ──────────────────────────→ Model  (~85%)
Cart:       FormRequest → Controller → Action ──────────────────────────→ Model  (~40% — IDOR)
Orders:     FormRequest → Controller → Action ──────────────────────────→ Model  (~60%)
Favorites:  —            —            —         (models/seeders only)     🔴 0%
```

### هيكل المجلدات الحالي

```
app/
├── Actions/          ← 19 actions ✅ (Auth 7, Category 3, Meal 3, Cart 4, Order 2)
├── Http/
│   ├── Controllers/  ← 7 controllers
│   ├── Requests/     ← 14 form requests ✅
│   └── Resources/    ← 10 JSON:API resources ✅
├── Models/           ← 10 models + factories + seeders ✅
├── Providers/
│   ├── ResponseServiceProvider.php   ← macros envelope ✅
│   └── ConfigurationServiceProvider  ← ❌ غير مسجّل
├── Support/          ← Pagination, TokenAbility, MediaUploader (unused)
└── (no Repositories, Services, DTOs, Contracts)
```

**نقطة قوة:** Action pattern مُطبَّق فعليًا — ليس مجرد scaffold.  
**نقطة ضعف:** لا abstraction layer، و Cart/Orders بدون authorization checks على resource ownership.

---

## 3. تحليل SOLID

### S — Single Responsibility Principle

| الملف | التقييم |
|-------|---------|
| Auth Actions | ✅ مسؤولية واحدة لكل action |
| Category / Meal Actions | 🟡 pass-through لـ Eloquent — thin جدًا |
| Cart Actions | 🟡 CRUD بدون ownership validation |
| `AuthController::refresh` | 🟡 token-revocation logic في controller |
| `AuthController::logout` | 🟡 token deletion في controller |
| `CartItemController::index` | ❌ query + validation في controller + **no user scope** |

### O — Open/Closed Principle

- Actions تعتمد على Eloquent مباشرة — لا extension points ❌
- `ResponseServiceProvider` macros — قابل للتوسيع ✅
- `SendOtp` hardcoded — أي تغيير gateway يتطلب تعديل action مباشرة ❌

### L — Liskov Substitution Principle

- N/A — لا inheritance hierarchies مهمة

### I — Interface Segregation Principle

- لا interfaces على الإطلاق ❌

### D — Dependency Inversion Principle

| الوضع الحالي | المطلوب |
|--------------|---------|
| Actions → `Model::query()` مباشرة | `RepositoryInterface` per aggregate |
| `RegisterUser` → `SendOtp` (concrete) | مقبول داخل نفس layer |
| `AppServiceProvider` فارغ | DI bindings |
| `ConfigurationServiceProvider` موجود لكن dead code | register in `bootstrap/providers.php` |

---

## 4. مراجعة ملف بملف

### 4.1 `AuthController` — 🟡

```33:38:app/Http/Controllers/AuthController.php
    public function login(LoginRequest $request, LoginUser $action): JsonResponse
    {
        $result = $action->handle($request->validated());

        return response()->success($result, 'Logged in successfully');
    }
```

**إيجابيات:**
- `register`, `login`, `resetPassword` تستخدم Actions ✅
- `refresh` مع token ability middleware ✅
- Sanctum token abilities (`TokenAbility` enum) ✅

**مشاكل:**

| Method | المشكلة |
|--------|---------|
| `register` | يرجع raw `User` model — مش `UserResource` |
| `logout` | token deletion في controller — ينقل لـ `LogoutUser` action |
| `forgotPassword`, `resendOtp` | `response()->json()` يدوي — مش macro موحّد |
| `verifyOtp` | يمرر `$request->phone` مباشرة — مش `$request->validated()` |
| `resetPassword` | **🐛 swapped macro arguments** (سطر 94) |

**🐛 Bug — `resetPassword` (سطر 90–94):**

```90:95:app/Http/Controllers/AuthController.php
    public function resetPassword(ResetPasswordRequest $request, ResetPassword $action): JsonResponse
    {
        $result = $action->handle($request->validated());

        return response()->success('Password reset successful', $result);
    }
```

الـ macro signature: `success($data, $message)` — **الـ arguments مقلوبة**. الـ client يستقبل message في `data` والعكس.

**الإصلاح:** `return response()->success($result, 'Password reset successful');`

---

### 4.2 `CategoryController` / `MealController` — 🟡

**إيجابيات:**
- `store`, `update`, `destroy` تستخدم Actions ✅
- JSON:API Resources (`CategoryResource`, `MealResource`) ✅
- Form Requests للـ mutations ✅

**مشاكل:**

```23:31:app/Http/Controllers/CategoryController.php
    public function index(Request $request): AnonymousResourceCollection
    {
        $request->validate([
            'per_page' => [Pagination::PER_PAGE_RULES],
        ]);

        $categories = Category::query()->paginate($request->integer('per_page', Pagination::DEFAULT_PER_PAGE));

        return CategoryResource::collection($categories);
    }
```

- `index` و `show` بدون Actions — inline validation + direct Eloquent ❌
- لا `IndexCategoryRequest` / `ListCategories` action
- **Categories و Meals public CRUD بالكامل** — أي anonymous user يقدر create/update/delete

```32:36:routes/api.php
// categories
Route::apiResource('categories', CategoryController::class);

// products
Route::apiResource('meals', MealController::class);
```

---

### 4.3 `CartItemController` — 🔴 IDOR Critical

```22:31:app/Http/Controllers/CartItemController.php
    public function index(Request $request): AnonymousResourceCollection
    {
        $request->validate([
            'per_page' => [Pagination::PER_PAGE_RULES],
        ]);

        $cartItems = CartItem::query()->paginate($request->integer('per_page', Pagination::DEFAULT_PER_PAGE));

        return CartItemResource::collection($cartItems);
    }
```

**🐛 Bug حرج #1 — `index` lists ALL users' cart items:**

`CartItem::query()->paginate(...)` بدون `whereHas('cart', fn ($q) => $q->where('user_id', auth()->id()))`. أي authenticated user يرى cart items لكل المستخدمين.

**🐛 Bug حرج #2 — `update` / `destroy` بدون ownership:**

Route model binding على `CartItem $cartItem` بدون policy أو `authorize()` — user A يقدر يعدّل/يمسح cart item لـ user B لو عرف الـ ID.

`UpdateCartItemRequest` — rules فقط، **لا `authorize()` method**.

---

### 4.4 `CartController` — 🔴 IDOR Critical

```15:27:app/Http/Controllers/CartController.php
    public function clear(Cart $cart, ClearCart $action): JsonResponse
    {
        $action->handle($cart);

        return CartResource::make($cart)
            ->additional([
                'meta' => [
                    'message' => 'Cart cleared successfully.',
                ],
            ])
            ->response()
            ->setStatusCode(Response::HTTP_OK);
    }
```

**🐛 Bug حرج #3 — clear any user's cart:**

`DELETE /api/carts/{cart}/clear` — route model binding على `Cart $cart` بدون التحقق من `$cart->user_id === auth()->id()`. أي authenticated user يقدر يفرّغ cart أي user آخر.

---

### 4.5 Auth Actions — 🟡 + 🔴

| Action | التقييم | ملاحظات |
|--------|---------|---------|
| `RegisterUser` | ✅ | E164 phone formatting, DI على SendOtp, auto-create cart |
| `LoginUser` | 🟡 | verified check ✅ — لكن raw phone lookup ❌ |
| `SendOtp` | 🔴 | **hardcoded OTP `'123456'`** |
| `VerifyOtp` | ✅ | cache-based, hashed OTP, attempts tracking |
| `ResetPassword` | 🔴 | **double-hash password bug** |
| `ResendOtp` | ✅ | E164 formatting |
| `IssueTokens` | ✅ | access + refresh token pair |

**🐛 Bug حرج #4 — `SendOtp` (سطر 15–17):**

```15:17:app/Actions/Auth/SendOtp.php
        // FIXME: fix in production
        // $otp = (string) random_int(100000, 999999);
        $otp = '123456';
```

OTP ثابت — **ثغرة أمنية حرجة** في أي بيئة غير local. أي attacker يعرف OTP بدون SMS.

**🐛 Bug حرج #5 — `ResetPassword` double-hash (سطر 32):**

```31:32:app/Actions/Auth/ResetPassword.php
        return DB::transaction(function () use ($cacheKey, $data, $user): array {
            $user->update(['password' => Hash::make($data['password'])]);
```

`User` model عنده `'password' => 'hashed'` cast:

```34:40:app/Models/User.php
    protected function casts(): array
    {
        return [
            'phone_verified_at' => 'datetime',
            'password' => 'hashed',
        ];
    }
```

`Hash::make()` + `hashed` cast = **double hashing**. المستخدم لن يقدر يسجّل دخول بعد reset.

**الإصلاح:** `$user->update(['password' => $data['password']]);` — الـ cast يعمل hash تلقائيًا.

**🐛 Bug #6 — Phone E164 mismatch:**

| Action / Request | Phone format |
|------------------|-------------|
| `RegisterUser` | E164 ✅ (`PhoneNumber::formatE164()`) |
| `ResendOtp` | E164 ✅ |
| `LoginUser` | raw `$data['phone']` ❌ |
| `SendOtp` (forgot) | raw من `$request->phone` ❌ |
| `VerifyOtp` | raw ❌ |
| `LoginRequest` | no E164 validation ❌ |

User يسجّل بـ `+201234567890` لكن login بـ `01234567890` → **login fails** رغم نفس الرقم. Cache keys `"{$purpose}_otp_{$phone}"` تتكسر لو نفس الرقم بصيغ مختلفة.

---

### 4.6 Order Actions — 🟡

**`CreateOrder` — empty cart checkout:**

```16:35:app/Actions/Order/CreateOrder.php
        return DB::transaction(function () use ($data, $user) {
            $cart = $user->cart()->firstOrFail();

            $cartItems = CartItem::query()
                ->where('cart_id', $cart->id)
                ->with('meal')
                ->get();

            $subtotal = $cartItems->sum(fn (CartItem $item) => $item->meal->price * $item->quantity);
            // ...
            $order = Order::query()->create([
                'user_id' => $user->id,
                'subtotal' => $subtotal,
                // ...
            ]);
```

- لا validation على `$cartItems->isNotEmpty()` — **empty cart → order بـ subtotal = 0**
- `$deliveryFee = 5.00` hardcoded — not configurable
- لا payment step — order ينشأ `pending` مباشرة

**`UpdateOrder` — no ownership in action:**

Authorization موجود في `UpdateOrderRequest::authorize()` ✅ — لكن action نفسه pass-through بدون business rules إضافية.

**`OrderController` — no `show`:**

```46:48:routes/api.php
    Route::get('orders', [OrderController::class, 'index']);
    Route::post('orders', [OrderController::class, 'store']);
    Route::put('orders/{order}', [OrderController::class, 'update']);
```

لا `GET /api/orders/{order}` — user لا يقدر يشوف order details.

---

### 4.7 Form Requests — ✅ جيد (مع فجوات)

| Request | الحالة |
|---------|--------|
| `RegisterRequest` | ✅ phone formatting, uniqueness, Password::defaults() |
| `LoginRequest` | 🟡 phone بدون E164 validation |
| `ForgotPasswordRequest` | 🟡 `exists:users,phone` على raw input |
| `VerifyOtpRequest` | 🟡 phone بدون format validation |
| `ResetPasswordRequest` | 🟡 نفس مشكلة phone |
| `StoreCategoryRequest` / `UpdateCategoryRequest` | ✅ |
| `StoreMealRequest` / `UpdateMealRequest` | ✅ |
| `StoreCartItemRequest` | ✅ |
| `UpdateCartItemRequest` | 🔴 no `authorize()` — IDOR |
| `StoreOrderRequest` | 🟡 delivery_address فقط — no empty-cart rule |
| `UpdateOrderRequest` | ✅ authorize + cancel rules |

**ناقص:** `IndexCategoryRequest`, `IndexMealRequest`, `IndexCartItemRequest`, `IndexOrderRequest` للـ pagination validation.

**مشكلة:** translation keys مثل `auth.unverified`, `auth.otp_resent`, `lang.unauthorized` — **لا `lang/` files موجودة**.

---

### 4.8 Repositories — 🔴 غير موجود

لا `app/Repositories/` ولا interfaces. كل Actions تستخدم `Model::query()` مباشرة.

---

### 4.9 Services — 🔴 غير موجود

OTP logic في `SendOtp`/`VerifyOtp` — يمكن استخراجها لـ `OtpService`.  
`MediaUploader` في `app/Support/` — **غير مستخدم** رغم Spatie Media Library migration.

---

### 4.10 Response Layer — 🟡 `ResponseServiceProvider`

```45:51:app/Providers/ResponseServiceProvider.php
        Response::macro('success', function ($data = null, $message = null) use ($buildResponse): JsonResponse {
            return $buildResponse(true, Status::HTTP_OK, $message, $data);
        });

        Response::macro('created', function ($message = null, $data = null) use ($buildResponse): JsonResponse {
            return $buildResponse(true, Status::HTTP_CREATED, $message, $data);
        });
```

**إيجابيات:** envelope موحّد `{ success, message, data, paginate }` ✅  
**مشاكل:**
- Auth mix بين macros و `response()->json()` ❌
- Catalog/Cart/Orders تستخدم JSON:API format — **شكل مختلف تمامًا**
- Error macros تستخدم `__('lang.unauthorized')` — **لا lang files**

---

### 4.11 Models — ✅ ممتاز

10 models مع relationships, factories, seeders:

| Model | Resource | Factory | Seeder | API Routes |
|-------|:--------:|:-------:|:------:|:----------:|
| User | ✅ | ✅ | — | ✅ auth |
| Category | ✅ | ✅ | ✅ | ✅ CRUD (public) |
| Meal | ✅ | ✅ | ✅ | ✅ CRUD (public) |
| Cart / CartItem | ✅ | ✅ | ✅ | 🟡 IDOR bugs |
| Order / OrderItem | ✅ | ✅ | ✅ | 🟡 partial |
| Favorite | ✅ | ✅ | ✅ | ❌ |
| Review | ✅ | ✅ | ✅ | ❌ |
| Ingredient | ✅ | ✅ | ✅ | ❌ |

**إيجابيات:** `declare(strict_types=1)` ✅, `#[UseFactory]` ✅, `password` hashed cast ✅, relationships كاملة ✅

---

### 4.12 `AppServiceProvider` / `ConfigurationServiceProvider` — 🔴

```7:11:bootstrap/providers.php
return [
    AppServiceProvider::class,
    ResponseServiceProvider::class,
    TelescopeServiceProvider::class,
];
```

`ConfigurationServiceProvider` **غير مسجّل** — `Model::shouldBeStrict()`, `FormRequest::failOnUnknownFields()`, `URL::forceHttps()` كلها dead code.

---

### 4.13 Routes — 🟡

**إيجابيات:**
- Auth routes مجمّعة ✅
- `throttle:3,1` على forgot/resend ✅
- `guest` middleware على OTP routes ✅
- Sanctum + `TokenAbility` على cart/orders ✅

**مشاكل:**
- Categories + Meals **بدون auth** — public CRUD 🔴
- Cart IDOR — no ownership policies 🔴
- لا rate limiting على register/login/verify-otp
- لا API versioning
- Favorites, Reviews, Ingredients — **no routes**

---

### 4.14 Tests — 🔴

فقط Pest scaffold (`ExampleTest.php`). لا feature tests لـ auth, categories, cart IDOR, orders رغم infrastructure جاهزة.

---

## 5. مشاكل أمنية

| # | المشكلة | الخطورة | الملف |
|---|---------|---------|-------|
| 1 | Hardcoded OTP `'123456'` | 🔴 Critical | `SendOtp.php:17` |
| 2 | Cart IDOR — list all users' items | 🔴 Critical | `CartItemController.php:28` |
| 3 | Cart IDOR — update/delete any item | 🔴 Critical | `CartItemController.php:47-63` |
| 4 | Cart IDOR — clear any user's cart | 🔴 Critical | `CartController.php:15-17` |
| 5 | Categories/Meals public CRUD | 🔴 Critical | `routes/api.php:33-36` |
| 6 | Password reset double-hash | 🔴 Critical | `ResetPassword.php:32` + `User.php:38` |
| 7 | Phone E164 mismatch login vs register | 🟡 High | `LoginUser.php:13`, `RegisterUser.php:14` |
| 8 | `resetPassword` swapped macro args | 🟡 High | `AuthController.php:94` |
| 9 | Empty cart checkout (zero-total order) | 🟡 High | `CreateOrder.php:19-35` |
| 10 | No throttle على register/login | 🟡 Medium | `routes/api.php:13-14` |
| 11 | Raw User model in register response | 🟡 Medium | `AuthController.php:28-29` |

---

## 6. ما هو جيد ✅

1. **Action pattern** — أفضل تطبيق معماري في الـ bootcamp: 19 actions across 5 domains.
2. **Form Requests** — 14 requests، validation خارج controllers (معظم endpoints).
3. **JSON:API Resources** — 10 resources جاهزة مع relationships.
4. **Factories + Seeders** — كاملة لكل models — جاهز للـ testing.
5. **ResponseServiceProvider** — envelope macros (فكرة جيدة للـ auth layer).
6. **Domain schema** — 10 models, migrations, DBML documentation.
7. **Strict types** — `declare(strict_types=1)` على models و cart/order controllers.
8. **E164 phone formatting** — في register/resend (مع propaganistas/laravel-phone).
9. **OTP hashed in cache** — مش plain text (مع attempts tracking).
10. **Sanctum token abilities** — access vs refresh vs issue-access separation.
11. **Telescope + Larastan + Pint** — أدوات جودة في dev dependencies.
12. **Spatie Media Library** — migration جاهزة لـ meal images.
13. **Auto cart creation** — `RegisterUser` ينشئ cart للـ user تلقائيًا.
14. **Order authorization** — `UpdateOrderRequest::authorize()` checks ownership.
15. **PR-based workflow** — feature branches merged via PRs.

---

## 7. خطة Refactor

### المرحلة 0 — Security Bugs (P0) 🔴

- [ ] إصلاح Cart IDOR — scope `index` to `auth()->user()->cart->items()`
- [ ] إضافة `authorize()` على `UpdateCartItemRequest` + Cart policy
- [ ] إضافة ownership check في `CartController::clear`
- [ ] إزالة hardcoded OTP — استخدم `random_int(100000, 999999)` + SMS gateway
- [ ] إصلاح double-hash في `ResetPassword` — `$user->update(['password' => $data['password']])`
- [ ] إصلاح `resetPassword` swapped macro arguments
- [ ] `auth:sanctum` + admin middleware على category/meal mutations
- [ ] Validate non-empty cart في `CreateOrder`

### المرحلة 1 — Consistency (P1) 🟡

- [ ] توحيد phone normalization (E164) في كل auth flow
- [ ] `Index*Request` + `List*` actions لكل index endpoints
- [ ] `LogoutUser` action
- [ ] استخدام `UserResource` في auth responses
- [ ] توحيد response format — macros أو JSON:API (مش الاتنين)
- [ ] إضافة `lang/en/auth.php` + `lang/en/lang.php` للـ translation keys
- [ ] throttle على register/login/verify-otp
- [ ] تسجيل `ConfigurationServiceProvider` في `bootstrap/providers.php`
- [ ] `GET /api/orders/{order}` show endpoint

### المرحلة 2 — Architecture (P2) 🟢

- [ ] Repository interfaces + implementations
- [ ] DTOs (`RegisterUserData`, `CreateOrderData`)
- [ ] Bind interfaces في `AppServiceProvider`
- [ ] Policies: `CartItemPolicy`, `CartPolicy`, `OrderPolicy`
- [ ] Feature tests (auth flow + cart ownership + order create)
- [ ] README documentation لـ Foodify

### المرحلة 3 — Remaining Domain (P3) 🟢

| # | Module | السبب |
|---|--------|-------|
| 1 | Favorites | schema + resource جاهز — routes فقط |
| 2 | Reviews | يعتمد على Order completion |
| 3 | Profile | update name/phone/password |
| 4 | Payment gateway | Stripe/Paymob checkout |
| 5 | Notifications | order status changes |
| 6 | Settings | app config |
| 7 | Admin | CRUD categories/meals/orders |
| 8 | Ingredient | many-to-many مع Meal |

---

## 8. Controller المستهدف (مثال: Cart)

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Actions\Cart\ListCartItems;
use App\Http\Requests\IndexCartItemRequest;
use App\Http\Resources\CartItemResource;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;

final class CartItemController extends Controller
{
    public function index(IndexCartItemRequest $request, ListCartItems $action): AnonymousResourceCollection
    {
        $cartItems = $action->handle($request->user(), $request->integer('per_page'));

        return CartItemResource::collection($cartItems);
    }
}
```

```php
// app/Actions/Cart/ListCartItems.php
public function handle(User $user, int $perPage): LengthAwarePaginator
{
    return $user->cart
        ->items()
        ->with('meal')
        ->paginate($perPage);
}
```

```php
// app/Policies/CartItemPolicy.php
public function update(User $user, CartItem $cartItem): bool
{
    return $cartItem->cart->user_id === $user->id;
}
```

---

## 9. File-by-File Scorecard

| File | Thin Controller | Form Request | Action | Auth/Policy | Response | Verdict |
|------|:-:|:-:|:-:|:-:|:-:|---------|
| `AuthController` | 🟡 refresh/logout fat | ✅ | ✅ | 🟡 partial | 🟡 mixed | 🟡 Partial |
| `CategoryController` | 🟡 index fat | 🟡 partial | 🟡 partial | ❌ public writes | ✅ JSON:API | 🟡 Partial |
| `MealController` | 🟡 index fat | 🟡 partial | 🟡 partial | ❌ public writes | ✅ JSON:API | 🟡 Partial |
| `CartItemController` | 🟡 index fat | 🟡 no authorize | ✅ | 🔴 IDOR | ✅ JSON:API | 🔴 Critical |
| `CartController` | ✅ | — | ✅ | 🔴 IDOR | ✅ JSON:API | 🔴 Critical |
| `OrderController` | 🟡 index fat | ✅ | ✅ | 🟡 partial | ✅ JSON:API | 🟡 Partial |
| Auth Actions (7) | — | — | ✅ | — | — | 🟡 Good |
| Category/Meal Actions (6) | — | — | 🟡 pass-through | — | — | 🟡 OK |
| Cart Actions (4) | — | — | 🟡 no ownership | — | — | 🔴 IDOR |
| Order Actions (2) | — | — | 🟡 empty cart | — | — | 🟡 Partial |
| Form Requests (14) | — | ✅ | — | 🟡 gaps | — | 🟡 Good |
| Models (10) | — | — | — | — | — | ✅ Good |
| JSON:API Resources (10) | — | — | — | — | ✅ | ✅ Good |
| `ResponseServiceProvider` | — | — | — | — | 🟡 | 🟡 Partial |
| `AppServiceProvider` | — | — | — | ❌ | — | 🔴 Empty |
| Tests | — | — | — | — | — | 🔴 Missing |

---

## 10. مقارنة مع مشاريع Bootcamp

| المعيار | Elham | Abdella | Mohamed Esmail | 3bd-ulrahman |
|---------|:-:|:-:|:-:|:-:|
| Feature completeness | ~68% | ~70% | ~83% | **~42%** |
| Actions layer | ✅ | ❌ | ❌ | ✅ **19 actions** |
| Form Requests | ✅ many | 🟡 3 | 🟡 partial | ✅ **14** |
| JSON:API Resources | ✅ | ❌ | ❌ | ✅ **10** |
| Repositories | ❌ | ❌ | ❌ | ❌ |
| Factories/Seeders | ✅ | 🟡 | 🟡 | ✅ كاملة |
| Feature Tests | 🟡 | ❌ | ✅ | ❌ |
| Payment gateway | ✅ Paymob | ✅ Stripe | 🟡 | ❌ |
| Favorites / Reviews | 🟡 | ✅ | ✅ | ❌ schema only |
| Cart security | 🟡 | 🟡 | ✅ | 🔴 **IDOR** |
| Admin | ✅ | ❌ | ✅ | ❌ |
| Response standard | macros | ApiTrait | manual | macros + JSON:API mixed |

**نقطة قوة:** أفضل Action pattern + Resources + Seeders infrastructure في الـ bootcamp.  
**نقطة ضعف:** أقل feature completeness + ثغرات IDOR حرجة في Cart + public catalog writes.

---

## 11. ملخص الأولويات (Action Items)

### 🔴 Critical (اعملها فورًا)

1. إصلاح **Cart IDOR** — scope index + policies على update/delete/clear
2. إزالة **hardcoded OTP** `'123456'` في `SendOtp.php`
3. إصلاح **double-hash password** في `ResetPassword.php`
4. **Auth على category/meal mutations** — لا تسيب CRUD public
5. **Empty cart validation** في `CreateOrder` — reject zero-item orders
6. إصلاح **`resetPassword`** swapped macro arguments

### 🟡 High

7. توحيد **phone normalization** (E164) في كل auth flow
8. تسجيل **`ConfigurationServiceProvider`**
9. **`Index*Request`** + `List*` actions لكل index endpoints
10. توحيد **response format** (macros أو JSON:API)
11. **`lang/` files** للـ translation keys
12. **throttle** على register/login/verify-otp
13. **`UserResource`** في auth responses
14. **`GET /api/orders/{order}`** show endpoint

### 🟢 Medium

15. Repository interfaces + DI bindings
16. DTOs لكل action
17. Feature tests (auth + cart ownership + order flow)
18. Implement Favorites + Reviews APIs
19. Payment gateway integration
20. Profile + Notifications + Settings + Admin modules
21. README documentation

---

## 13. تقرير Feature Completeness — النواقص في الـ Application

> **مرجع المتطلبات:** Authentication, Profile, Cart, My Orders, Notifications, Favorites, Meals/Categories, Reset Password, Category Details, Meal Details, Settings, Payments/Checkout.

### 13.1 Feature Matrix

| # | Feature | الحالة | Route / Implementation | النواقص |
|---|---------|--------|------------------------|---------|
| 1 | **Authentication** | 🟡 **78%** | register, login, logout, refresh, verify/resend OTP | OTP hardcoded, phone E164 mismatch |
| 2 | **Reset Password** | 🔴 **55%** | forgot → verify → reset | double-hash bug, swapped response args |
| 3 | **Profile** | 🔴 **0%** | — | no profile update route |
| 4 | **Categories** | 🟡 **70%** | `apiResource categories` | writes بدون admin guard |
| 5 | **Category Details** | ✅ **85%** | `GET /api/categories/{id}` | — |
| 6 | **Meals** | 🟡 **70%** | `apiResource meals` | writes بدون admin guard |
| 7 | **Meal Details** | ✅ **85%** | `GET /api/meals/{meal}` | — |
| 8 | **Favorites** | 🔴 **0%** | — | schema + resource + seeder exist, **no routes** |
| 9 | **Cart** | 🔴 **35%** | `cart-items` apiResource + clear cart | **IDOR critical**, no ownership |
| 10 | **Checkout** | 🟡 **40%** | `POST /api/orders` | no payment, empty cart allowed |
| 11 | **Payment** | 🔴 **0%** | — | no Stripe/gateway endpoint |
| 12 | **My Orders** | 🟡 **60%** | `GET /api/orders` | scoped to user ✅ |
| 13 | **Order Details** | 🔴 **30%** | `PUT /api/orders/{order}` only | **no GET show** |
| 14 | **Reviews** | 🔴 **0%** | — | schema + resource exist, **no routes** |
| 15 | **Notifications** | 🔴 **0%** | — | — |
| 16 | **Settings** | 🔴 **0%** | — | — |
| 17 | **Admin** | 🔴 **0%** | — | — |

**Overall Feature Completeness: ~42%**

### 13.2 Route Map

```
POST   /api/register, /login, /logout, /refresh          ✅
POST   /api/forgot-password, /verify-otp, /resend-otp   ✅ (OTP hardcoded)
POST   /api/reset-password                                🟡 (double-hash bug)

GET/POST/PUT/DELETE  /api/categories                      ✅ (public writes 🔴)
GET/POST/PUT/DELETE  /api/meals                           ✅ (public writes 🔴)

GET/POST/PUT/DELETE  /api/cart-items  [auth]              🟡 (IDOR 🔴)
DELETE /api/carts/{cart}/clear          [auth]              🟡 (IDOR 🔴)

GET    /api/orders                      [auth]            ✅
POST   /api/orders                      [auth]            🟡 (no payment, empty cart)
PUT    /api/orders/{order}              [auth]            ✅ (authorize ✅)

/api/favorites, /api/reviews, /api/ingredients            ❌
/api/profile, /api/notifications, /api/settings           ❌
/api/admin/*, /api/payments/*                             ❌
GET    /api/orders/{order}                                ❌
```

### 13.3 Database Tables

| Table | موجود | API Wired |
|-------|-------|-----------|
| `users` | ✅ | Auth only |
| `categories` | ✅ | CRUD (public) |
| `meals` | ✅ | CRUD (public) |
| `ingredients` + pivot | ✅ | ❌ no routes |
| `carts`, `cart_items` | ✅ | 🟡 IDOR bugs |
| `orders`, `order_items` | ✅ | 🟡 partial (no show) |
| `favorites` | ✅ | ❌ schema only |
| `reviews` | ✅ | ❌ schema only |
| `media` (Spatie) | ✅ migration | ❌ unused |
| `settings` | ❌ | — |
| `notifications` | ❌ | — |
| payment tables | ❌ | — |

### 13.4 Feature Completeness Scorecard

| Category | Score |
|----------|-------|
| Auth + Reset Password | 68% |
| Catalog (Categories + Meals) | 72% |
| Cart | 35% |
| Orders + Checkout | 45% |
| Favorites / Reviews | 0% |
| Payment | 0% |
| Profile / Notifications / Settings / Admin | 0% |
| **Overall** | **~42%** |

---

## 14. المراجع

- [Laravel Form Requests](https://laravel.com/docs/validation#form-request-validation)
- [Laravel JSON:API Resources](https://laravel.com/docs/eloquent-resources#json-api)
- [Laravel Authorization — Policies](https://laravel.com/docs/authorization#creating-policies)
- [Laravel Sanctum — Token Abilities](https://laravel.com/docs/sanctum#token-abilities)
- [propaganistas/laravel-phone — E164 formatting](https://github.com/Propaganistas/Laravel-Phone)
- مراجعة Abdella: [abdella/FoodIfy/CODE_REVIEW.md](../../abdella/FoodIfy/CODE_REVIEW.md)
- مراجعة Elham: [elham/FoodIfy/CODE_REVIEW.md](../../elham/FoodIfy/CODE_REVIEW.md)
- مراجعة Mohamed Esmail: [mohamedEsmail/FoodIfy/CODE_REVIEW.md](../../mohamedEsmail/FoodIfy/CODE_REVIEW.md)
- مراجعة Osama: [osama/OnlineRestaurant/CODE_REVIEW.md](../../osama/OnlineRestaurant/CODE_REVIEW.md)

---

*هذا الملف مرجع حي — حدّثه مع كل refactor جديد.*
