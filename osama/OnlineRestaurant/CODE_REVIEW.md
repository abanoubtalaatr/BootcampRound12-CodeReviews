# Online Restaurant — Code Review & Architecture Guide

> **الهدف:** Controllers رفيعة تمامًا — بدون validation وبدون business logic.  
> كل المنطق في Actions / Services / Repositories مع تطبيق SOLID في كل الطبقات.

**تاريخ المراجعة:** 5 يوليو 2026 — **مراجعة كاملة**
**الطالب:** Osama  
**المستودع:** [osama726/online-restaurant](https://github.com/osama726/online-restaurant)  
**النطاق:** `app/` — Domain layer (models, migrations, enums) مُنفَّذ + Application layer (API) غير مُنفَّذ بعد

---

## 1. ملخص تنفيذي

| المنطقة | الحالة | التقييم |
|---------|--------|---------|
| Database Schema | مُنفَّذ بالكامل | ✅ ممتاز |
| Eloquent Models | 8 models مع relationships | ✅ جيد جدًا |
| Enums | OrderStatus, PaymentStatus, PaymentMethod | ✅ |
| Docker Setup | PHP + Nginx + MySQL + phpMyAdmin | ✅ |
| Spatie Permission | مُثبَّت + RolesSeeder | 🟡 جزئي |
| Auth Module | غير موجود | 🔴 |
| Controllers | base فقط — لا feature controllers | 🔴 |
| Form Requests | غير موجود | 🔴 |
| Actions / Services | غير موجود | 🔴 |
| Repositories | غير موجود | 🔴 |
| DTOs | غير موجود | 🔴 |
| API Resources | غير موجود | 🔴 |
| `ApiResponse` Trait | غير موجود | 🔴 |
| API Routes | route واحد (closure) | 🔴 |
| Feature Tests | scaffold فقط | 🔴 |
| SOLID (Application Layer) | غير مُطبَّق | 🔴 |

**الخلاصة:** المشروع في مرحلة **domain scaffold** قوية — migrations، models، enums، و Docker setup ممتازة. لكن **طبقة الـ Application غير موجودة**: لا controllers، لا actions، لا repositories، لا API endpoints (عدا `/api/user` closure). الـ README يذكر مبادئ (Service Layer, Form Requests, API Resources) لكن **لم تُنفَّذ بعد**. الخطوة التالية: بناء الهيكل المعماري **قبل** إضافة endpoints.

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

### الوضع الحالي vs المستهدف

```
المستهدف:  FormRequest → Controller → Action → Service → RepositoryInterface → Model
الحالي:    (لا يوجد) ──────────────────────────────────────────────→ Model فقط
           migrations + models + enums — بدون HTTP layer
```

**ملاحظة إيجابية:** عدم وجود fat controllers يعني **لا technical debt** في طبقة الـ HTTP — فرصة ممتازة لبناء المعمارية صح من البداية.

---

## 3. تحليل SOLID

### S — Single Responsibility Principle

| الطبقة | التقييم |
|--------|---------|
| Models | ✅ جيد — relationships + casts فقط |
| Enums | ✅ مسؤولية واحدة لكل enum |
| Migrations | ✅ منفصلة per table |
| Application layer | ❌ غير موجود للتقييم |

### O — Open/Closed Principle

- لا interfaces — أي توسيع (cache layer, mock repos) يتطلب تعديل مباشر على Eloquent.
- **المطلوب:** Repository interfaces من البداية.

### L — Liskov Substitution Principle

- غير قابل للتطبيق — لا abstractions بعد.

### I — Interface Segregation Principle

- غير قابل للتطبيق — لا interfaces.

### D — Dependency Inversion Principle

| الوضع الحالي | المطلوب |
|--------------|---------|
| `AppServiceProvider::register()` فارغ | يربط كل Interface → Implementation |
| لا controllers بعد | عند الإنشاء: تعتمد على interfaces مش models مباشرة |

---

## 4. مراجعة ملف بملف

### 4.1 Controllers — 🔴 غير موجود (feature controllers)

**الموجود:** `app/Http/Controllers/Controller.php` فقط (abstract base فارغ).

```1:8:app/Http/Controllers/Controller.php
<?php

namespace App\Http\Controllers;

abstract class Controller
{
    //
}
```

**إيجابية:** لا fat controllers — slate نظيف ✅

**المطلوب قبل أي endpoint:**

```
app/Http/Controllers/Api/V1/
├── Auth/
│   ├── LoginController.php
│   └── RegisterController.php
├── ProductController.php
├── CategoryController.php
├── CartController.php
├── OrderController.php
└── ReviewController.php
```

كل controller يستخدم `ApiResponse` trait ويستدعي Actions فقط.

---

### 4.2 Routes — 🔴 شبه فارغ

```1:8:routes/api.php
<?php

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

| المشكلة | التفاصيل |
|---------|----------|
| Closure route | بدل thin controller |
| Raw model response | يرجع User model مباشرة بدون Resource |
| No versioning | لا `v1` prefix |
| No route groups | لا تجميع auth/customer/admin/rider |
| No ApiResponse | response format غير موحّد |

**المستهدف:**

```php
Route::prefix('v1')->group(function () {
    Route::prefix('auth')->group(function () {
        Route::post('register', [RegisterController::class, 'store']);
        Route::post('login', [LoginController::class, 'store']);
    });

    Route::middleware(['auth:sanctum', 'role:customer'])->group(function () {
        Route::apiResource('products', ProductController::class)->only(['index', 'show']);
        Route::apiResource('cart', CartController::class);
        Route::apiResource('orders', OrderController::class);
    });

    Route::middleware(['auth:sanctum', 'role:admin'])->prefix('admin')->group(function () {
        Route::apiResource('products', Admin\ProductController::class);
        Route::apiResource('categories', CategoryController::class);
    });
});
```

---

### 4.3 Models — ✅ جيد جدًا

#### `User` — ✅

```16:85:app/Models/User.php
class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable, SoftDeletes, HasRoles;
    // fillable, casts, relationships: favorites, cartItems, orders, deliveries, reviews
}
```

**إيجابيات:**
- Sanctum (`HasApiTokens`) + Spatie Permission (`HasRoles`) ✅
- Soft deletes ✅
- Relationships كاملة ✅
- `'password' => 'hashed'` cast ✅

**مشاكل بسيطة:**
- Unused import `BelongsToMany` (سطر 6)

---

#### `Product` — ✅

- fillable شامل (nutrition fields, stock, availability)
- `belongsToMany` categories
- casts صحيحة (`decimal:2`, `boolean`)
- relationships: favorites, cartItems, orderItems, reviews

**ناقص:** `scopeAvailable()` scope للـ queries الشائعة.

---

#### `Order` — ✅ ممتاز

```34:44:app/Models/Order.php
    protected function casts(): array
    {
        return [
            'status' => OrderStatus::class,
            'payment_status' => PaymentStatus::class,
            'payment_method' => PaymentMethod::class,
            // ...
        ];
    }
