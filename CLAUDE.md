# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**Active eCommerce CMS** — a Laravel 10 / PHP 8.2 multi-vendor e-commerce platform (ActiveITzone commercial CMS, not a vanilla Laravel app). It serves four distinct panels from one codebase: customer storefront, seller dashboard, admin backend, and a mobile-app REST API (`/api/v2/*`). Local environment is XAMPP (`C:\xampp\php`).

## Commands

```bash
php artisan serve                      # dev server (APP_URL is http://localhost:8000)
npm run watch                          # Laravel Mix watch (resources/js + resources/sass only)
npm run prod                           # production asset build
php artisan optimize:clear             # clear config/route/view caches — needed often, see Caching
php vendor/bin/phpunit                 # run the test suite
php vendor/bin/phpunit --filter=TestName tests/Feature/SomeTest.php   # single test
```

There is no lint/format tooling configured. `.editorconfig` sets 4-space indent for PHP, 2 for most other files.

### Testing reality check

`tests/` contains only the stock `ExampleTest.php` placeholders — there is no real test coverage. Don't assume a passing suite validates anything. Verify changes by exercising the actual routes in a browser.

### Database

**The schema does NOT come from migrations.** `database/migrations/` holds only 5 files (users, password_resets, personal_access_tokens, product_queries, permission_tables); the ~200 remaining tables come from `shop.sql` imported by the web installer. **Never run `php artisan migrate:fresh`** — it will destroy the schema. Schema changes in this codebase are made as raw SQL against the live DB (the addon/update system does the same via `DB::unprepared()`).

The installer and updater route groups are commented out in `app/Providers/RouteServiceProvider.php::map()` (`mapInstallRoutes`, `mapUpdateRoutes`) — uncomment to re-expose them.

## Architecture

### Document root is the project root, not `public/`

`index.php` at the repo root bootstraps `vendor/autoload.php` and `bootstrap/app.php` directly (shared-hosting layout). Consequently `my_asset()` prefixes paths with `public/` while `static_asset()` does not. Keep this distinction when writing views:

- `static_asset('assets/js/foo.js')` → bundled theme assets under `public/assets/` (css, js, img, fonts — shipped with the product, gitignored)
- `my_asset($path)` → user-uploaded files; transparently switches to S3 when `config('filesystems.default') != 'local'`
- `uploaded_asset($id)` → takes an **`uploads` table ID**, not a path. Uploaded files are rows in `uploads` and models store the integer ID (or comma-separated IDs) — see `AizUploadController` and the `resources/views/uploader/` modal.

### Settings live in the database, not config files

`get_setting('key')` reads the `business_settings` table (cached 24h under the `business_settings` key). This drives nearly all behavior, including **which frontend theme renders**:

```php
return view('frontend.' . get_setting('homepage_select') . '.index', ...);
```

Themes are sibling view directories under `resources/views/frontend/`: `classic`, `reclassic`, `thecore`, `megamart`, `metro`, `minima`. A storefront change usually has to be made in the active theme's folder — and replicated across themes if it should apply everywhere. Theme-agnostic frontend views sit directly in `resources/views/frontend/`.

`get_setting($key, $default, $lang)` also supports per-language setting values (`lang` column), falling back to the language-less row.

### Everything goes through `app/Http/Helpers.php`

A single ~3,200-line global helper file, autoloaded via composer `files`. Before writing any pricing, cart, product-fetch, or formatting logic, grep here — the function almost certainly exists. Key families:

- Pricing: `format_price`, `convert_price`, `home_price`, `home_discounted_price`, `home_base_price_by_stock_id`, `cart_product_price`, `cart_product_tax`, `cart_product_discount`
- Fetchers (all cached): `get_single_product`, `get_featured_products`, `get_todays_deal_products`, `get_single_category`, `get_user_cart`
- Checkout completion side effects: `checkout_done`, `order_re_payment_done`, `wallet_payment_done`, `product_restock`, `calculateCommissionAffilationClubPoint`
- Role checks: `isAdmin()`, `isSeller()`, `isCustomer()`
- Shipping: `getShippingCost`, `carrier_base_price`, `seller_base_carrier_list`

### Translation: DB-backed, self-populating

`translate('Some string')` looks up a normalized key in the `translations` table and **auto-inserts a missing English row on first call** (plus an `app_translations` row when not in demo mode). All user-facing strings in views and flash messages must be wrapped in `translate()`. Translations are cached with `rememberForever`, so adding/editing a language requires `Cache::forget('translations-{code}')` — `LanguageController` does this.

