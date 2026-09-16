# WISECP Codebase Guide

**Audience:** an AI agent (or developer) asked to build or change something in a WiseCP installation — an addon module, an API endpoint, a theme, a cron job, a core patch — **without scanning the codebase first**.

**Source of truth:** this document was compiled by reading the actual WiseCP **v5 (5.0-beta.1)** source tree (`/wisecp v5/`). Everything marked **[verified]** was confirmed by reading the corresponding core file. Items marked **[doc]** come from the older `module guide.md` / `theme guide.md` in this workspace and may lag v5. Where the guides and v5 disagree, **trust this file and the source**.

Companion docs in this workspace:

| File | Covers |
| --- | --- |
| `WISECP-CODEBASE-GUIDE.md` (this file) | Platform architecture, DB schema, core APIs — the v5 ground truth |
| `module guide.md` | Older official module docs — conceptually useful, but v5 differs in several places (flagged below) |
| `theme guide.md` | Front-end/theme conventions for this workspace's `WStyle` theme |
| `ProductCatalog/` | A complete, working v5 addon module — use it as a skeleton |

A live v5 source tree is available at `/Users/tahmied/Web-Development/others/wisecp v5/` — read it when this guide doesn't answer something. **But note:** many core files are ionCube-encoded and unreadable (see §2.3).

---

## 1. What WiseCP is

WiseCP is a hosting billing & automation platform (WHMCS competitor): clients, orders, services (hosting accounts, servers, domains), invoices, support tickets, a client area, an admin panel, a REST API, a module system (server provisioning, payment gateways, registrars, addons, …), a theme system, and a cron system.

- **PHP 8+** (the core uses `match`, arrow fns, typed properties).
- Core business logic is **ionCube-encrypted**; the extension surface (classes you build on, models, API resources, controllers, operations, templates) is **plain readable PHP**.
- One installation serves: the public **website** (cart/catalog), the **client area**, and the **admin panel**, plus **cron jobs** and the **REST API** (`/api/v1/...`).

## 2. Repository layout

### 2.1 Top level [verified]

```
wisecp/
├── index.php          # the single HTTP entry point (front controller)
├── bootstrap.php      # boots the core (constants, autoload, init)
├── cronjobs.php       # CLI/web cron entry point
├── coremio/           # THE application folder ("core")
├── templates/         # admin panel theme, website theme, system templates, notifications
├── resources/         # public assets served to the browser
└── temp/, backups/
```

### 2.2 `coremio/` — the application [verified]

| Path | What lives there |
| --- | --- |
| `api/` | The REST API subsystem: `Kernel.php`, `Core/` (Request, Response, Router, Routes, ClientRoutes, ModuleRoutes, ApiException, Idempotency, DocLinks), `Auth/` (Authenticator, RateLimiter, AbuseGuard, Scope), `Resources/` (Admin/, Client/, System.php, `_Resource.php`) |
| `classes/` | Core classes, one per file: `Modules.php`, `AddonModule.php`, `Database.php` (the WDB query builder), `WDB.php` (static proxy), `Cache.php`, `Config.php`, `Hook.php` *(ionCube)*, `Crypt.php`, `Filter.php`, `Models.php`, `Language.php`, `Theme.php`, `TemplateEngine.php`, `ModuleAdminArea.php`, `ModulesAdminFormBuilder`/form builder, `Logger.php`, `Session.php`, `Auth.php`, … |
| `models/` | Data access classes grouped by area: `models/admin/*.php`, `models/website/*.php`, `models/system/`. Namespace `WISECP\models\{area}`. Each extends base `Models` (`classes/Models.php`) which provides `$this->db` (a builder) and `$this->pfx` (table prefix). **This is where correct SQL lives — always mirror a model query before writing your own.** |
| `controllers/` | Page controllers: `controllers/admin/`, `controllers/website/`, `controllers/system/`. One file per page group (`products.php`, `cart.php`, …). |
| `operations/` | Panel POST action handlers (`AdminProducts.php`, `ClientTickets.php`, …). The templates' forms post `operation=<name>`; the operation file implements the action, reads input with `Filter::init()`, emits JSON envelopes, and fires hooks. |
| `helpers/` | Static helper classes, one per file: `products.php` (class `Products`), `money.php` (`Money`), `cart.php`, `invoices.php`, `orders.php`, `services.php`, `license.php` *(ionCube)*, `menus.php`, `events.php`, `modulequeue.php`, … Global-namespace statics, used everywhere as `\Products::…`, `\Money::…`. |
| `modules/` | The module system: one subdirectory per type (`Addons/`, `Servers/`, `Payment/`, `Registrars/`, `Product/`, `SMS/`, `Mail/`, `Authentication/`, `Captcha/`, `Currency/`, `Fraud/`, `IP/`, `Imports/`, `Pipe/`, `SocialAuth/`, `Storage/`), one folder per module inside. |
| `configuration/` | PHP files that each `return` an array — the platform's settings store: `general.php`, `database.php`, `crypt.php`, `options.php`, `api-actions.php`, `client-api-actions.php`, `modules.php`, `privileges.php`, `routes.php`, `theme.php`, `sms.php`, `notifications.php`, `pictures.php`, `cronjobs.php`, `debug.php`, `constants.php`, `contact.php`, `blocked_ips.php`, `info.php`. Read with `Config::get("file/key")`, written by the panel via `Config::set()/save()`. |
| `locale/` | Translations: `locale/{en,tr,de,fr}/…` PHP arrays. Read with `Language::g("path/to/key")` (dot paths, no `.php`). |
| `hooks/` | Global hook listener files (independent of modules). |
| `cronjobs/` | Cron job classes (`*Discover.php` / `*Execute.php` pairs + `CronJobHandler.php`). |
| `components/` | Reusable UI components (e.g. `Tab`). |
| `storage/`, `vendor/`, `UPDATE`, `VERSION`, `errlog.php`, `init.php`, `pipe.php`, `restore.php` | Support files. `init.php` wires constants (`CORE_FOLDER`, `MODULE_DIR`, `TEMPLATE_DIR`, `CACHE_DIR`, `STORAGE_DIR`, `MODEL_DIR`, `DS`, …) and autoloads `coremio/classes` + `helpers`. |