```

- Enum casts — type safety ممتاز ✅
- Relationships: user, rider, items, payment ✅
- Soft deletes ✅

**ملاحظة:** `payment_status` و `payment_method` موجودة على `orders` **و** `payments` table — denormalization مقبول لكن checkout logic لازم يحافظ على sync.

---

#### `Category` — 🟡

```10:12:app/Models/Category.php
class Category extends Model
{
use HasFactory, SoftDeletes;
```

- **Indentation bug:** `use HasFactory` بدون indent (سطر 12)
- باقي الـ model صحيح

---

#### Models أخرى

| Model | الحالة | ملاحظات |
|-------|--------|---------|
| `CartItem` | ✅ | relationships موجودة |
| `Favorite` | ✅ | unique constraint في migration |
| `OrderItem` | ✅ | — |
| `Payment` | ✅ | enum casts |
| `Review` | ✅ | rating + product + user |

**ناقص عام:** factories لمعظم models (عدا User).

---

### 4.4 Enums — ✅ ممتاز

```5:18:app/Enums/OrderStatus.php
enum OrderStatus:string
{
    case Pending = 'pending';
    case Confirmed = 'confirmed';
    case Preparing = 'preparing';
    case OnTheWay = 'on_the_way';
    case Delivered = 'delivered';
    case Cancelled = 'cancelled';
}
```

- Lifecycle كامل للـ order ✅
- `PaymentStatus`: pending, paid, failed ✅
- `PaymentMethod`: card, apple_pay ✅

**تحسين بسيط:** `enum OrderStatus:string` → `enum OrderStatus: string` (مسافة بعد colon — Pint fix).

---

### 4.5 Database / Migrations — ✅ ممتاز

| Migration | الحالة |
|-----------|--------|
| `users` | ✅ phone unique, soft deletes, geo fields |
| `categories` | ✅ |
| `products` | ✅ nutrition + stock + availability |
| `category_product` | ✅ pivot table |
| `favorites` | ✅ unique(user_id, product_id) |
| `cart_items` | ✅ unique(user_id, product_id) |
| `orders` | ✅ status enums, payment fields, rider |
| `order_items` | ✅ |
| `payments` | ✅ |
| `reviews` | ✅ unique(user_id, product_id) |
| `permission_tables` | ✅ Spatie |
| `personal_access_tokens` | ✅ Sanctum |

**إيجابيات:**
- Foreign keys + indexes ✅
- Unique constraints على cart/favorites/reviews ✅
- Soft deletes على entities الرئيسية ✅

---

### 4.6 Form Requests — 🔴 غير موجود

المجلد `app/Http/Requests/` **غير موجود**.

README يذكر "Form Requests Validation" كمبدأ — لم يُنفَّذ.

**مطلوب لكل module:**

```
app/Http/Requests/
├── Auth/
│   ├── RegisterRequest.php
│   └── LoginRequest.php
├── Product/
│   ├── StoreProductRequest.php
│   └── UpdateProductRequest.php
├── Cart/
│   └── AddToCartRequest.php
└── Order/
    └── PlaceOrderRequest.php
```

كل Request يحتوي `toDto()` method.

---

### 4.7 Actions / Services / Repositories — 🔴 غير موجود

| المجلد | الحالة |
|--------|--------|
| `app/Actions/` | ❌ |
| `app/Services/` | ❌ |
| `app/Repositories/` | ❌ |
| `app/Contracts/` | ❌ |
| `app/DTOs/` | ❌ |

**المطلوب لكل domain entity:**

```
Product/
├── ListProductsAction.php
├── StoreProductAction.php
├── ProductRepositoryInterface.php
├── ProductRepository.php
└── ProductData.php (DTO)
```

---

### 4.8 `AppServiceProvider` — 🔴 فارغ

```12:23:app/Providers/AppServiceProvider.php
    public function register(): void
    {
        //
    }
```

**المطلوب عند بناء الـ layers:**

```php
public function register(): void
{
    $this->app->bind(UserRepositoryInterface::class, UserRepository::class);
    $this->app->bind(ProductRepositoryInterface::class, ProductRepository::class);
    $this->app->bind(OrderRepositoryInterface::class, OrderRepository::class);
    // ...
}
```

---

### 4.9 Middleware — 🔴 غير مُعدّ

```14:16:bootstrap/app.php
    ->withMiddleware(function (Middleware $middleware): void {
        //
    })
```

**Spatie Permission مُثبَّت لكن middleware aliases غير مسجّلة.**

**المطلوب:**

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->alias([
        'role' => \Spatie\Permission\Middleware\RoleMiddleware::class,
        'permission' => \Spatie\Permission\Middleware\PermissionMiddleware::class,
        'role_or_permission' => \Spatie\Permission\Middleware\RoleOrPermissionMiddleware::class,
    ]);
})
```

**مفقود أيضًا:**
- JSON exception rendering لـ `api/*`
- Rate limiting على auth routes
- Custom middleware (e.g. `EnsurePhoneVerified`)

---

### 4.10 Seeders & Factories — 🟡 مشاكل

#### 🐛 `UserFactory` — Bug حرج (P0)

```27:33:database/factories/UserFactory.php
        return [
            'name' => fake()->name(),
            'email' => fake()->unique()->safeEmail(),
            // ❌ phone مفقود — مطلوب و unique في migration
            'password' => static::$password ??= Hash::make('password'),
        ];
```

Migration يتطلب `phone` unique (سطر 17) لكن Factory لا يُولّده.

`DatabaseSeeder` (سطر 23–26) ينشئ user بدون `phone` → **سيفشل `php artisan db:seed`**.

**الإصلاح:**

```php
return [
    'name' => fake()->name(),
    'phone' => '01' . fake()->numerify('##########'),
    'email' => fake()->unique()->safeEmail(),
    'password' => static::$password ??= Hash::make('password'),
];
```

#### `RolesSeeder` — 🟡

- ينشئ roles (admin, customer, rider) ✅
- لكن لا permissions ولا role assignment لـ test user

---

### 4.11 Docker — ✅ جيد

- PHP + Nginx + MySQL + phpMyAdmin
- `docker-compose.yml` جاهز للتطوير

**تناقض:** README يذكر Redis لكن:
- لا Redis service في `docker-compose.yml`
- `.env.example` default = sqlite (مش MySQL زي Docker)

---

### 4.12 Tests — 🔴

فقط scaffold (`ExampleTest.php`). لا feature tests.

---

## 5. README vs الواقع

| README يذكر | الواقع |
|-------------|--------|
| RESTful API for menus, orders | route واحد فقط |
| Service Layer | `app/Services/` غير موجود |
| Form Requests Validation | `app/Http/Requests/` غير موجود |
| API Resources | `app/Http/Resources/` غير موجود |
| Redis | غير مُعدّ في Docker |
| MySQL | Docker ✅ لكن `.env.example` = sqlite |
| User Authentication | غير مُنفَّذ |
| Role & Permission Management | roles seeded فقط — لا API |

---

## 6. ما هو جيد ✅

1. **Domain schema ممتاز** — migrations تغطي المشروع بالكامل (menu, cart, orders, payments, reviews, roles).
2. **Eloquent models** — relationships و enum casts مدروسة.
3. **Package choices** — Sanctum + Spatie Permission مناسبة للمشروع.
4. **Docker setup** — بيئة تطوير كاملة.
5. **Enums** — OrderStatus lifecycle كامل، Payment enums.
6. **Clean slate** — لا technical debt في controllers (لأنها غير موجودة).
7. **Soft deletes** — على entities الرئيسية.
8. **Unique constraints** — cart, favorites, reviews محمية من التكرار.
9. **Larastan + Pint** — في dev dependencies (جودة كود).

---

## 7. خطة التنفيذ

### المرحلة 0 — إصلاحات فورية (P0) 🔴

- [ ] إصلاح `UserFactory` — إضافة `phone` field
- [ ] إصلاح `DatabaseSeeder` — test user مع `phone`
- [ ] إصلاح indentation في `Category.php`
- [ ] تسجيل Spatie middleware aliases في `bootstrap/app.php`
- [ ] إضافة JSON exception rendering لـ `api/*`

### المرحلة 1 — Architecture Skeleton (P1) 🟡

**ابنِ الهيكل قبل أي endpoint:**

- [ ] `app/Traits/ApiResponse.php`
- [ ] `app/Contracts/Repositories/` — interfaces لكل entity
- [ ] `app/Repositories/Eloquent/` — implementations
- [ ] `app/DTOs/` — readonly classes
- [ ] `app/Actions/` — single-responsibility actions
- [ ] `app/Http/Requests/` — validation + `toDto()`
- [ ] `app/Http/Resources/` — API output transformation
- [ ] Bind interfaces في `AppServiceProvider`
- [ ] Base `Controller` يستخدم `ApiResponse`

### المرحلة 2 — Auth Module (P1) 🟡

- [ ] `RegisterAction`, `LoginAction`, `LogoutAction`
- [ ] `UserRepositoryInterface` + implementation
- [ ] `RegisterRequest`, `LoginRequest` مع phone validation
- [ ] `AuthController` thin
- [ ] Role assignment عند التسجيل (customer role)
- [ ] Feature tests

### المرحلة 3 — First Vertical Slice (P2) 🟢

ترتيب الأولوية:

| # | Module | السبب |
|---|--------|-------|
| 1 | Category | بسيط — CRUD |
| 2 | Product | يعتمد على Category |
| 3 | Favorite | يعتمد على Product + User |
| 4 | Cart | business logic (quantities) |
| 5 | Order | أعقد — transactions, status workflow |
| 6 | Payment | تكامل خارجي |
| 7 | Review | يعتمد على Order/Product |

### المرحلة 4 — Quality (P3) 🟢

- [ ] Factories لكل models
- [ ] Permission seeder (مش roles فقط)
- [ ] Align `.env.example` مع Docker MySQL
- [ ] Redis service (لو مطلوب)
- [ ] Feature tests لكل endpoint
- [ ] `APP_NAME=Online Restaurant` في `.env.example`

---

## 8. Checklist لكل Endpoint جديد

```
□ Form Request (validation + authorize + toDto)
□ DTO class
□ Action class (business logic)
□ Repository Interface + Implementation
□ API Resource (response transformation)
□ ApiResponse method
□ Controller method (سطرين: execute + respond)
□ Feature Test
□ Route registration (versioned + middleware)
□ Permission/Role check
```

---

## 9. Controller مثالي (Target)

```php
<?php

namespace App\Http\Controllers\Api\V1;

use App\Actions\Product\ListProductsAction;
use App\Http\Controllers\Controller;
use App\Http\Requests\Product\ListProductsRequest;
use App\Traits\ApiResponse;
use Illuminate\Http\JsonResponse;

final class ProductController extends Controller
{
    use ApiResponse;

    public function index(ListProductsRequest $request, ListProductsAction $action): JsonResponse
    {
        return $this->paginatedResponse($action->execute($request->toDto()));
    }
}
```

---

## 10. File-by-File Scorecard

| File / Area | Models | Migrations | Application Layer | Verdict |
|-------------|:-:|:-:|:-:|---------|
| `User` | ✅ | ✅ | ❌ | 🟡 Domain OK |
| `Product` | ✅ | ✅ | ❌ | 🟡 Domain OK |
| `Order` | ✅ | ✅ | ❌ | 🟡 Domain OK |
| `Category` | 🟡 indent | ✅ | ❌ | 🟡 Minor fix |
| Enums | ✅ | — | — | ✅ Good |
| Controllers | — | — | ❌ | 🔴 Missing |
| Form Requests | — | — | ❌ | 🔴 Missing |
| Actions/Services | — | — | ❌ | 🔴 Missing |
| Repositories | — | — | ❌ | 🔴 Missing |
| `ApiResponse` | — | — | ❌ | 🔴 Missing |
| `routes/api.php` | — | — | 🔴 closure | 🔴 Fail |
| `UserFactory` | — | — | 🐛 no phone | 🔴 Bug |
| `AppServiceProvider` | — | — | ❌ empty | 🔴 Fail |
| Docker | — | — | ✅ | ✅ Good |
| Tests | — | — | ❌ | 🔴 Missing |

---

## 11. ملخص الأولويات (Action Items)

### 🔴 Critical (اعملها فورًا)

1. إصلاح **`UserFactory`** — `phone` field مفقود (يفشل seeding)
2. إصلاح **`DatabaseSeeder`** — test user بدون phone
3. تسجيل **Spatie middleware** aliases
4. إضافة **JSON exception handling** لـ API routes

### 🟡 High (قبل بناء endpoints)

5. بناء **Architecture skeleton** (ApiResponse, Repositories, Actions, DTOs, Requests)
6. **Auth module** كأول vertical slice
7. **Route versioning** (`v1`) + middleware groups per role
8. Bind **interfaces** في `AppServiceProvider`
9. إصلاح **Category.php** indentation

### 🟢 Medium (أثناء التطوير)

10. Factories لكل models
11. Permission seeder (مش roles فقط)
12. Feature tests
13. Align `.env.example` مع Docker
14. API Resources لكل entity
15. README update ليعكس الواقع

---

## 12. مقارنة مع مشاريع أخرى في الـ Bootcamp

| المعيار | Elham (FoodIfy) | Abdella (Foodify) | Osama (Online Restaurant) |
|---------|:-:|:-:|:-:|
| Auth مُنفَّذ | جزئي + Actions | functional prototype | ❌ |
| Models | scaffold | User فقط | ✅ 8 models كاملة |
| Migrations | جزئي | users فقط | ✅ domain كامل |
| Actions layer | موجود | ❌ | ❌ |
| Repositories | بدون interface | ❌ | ❌ |
| Docker | ❌ | ❌ | ✅ |
| Enums | ❌ | ❌ | ✅ |
| Spatie Permission | ❌ | ❌ | ✅ |

**نقطة قوة Osama:** أفضل domain layer في الـ bootcamp — schema و models جاهزة للبناء عليها.  
**نقطة ضعف:** لا application layer بعد — يحتاج بناء الهيكل المعماري قبل features.

---

## 13. تقرير Feature Completeness — النواقص في الـ Application

> **مرجع المتطلبات:** Authentication, Profile, Cart, My Orders, Notifications, Favorites, Products/Categories, Reset Password, Category Details, Product Details, Settings, Payments/Checkout.

**Overall Feature Completeness: ~19%**

### 13.1 Summary

| Layer | Score |
|-------|-------|
| Database schema + models | ~65% |
| Routes / controllers | ~3% |
| **Overall features** | **~19%** |
---

## 14. المراجع

- [Laravel Form Requests](https://laravel.com/docs/validation#form-request-validation)
- [Laravel Sanctum](https://laravel.com/docs/sanctum)
- [Spatie Permission](https://spatie.be/docs/laravel-permission)
- مراجعة Elham: [elham/FoodIfy/CODE_REVIEW.md](../../elham/FoodIfy/CODE_REVIEW.md)
- مراجعة Abdella: [abdella/FoodIfy/CODE_REVIEW.md](../../abdella/FoodIfy/CODE_REVIEW.md)

---

*هذا الملف مرجع حي — حدّثه مع كل module جديد يتنفّذ.*