### Caching is manual and easy to get wrong

Long-lived caches that must be explicitly invalidated when you write to the underlying table:

| Cache key | Invalidate when |
|---|---|
| `business_settings` | any `business_settings` write |
| `addons` | addon install/activate (`AddonController`) |
| `translations-en`, `translations-{lang}` | translation or language change |
| `featured_categories`, `hot_categories` | category flags (`CategoryController`) |
| `verified_sellers_id` | seller verification (`SellerController`) |
| `todays_deal_products` | product deal flag (`ProductController`) |

Admin controllers additionally call `CoreComponentRepository::initializeCache()` / `instantiateShopRepository()` (from `mehedi-iitdu/core-component-repository`) on entry.

### Routing: 23 feature-split route files

`routes/web.php` is only part of the picture. `RouteServiceProvider::map()` explicitly registers each file: `admin.php`, `seller.php`, `api.php`, `api_seller.php`, `pos.php`, `auction.php`, `wholesale.php`, `preorder.php`, `affiliate.php`, `club_points.php`, `refund_request.php`, `delivery_boy.php`, `seller_package.php`, `otp.php`, `offline_payment.php`, plus per-gateway files (`paytm.php`, `african_pg.php`, `cybersource.php`). **When adding a route file you must add a `map*Routes()` method** — there is no auto-discovery. When hunting for a route, grep all of `routes/`.

Route group conventions:
- admin: `['prefix' => 'admin', 'middleware' => ['auth', 'admin', 'prevent-back-history']]`
- seller: `['prefix' => 'seller', 'middleware' => ['seller', 'verified', 'user', 'prevent-back-history'], 'as' => 'seller.']`

### Authorization: one `users` table, `user_type` column

`user_type` is `admin` | `staff` | `seller` | `customer` | `delivery_boy`, gated by the `IsAdmin` / `IsSeller` / `IsCustomer` / `IsUser` / `IsUnbanned` middleware in `app/Http/Kernel.php`.

Staff granularity uses **spatie/laravel-permission**, declared in **controller constructors**, not routes:

```php
$this->middleware(['permission:add_new_product'])->only('create');
```

New admin screens need a matching permission row and a check in the constructor, plus a guard in `resources/views/backend/inc/admin_sidenav.blade.php`.

### Models: translation tables + demo-mode trait

Most content models have a paired `*Translation` model (`ProductTranslation`, `CategoryTranslation`, …) and expose `getTranslation($field, $lang)` which falls back to the base column. Always render user-facing model text via `getTranslation()`, and write/update the translation row alongside the base row on save.

Nearly all models `use PreventDemoModeChanges`, a trait that overrides `save`/`create`/`update`/`delete`/`insert`/`truncate` to flash a warning and throw `RedirectingException` when `DEMO_MODE=On`. If writes mysteriously no-op, check `.env`.

### The cart → order domain model (multi-vendor split)

This is the single most important flow to understand, and it spans several files:

```
Cart rows (one per product, carries owner_id + variation)
  └─ grouped by product->user_id (the seller) in OrderController::store_order
       CombinedOrder  (one per checkout, holds JSON shipping_address + summed grand_total)
         └─ Order     (ONE PER SELLER — carries seller_id, carrier, pickup_point, coupon_discount)
              └─ OrderDetail  (per line item, also stamped with seller_id)
```

Consequences worth internalizing:

- A customer's single checkout produces **multiple `Order` rows**. "An order" is ambiguous — admin order lists filter on `orders.seller_id` (`== admin_id` for in-house, `!=` for seller orders), and seller views filter `orderDetails->where('seller_id', Auth::user()->id)`.
- `shipping_address` is **denormalized JSON** copied onto both `CombinedOrder` and every `Order`; it is not a relation to `addresses`.
- Totals are computed per-`Order` then summed into `CombinedOrder->grand_total`. Coupons apply per seller-order.
- `combined_order_id` is put in the session after order creation and is what the payment gateways and `checkout_done()` key off.

**Guest carts** use a session `temp_user_id` (random hex from `bin2hex(random_bytes(10))`) instead of `user_id`. On login, `CartController` claims the rows — sets `user_id`, nulls `temp_user_id`, forgets the session key. Any new cart query must handle both cases.

### Product variants

