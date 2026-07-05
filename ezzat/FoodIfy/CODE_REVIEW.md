# FoodIfy — Code Review & Architecture Guide

> **الهدف:** Controllers رفيعة تمامًا — بدون validation وبدون business logic.  
> كل المنطق في Actions / Services / Repositories مع تطبيق SOLID في كل الطبقات.

**تاريخ المراجعة:** 5 يوليو 2026  
**الطالب:** Ezzat (EzzatCodes)  
**المستودع:** [EzzatCodes/Foodify](https://github.com/EzzatCodes/Foodify)  
**النطاق:** `app/` — Auth + Catalog + Favorites + Cart + Checkout/Payment + Profile + Admin

---

## 1. ملخص تنفيذي

| المنطقة | الحالة | التقييم |
|---------|--------|---------|
| Auth Module | مُنفَّذ مع Actions | ✅ أفضل auth architecture في الـ bootcamp |
| Form Requests (Auth) | 6 requests | ✅ |
| Form Requests (Business) | 1 فقط (`SearchRequest`) | 🔴 |
| Actions (Auth) | 6 actions | ✅ |
| Actions (Business) | غير موجود | 🔴 |
| Repositories | غير موجود | 🔴 |
| Services | `CheckoutService` + `StripeService` | 🟡 بدون interfaces |
| DTOs | غير موجود | 🔴 |
| `ApiResponse` Trait | غير موجود | 🔴 response shapes مختلفة |
| Catalog / Favorites | مُنفَّذ | 🟡 fat controller — catalog على `web.php` |
| Cart | مُنفَّذ | 🔴 `User::first()` + `attach()` على `hasMany` |
| Checkout + Payment | مُنفَّذ جزئيًا | 🟡 service layer موجود — bugs حرجة |
| My Orders | غير موجود | 🔴 0% |
| Notifications | غير موجود | 🔴 0% |
| Settings | غير موجود | 🔴 0% |
| Admin CRUD | categories ✅ / products 🔴 | 🟡 routing bug + no auth |
| Feature Tests | scaffold فقط | 🔴 |
| SOLID Compliance | جزئي — auth جيد، business ضعيف | 🟡 |

**الخلاصة:** Ezzat عنده **أفضل Auth layer في الـ bootcamp** — 6 Actions + 6 Form Requests مع DI وOTP hashed وpurpose enum. لكن الـ business domain (catalog, cart, checkout) **fat controllers** مع bugs حرجة: `User::first()` بدل المستخدم المُصادَق، `attach()`/`detach()` على `hasMany` relationship، admin product routes تشير لـ `AdminController` بينما الـ methods في `ProductsAdminController`. Catalog على **web routes** مش API. My Orders، Notifications، Settings **غير موجودين**.

**Overall Feature Completeness: ~50%**

---

## 2. المعمارية المستهدفة vs الحالية

```
المستهدف:  FormRequest → Controller → Action → Service → RepositoryInterface → Model

Auth:      FormRequest → Controller → Action ──────────────────────────→ Model  (~55%)
Catalog:   SearchRequest → Controller ─────────────────────────────────→ Model  (~15%)
Cart/Fav:  (لا شيء)    → Controller ──────────────────────────────────→ Model  (0%)
Checkout:  (inline)    → Controller → CheckoutService → StripeService → Model  (~35%)
Admin:     FormRequest → Controller ──────────────────────────────────→ Model  (~20%)
```

### قواعد الـ Controller (إلزامية)

```php
// ✅ Controller مثالي — Auth (Ezzat قريب من هذا)
public function register(RegisterRequest $request, RegisterUser $action): JsonResponse
{
    $user = $action->handle($request->validated());
    return $this->data(['user' => new UserResource($user)], 'User registered', 201);
}

// ❌ الوضع الحالي في HomeController::addToCart
$user = User::first();  // always first user in DB
$user->cart()->attach($productId);  // attach() on hasMany — runtime error
```

### هيكل المجلدات المقترح

```
app/
├── Actions/
│   ├── Auth/              ← موجود ✅ (6 actions)
│   ├── Cart/
│   ├── Catalog/
│   ├── Order/
│   └── Favorite/
├── Contracts/
│   ├── Repositories/
│   └── Services/            ← StripeServiceInterface
├── DTOs/
├── Http/
│   ├── Controllers/
│   │   ├── Auth/            ← إصلاح casing: auth/ → Auth/
│   │   └── Api/
│   ├── Requests/            ← Form Request لكل endpoint
│   └── Resources/           ← ProductResource, OrderResource
├── Repositories/
├── Services/
│   ├── CheckoutService.php  ← موجود ✅
│   └── StripeService.php    ← موجود ✅
└── Traits/
    └── ApiResponse.php      ← مفقود 🔴
```

---

## 3. تحليل SOLID

### S — Single Responsibility Principle

| الملف | المشكلة |
|-------|---------|
| Auth Actions | ✅ مسؤولية واحدة لكل action |
| `AuthController` | 🟡 يستدعي actions لكن فيه user lookup + token creation خارج actions |
| `HomeController` | ❌ catalog + search + favorites + cart — 200+ سطر |
| `CheckoutController` | 🟡 thin wrapper لكن `User::first()` bug |
| `CheckoutService` | ✅ order + items + payment intent |
| `VerifyOtp` | 🟡 `handlePasswordReset()` dead code — never called |

### O — Open/Closed Principle

- OTP logic في `SendOtp` + duplicate في `ResendOtpAction` — أي تغيير gateway يتطلب تعديل ملفين.
- `StripeService` concrete — لا interface للـ mock في tests.

### D — Dependency Inversion Principle

| الوضع الحالي | المطلوب |
|--------------|---------|
| Auth Actions → Eloquent مباشرة | Repository interfaces |
| `RegisterUser` → `SendOtp` (concrete) | مقبول داخل نفس layer |
| `CheckoutService` → `StripeService` (concrete) | `PaymentGatewayInterface` |
| `AppServiceProvider::register()` فارغ | DI bindings |
| Admin بدون auth middleware | Policies + `auth:sanctum` + role |

---

## 4. مراجعة ملف بملف

### 4.1 `AuthController` — ✅ جيد + 🐛 Bugs

```29:39:app/Http/Controllers/auth/AuthController.php
    public function register(RegisterRequest $request, RegisterUser $action): JsonResponse
    {
        $validatedData = $request->validated();
        $user = $action->handle($validatedData);
        $token = $user->createToken('auth_token')->plainTextToken;
        return response()->json(['message' => 'User registered successfully', 'user' => $user, 'access_token' => $token, 'token_type' => 'Bearer'], 201);
    }
```

**إيجابيات:**
- Form Requests + Actions injection ✅
- `register`, `login` thin نسبيًا ✅
- `resetPassword` يستخدم action chain (VerifyOtp → ResetPasswordAction) ✅
- `logout` يحذف Sanctum token ✅

**🐛 Bugs حرجة:**

| # | المشكلة | التفاصيل |
|---|---------|----------|
| 1 | Namespace vs folder casing | File: `Controllers/auth/` — Namespace: `Auth` — Route: `auth\AuthController` — **يفشل autoload على Linux** |
| 2 | `use App\models\User` | casing خاطئ — يجب `App\Models\User` |
| 3 | `use App\Actions\Auth\sendOtp` | class name هو `SendOtp` — casing خاطئ |
| 4 | `verifyOtp` يبحث بـ `email` | `resetPasswordOtp` يستخدم `phone` — **inconsistent identifier** |
| 5 | Token قبل phone verify | register يصدر token قبل `verify-otp` |

**مخالفات:**

| Method | المشكلة |
|--------|---------|
| `verifyOtp`, `resendOtp` | `User::where()` في controller — يجب action |
| `resetPasswordOtp` | phone-based lookup بينما verify يستخدم email |
| كل methods | `response()->json()` يدوي — لا `ApiResponse` trait |

---

### 4.2 Auth Actions — ✅ أفضل في الـ bootcamp (مع bugs)

| Action | التقييم | ملاحظات |
|--------|---------|---------|
| `RegisterUser` | ✅ | DI على `SendOtp`, creates user + sends OTP |
| `LoginUser` | ✅ | credential check + `is_phone_verified` gate + token |
| `SendOtp` | ✅ | OTP hashed, expiry 1 min, invalidates old codes |
| `VerifyOtp` | 🔴 | **لا يضع `used_at`** — OTP reusable رغم `whereNull('used_at')` check |
| `VerifyOtp` | 🔴 | دائمًا `is_phone_verified=true` حتى لـ `reset_password` purpose |
| `VerifyOtp` | 🔴 | `handlePasswordReset()` **dead code** — never called from controller |
| `ResendOtpAction` | 🟡 | duplicates `SendOtp` logic |
| `ResetPasswordAction` | 🟡 | لا يبطل OTP بعد reset — لا `used_at` update |

```33:38:app/Actions/Auth/VerifyOtp.php
        $user->update([
            'is_phone_verified' => true,  // ❌ حتى لو purpose = reset_password
            'phone_verified_at' => now(),
        ]);
        return true;  // ❌ لا $otpCode->update(['used_at' => now()])
```

---

### 4.3 Form Requests (Auth) — ✅

| Request | الحالة |
|---------|--------|
| `RegisterRequest` | ✅ name, email, phone, password confirmed |
| `LoginRequest` | ✅ |
| `VerifyRequest` | ✅ purpose enum |
| `ResendOtpRequest` | ✅ |
| `ResetPasswordRequestOtp` | 🟡 phone بدون `exists:users,phone` |
| `ResetPasswordRequest` | 🟡 phone + otp + new_password |

**ناقص:** `toDto()` methods — Actions تقبل `array` مباشرة.

**مفقود لـ business:** favorites, cart, checkout, payment, profile update.

---

### 4.4 `HomeController` — 🔴 Fat + Critical Bugs

```153:179:app/Http/Controllers/Api/HomeController.php
    public function getCart()
    {
        $user = User::first();  // 🔴 MUST be auth()->user()
        $cartItems = $user->cart()->with('product.category')->get();
        ...
    }

    public function addToCart($id)
    {
        $user = User::first();  // 🔴
        ...
        $user->cart()->attach($productId);  // 🔴 attach() on hasMany
    }
```

| Method | المشاكل |
|--------|---------|
| `index`, `getCategories`, `category`, `product` | direct Eloquent — catalog على `web.php` مش `api.php` |
| `search` | `SearchRequest` ✅ — sole business Form Request |
| `addToFavorites`, `removeFromFavorites`, `getFavorites` | business logic في controller — لكن `auth()->user()` ✅ |
| `getCart`, `addToCart`, `removeFromCart` | **`User::first()`** + **`attach()`/`detach()` on `hasMany`** 🔴 |
| كل methods | لا API Resources — raw models |

**Cart relationship bug:** `User::cart()` is `hasMany(Cart::class)` — يجب `Cart::create(['user_id' => ..., 'product_id' => ...])` مش `attach()`.

---

### 4.5 Admin Controllers — 🟡 Partial + Routing Bug

**`routes/admin.php` — مربوط** لكن بدون auth middleware.

| File | المشكلة |
|------|---------|
| `routes/admin.php:18-22` | Product routes → **`AdminController`** لكن methods في **`ProductsAdminController`** — **runtime error** |
| `CategoriesAdminController::updateCategory` | `$request->all()` بدل `$request->validated()` — mass assignment risk |
| `ProductsAdminController` | CRUD logic صحيح + `ProductRequest` ✅ — لكن **unrouted** |
| `CategoriesAdminController:8` | `use App\Http\Requests\admin;` — unused import |
| كل admin | لا `auth:sanctum` ولا role check |

```17:22:routes/admin.php
    // Product routes — ❌ wrong controller
    Route::get('/products', [AdminController::class, 'allProducts']);
    Route::get('/products/{id}', [AdminController::class, 'product']);
    Route::post('/products/create', [AdminController::class, 'createProduct']);
    Route::put('/products/{id}/update', [AdminController::class, 'updateProduct']);
    Route::delete('/products/{id}/delete', [AdminController::class, 'deleteProduct']);
```

---

### 4.6 `CheckoutController` + Services — 🟡 Good Start, Critical Bugs

```13:16:app/Http/Controllers/Api/CheckoutController.php
    public function checkout(CheckoutService $checkoutService)
    {
        $user = User::first();  // 🔴 MUST be auth()->user() or $request->user()
        return $checkoutService->checkout($user);
    }
```

**إيجابيات:**
- `CheckoutService` — DB transaction + order + order items + payment record ✅
- `StripeService` — PaymentIntent via Stripe SDK ✅
- أول business service layer خارج auth ✅

**مشاكل:**

| # | المشكلة | Severity |
|---|---------|----------|
| 1 | `User::first()` in checkout + trackOrder | 🔴 |
| 2 | `trackOrder` يغيّر status لـ `delivered` — مش read-only tracking | 🟡 |
| 3 | `CheckoutService` يرجع `response()->json()` — service layer shouldn't return HTTP | 🟡 |
| 4 | Cart clear في `PaymentController` فقط — مش في checkout | 🟡 |
| 5 | `PaymentController` imports `PaymentRequest` — **file missing** | 🔴 |
| 6 | `PaymentController` inline `$request->validate()` | 🟡 |
| 7 | Payment status update بدون Stripe webhook verification | 🟡 |

---

### 4.7 `ProfileController` — 🟡

- `GET/POST /api/profile` ✅ behind `auth:sanctum`
- Inline `$request->validate()` — no `UpdateProfileRequest`
- Update focuses on `name` + `email` — inconsistent with phone-centric auth
- لا upload-photo، لا change-password

---

### 4.8 Models — 🟡

#### `User.php` — ✅

```35:48:app/Models/User.php
    public function favorites()
    {
        return $this->belongsToMany(Product::class, 'favorites', 'user_id', 'product_id');
    }

    public function cart()
    {
        return $this->hasMany(Cart::class);  // hasMany — NOT belongsToMany
    }
```

| Model | الحالة | ملاحظات |
|-------|--------|---------|
| `User` | ✅ | Sanctum, favorites belongsToMany, cart hasMany |
| `Product` | 🟡 | fillable ناقص (`calories`, `protein`, etc. من migration) |
| `Category` | ✅ | with products relationship |
| `OtpCode` | ✅ | hashed codes, purpose, expiry, `used_at` column exists |
| `Cart` | ✅ model | table `cart` — لكن controller يستخدم pivot API |
| `Order` / `OrderItem` / `Payment` | ✅ | wired in checkout flow |
| `Favorites` | 🟡 | pivot model exists — relationship via User |

---

### 4.9 Routes — 🟡

```14:33:routes/api.php
Route::middleware('auth:sanctum')->group(function () {
    // favorites, cart, checkout, pay, profile ✅ sanctum enabled
});
```

| File | المشكلة |
|------|---------|
| `web.php` | Catalog (`/`, `/categories`, `/category/{id}`, `/product/{id}`, `/products/search`) — **مش API** |
| `api.php` | لا `GET /orders` — My Orders missing |
| `api.php` | لا notifications / settings |
| `api.php:6-7` | `checkoutController`, `profileController` — casing inconsistency |
| `admin.php` | product routes → wrong controller |
| `auth.php:4` | `use App\Http\Controllers\auth\AuthController` — lowercase `auth` |
| `auth.php:22` | `/me` endpoint commented out |

---

### 4.10 `ApiResponse` / Repositories — 🔴

Response shapes مختلفة:

| Layer | Format |
|-------|--------|
| Auth | `{ message, user, access_token, token_type }` |
| Home/Admin | `{ message, status, data/categories/products }` |
| Checkout | `{ order_id, client_secret, payment_intent_id }` |

لا envelope موحّد `{ message, error, data }`.

---

### 4.11 `AppServiceProvider` — 🔴 فارغ

- لا repository bindings
- لا rate limiting على OTP routes
- لا custom exception handling

---

## 5. مشاكل أمنية

| # | المشكلة | الملف | Severity |
|---|---------|-------|----------|
| 1 | `User::first()` — any authenticated user gets first DB user's cart/orders | `HomeController`, `CheckoutController` | 🔴 |
| 2 | Cart `attach()`/`detach()` on `hasMany` — broken + potential data corruption | `HomeController` | 🔴 |
| 3 | OTP reusable — `used_at` never set after verify | `VerifyOtp.php` | 🔴 |
| 4 | Admin product routes → `AdminController` (methods don't exist) | `routes/admin.php` | 🔴 |
| 5 | `$request->all()` في admin category update | `CategoriesAdminController:67` | 🔴 |
| 6 | Namespace/folder casing — deploy fail on Linux | `AuthController` path + route import | 🔴 |
| 7 | Admin routes بدون auth middleware | `routes/admin.php` | 🟡 |
| 8 | Token issued on register before phone verify | `AuthController::register` | 🟡 |
| 9 | No rate limiting على OTP endpoints | `routes/auth.php` | 🟡 |
| 10 | Payment marked paid without Stripe webhook verification | `PaymentController` | 🟡 |
| 11 | OTP logged in plain text | `SendOtp.php:31` — `Log::info("OTP ... {$otp}")` | 🟡 |
| 12 | No API Resources — may leak internal fields | all controllers | 🟡 |

---

## 6. ما هو جيد ✅

1. **Auth Actions layer** — 6 actions مع DI — **أفضل auth architecture في الـ bootcamp** (مع 3bd-ulrahman).
2. **Form Requests للـ auth** — 6 requests مع validation قوية + purpose enum.
3. **OTP hashed** في `otp_codes` table — مش plain text (مع `Hash::make`).
4. **OTP purpose enum** — `registration` vs `reset_password`.
5. **`LoginUser` blocks unverified phones** — `is_phone_verified` check ✅.
6. **Factories + Seeders** — Product, Category seeded.
7. **DBML documentation** — `databas.dbml`.
8. **`resetPassword` action chain** — verify + reset منفصلين.
9. **Sanctum** على logout + business routes.
10. **`CheckoutService` + `StripeService`** — أول business service layer خارج auth.
11. **Profile endpoints** — get/update behind sanctum.
12. **Favorites** — `auth()->user()` + belongsToMany correctly.
13. **Admin categories CRUD** — wired + `CategoryRequest` validation.

---

## 7. خطة Refactor

### Sprint 0 — Bugs فورية (P0) 🔴

1. Fix **`User::first()`** → `$request->user()` in cart + checkout + trackOrder
2. Fix **cart operations** — `Cart::create()` / `delete()` instead of `attach()`/`detach()`
3. **`VerifyOtp`**: set `$otpCode->update(['used_at' => now()])`, branch on purpose (register vs reset)
4. Fix **admin product routes** → `ProductsAdminController::class`
5. Fix **namespace casing**: `auth/` → `Auth/`, `App\models\User` → `Models`, `sendOtp` → `SendOtp`
6. Create missing **`PaymentRequest`** or remove import
7. **`CategoriesAdminController::updateCategory`** — `$request->validated()` not `all()`

### Sprint 1 — Missing Features (P0/P1)

8. **`GET /api/orders`** — My Orders list (paginated)
9. **`GET /api/orders/{id}`** — order details with items
10. **Notifications module** — migration + `GET /api/notifications` + mark read
11. **Settings module** — `GET/PATCH /api/settings`
12. Move **catalog routes** from `web.php` to `api.php`

### Sprint 2 — Architecture (P1)

13. `ApiResponse` trait — unify JSON envelope
14. Repository interfaces + `AppServiceProvider` bindings
15. Actions: `ListProductsAction`, `AddToCartAction`, `ToggleFavoriteAction`
16. Form Requests لـ cart, payment, profile, admin
17. API Resources: `UserResource`, `ProductResource`, `OrderResource`
18. Admin auth middleware + role policy
19. `OtpService` — unify SendOtp + ResendOtp + mark used
20. `PaymentGatewayInterface` for StripeService

### Sprint 3 — Quality (P2)

21. Feature tests: auth flows, cart, checkout (mock Stripe)
22. Rate limiting على OTP routes
23. Remove OTP from logs in production
24. Stripe webhook for payment confirmation
25. README documentation
26. DTOs for checkout flow

---

## 8. Controller المستهدف (مثال: Add to Cart)

```php
public function store(AddToCartRequest $request, AddToCartAction $action): JsonResponse
{
    $cartItem = $action->execute(
        user: $request->user(),
        productId: $request->validated('product_id')
    );

    return $this->data(
        ['cart_item' => new CartItemResource($cartItem)],
        'Product added to cart.',
        201
    );
}

// AddToCartAction — uses hasMany correctly
public function execute(User $user, int $productId): Cart
{
    return $user->cart()->firstOrCreate(
        ['product_id' => $productId],
        ['quantity' => 1]
    );
}
```

---

## 9. File-by-File Scorecard

| File | Thin | Form Request | Action/Service | Repository | ApiResponse | Verdict |
|------|:----:|:------------:|:--------------:|:----------:|:-----------:|---------|
| `AuthController` | 🟡 | ✅ | ✅ | ❌ | ❌ | 🟡 Good + bugs |
| Auth Actions (×6) | — | — | ✅ | ❌ | — | ✅ Best in bootcamp |
| Form Requests (auth) | — | ✅ | — | — | — | ✅ |
| `HomeController` | ❌ | 🟡 1/8 | ❌ | ❌ | ❌ | 🔴 Critical bugs |
| `CheckoutController` | ✅ | ❌ | ✅ Service | ❌ | ❌ | 🔴 User::first() |
| `PaymentController` | 🟡 | ❌ missing | 🟡 Stripe | ❌ | ❌ | 🔴 |
| `ProfileController` | 🟡 | ❌ | ❌ | ❌ | ❌ | 🟡 |
| `CategoriesAdminController` | ❌ | ✅ | ❌ | ❌ | ❌ | 🟡 |
| `ProductsAdminController` | ❌ | ✅ | ❌ | ❌ | ❌ | 🔴 Unrouted |
| `AdminController` | ✅ | — | — | ❌ | ❌ | 🟡 index only |
| `CheckoutService` | — | — | ✅ | ❌ | — | 🟡 |
| `StripeService` | — | — | ✅ | ❌ | — | 🟡 |
| Routes | — | — | — | — | — | 🔴 catalog web + admin bug |
| Tests | — | — | — | — | — | 🔴 |
| `AppServiceProvider` | — | — | ❌ | ❌ | — | 🔴 |

---

## 10. مقارنة مع مشاريع Bootcamp

| المعيار | Ezzat | 3bd-ulrahman | Abdella | Mohamed Esmail |
|---------|:-----:|:------------:|:-------:|:--------------:|
| Auth Actions | ✅ 6 | ✅ 6 | ❌ | ❌ |
| Auth Form Requests | ✅ 6 | ✅ 8 | 🟡 3 | 🟡 partial |
| Business Actions | ❌ | 🟡 categories | ❌ | ❌ |
| Business Services | 🟡 checkout | ❌ | 🟡 WhatsApp | ❌ |
| ApiResponse | ❌ | macros | ✅ trait | ❌ |
| Feature completeness | ~50% | ~45% | ~70% | ~83% |
| Stripe / Payment | 🟡 PaymentIntent | ❌ | ✅ Cashier | 🟡 pending |
| Admin panel | 🟡 broken products | ❌ | ❌ | ✅ web |
| OTP security | 🔴 reusable | 🔴 hardcoded | ✅ hashed | ✅ Vonage |
| Cart / Orders | 🔴 bugs | ❌ | 🟡 works | ✅ |
| Feature tests | ❌ | ❌ | ❌ | ✅ |

**نقطة قوة Ezzat:** Auth Actions + Form Requests — **أفضل auth architecture** في الـ bootcamp. CheckoutService + StripeService بداية service layer.  
**نقطة ضعف:** `User::first()` bug + cart relationship misuse + admin routing broken + 50% features missing (orders list, notifications, settings).

---

## 11. ملخص الأولويات (Action Items)

### 🔴 Critical

1. Fix **`User::first()`** → `$request->user()` in Cart + Checkout + trackOrder
2. Fix **cart relationship** — stop using `attach()`/`detach()` on `hasMany`; use `Cart::create()` / `delete()`
3. إصلاح **OTP `used_at`** — mark OTP used after verify; branch purpose handling
4. Fix **admin product routes** → `ProductsAdminController::class`
5. إصلاح **namespace casing** (`auth/` → `Auth/`, `App\models\User`, `sendOtp`)
6. Create **`PaymentRequest`** or fix `PaymentController` import
7. Build **`GET /api/orders`** — My Orders list

### 🟡 High

8. **Notifications module** (migration + API + order triggers)
9. **Settings module** (`GET/PATCH /api/settings`)
10. **`GET /api/orders/{id}`** — order details with items
11. `ApiResponse` trait + API Resources
12. Form Requests لـ profile, cart, payment
13. Admin auth middleware + role check
14. Move catalog from `web.php` to `api.php`
15. Unify email vs phone in auth OTP flows
16. Rate limiting على OTP endpoints

### 🟢 Medium

17. DTOs + Repository interfaces
18. Feature tests (auth + cart + checkout mock Stripe)
19. Stripe webhook for payment confirmation
20. Remove plain-text OTP from logs
21. `ResendOtpAction` delegates to `SendOtp`
22. README documentation

---

## 13. تقرير Feature Completeness — النواقص في الـ Application

> **مرجع المتطلبات:** Authentication, Profile, Cart, My Orders, Notifications, Favorites, Meals/Categories, Reset Password, Category Details, Meal Details, Settings, Payments/Checkout.

### 13.1 Feature Matrix

| # | Feature | الحالة | Route / Implementation | النواقص |
|---|---------|--------|------------------------|---------|
| 1 | **Authentication** | ✅ **88%** | `POST /api/auth/register`, login, logout, verify-otp | token before verify; namespace bugs |
| 2 | **Reset Password** | 🟡 **65%** | reset-password-otp, reset-password | phone/email inconsistency; OTP reusable |
| 3 | **Profile** | 🟡 **60%** | `GET/POST /api/profile` | inline validation; no photo/password |
| 4 | **Categories** | 🟡 **70%** | `GET /categories` (web) | not on API prefix |
| 5 | **Category Details** | 🟡 **70%** | `GET /category/{id}` (web) | not on API |
| 6 | **Products/Meals** | 🟡 **70%** | `GET /` home + search (web) | catalog on web not api |
| 7 | **Product Details** | 🟡 **70%** | `GET /product/{id}` (web) | not on API |
| 8 | **Favorites** | ✅ **80%** | add/remove/get behind sanctum | no Form Request; fat controller |
| 9 | **Cart** | 🔴 **25%** | get/add/remove behind sanctum | **`User::first()`** + **`attach()` bug** |
| 10 | **Checkout** | 🟡 **55%** | `POST /api/checkout` | User::first(); service returns HTTP |
| 11 | **Payment** | 🟡 **50%** | `POST /api/pay` | missing PaymentRequest; no webhook |
| 12 | **My Orders** | 🔴 **0%** | — | no list endpoint |
| 13 | **Order Details** | 🔴 **0%** | — | track-order mutates status |
| 14 | **Notifications** | 🔴 **0%** | — | — |
| 15 | **Settings** | 🔴 **0%** | — | — |
| 16 | **Admin** | 🟡 **45%** | categories CRUD ✅; products broken | wrong controller; no auth |

**Overall Feature Completeness: ~50%**

### 13.2 Route Map

```
POST /api/auth/register, /login, /logout                    ✅
POST /api/auth/verify-otp, /resend-otp                     ✅
POST /api/auth/reset-password-otp, /reset-password         🟡 (OTP bugs)

GET  /  /categories  /category/{id}  /product/{id}         🟡 web.php — NOT /api/*
GET  /products/search                                       🟡 web.php

POST /api/add-to-favorites/{id}  /remove-from-favorites/{id}  ✅ sanctum
GET  /api/favorites                                           ✅ sanctum
GET  /api/cart  POST /add-to-cart  /remove-from-cart         🔴 User::first() bug
POST /api/checkout  /pay  /track-order/{orderId}              🟡 partial
GET  /api/profile  POST /api/profile                          🟡 sanctum

GET  /api/admin/categories/*                                  🟡 no auth
GET  /api/admin/products/*                                    🔴 wrong controller

GET  /api/orders  /api/orders/{id}                           ❌
/api/notifications/*  /api/settings/*                         ❌
```

### 13.3 Database Tables

| Table | موجود | API Wired |
|-------|-------|-----------|
| `users` | ✅ | Auth + Profile |
| `otp_codes` | ✅ | Auth (used_at not set) |
| `categories`, `products` | ✅ | Catalog (web routes) |
| `favorites` | ✅ | Favorites API |
| `cart` | ✅ | Cart API (broken logic) |
| `orders`, `order_items` | ✅ | Checkout creates — no list/show |
| `payments` | ✅ | Checkout + pay |
| `notifications` | ❌ | — |
| `settings` | ❌ | — |

### 13.4 Feature Completeness Scorecard

| Category | Score |
|----------|-------|
| Auth + Reset + Phone Verify | 78% |
| Profile + Catalog | 65% |
| Favorites | 80% |
| Cart | 25% |
| Checkout + Payment | 52% |
| My Orders | 0% |
| Notifications | 0% |
| Settings | 0% |
| Admin | 45% |
| **Overall** | **~50%** |

---

## 14. المراجع

- [Laravel Sanctum](https://laravel.com/docs/sanctum)
- [Stripe Payment Intents](https://stripe.com/docs/payments/payment-intents)
- [Laravel Form Requests](https://laravel.com/docs/validation#form-request-validation)
- مراجعة 3bd-ulrahman: [3bd-ulrahman/FoodIfy/CODE_REVIEW.md](../../3bd-ulrahman/FoodIfy/CODE_REVIEW.md)
- مراجعة Abdella: [abdella/FoodIfy/CODE_REVIEW.md](../../abdella/FoodIfy/CODE_REVIEW.md)
- مراجعة Mohamed Esmail: [mohamedEsmail/FoodIfy/CODE_REVIEW.md](../../mohamedEsmail/FoodIfy/CODE_REVIEW.md)

---

*هذا الملف مرجع حي — حدّثه مع كل refactor جديد.*