### 2.3 What is ionCube-encoded (do NOT try to read it) [verified]

`Hook.php`, `helpers/license.php` (`License`), and the **main class files of all shipped addons** (`modules/Addons/*/X.php`). Encoded files start with `<?php //003fe` and an ionCube loader notice.

Consequences:

- `Hook::add/run/runRefs` and `License::api_resource/has_*` are **contracts, not readable code** — this guide documents their observed behavior.
- To learn hook/route conventions, read the **readable** parts of shipped modules: their `hooks.php`, `config.php`, `manifest.json`, and `src/` classes are plain PHP (e.g. `modules/Addons/WChat/hooks.php`).
- Everything else listed in §2.2 (`Modules.php`, `Database.php`, `Cache.php`, `Config.php`, `AddonModule.php`, models, API resources, controllers, operations, templates) is readable — read those freely.

## 3. Request lifecycle [verified at the entry points]

1. `index.php` hardens the session (secure/Httponly cookie, optional cookie domain), determines the path from `$_GET['route']` or `REQUEST_URI`, then boots `bootstrap.php` → `coremio/init.php` (constants, class/helper autoload, configuration, session, language).
2. The router maps the path to a **controller** in `controllers/{admin|website|system}/`. Admin routes require an admin session; website routes are public/client.
3. A page controller renders through the **theme** (`templates/website/{theme}` for the client side, `templates/admin` for the panel) using the template engine; it may call **models** for data.
4. Forms/JSON actions post `operation=<name>`; the matching file in `coremio/operations/` executes the change, guards privileges (`Admin::isPrivilege`, or the operation's own checks), writes via models, fires hooks, and returns a JSON envelope (`{"status":"successful","message":…,"data":…}` or an error).
5. Paths starting `/api/v1/…` bypass the page pipeline and are handed to `WISECP\Api\Kernel::handle()` (see §6).
6. Long-running work is queued through **cron** (`cronjobs.php` → `coremio/cronjobs/*`, plus `CronJobQueue` for module jobs).

## 4. Database layer

### 4.1 WDB — the query builder [verified from `classes/Database.php` + `classes/WDB.php`]

`WDB` is a static proxy to a singleton builder (`Database`). Core code (models, helpers) uses exactly these forms:

```php
// SELECT — build() executes and returns bool (rows > 0); fetch on the SAME instance.
$stmt = WDB::select('t1.*, t2.title')          // column list string; WDB::select() = '*'
    ->from('products AS t1')
    ->join('LEFT', 'products_lang AS t2', "t2.owner_id=t1.id AND t2.lang='en'")
    ->where('t1.status', '=', 'active')        // ->where($column, $mark, $value, $logical = '')
    ->where('t1.visibility', '=', 'visible')
    ->whereGroup(function ($q) use ($cat) {    // grouped (…) conditions
        $q->where('t1.category', '=', $cat, '||');
        $q->where("FIND_IN_SET('{$cat}', t1.categories)", '', '');
    }, '&&')
    ->where("(t2.id IS NOT NULL)", '', '', '&&')   // raw SQL condition: empty $mark
    ->group_by('t1.id')
    ->order_by('t1.rank ASC, t1.id ASC')       // ONE string argument
    ->limit($start, $end);                     // limit(offset_or_count, optional_count)

$rows = $stmt->build() ? $stmt->fetch_assoc() : [];   // ALL rows (PDO::FETCH_ASSOC)
$row  = $stmt->build() ? $stmt->getAssoc()  : [];     // FIRST row
$n    = $stmt->build() ? $stmt->rowCounter() : 0;     // row count

// WRITE
WDB::insert('table', $data);                       // then WDB::lastID()
WDB::update('table', $data)->where('id', '=', $id)->save();
WDB::delete('table')->where('id', '=', $id)->run();

// RAW / discovery
WDB::query('SHOW COLUMNS FROM `table`');           // PDOStatement | false
WDB::exec('ALTER TABLE …');                        // affected rows
WDB::hasTable('table');                            // bool
```

Rules that bite:

- `where()` chaining: the 4th argument is the **join operator to the previous condition** (`'&&'`, `'||'`; empty = AND). Raw conditions use an empty `$mark`.
- `order_by()` takes a single `"col DIR, col2 DIR"` string — there is no two-argument `orderBy()`.
- `build()` **throws `DatabaseException`** on SQL errors — wrap module DB code in try/catch.
- The builder mutates shared singleton state; never interleave two half-built statements. Finish one (`build()` + fetch) before starting the next, or use `pushState()`/`popState()`.
- Inside models, prefix **subquery** table references with `$this->pfx` (the installation's table prefix). The builder handles prefixing for `from()/join()` targets, not for raw SQL strings.
- `build()` special-cases the column name `rank` (quotes/qualifies it) — `rank` is a reserved word in some MySQL setups.

### 4.2 Schema dictionary

Column lists below are **[verified]** — read from the models/API resources and (for `products`) confirmed against a live v5 database. Tables marked (partially) were only seen in queries; inspect `models/` before writing schema-dependent code.

**`products`** [verified — the catalog]

```
id, rank, type ('hosting'|'server'|'software'|'special'|'sms'|…), type_id,
category (int FK → categories.id), categories (csv of category ids),
group_type, group_id,
status ('active'|'inactive'), visibility ('visible'|'hidden'),
override_usrcurrency (0/1 — force the product's own price currency),
taxexempt, additional_tax (JSON string), upgrade (0/1),
options (JSON string: disk_limit, bandwidth_limit, free_domain, auto_install, popular, …),
affiliate_disable, affiliate_rate, upgradeable_products, addons, requirements,
module (module class name or 'none'), module_data (JSON string),
notes, subdomains, stock (null = unlimited, 0 = sold out),
prorate_enabled, prorate_days, recurring_cycles_limit_enabled, recurring_cycles_limit,
auto_terminate_enabled, auto_terminate_days, auto_terminate_enabled_at,
ctime (DATETIME string — created at)
```

**`products_lang`** — one row per language [verified]

```
id, owner_id (→ products.id), lang ('en','tr',…),
title, tagline, content, route (URL slug), features (plain text, \r\n lines),
options (JSON string), seo_title, seo_keywords, seo_description
```

**`categories` / `categories_lang`** [verified]

```
categories:       id, parent, type ('products'|…), kind ('hosting'|'server'|'special'|…),
                  kind_id, rank, status, visibility, options (JSON), ctime
categories_lang:  id, owner_id (→ categories.id), lang, title, sub_title, route,
                  content, faq (JSON), options (JSON), seo_title, seo_keywords, seo_description
```

**`prices`** — ALL money amounts for everything (products, domains, addons) [verified]

```
id, owner ('products'|'tld'|'addon'), owner_id (→ owner table's id),
type ('periodicals' = recurring, 'sale' = one-time; domains use
      'register'|'transfer'|'renewal'|'grace'|'redemption'),
status (1 = active), 
period ('hour'|'day'|'week'|'month'|'year'|'none'), time (multiplier: 1,3,6,12,24,36; 0/1+none = one-time),
amount, setup, promotion, promotion_status (1 = promotion applies),
cid (→ currencies.id), rank
```

Cycle resolution — use the core helper, never reimplement [verified, `helpers/products.php`]:

```php
Products::$cycles = ['1-hour'=>'hourly','1-day'=>'daily','1-week'=>'weekly',
    '1-month'=>'monthly','3-month'=>'quarterly','6-month'=>'semiannually',
    '1-year'=>'annually','2-year'=>'biennially','3-year'=>'triennially',
    '0-none'=>'onetime','1-none'=>'onetime'];

Products::cycle($time, $period);      // → 'monthly', 'annually', 'onetime', …
Products::price_suffix($cycle, $amount);  // localized '/mo', '/yr', …
```

**Effective price rule** [verified, `helpers/invoices.php` + `models/website/index.php`]: if `promotion_status == 1 && promotion > 0 && promotion < amount` → effective = `promotion`, else `amount`.

**`currencies`** [verified]

```
id, code ('USD'…), name, rate, status ('active'|'inactive'), local (1 = the default currency)
```

Default currency id: `Config::get('general/currency')`. Currency helpers: `Money::selected()` (visitor's currency id), `Money::exChange($amount, $fromCid, $toCid)`, `Money::currency_code($cid)`, `Money::formatter($amount, $cid, …)`, `Money::formatter_symbol($amount, $cid)` (symbol-formatted string).

**Other tables seen in core queries** (partially verified — read `models/` before use):

- `users` (clients), `users_products` (**services** ordered by clients: `type`, `type_id`, `product_id`, `status`, `period`, `period_time`, `renewaldate`, `options` JSON…)
- `invoices`, `invoices_items` (`amount`, `cid`, `status`, `datepaid`, `user_pid`…)
- `servers`, `servers_groups`, `products_addons`, `products_requirements`, `products_metrics`
- `tldlist`, `tldlist_docs` (domains), `privileges` (`id`, `name` — admin ACL), `events` (audit), `sms_logs`, `cookie_consent_log`

### 4.3 The pricing pipeline (how the website prices a plan) [verified, `helpers/products.php`]

`Products::catalog_plans($categoryId, $type, $layout)` → cached (3600 s) → `catalog_products()` (active+visible products of the category, localized via `products_lang`) → `catalog_prices_bulk()` (price rows where `owner='products' AND status=1 AND type='periodicals'`, `order_by rank`) → **`plan_price_map($productId, $override, $ucid, $rows)`**:

1. cycle = `Products::cycle(time, period)`; skip rows with `amount <= 0`.
2. If `products.override_usrcurrency == 1`: prices are shown **only** in the currency of the product's own price rows (`cid` of the first row) — no conversion.
3. Otherwise each row is **converted** from its `cid` to the selected currency: `Money::exChange($amount, $cid, $ucid)`.
4. Keys are the display cycles (`monthly`, `quarterly`, `semiannually`, `annual`, `biennial`, `triennial`, `onetime` — note the website shortens *annually/biennially/triennially*).

The `ProductCatalog` addon module in this workspace mirrors this pipeline 1:1 (and adds `promotion` resolution + `setup`) — read its `src/Catalog.php` before writing any product/pricing code.

## 5. Module system

### 5.1 Types and layout

Sixteen types = subdirectories of `coremio/modules/` (§2.2). Each module is a folder:

```
coremio/modules/{Type}/{Name}/
├── {Name}.php        # the class — REQUIRED. Class name MUST equal the folder name.
├── config.php        # returns the config array (settings live here; the panel rewrites it)
├── hooks.php         # included on EVERY request — hook registrations only
├── manifest.json     # optional; module-updater metadata only
├── lang/en.php       # 'name' + 'description' (+ any strings); one file per language
├── logo.png/svg
├── src/              # your own classes — NOT autoloaded; include_once before use
├── views/            # templates rendered via $this->view()
├── assets/           # css/js/images, public URL via $module->url
└── cronjobs/         # optional job classes (register via 'register:cronjobs')
```

### 5.2 Class resolution — the #1 gotcha [verified, `classes/Modules.php::getInstance`]

The loader tries, in order:

```php
$name . "_Module"                                   // legacy global
$name                                               // bare GLOBAL class
"WISECP\\Modules\\" . ucfirst($type) . "\\" . $name // ← write new modules this way
```

⚠️ The namespace is **`WISECP\Modules\{Type}` — WITHOUT the module name**. A class file `ProductCatalog.php` must declare `namespace WISECP\Modules\Addons;` + `class ProductCatalog`. Writing `namespace WISECP\Modules\Addons\ProductCatalog;` makes the FQCN `…\ProductCatalog\ProductCatalog`, matches nothing, `getInstance()` returns null, and every panel screen that builds the module dies with **HTTP 500 / "System error"** (this exact bug shipped once — learn from it). Your `src/` classes, by contrast, SHOULD carry the full sub-namespace (`WISECP\Modules\Addons\ProductCatalog\Src\…`) — they are `include_once`'d manually, not autoloaded.

Never `new` a module: the factory (`Modules::getInstance($type, $name, $params)`) includes the class file, loads config/lang into the instance, and caches one instance per request. `Modules::Load($type, $name, true)` = read config/lang only (no class include). `Modules::Config($type,$name)` reads the cache only (load first!).

### 5.3 Addon modules — the base class [verified, `classes/AddonModule.php`]

```php
namespace WISECP\Modules\Addons;
class MyAddon extends \AddonModule { … }
```

The constructor fills: `$this->_name`, `$this->dir` (filesystem, trailing DS), `$this->url` (public URL), `$this->area_link`, `$this->config` (whole config.php), `$this->lang`, `$this->admin`/`$this->user` (when sessions exist). Available members/methods:

- `isEnabled()` — reads `config['status']`.
- `fields()` — **declare it** to get a settings form. Returns field descriptors: `'key' => ['name' => label, 'type' => text|password|textarea|dropdown|approval|switch|output, 'value' => current, 'description' => …, 'options' => […], 'checked' => bool]`. The panel renders via `Modules::fields_output_wBuilder()` and saves through the base `save_settings($pFields, $accessPs)`, which normalizes posted values (approval → 0/1, options allow-listed) into `config['settings']` and rewrites `config.php`.
- `save_config($data)` — rewrites config.php (`FileManager::file_write` + `Utility::array_export(['pwith' => true])`). Always merge with `$this->config` first — it is a full-file write.
- `change_addon_status('enable'|'disable')` — the enable chain: `activate()` → `enable()` → write `status` (or `deactivate()` → `disable()`). Optional methods; return falsy to veto.
- `privileges()` — admin ACL rows; `config['access_ps']` stores granted ids.
- `view($file, $variables)` — renders `views/{file}`.
- `encode_str()/decode_str()` — symmetric encryption under `Config::get("crypt/system")` (addon default sub-key) — use for stored secrets.

`config.php` shape [verified against shipped addons]:

```php
return [
    'created_at' => 169…,
    'meta' => ['name' => 'Human Name', 'version' => '1.0', 'author' => '…',
               'logo' => 'logo.svg', 'icon' => 'bi bi-…', 'slug' => 'optional-slug'],
    'show_on_adminArea' => true, 'show_on_clientArea' => false,
    'status' => false,          // the enable flag; ships false
    'access_ps' => [],
    'settings' => [ 'key' => 'safe default', … ],   // every key your code reads
];
```

`lang/en.php`: `'name'` and `'description'` are what the addon list prints.

`hooks.php` rules: it is included on **every request, for every module on disk, enabled or not**. Keep it to `Modules::Load(...)` + `Modules::Config(...)` + `Hook::add(...)` registrations, gated on `if ($config['status'] ?? false)` for anything that should only run when enabled (route registration, cron registration, license checks — see WChat's hooks.php for the canonical pattern).

Other module types (Servers, Payment, Registrars, Product, …) extend their base classes (`ServerModule`, `PaymentGatewayModule`, `RegistrarModule`, `ProductModule`, …, all readable in `coremio/classes/`) and implement the provisioning/billing contract documented in `module guide.md` — that part of the old guide is still accurate.

## 6. The REST API (`/api/v1/...`)

### 6.1 Kernel flow [verified, `coremio/api/Kernel.php`, `Core/Request.php`, `Core/Router.php`]

```
request → Kernel::handle()
  → Request::capture(): strips 'api','v1', then 'admin'|'client' → audience + segments
  → OPTIONS? → Kernel answers preflight itself (204 + CORS) and stops
  → Router::match(): admin/client registries, or ModuleRoutes (free surface)
  → public route?  → optional token auth, then per-IP throttle
    non-public?    → Authenticator (API credential) → RateLimiter → scope check
                     (scope enforced only when tuple index 5 authOnly == false)
  → dispatch: group 'Module:{Type}/{Name}' → module instance → $obj->{'api_'.$action}($request, $match)
               core groups → License::api_resource() → $obj->{$action}($request->body)
  → Response::send() with CORS headers + Api::save_log()
```

Key facts:

- **Free (module) surface** = audience `'module'`, addresses `/api/v1/{pattern}` — no `/admin` or `/client` prefix.
- Route tuples come from the `filter:api.routes` hook, **fired per audience by reference** (`Hook::runRefs`); append, never return:

```php
// $routes[] = [method, pattern, group, action, public, authOnly?, audience?]
$routes[] = ['GET', 'products/catalog', 'Module:Addons/ProductCatalog', 'catalog', true];
// index 4 public=true → no credential required. Index 5 authOnly (default false) =
// skip the per-endpoint scope check when a credential IS presented.
// Index 6 (module surface only) = 'admin'(default)|'client'|'any'.
```

- The router is **first match wins at a given segment count** — declare literal paths (`products/catalog/categories`) **before** parametric ones (`products/catalog/{id}`).
- `$match['params']` is a **numeric array** of captured values (`$match['params'][0]`), not keyed by placeholder name.
- The handler method MUST exist as a real method (`method_exists` is checked) named `api_{action}`, signature `(Request $request, array $match): Response|array`.
- `Request` public fields [verified]: `method`, `originalMethod`, `audience`, `segments`, `resource`, `query` ($_GET minus route), `body` (JSON or form), `headers` (lower-cased keys), `ip`, `token`, `idempotencyKey`, `rawBody`.
- **CORS is global**: the Kernel sets `Access-Control-Allow-Origin` from `Config::get('options/api-cors-origins')` (empty or `*` = allow all) on every response, and answers preflight for all surfaces. **Modules must not send CORS headers themselves.**
- **Public rate limit**: `options/api-module-public-rate-limit-per-minute` (default 120/min per IP). Credentialed requests use `RateLimiter` with `X-RateLimit-*` headers.
- Errors: throw `ApiException` (factories `badRequest/unauthorized/forbidden/notFound/validation/rateLimited/server`) or return `Response::error($code, $message, $status)`. Success: `Response::success($data, 200, $meta)` → `{"data":…,"meta":…}`; payload is JSON-encoded by `Response::send()`.
- In-process calls to core endpoints: `Kernel::internal('Group/Action', $input, $query)` returns the payload array (or `['error'=>…]`); `client:` prefix + `owner_id` for client-surface endpoints. Module (`Module:`) endpoints are NOT reachable via `internal` — call the module class directly.

### 6.2 Registering endpoints — canonical pattern (from shipped `WChat`)

In the module's `hooks.php`:

```php
Modules::Load('Addons', 'MyAddon', true);
$cfg = Modules::Config('Addons', 'MyAddon') ?: [];

if ($cfg['status'] ?? false) {                      // only when enabled
    Hook::add('filter:api.routes', 1, function (&$routes, &$audience) {
        if ($audience !== 'module') return;         // free surface only
        $routes[] = ['GET',  'myaddon/thing',       'Module:Addons/MyAddon', 'thing_list', true];
        $routes[] = ['GET',  'myaddon/thing/{id}',  'Module:Addons/MyAddon', 'thing_detail', true];
    });
}
```

Handlers live on the module class; permission checkboxes for credentialed use are managed through `configuration/api-actions.php` (`Config::set('api-actions', [group => [actions]], true)` — in-memory publish from hooks body if needed).

## 7. Hook system

`Hook::add($name, $priority, callable)`, `Hook::run($name, …$args)` (listeners' returns collected — a non-empty string return from a `gate:` hook vetoes), `Hook::runRefs($name, &…$args)` (reference filters, e.g. route lists). `Hook.php` itself is ionCube — treat signatures as contract. Naming conventions observed in core:

| Prefix | Meaning | Verified examples |
| --- | --- | --- |
| `action:` | something happened | `action:product.created|updated|deleted|status_changed|bulk_action_applied`, `action:product.group_saved`, `action:product.server_updated|created|deleted|imported`, `action:module.addon_settings_saved`, `action:addon.status_changed` |
| `gate:` | veto point — return non-empty string to refuse | `gate:product.create`, `gate:product.delete`, `gate:product.group_delete`, `gate:product.server_delete`, `gate:module.activate/addon_delete/…` *(doc)* |
| `filter:` | modify data by reference / return | `filter:api.routes` (by ref), `filter:product.catalog_plans` (by ref) |
| `ui:` | inject HTML into layouts | `ui:client.head.css`, `ui:client.head.js`, `ui:client.body.end` (WChat) |
| `register:` | contribute registrations | `register:cronjobs` (WDNS: `CronJobQueue::register($class::TYPE, $class)`) |

Hook listeners in a module go in `hooks.php` (every request) and must be side-effect-free at registration time.

## 8. Caching [verified, `classes/Cache.php`]

File-based, under `CACHE_DIR`, entries stored as one encrypted JSON file per **cache name** (group). Global toggle: `Config::get("general/cache")` — when off, `Cache::remember` just invokes the producer (cache layer is safely bypassable).

```php
// THE API to use:
$value = Cache::remember(string $name, string $key, int $ttl, callable $producer);

// manual:
$c = new Cache('mygroup');           // or Cache::getInstance() for 'default'
$c->store($key, $data, $ttl);        // $ttl seconds (default 86400)
$c->retrieve($key);                  // null if absent/expired
$c->erase($key);                     // throws if the key doesn't exist
$c->eraseAll();                      // wipes the whole named file
Cache::getInstance()->clear('mygroup'); // wipes one named group
```

Core usage example: `Products::catalog_plans()` caches per category/type/layout/currency/language for 3600 s. Pattern for module payloads: cache name = module key (e.g. `'productcatalog'`), key = `md5(json_encode([$filters, Language::selected()]))` — **always include language** (and any currency/currency-display inputs) in the key. Invalidate on data changes by listening to the relevant `action:*` hooks and calling `Cache::getInstance()->clear('group')` (see `ProductCatalog/hooks.php`).

## 9. Themes & templates

- `templates/admin/` — panel theme; `templates/website/` — the active client-side theme; `templates/system/` — **shared system templates** (module settings screens, install, maintenance, error pages). Module settings render through `templates/system/module/addon-settings.php` [verified], which builds an `AdminFormBuilder` form: status checkbox, privileges grid, then your `fields()` descriptors via `Modules::fields_output_wBuilder()`.
- `templates/notifications/` — email/SMS templates.
- A website theme is a folder under `templates/website/` (`theme.php`, `config.php`, `hooks.php`, `views/` as `.tpl` templates, `assets/`, `locale/`, `components/`, `layouts/`, `partials/`, `tables/`). Front-end data conventions (plan cards, billing toggle, `Products::seed_plan_prices`) are documented in this workspace's `theme guide.md`.
- `Theme` class (`classes/Theme.php`) exposes theme settings (`Theme::price_monthly_equivalent()` etc.); theme configuration lives in `configuration/theme.php`.

## 10. Operations, input, sessions, cron

- **Operations** (`coremio/operations/*.php`) implement panel actions. Read input through `Filter::init("POST/name", "filter")` (filters: `route`, `rnumbers`, `text`, … — see `classes/Filter.php`); emit `{"status":"successful", …}` envelopes; throw `Exception` with a human message on failure; guard with `Admin::isPrivilege([...])`.
- **Sessions/identities**: `UserManager::LoginData('member'|'admin')`, `User::getData($id, $fields, 'array')`, `User::getInfo/AddInfo` (meta), `Admin::info()`, `Admin::isPrivilege([...])`. In API context, the actor is `ApiActor::adminId()`.
- **Cron**: `cronjobs.php` boots the cron runner; jobs are classes in `coremio/cronjobs/` (pattern: `*Discover` finds work, `*Execute` does it; `CronJobHandler.php` orchestrates; schedule config in `configuration/cronjobs.php`). Modules register jobs via the `register:cronjobs` hook + `CronJobQueue::register($type, $class)` (WDNS pattern), keeping job classes inside the module (`cronjobs/` subfolder).

## 11. Internationalization

- Platform strings: `locale/{lang}/…` arrays, read with `Language::g("path/to/key")` (slash dot-paths, e.g. `date/cycles/monthly`) and `Language::gc()` (with fallback) in panel code.
- Active language: `Language::selected()`; fallback language: `Language::primary()`.
- DB localization: `{table}_lang` rows keyed by `owner_id` + `lang`. Models use the `Language::localize($stmt, $langTable, $alias, $ownerColumn, $lang)` / `Language::localized(...)` JOIN helpers, then `COALESCE` between selected and primary language. Simpler equivalent used by modules: fetch all lang rows for the ids and group in PHP (see `ProductCatalog/src/Catalog.php`).

## 12. Utility classes you will constantly need [verified signatures in `coremio/`]

- `Utility::jdecode($json, true)` / `Utility::jencode($data)` — JSON with error tolerance.
- `FileManager::file_write($path, $data)` — writes AND invalidates any compiled/OPcache-cached copy. **Always write `config.php` through it** (never raw `file_put_contents`).
- `Utility::array_export($array, ['pwith' => true])` — array → `<?php return […]` file content.
- `Crypt::encode/decode($str, $key)`.
- `LinkGenerator::client($path, $params)` / `LinkGenerator::admin($path, $params)` — build correct front/controller URLs (respects SEF settings; usually returns absolute URLs).
- `Money::*` (see §4.2), `Products::*` (see §4.3), `Services::*`, `Invoices::*` (see `helpers/`).

## 13. Worked reference: the ProductCatalog addon

This workspace contains a complete, production-tested v5 addon that exercises most of the above:

- Public API on the free surface with **prices** (`/api/v1/products/catalog`), correct namespace + route tuples + `api_*` handlers.
- Settings via `fields()`; optional access-token guard; `isEnabled()` gating.
- Direct WDB reads mirroring the core models (products + `products_lang` + `prices` + `currencies` + `categories[_lang]`), promotion/currency logic copied from `Products::plan_price_map` and the invoice effective-price rule.
- 24 h `Cache::remember` caching with `action:product.*` invalidation hooks.

Read `ProductCatalog/` (especially `src/Catalog.php`) before writing anything that touches products, pricing, or the API.

## 14. Pitfall checklist (each of these has caused a real failure)

1. **Module class namespace**: `WISECP\Modules\Addons;` + `class ProductCatalog` — never include the module name in the namespace (→ 500 on enable/settings).
2. Never `new` a module — use `Modules::getInstance()`.
3. `src/` classes are not autoloaded — `include_once __DIR__ . DS . 'src' . DS . 'File.php';` before use.
4. Core class names inside a module namespace need a leading backslash (`\WDB`, `\Money`, `\Cache`) or a `use` import — otherwise PHP resolves them into your module namespace and fatals at runtime.
5. Route tuples: literal paths **before** `{param}` twins; `$match['params']` is **numeric**; register routes only while the addon is enabled.
6. Don't send CORS headers or handle OPTIONS in module code — the Kernel owns both.
7. `WDB::order_by()` takes one string; `build()` returns bool and **throws on SQL errors**; fetch immediately after building on the same instance.
8. Product prices: cycle = `Products::cycle(time, period)`; honor `override_usrcurrency`; promotion applies only when `promotion_status==1 && promotion>0 && promotion<amount`.
9. `config.php` writes go through `save_config()`/`FileManager::file_write` and must merge with the existing array.
10. Cache keys must include language (and currency) or bilingual/multi-currency installs will serve the wrong locale.
11. `hooks.php` runs for **every** module on **every** request — registration only, no work.
12. ionCube files (`Hook.php`, `helpers/license.php`, shipped addons' main classes) are contracts, not readable code — don't waste time trying to open them.

## 15. Where to look when extending something

| Task | Read first |
| --- | --- |
| New API endpoint | §6 + `modules/Addons/WChat/hooks.php` + `ProductCatalog/` |
| Panel list/table | `controllers/admin/*.php` + `templates/system/module/*` + `classes/Modules.php::fields_output*` |
| Product/pricing change | `models/admin/products.php`, `models/website/products.php`, `helpers/products.php` |
| Billing/invoice logic | `helpers/invoices.php`, `helpers/orders.php`, `models/admin/financial.php` |
| Client-area page | `controllers/website/*.php` + `theme guide.md` |
| Email/notification | `templates/notifications/` + `configuration/notifications.php` |
| New cron job | `coremio/cronjobs/CronJobHandler.php` + WDNS `hooks.php` pattern |
| Settings storage | `coremio/configuration/*.php` + `classes/Config.php` (`get/set/save`; module settings live in the module's own `config.php` instead) |