`products.variant_product` flags a variant product; stock lives in `product_stocks` rows keyed by a `variant` **string** built from the chosen attribute/color combination (`ProductUtility::get_combination_string`, combinations generated by `aiz-packages/combination-generate` via `ProductStockService`).

The variant form posts **dynamically named fields** that the service reads straight off the request, with `.` replaced by `_`:

```php
$product_stock->price = request()['price_' . str_replace('.', '_', $str)];   // also sku_, qty_, img_
```

So variant inputs are not in any FormRequest and won't show up by grepping for a fixed field name. Note the naming inconsistency: carts call it `variation`, product stocks call it `variant`. Price lookups for a specific stock row go through `home_base_price_by_stock_id()` / `home_discounted_base_price_by_stock_id()`.

### Service / utility layer

Thin but real: `app/Services/` (`ProductService`, `ProductStockService`, `ProductTaxService`, `ProductFlashDealService`, `OrderService`, OTP + social-revoke providers) is constructor-injected into controllers — `ProductController` takes five. `app/Utility/` holds static-style helpers (`CartUtility`, `CategoryUtility`, `EmailUtility`, `NotificationUtility`, `SearchUtility`, plus per-gateway utilities like `NagadUtility`, `PayhereUtility`, `NgeniusUtility`). Put new product/order/cart logic in these rather than growing `Helpers.php` or the controllers.

### Mobile API (`/api/v2/*`)

`routes/api.php` declares `namespace App\Http\Controllers\Api\V2` at the top (not a Laravel route namespace — it's a PHP namespace so bare `AuthController::class` resolves). Seller endpoints live in `routes/api_seller.php` → `Api\V2\Seller\*`. Auth is **Sanctum** tokens; responses are shaped by `app/Http/Resources/V2/`.

Two gotchas:
- The `api` middleware group includes `EnsureSystemKey`, which requires a `System-Key` request header matching `config('app.system_key')` (`SYSTEM_KEY` in `.env`, currently empty).
- Locale on API routes comes from the `app_language` middleware (`AppLanguage`), not the web `Language` middleware.

### Payment gateways

~30 gateways, each with its own controller in `app/Http/Controllers/Payment/` (web) and `app/Http/Controllers/Api/` (app), optional `config/*.php` (`nagad.php`, `rave.php`, `paystack.php`, `toyyibpay.php`, `mercadopago.php`, `flutterwave.php`, `laravel-payku.php`), redirect views under `resources/views/frontend/<gateway>/`, and env credentials. All converge on the `checkout_done()` / `order_re_payment_done()` / `wallet_payment_done()` helpers — **add a new gateway by routing its success callback into those**, not by reimplementing order finalization.

### Addon system

Paid features ship as zip addons (`AddonController::store`): the zip's `config.json` lists directories/files/folders to copy into the app tree plus a `sql/update.sql` run through `DB::unprepared()`, and an `addons` table row is created. Guard addon-dependent code with `addon_is_activated('unique_identifier')`. Because addons overwrite real application files, **edits to core files can be clobbered by an addon install or a CMS update** — prefer additive changes.

### Emails and notifications

Email bodies are **database rows, not Blade files**: `EmailUtility` static methods look up `EmailTemplate::whereIdentifier($identifier)` and render the stored template. Adding a transactional email means adding an `email_templates` row with a new identifier plus an `EmailUtility` method — editing a view in `resources/views/emails/` alone won't change what ships. `app/Mail/*MailManager` classes are the Mailable wrappers; `app/Notifications/` holds the Laravel notifications (`OrderNotification`, `DbNotification`, `ShopProductNotification`, FCM push via `FirebaseNotification`).

### Views

- `resources/views/backend/` — admin (layouts: `app`, `layout`, `blank`); nav/permission wiring in `backend/inc/`
- `resources/views/seller/` — seller dashboard
- `resources/views/frontend/<theme>/` — storefront themes; `frontend/layouts/app.blade.php` + `user_panel.blade.php`
- `resources/views/uploader/` — the AIZ file-manager modal reused everywhere a file field appears

Frontend behavior is jQuery + Bootstrap 4 driven by `public/assets/js/aiz-core.js` (not built by Mix). `webpack.mix.js` only compiles `resources/js/app.js` and `resources/sass/app.scss`, which are largely vestigial — most UI work happens in Blade plus `public/assets/`.

### Flash messages

Two mechanisms coexist: `laracasts/flash` (`flash(translate('...'))->success()`) in controllers, and the custom `flash_message($message, $level)` helper (used by `PreventDemoModeChanges`) which dedupes by message text.
