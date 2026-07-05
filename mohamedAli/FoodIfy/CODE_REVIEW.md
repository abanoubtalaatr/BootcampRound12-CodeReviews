# Foodify — Code Review & Architecture Guide

> **الهدف:** Controllers رفيعة تمامًا — بدون validation وبدون business logic.  
> كل المنطق في Actions / Services / Repositories مع تطبيق SOLID في كل الطبقات.

**تاريخ المراجعة:** 5 يوليو 2026  
**الطالب:** Mohamed Ali (MohamedAli-Elmorsy)  
**المستودع:** [MohamedAli-Elmorsy/Foodify](https://github.com/MohamedAli-Elmorsy/Foodify)  
**النطاق:** `app/` — Auth (OTP + WhatsApp) + Catalog + Favorites + Cart + Checkout (Stripe) + Orders (create only)

---

## 1. ملخص تنفيذي

| المنطقة | الحالة | التقييم |
|---------|--------|---------|
| Auth Module | مُنفَّذ (phone + OTP + WhatsApp) | 🟡 يعمل — fat controllers + ثغرات |
| OTP / Phone Verify | UltraMsg + `ichtrojan/laravel-otp` | 🔴 **OTP يُسَرَّب في response** |
| Reset Password | OTP flow كامل | 🟡 OTP leak + double hash |
| Form Requests | `LoginRequest` + `orderRequest` | 🔴 باقي flows inline |
| Actions / Services | غير موجود | 🔴 |
| Repositories | غير موجود | 🔴 |
| `ApiResponse` Trait | غير موجود | 🔴 JSON shapes مختلفة |
| API Resources | `UserResource` فقط | 🟡 |
| Catalog (Home/Meals) | مُنفَّذ | ✅ جيد — filters + related meals |
| Cart | مُنفَّذ | ✅ add/update/remove + summary |
| Checkout + Stripe | مُنفَّذ | 🟡 visa → Stripe session |
| My Orders | **غير مُنفَّذ** | 🔴 لا list/show |
| Favorites | مُنفَّذ | 🟡 fat controller |
| Profile | غير موجود | 🔴 |
| Notifications | جزئي | 🟡 `Notifiable` فقط — لا API |
| Payment Methods CRUD | غير موجود | 🔴 |
| Settings | غير موجود | 🔴 |
| Admin | غير موجود | 🔴 |
| Feature Tests | Breeze scaffold | 🔴 out of sync (email vs phone) |
| SOLID Compliance | ضعيف | 🔴 fat controllers everywhere |

**الخلاصة:** Mohamed Ali بنى **Foodify API functionally متوسط (~55%)** — Auth (register/login/logout/OTP/reset) شغال، Catalog ممتاز (home + meals + filters)، Cart كامل، و Checkout يعمل مع Stripe لمسار `visa` + Cash on delivery. لكن **ثغرات أمنية حرجة**: OTP يُرجَع في API responses، و **double password hashing** (`Hash::make()` + `'password' => 'hashed'` cast) يكسر login بعد register/reset. **My Orders ناقص بالكامل** — لا `GET /orders` ولا order details. Notifications جزئية، ولا Admin ولا Profile.

**Overall Feature Completeness: ~55%**

---

## 2. المعمارية المستهدفة vs الحالية

```
المستهدف:  FormRequest → Controller → Action → Service → RepositoryInterface → Model

Auth:      FormRequest → Controller ──────────────────────────────→ Model  (~25%)
           (LoginRequest فقط)  (+ inline validate, OTP, WhatsApp HTTP)

Business:  Request     → Controller ──────────────────────────────→ Model  (0%)
           (orderRequest فقط) (+ pricing, DB transaction, Stripe URL)
```

### قواعد الـ Controller (إلزامية)

```php
// ✅ Controller مثالي
public function checkout(CheckoutRequest $request, CheckoutAction $action): JsonResponse
{
    $result = $action->execute($request->user(), $request->toDto());

    return $this->data($result, 'Order placed successfully', 201);
}

// ❌ الوضع الحالي في OrderController
$cartItems = Cart::where('user_id', $user->id)->with('meal')->get();
$subTotal = $cartItems->sum(fn ($item) => $item->quantity * $item->meal->price);
DB::beginTransaction();
$order = Order::create([...]);
// ... mock/stripe payment_url hardcoded in controller
```

### هيكل المجلدات المقترح

```
app/
├── Actions/
│   ├── Auth/
│   ├── Cart/
│   └── Order/
├── Contracts/
│   ├── Repositories/
│   └── Services/
├── DTOs/
├── Http/
│   ├── Controllers/
│   ├── Middleware/
│   ├── Requests/          ← Form Request لكل endpoint
│   └── Resources/         ← MealResource, OrderResource, CartResource
├── Repositories/
├── Services/
│   ├── OtpService.php
│   ├── WhatsAppService.php
│   └── CartPricingService.php
└── Traits/
    └── ApiResponse.php      ← غير موجود 🔴
```

---

## 3. تحليل SOLID

### S — Single Responsibility Principle

| الملف | المشكلة |
|-------|---------|
| `RegisteredUserController` | validation + user create + OTP + WhatsApp HTTP + OTP في response |
| `PasswordResetLinkController` | validation + user lookup + OTP + WhatsApp + OTP leak |
| `OrderController` | cart pricing + DB transaction + order items + cart clear + Stripe redirect |
| `CartController` | validation + pricing (delivery 30) + upsert + quantity logic |
| `MealController` | query building + filters + related meals |
| `FavoriteController` | validation + toggle logic |

### O — Open/Closed Principle

- OTP + WhatsApp logic مكرر في `RegisteredUserController` و `PasswordResetLinkController` — أي تغيير gateway يتطلب تعديل controllers متعددة ❌
- لا interfaces — لا extension points للـ payment أو SMS ❌

### D — Dependency Inversion Principle

| الوضع الحالي | المطلوب |
|--------------|---------|
| `env('ULTRAMSG_*')` مباشرة في controllers | `WhatsAppService` via interface |
| `(new Otp)` في controllers | `OtpService` مُحقَن |
| `Cart::where()` / `Order::create()` مباشرة | Repository interfaces |
| `AppServiceProvider::register()` فارغ | DI bindings |

---

## 4. مراجعة ملف بملف

### 4.1 Auth Controllers — 🔴 Fat + Security Issues

#### `RegisteredUserController` — 🔴

```43:81:app/Http/Controllers/Auth/RegisteredUserController.php
        $user = User::create([
            'name' => $request->name,
            'phone' => $request->phone,
            'password' => Hash::make($request->password),  // 🔴 double hash
        ]);

        $otp = (new Otp)->generate($request->phone , 'numeric' , 6 ,10);
        // ... WhatsApp HTTP via env() + hardcoded instance183215
        return response()->json([
                'message' => 'Registration successful...',
                'phone' => $request->phone,
                'otp'=>$otp->token   // 🔴 OTP leak
            ],201);
```

| # | المشكلة | Severity |
|---|---------|----------|
| 1 | OTP في API response (success + failure) | 🔴 Critical |
| 2 | `Hash::make()` + `'password' => 'hashed'` cast | 🔴 login broken |
| 3 | Inline validation — لا `RegisterRequest` | 🟡 |
| 4 | `env()` مباشرة — يكسر `config:cache` | 🟡 |
| 5 | Hardcoded `instance183215` في URL | 🟡 |
| 6 | Unused imports (`Auth`, `Cache`, `UserResource`) | 🟡 |

---

#### `PasswordResetLinkController` — 🔴

```57:67:app/Http/Controllers/Auth/PasswordResetLinkController.php
        if($response->failed()){
            return response()->json([
                'error'=>'Faild to send WhatsApp message',
                'otp_debug' => $otp->token   // 🔴
            ],500);
        }
        return response()->json([
            'status' => 'Verify Code has been sent...',
            'otp' => $otp->token   // 🔴
        ]);
```

- OTP logic مكرر مع register
- User enumeration: `"This number doesnt match our recordes"` — typo + reveals if phone exists

---

#### `NewPasswordController` — 🔴

```49:51:app/Http/Controllers/Auth/NewPasswordController.php
        $user->forceFill([
            'password' => Hash::make($request->password),  // 🔴 double hash again
        ])->save();
```

- OTP validation via `ichtrojan/laravel-otp` ✅
- لا Form Request — inline validation

---

#### `AuthenticatedSessionController` — 🟡

**إيجابيات:**
- `LoginRequest` مع rate limiting (5 attempts) ✅ — **أفضل Form Request في الـ bootcamp**
- `UserResource` في login response ✅

**مشاكل:**
- Response keys غير موحّدة (`user_data`, `signin_token`)
- Token deletion في `destroy()` — يجب Action

---

#### `VerifyOtpController` — 🟡

- Invokable controller — pattern جيد ✅
- Inline validation — لا `VerifyOtpRequest`
- OTP validation + token creation في controller
- لا يحدّث `phone_verified_at` — column غير موجود أصلًا في migration

---

#### Breeze leftovers — 🟡

- `VerifyEmailController`, `EmailVerificationNotificationController` — email flow غير relevant
- `EnsureEmailIsVerified` middleware — **غير مستخدم**
- `User` لا يطبّق `MustVerifyEmail`

---

### 4.2 Catalog — ✅ جيد

#### `MealController` — 🟡 (functionally ✅)

| Method | الحالة | ملاحظات |
|--------|--------|---------|
| `home` | ✅ | categories + trending meals |
| `index` | ✅ | filter by category slug + `high-protein` / `low-calories` |
| `show` | ✅ | meal + related meals (same category) |

**ناقص:** `ListMealsRequest`, model scopes, `MealResource`, pagination  
**ملاحظة:** كل catalog محمي بـ `auth:sanctum` — intentional لكن غير معتاد

---

### 4.3 `CartController` — 🟡 Fat (functionally ✅)

| Method | الحالة | المشاكل |
|--------|--------|---------|
| `index` | ✅ | subtotal + delivery `30.00` hardcoded |
| `add` | ✅ | upsert logic في controller |
| `updateQuantity` | ✅ | increase/decrease — stale `quantity` بعد delete branch |
| `remove` | ✅ | inline validation |

**Delivery fee `30`** مكرر في `CartController` و `OrderController` — يجب config.

---

### 4.4 `OrderController` — 🟡 Checkout only

```16:86:app/Http/Controllers/Api/OrderController.php
    public function checkout(orderRequest $request): JsonResponse
    {
        // cart load + pricing + DB transaction + order items + cart clear
        if ($request->payment_method === 'visa') {
            return response()->json([
                'payment_url' => 'https://sandbox.checkout.stripe.com/pay/mock_session_foodify_' . $order->id,
                ...
            ]);
        }
        // cash on delivery response
    }
```

| # | المشكلة | Severity |
|---|---------|----------|
| 1 | **لا `index()` / `show()`** — My Orders مفقود | 🔴 |
| 2 | Stripe/payment logic في controller | 🟡 |
| 3 | DB transaction في controller | 🟡 |
| 4 | `$e->getMessage()` exposed في catch | 🟡 |
| 5 | `orderRequest` — naming convention (`OrderRequest`) | 🟡 |
| 6 | Delivery fee hardcoded `30.00` | 🟡 |

**إيجابي:** Checkout flow يعمل — cash + Stripe (`visa`) paths، transaction + cart clear ✅

---

### 4.5 `FavoriteController` — 🟡

- Inline validation + toggle logic في controller
- Raw models في response — لا `MealResource`

---

### 4.6 Form Requests — 🔴 جزئي

| Request | الحالة |
|---------|--------|
| `LoginRequest` | ✅ validation + authenticate + rate limiting |
| `orderRequest` | ✅ checkout validation |
| `RegisterRequest` | 🔴 مفقود |
| `VerifyOtpRequest` | 🔴 مفقود |
| `ForgotPasswordRequest` | 🔴 مفقود |
| `ResetPasswordRequest` | 🔴 مفقود |
| `AddToCartRequest` | 🔴 مفقود |
| `ToggleFavoriteRequest` | 🔴 مفقود |

---

### 4.7 Actions / Repositories / Services — 🔴 غير موجود

| المجلد | الحالة |
|--------|--------|
| `app/Actions/` | ❌ |
| `app/Repositories/` | ❌ |
| `app/Services/` | ❌ |
| `app/DTOs/` | ❌ |

---

### 4.8 `ApiResponse` Trait — 🔴 غير موجود

| Endpoint | Response shape |
|----------|---------------|
| Login | `{ user_data, signin_token }` |
| Register | `{ message, phone, otp }` |
| Cart | `{ cart_items, summary }` أو `{ status, message }` |
| Meals | `{ count, meals }` |
| Checkout | `{ status, message, payment_url, order_id }` |
| OTP verify | `{ access_token, token_type, user }` |

**لا envelope موحّد `{ message, error, data }`.**

---

### 4.9 Models — 🟡

#### `User.php`

```44:49:app/Models/User.php
        return [
            'phone_verified_at' => 'datetime',
            'password' => 'hashed',   // 🔴 + Hash::make() في controllers = double hash
        ];
```

| المشكلة | الإصلاح |
|---------|---------|
| Double hashing | إما cast `'hashed'` **أو** `Hash::make()` — ليس الاثنين |
| `phone_verified_at` cast بدون column | add migration أو remove cast |
| Relationships | `favoriteMeals()`, `cartItems()`, `orders()` ✅ |

**Migration issues:**

```17:19:database/migrations/0001_01_01_000000_create_users_table.php
           $table->string('phone');           // ❌ no unique index
            $table->string('is_active')->default(false);  // ❌ string not boolean
```

---

### 4.10 Routes — 🟡

```15:31:routes/api.php
Route::middleware(['auth:sanctum'])->group(function(){
    Route::get('/home', ...);
    Route::get('/meals', ...);
    Route::get('/favorites', ...);
    Route::get('/cart', ...);
    Route::post('/checkout', [OrderController::class, 'checkout']);
    // ❌ لا GET /orders, /orders/{id}
    // ❌ لا /profile, /notifications, /admin
});
```

**ناقص:** rate limiting على OTP routes (بخلاف login)، named routes على business endpoints، API versioning.

---

### 4.11 Tests — 🔴 Out of Sync

| Test | المشكلة |
|------|---------|
| `AuthenticationTest` | يستخدم `email` — app يستخدم `phone` |
| `RegistrationTest` | Breeze email flow |
| `EmailVerificationTest` | غير relevant |

Tests scaffold موجود (Pest) — إيجابي — لكن **لا تعكس التنفيذ الفعلي**.

---

### 4.12 Seeders — 🟡

- `FoodifySeeder` — categories + meals ✅
- **غير مربوط** في `DatabaseSeeder`
- `DatabaseSeeder` ينشئ user بـ `email` — field غير موجود في schema

---

## 5. مشاكل أمنية

| # | المشكلة | الملف | Severity |
|---|---------|-------|----------|
| 1 | **OTP في API response** (register success + failure) | `RegisteredUserController:72,80` | 🔴 Critical |
| 2 | **OTP في API response** (forgot password + debug) | `PasswordResetLinkController:60,66` | 🔴 Critical |
| 3 | **Double password hashing** — login fails after register/reset | `User` cast + `RegisteredUserController` + `NewPasswordController` | 🔴 Critical |
| 4 | `env()` مباشرة — secrets exposure risk | register + forgot password | 🟡 |
| 5 | Hardcoded UltraMsg instance ID | `instance183215` | 🟡 |
| 6 | User enumeration في forgot password | `PasswordResetLinkController:35-37` | 🟡 |
| 7 | No rate limiting على register/OTP send | routes | 🟡 |
| 8 | Exception message leaked to client | `OrderController:84` | 🟡 |
| 9 | `phone` بدون unique index في migration | users migration | 🟡 |
| 10 | Raw models في responses — may leak fields | all API controllers | 🟡 |

---

## 6. ما هو جيد ✅

1. **Customer journey جزئي** — register → OTP verify → browse → cart → checkout (Stripe/cash).
2. **`LoginRequest`** — validation + authenticate + rate limiting — reference implementation للـ bootcamp.
3. **`UserResource`** — مستخدم في login و OTP verify.
4. **WhatsApp OTP integration** — UltraMsg + `ichtrojan/laravel-otp` — real notification channel.
5. **Catalog quality** — home + meals list + filters (`high-protein`, `low-calories`) + related meals.
6. **Cart CRUD** — add, update quantity, remove, summary with delivery fee.
7. **Checkout with Stripe** — `visa` payment path + cash on delivery + DB transaction + cart clear.
8. **Relationships** — User ↔ favorites/cart/orders; Meal ↔ category.
9. **`FoodifySeeder`** — بيانات تجريبية جاهزة.
10. **Sanctum** — token auth صحيح + logout per device.
11. **`orderRequest`** — Form Request للـ checkout (خطوة نحو المعمارية المستهدفة).
12. **Invokable `VerifyOtpController`** — single-action pattern.

---

## 7. خطة Refactor

### Sprint 0 — أمن فوري (P0) 🔴

1. **إزالة OTP من كل API responses** — register + forgot password
2. **إصلاح double hashing** — احذف `Hash::make()` من controllers **أو** احذف `'hashed'` cast
3. نقل `ULTRAMSG_*` لـ `config/services.php`
4. إزالة hardcoded `instance183215`
5. إضافة `unique()` على `phone` + إصلاح `is_active` → boolean

### Sprint 1 — Architecture Foundation (P1)

6. `ApiResponse` trait على base `Controller`
7. Form Requests لكل endpoint (اتبع `LoginRequest` كمرجع)
8. `OtpService` + `WhatsAppService` — unify OTP flows
9. `CheckoutAction` — extract OrderController logic
10. `CartPricingService` — centralize delivery fee
11. API Resources: `MealResource`, `CategoryResource`, `CartResource`, `OrderResource`

### Sprint 2 — Missing Features (P1)

12. **`GET /api/orders`** — order history list (paginated)
13. **`GET /api/orders/{id}`** — order details + ownership check
14. Profile module (`GET/PUT /api/profile`)
15. Notifications API (index, unread, mark read)
16. إزالة Breeze email leftovers

### Sprint 3 — Quality (P2)

17. Feature tests — phone-based auth + cart + checkout
18. Wire `FoodifySeeder` في `DatabaseSeeder`
19. Rate limiting على OTP routes
20. Settings + Admin modules
21. Repository interfaces + DI
22. README documentation

---

## 8. Controller المستهدف (مثال: Register)

```php
<?php

namespace App\Http\Controllers\Auth;

use App\Actions\Auth\RegisterUserAction;
use App\Http\Controllers\Controller;
use App\Http\Requests\Auth\RegisterRequest;
use App\Traits\ApiResponse;
use Illuminate\Http\JsonResponse;

final class RegisteredUserController extends Controller
{
    use ApiResponse;

    public function store(RegisterRequest $request, RegisterUserAction $action): JsonResponse
    {
        $action->execute($request->toDto());

        return $this->data(
            ['phone' => $request->phone],
            'Registration successful. Please verify your phone via WhatsApp.',
            201
        );
        // ❌ NEVER return OTP in response
    }
}
```

---

## 9. File-by-File Scorecard

| File | Thin | Form Request | Service/Action | Repository | ApiResponse | Verdict |
|------|:----:|:------------:|:--------------:|:----------:|:-----------:|---------|
| `RegisteredUserController` | ❌ | ❌ | ❌ | ❌ | ❌ | 🔴 OTP leak + double hash |
| `PasswordResetLinkController` | ❌ | ❌ | ❌ | ❌ | ❌ | 🔴 OTP leak |
| `NewPasswordController` | ❌ | ❌ | ❌ | ❌ | ❌ | 🔴 double hash |
| `AuthenticatedSessionController` | 🟡 | ✅ | ❌ | ❌ | ❌ | 🟡 Partial |
| `VerifyOtpController` | ❌ | ❌ | ❌ | ❌ | ❌ | 🟡 |
| `MealController` | ❌ | — | ❌ | ❌ | ❌ | 🟡 OK functionally |
| `FavoriteController` | ❌ | ❌ | ❌ | ❌ | ❌ | 🔴 |
| `CartController` | ❌ | ❌ | ❌ | ❌ | ❌ | 🟡 OK functionally |
| `OrderController` | ❌ | ✅ | ❌ | ❌ | ❌ | 🟡 checkout only |
| `LoginRequest` | — | ✅ | — | — | — | ✅ Best in bootcamp |
| `orderRequest` | — | ✅ | — | — | — | 🟡 naming |
| `UserResource` | — | — | — | — | 🟡 | 🟡 partial use |
| Models | — | — | — | — | — | 🟡 double hash |
| Tests | — | — | — | — | — | 🔴 out of sync |
| Seeders | — | — | — | — | — | 🟡 not wired |

---

## 10. مقارنة مع مشاريع Bootcamp

| المعيار | Mohamed Ali | Abdella | Mohamed Esmail | Ali |
|---------|:-----------:|:-------:|:--------------:|:---:|
| Feature completeness | **~55%** | ~70% | ~83% | ~68% |
| Stripe / Payment | 🟡 checkout | ✅ Cashier | 🟡 pending | ✅ strategy |
| My Orders list/show | 🔴 missing | ✅ | ✅ | 🔴 empty controller |
| Catalog quality | ✅ good | 🟡 | ✅ | ✅ |
| Cart | ✅ full | ✅ | ✅ | 🟡 bugs |
| OTP channel | ✅ WhatsApp | ✅ WhatsApp | Vonage SMS | ✅ |
| OTP leak | **🔴 in response** | ❌ | ❌ | bypass only |
| Double password hash | **🔴 yes** | ❌ | ❌ | ❌ |
| Form Requests | 🟡 2 files | 🟡 3 files | 🟡 partial | ✅ many |
| Actions layer | ❌ | ❌ | ❌ | 🟡 partial |
| ApiResponse trait | ❌ | ✅ | ❌ | ✅ |
| LoginRequest quality | ✅ best | 🟡 | — | 🟡 |
| Admin | ❌ | ❌ | ✅ web | 🟡 partial |
| Feature tests | 🔴 out of sync | 🔴 | ✅ | ✅ |

**نقطة قوة Mohamed Ali:** Catalog + Cart + Checkout (Stripe) + `LoginRequest` ممتاز + WhatsApp OTP flow.  
**نقطة ضعف:** OTP leak + double hashing (auth broken) + لا My Orders + fat controllers + لا Actions/Repositories.

---

## 11. ملخص الأولويات (Action Items)

### 🔴 Critical

1. **إزالة OTP من API responses** — register + forgot password + `otp_debug`
2. **إصلاح double password hashing** — `Hash::make()` **أو** `'hashed'` cast — not both
3. **`GET /api/orders`** + **`GET /api/orders/{id}`** — My Orders مفقود
4. لا ترجع `$e->getMessage()` للـ client في production

### 🟡 High

5. **`ApiResponse` trait** — توحيد JSON envelope
6. **Form Requests** لكل endpoint (Register, VerifyOtp, Cart, Favorite)
7. **`OtpService` + `WhatsAppService`** — unify OTP flows
8. **Actions layer** — Register, Checkout, AddToCart
9. إضافة **`unique()`** على `phone` + إصلاح **`is_active`** type
10. نقل secrets لـ **`config/services.php`**
11. Fix **tests** لـ phone auth
12. **`phone_verified_at`** column + update on verify

### 🟢 Medium

13. API Resources لـ Meal, Category, Cart, Order
14. Profile module
15. Notifications API (index, mark read)
16. Wire `FoodifySeeder`
17. Rate limiting على OTP routes
18. Settings + Admin modules
19. Repository interfaces + DI
20. README documentation
21. إزالة Breeze email leftovers

---

## 13. تقرير Feature Completeness — النواقص في الـ Application

> **مرجع المتطلبات:** Authentication, Profile, Cart, My Orders, Notifications, Favorites, Meals/Categories, Reset Password, Category Details, Meal Details, Settings, Payments/Checkout.

### 13.1 Feature Matrix

| # | Feature | الحالة | Route / Implementation | النواقص |
|---|---------|--------|------------------------|---------|
| 1 | **Authentication** | 🟡 **82%** | register, login, logout, verify-otp | OTP leak, double hash |
| 2 | **Reset Password** | 🟡 **72%** | forgot-password, reset-password | OTP leak, double hash |
| 3 | **Profile** | 🔴 **0%** | — | — |
| 4 | **Categories** | ✅ **85%** | in `/home` response | no standalone endpoint |
| 5 | **Category Details** | 🟡 **55%** | filter by slug in `/meals?category=` | no dedicated show |
| 6 | **Meals** | ✅ **90%** | home, index, show | auth required |
| 7 | **Meal Details** | ✅ **92%** | show + related meals | — |
| 8 | **Favorites** | ✅ **88%** | list + toggle | raw models |
| 9 | **Cart** | ✅ **90%** | index, add, update, remove | delivery fee hardcoded |
| 10 | **Checkout** | 🟡 **78%** | `POST /api/checkout` | Stripe in controller |
| 11 | **Payment** | 🟡 **65%** | cash + Stripe (`visa`) | no payment-methods CRUD |
| 12 | **My Orders** | 🔴 **15%** | — | **no GET /orders** |
| 13 | **Order Details** | 🔴 **15%** | — | **no show route** |
| 14 | **Notifications** | 🟡 **25%** | `Notifiable` trait only | no API endpoints |
| 15 | **Settings** | 🔴 **0%** | — | — |
| 16 | **Admin** | 🔴 **0%** | — | — |

**Overall Feature Completeness: ~55%**

### 13.2 Route Map

```
POST /api/register, /login, /logout                    ✅
POST /api/forgot-password, /reset-password             ✅ (OTP leak)
POST /api/verify-otp                                   ✅

GET  /api/home, /meals, /meals/{id}                    ✅ (auth required)
/api/favorites/*, /cart/*                              ✅
POST /api/checkout                                     ✅ (Stripe + cash)

GET  /api/orders, /api/orders/{id}                     ❌
/api/profile, /api/notifications, /api/settings        ❌
/api/admin/*, /api/payment-methods                     ❌
```

### 13.3 Database Tables

| Table | موجود | API Wired |
|-------|-------|-----------|
| `users` | ✅ | Auth |
| `categories`, `meals` | ✅ | Catalog |
| `favorites`, `carts` | ✅ | Favorites + Cart |
| `orders`, `order_items` | ✅ | Checkout create only — **no read** |
| `notifications` | ❌ | — |
| `settings` | ❌ | — |
| `payment_methods` | ❌ | — |

### 13.4 Feature Completeness Scorecard

| Category | Score |
|----------|-------|
| Auth + Reset + Phone Verify | 75% |
| Profile + Catalog | 70% |
| Cart + Favorites | 89% |
| Checkout + Payment (Stripe) | 72% |
| My Orders (list/show) | 15% |
| Notifications | 25% |
| Settings + Admin | 0% |
| **Overall** | **~55%** |

---

## 14. المراجع

- [ichtrojan/laravel-otp](https://github.com/ichtrojan/laravel-otp)
- [Laravel Sanctum](https://laravel.com/docs/sanctum)
- [Laravel Form Requests](https://laravel.com/docs/validation#form-request-validation)
- [Stripe Checkout](https://stripe.com/docs/checkout)
- مراجعة Abdella: [abdella/FoodIfy/CODE_REVIEW.md](../../abdella/FoodIfy/CODE_REVIEW.md)
- مراجعة Mohamed Esmail: [mohamedEsmail/FoodIfy/CODE_REVIEW.md](../../mohamedEsmail/FoodIfy/CODE_REVIEW.md)
- مراجعة Ali: [ali/FoodIfy/CODE_REVIEW.md](../../ali/FoodIfy/CODE_REVIEW.md)

---

*هذا الملف مرجع حي — حدّثه مع كل refactor جديد.*
