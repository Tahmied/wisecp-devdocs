# The Module System

https://dev.wisecp.com/en/the-module-system

Every integration is a module: a directory under a type folder. The platform discovers it by name, loads it on demand, and hands it to you through one factory.

## Overview

A module is never registered: no manifest to edit, no container entry, no install routine. The registry reads the file system. A directory whose name matches the class file inside it is a module the moment it exists on disk, and the panel lists it on the next request.

The directory one level up is the module's **type**, and the type is the whole contract. It decides which methods the core calls, which admin screen lists the module, and whether a base class exists to extend. Eight of the sixteen types have one. The other eight are plain classes whose contract is whatever the core looks for with `method_exists`.

- **coremio/modules**: The whole extension surface: one subdirectory per type, one per module inside it. 16 types, 300 modules, including the `Sample` sandbox modules that ship as templates.
- **Modules**: The registry and factory, all static. It scans directories, includes class files, caches configuration and language packs, and builds instances.
- **Module type**: The parent directory name, with its exact case (`Servers`, `Payment`, `Registrars`). It is a string in every registry call, so a typo gives a silent empty result rather than an error.
- **Module name**: The module directory name. The class file and the class carry the same name, character for character, including case.

## Structure

### The Sixteen Types

Counts include the sandbox modules whose names start with `Sample`. Those exist to be read and copied, not enabled in production.

| Type | Base class | What the core calls it for | Present |
| --- | --- | --- | --- |
| Servers | `ServerModule` | Provisioning and managing a hosting account, a virtual machine or a game server on a control panel | 50 |
| Payment | `PaymentGatewayModule` | Taking money: the payment screen, the capture, the callback and the settlement | 164 |
| Registrars | `RegistrarModule` | Registering, renewing and transferring a domain name at a registrar | 21 |
| Product | `ProductModule`, `SslProductModule` | A product that is provisioned without a server. All four non-sample modules here are SSL certificate products | 7 |
| Addons | `AddonModule` | A feature bolted onto the panel itself: its own settings page, privileges and hooks | 9 |
| SMS | none, plain class | Sending a text message through a provider | 11 |
| Mail | none, plain class | Delivering outgoing mail, by SMTP or through a provider API | 4 |
| Authentication | none, plain class | A second factor at login: mail code, SMS code or an authenticator app | 3 |
| Pipe | none, plain class | Pulling mail from a mailbox and turning it into support tickets | 3 |
| Imports | none, plain class | Migrating clients, services and invoices in from another platform | 3 |
| Fraud | `FraudModule` | Scoring an order or a signup and recording what was found | 2 |
| SocialAuth | `SocialAuthProvider` | Social login: the authorisation redirect, the code exchange and the identity token check | 3 |
| Captcha | none, plain class | Challenging a public form before it is accepted | 4 |
| Currency | none, plain class | Fetching exchange rates for the configured currencies | 7 |
| IP | none, plain class | Resolving an address to a country and a network, used for locale and risk | 3 |
| Storage | `StorageModule`, `CloudStorageModule` | A remote destination backups are uploaded to and restored from | 6 |

### Discovery and Class Resolution

Loading is two separate things. Configuration and the language pack are `include` calls into a static cache. The class file is included only when an instance is wanted, and the loader's third argument controls that split.

```bash
coremio/modules/{Type}/{Name}/
├── {Name}.php      # included only when an instance is built (nominc = false)
├── config.php      # returns an array, cached under Modules::$modules[Type][Name]['config']
└── lang/
    ├── en.php      # fallback, always tried when the active language file is missing
    └── {lang}.php  # cached under ['lang']
```

Three class names are tried in this order, and the first that exists wins. That is why the global namespace and the type namespace both work.

```php
$classList = [
    $name . "_Module",                                       // legacy suffix form
    $name,                                                   // global namespace
    "WISECP\\Modules\\" . ucfirst($type) . "\\" . $name,      // the form new modules use
];
```

## Reference

### The Registry

All static, all in one class. The exact signatures:

```php
// Build. Returns null when no class of any candidate name exists.
public static function getInstance(string $type, string $name, array $params = []): ?object;

// Load configuration and language into the static cache. Returns the loaded record(s),
// or false when a named module directory does not exist.
public static function Load($type = '', $name = '', $nominc = false, $status = '');
public static function add($file, $type, $nominc = false, $status = '');

// Read back from the cache. Config() does NOT load; Lang() does.
public static function Config($type, $module);
public static function Lang($type, $module, $lang = '');
public static function getName(string $type, string $module): string;
public static function getModules($type = '', $name = '');

// Surfaces the panel renders for a module.
public static function getPage(string $type, string $name, string $page, array $data = []): string;
public static function getController($type = '', $name = '', $cname = '');
public static function view($file, $variables = []): string;
public static function logo(string $name = '', $type = 'Servers'): string;

// Settings-field rendering, shared by every type that declares fields in config.php.
public static function fields_output($data = [], $input_name = ''): string;
public static function fields_output_wBuilder(AdminFormBuilder $form, $data = [], $input_name = ''): void;

// The module action log the panel shows under the client's action history.
public static function save_log($type = '', $module = '', $action = '', $request = '', $response = '', $processed = '');
```

### Arguments That Change the Result

- **$name = 'All'**: Scan the whole type directory instead of one module. The literal `'All'`, not an empty string: for `Mail` and `SMS` an empty name means something else (see Pitfalls).
- **$nominc = true**: No include. Configuration and language are read, the class file is not touched. The cheap call, used by every listing screen.
- **$status**: Left as the default empty string it filters nothing. Pass a non-string, in practice `true`, and only modules whose `config['status']` equals it load. That is how the panel lists enabled addons only.
- **$params**: Positional constructor arguments. The factory reflects the constructor and pads with `null` up to the required count. A module with a mandatory argument still builds when you pass nothing.

### Return Shapes

- **getInstance()**: The object, or `null`. Cached per type, name and serialised parameters, so two calls in one request give the same object.
- **Load()**: With a name: `['config' => [], 'lang' => [], 'config_file' => '']`, or `false` when the directory is missing. With `'All'`: a map of name to that same record, ordered by display name.
- **Config()**: The cached configuration array, or `null` when nothing has loaded it yet. It never reads the file itself.
- **Lang()**: The language array, or `[]`. Unlike `Config()` it does read the file, falling back to the active interface language, then the system default, then English.
- **getName()**: The display label: `lang['name']`, then `config['name']`, then the directory name. It loads for you, so it is safe to call cold.

## Example

Building one module and calling it, then listing a whole type without building anything.

```php
$module = Modules::getInstance("Registrars", "ExampleRegistrarModule");
if (!$module) throw new Exception(Language::gc("modules/error-not-found"));

// Guard on the contract, not on the class: a module is plain PHP and may predate a method.
if (!method_exists($module, "create")) throw new Exception(Language::gc("modules/error-unsupported"));

// Context is bound with setters, not passed as arguments.
$module->set_service($serviceId);

$result = $module->create();
```

```php
// nominc = true: read config.php and lang/, do not include any class file.
$installed = Modules::Load("Currency", "All", true) ?: [];

$rows = [];
foreach ($installed as $name => $record) {
    $rows[$name] = [
        'label'  => Modules::getName("Currency", $name),
        'active' => (bool) ($record['config']['status'] ?? false),
        'logo'   => Modules::logo($name, "Currency"),
    ];
}

// Enabled modules only, in one call: the fourth argument is compared to config['status'].
$enabled = Modules::Load("Addons", "All", true, true) ?: [];
```

The reading side, when you know the module and need only its settings:

```php
// Config() reads the cache only, so the load has to come first.
Modules::Load("Servers", "cPanel", true);

$config = Modules::Config("Servers", "cPanel") ?: [];
$fields = $config['fields'] ?? [];

// Same record, one call, when the load result is all you need.
$fields = Modules::Load("Servers", "cPanel", true)["config"]["fields"] ?? [];
```

## Pitfalls

> **Never build a module with new**
> 
> The factory includes the class file, fills the configuration and language properties, pads missing constructor arguments and caches the result. Constructing the class directly skips all of it. The module runs with an empty configuration, which looks exactly like a provider returning nothing.

> **Reading configuration without loading returns null, not an empty array**
> 
> The read is cache only. Called cold it returns `null`, which flows into your code as a missing setting rather than an error. Load first, or take the configuration from the load result.

> **An empty name means the active module for Mail and SMS**
> 
> For those two types only, a load with no name resolves the configured active driver and loads that one. Every other type reads the whole directory. For the full list, always pass `'All'`.

> **The instance is shared for the whole request**
> 
> The factory caches per type, name and parameters, so a loop over many services keeps handing back the same object, with whatever a previous iteration bound to it. Bind the new context with the setters at the top of every iteration.

> **Dashboard widgets are not a module type**
> 
> Panel widgets come from privilege checks in the dashboard controller and are extended with hooks. A directory under the modules folder does nothing, and there is no widget module type.

## Related Articles

- [Module Anatomy](https://dev.wisecp.com/en/module-anatomy)
- [Your First Module](https://dev.wisecp.com/en/your-first-module)
- [Module Lifecycle](https://dev.wisecp.com/en/module-lifecycle)
- [Module Configuration](https://dev.wisecp.com/en/module-configuration)
- [Registering Hooks from a Module](https://dev.wisecp.com/en/registering-hooks-from-a-module)
- [Bootstrap and Autoloading](https://dev.wisecp.com/en/bootstrap-and-autoloading)


# Module Anatomy

https://dev.wisecp.com/en/module-anatomy

The files a module is made of, the three names that must agree, and what the base class hands you.

## Overview

A module is a directory with one mandatory class file, plus optional files the platform looks up by name. Nothing is registered: the loader builds each path from the type and the module name. A file in the right place is found, and a misspelled one is ignored without a warning.

Most types give you a base class that has already done the setup. Paths, configuration and language are ready before your first method runs, and provisioning types also know which service they act on. Rebuilding any of it by hand is the most common beginner mistake.

## Structure

### The File Tree

```bash
coremio/modules/{Type}/{Name}/
├── {Name}.php          # REQUIRED: the class, same name as the directory
├── config.php          # returns an array; settings, metadata, field definitions
├── logo.png            # or .svg/.webp/.jpg; resolved by name when config does not name one
├── hooks.php           # Hook::add() registrations, included on EVERY request
├── AdminArea.php       # own admin page, registered from router.php
├── router.php          # include + ModuleAdminArea::register()
├── lang/
│   ├── en.php          # REQUIRED in practice: the fallback language
│   └── tr.php          # one file per translated language
├── pages/              # settings and management markup, resolved by page name
├── views/              # same role, alternative directory name
├── controllers/        # named entry points reached through the controller dispatcher
├── assets/
│   ├── style/          # css
│   ├── js/             # javascript
│   └── images/         # interface images, not the logo
└── src/                # your own helper classes, NOT autoloaded
```

### What Each File Is For

| Path | Required | Read when | What it holds |
| --- | --- | --- | --- |
| `{Name}.php` | Yes | An instance is built | The class. Included unless the loader is told to skip it. |
| `config.php` | In practice yes | Every load, with or without the class | An array: settings, metadata, field definitions, enabled flag. |
| `lang/en.php` | Yes | Every load | An array. The fallback whenever the active language has no file. |
| `lang/{code}.php` | Optional | That language is active | The same keys, translated. |
| `logo.png` and friends | Optional | The panel shows the module | The icon. Found by extension when configuration names no file. |
| `hooks.php` | Optional | Every request, for every module on disk | Hook registrations. They run whether or not the module is enabled. |
| `AdminArea.php` plus `router.php` | Optional | Every request, before routing | An admin page of your own: route, menu entry, privileges. |
| `pages/` or `views/` | Optional | A page is shown | Markup addressed by page name; both directory names are tried. |
| `controllers/` | Optional | A named controller is dispatched | One file per entry point, tried before the method of that name. |
| `assets/` | Optional | The browser requests it | Style, script and image files, addressed through the URL property. |
| `src/` | Optional | You include it yourself | Your API client and helpers. The autoloader does not reach here. |

## Reference

### The Three Names That Must Agree

- **Directory name**: The module name, and the only thing the loader is given. Case matters on a case-sensitive filesystem: `cPanel` is not `CPanel`.
- **Class file name**: The directory name plus `.php`. The loader builds this path and looks nowhere else.
- **Class name**: Three candidates are tried in order. The name with a `_Module` suffix, the bare name in the global namespace, then the name under the type namespace. Write new modules as the third form.
- **Namespace**: `WISECP\Modules\{Type}`, with the type spelled as the directory is. Inside it, a bare core class name resolves into the module namespace and fails at runtime. Import core classes, or prefix them with a backslash.

### Base Class by Type

Eight of the sixteen types have one. The rest inherit nothing; their contract is whatever methods the core looks for.

| Base class | Type | Shares the trait | What its constructor already did |
| --- | --- | --- | --- |
| `ServerModule` | Servers | Yes | Paths, configuration, language, the current admin, the default tool set, and the server when one was passed in. |
| `PaymentGatewayModule` | Payment | No | Paths, configuration, language, the pay button label, an empty client info object. |
| `RegistrarModule` | Registrars | Yes | Paths, configuration, language, and the crypt sub-key moved from the per-user key to the system key. |
| `ProductModule` | Product | Yes | Paths, configuration and language, nothing else. |
| `SslProductModule` | Product | Inherited | Abstract. The product base, plus the validation, reissue and SAN contract every certificate module answers. |
| `AddonModule` | Addons | No | Paths, configuration, language, the admin area link, the signed-in admin and member records. |
| `FraudModule` | Fraud | No | Paths, configuration, language, the current controller link, the signed-in identities. |
| `StorageModule` | Storage | No | Abstract. Takes the storage configuration as a constructor array; no directory or URL property. |
| `SocialAuthProvider` | SocialAuth | Yes | Abstract. Paths, configuration and language; every endpoint and the token check stay abstract. |

### Inherited Properties

From the shared trait, so identical in server, registrar, product and social login modules.

| Property | Type | Filled by | Holds |
| --- | --- | --- | --- |
| `$_name` | string | Constructor | Module name; the class short name when you do not set it. |
| `$_type` | string | Constructor | Type directory, as the base class declared it. |
| `$dir` | string | Constructor | Absolute path to the module directory, trailing separator included. |
| `$url` | string | Constructor | Public URL of the module directory, trailing slash. Build asset links from it. |
| `$config` | array | Constructor | The whole `config.php` array, already loaded. |
| `$lang` | array | Constructor | Module strings for the active language. |
| `$service` | array | Binding a service | The full service record; empty until you bind one. |
| `$product` | array | Binding a service or a product | The product the service was ordered from. |
| `$order` | array | Binding an order | The order record, while provisioning from an order. |
| `$user` | array | Binding a service | Owner identity, contact details and billing address. |
| `$admin` | array | Binding a service | The admin acting; empty outside the panel. |
| `$options` | array | Binding a service | The service options, where a module keeps its per-service state. |
| `$addons` | array | Binding a service | The service add-on records, keyed by add-on id. |
| `$addon_params` | array | Binding a service | Configurable add-on values merged into totals; cancelled and waiting excluded. |
| `$addon_params_by_id` | array | Binding a service | The same values per add-on, cancelled ones included. |
| `$requirement_params` | array | Binding a service | Customer answers to product requirements, keyed by your parameter name. |
| `$callable_methods` | array | You declare it | Allowlist of methods reachable by bare name from a URL; anything else is not. |
| `$error` | string | Nothing, in new code | Left from the previous major version. Report failures by throwing instead. |

### Inherited Helpers

```php
// Context binding. Each takes an id OR the already-loaded record; an int is fetched for you.
public function set_service(array|int $service = []): void;
public function set_product(array|int $product = []): void;
public function set_order(array|int $order = []): void;

// Persist $this->options back onto the bound service. False when nothing is bound.
public function save_options(): bool;

// Rewrite config.php. $auto_status = true flips status on when a settings array is present.
protected function save_config($data = [], $auto_status = true);

// One log row per provider call, shown in the panel's action history.
protected function save_log($action = '', $request = '', $response = '', $processed = ''): int|bool;

// Encrypt with the module's crypt sub-key; $key overrides it for one call.
protected function encode_str(string $str = '', string $key = ''): string;
protected function decode_str(string $str = '', string $key = ''): string;

// Resolved logo URL, or an empty string.
public function logo(): string;

// Metered metric limits enabled on the bound service, keyed by metric type.
protected function enabled_metrics(): array;
protected function enabled_metric_values(): array;
protected function reapply_enabled_metrics(): void;

// Data attributes for a dropdown that loads its options from one of your methods.
protected function method_url_data(string $method): array;

// Dispatch "reset-password" to handle_reset_password(). Returns null when it does not exist.
protected function use_method($param = '');

// Add-on values for one option, multiplied by quantity unless the param opts out.
public function resolveAddonConfigurable(array $moduleParams, int $quantity = 0): array;
```

- **save_config()**: Writes the whole array, so merge into the existing configuration first. Pass `false` as the second argument when a settings write should not also enable the module.
- **encode_str()**: The sub-key is a property, not an argument: `user` by default, switched to `system` by the registrar base. A value encrypted under one key does not decrypt under the other.
- **use_method()**: Maps a dashed action name onto a `handle_` method, dashes becoming underscores. Any other name is unreachable from the panel action buttons.
- **save_log()**: Arrays are encoded for you. Put the request URL under `api_url` in the request array; it is lifted into its own column.

### The Addon Exception

An addon module does not share the trait. None of the properties above exist on it, apart from the four its own base declares.

```php
// Properties: $config, $lang, $dir, $url, $area_link, $_name, $user, $admin, $error.
public function __construct();

// Render views/{file}.php with $variables extracted into it. Returns the markup.
protected function view($file = '', $variables = []): string;

// Privilege keys this addon adds to the admin role screen.
public function privileges();

// Called by the addon settings screen: $pFields is the posted settings,
// $accessPs the posted privilege selection.
public function save_settings($pFields, $accessPs): bool;

// Enable or disable from the addon list.
public function change_addon_status($arg = '');

// Rewrite config.php. Note the single argument: no auto-status flag here.
public function save_config($data = []): bool;

// Same crypt helpers, but the addon base defaults the sub-key to 'system'.
protected function encode_str(string $str = '', string $key = ''): string;
protected function decode_str(string $str = '', string $key = ''): string;

public function use_default_settings($formElements = null);
public function isEnabled();
```

## Example

A skeleton that declares nothing it inherits, and the caller that drives it.

```php
namespace WISECP\Modules\Registrars;

use Exception;
use Language;
use RegistrarModule;
use WISECP\Modules\Registrars\AcmeDomains\ApiClient;

class AcmeDomains extends RegistrarModule
{
    private ?ApiClient $api = null;

    // No constructor. The base one already set $_name, $_type, $dir, $url, $config and $lang.
    // Build the client lazily instead: the service is bound AFTER construction, so anything
    // that needs $this->service cannot run here.
    private function api(): ApiClient
    {
        if ($this->api) return $this->api;

        // src/ is not autoloaded, so the file is included explicitly.
        include_once $this->dir . 'src' . DS . 'ApiClient.php';

        $settings = $this->config['settings'] ?? [];

        $this->api = new ApiClient(
            (string) ($settings['username'] ?? ''),
            $this->decode_str((string) ($settings['apiKey'] ?? '')),
        );

        return $this->api;
    }

    // NOT create(). The base class owns create() and dispatches to register() or
    // transfer() depending on whether an EPP code is present. Overriding create()
    // would drop the transfer branch and both of its hooks.
    public function register(): array|bool
    {
        // The bound record carries the ordered domain in options; service['name']
        // is the fallback. There is no 'domain' column on the service itself.
        $domain = (string) ($this->options['domain'] ?? $this->service['name'] ?? '');
        if ($domain === '') throw new Exception(Language::gc("acme/error-no-domain"));

        // 'year' is the registration length. service['period'] is the billing cycle
        // string ('y', 'm', 'none'), never a number of years.
        $year = (int) ($this->options['year'] ?? $this->service['period_time'] ?? 1) ?: 1;

        $response = $this->api()->register($domain, $year);

        // Every provider call is recorded; api_url is lifted into its own column.
        $this->save_log('register', ['api_url' => $this->api()->last_url, 'domain' => $domain], $response);

        if (!($response['ok'] ?? false))
            throw new Exception($response['message'] ?? Language::gc("acme/error-refused"));

        return true;
    }
}
```

```php
$module = Modules::getInstance("Registrars", "AcmeDomains");
if (!$module) throw new Exception(Language::gc("modules/error-not-found"));

// Binding is a separate step, and it is what fills $service, $product, $user and $options.
$module->set_service($serviceId);

try {
    // The caller always asks for create(). On a registrar that is the base method,
    // which routes to register() or transfer() and runs the domain hooks around it.
    $result = $module->create();
}
catch (\Throwable $e) {
    // A module reports failure by throwing; the caller turns the message into a response.
    return $operation->output(['status' => "error", 'message' => $e->getMessage()]);
}

// Module state the module changed goes back to the service through its own helper.
$module->save_options();
```

## Pitfalls

> **The hook file of every module on disk runs on every request**
> 
> A disabled or half-finished module still registers its hooks. Gate the body on the module's own enabled flag, and keep expensive work out of the file.

> **Your own helper classes are not autoloaded**
> 
> The autoloader maps the type namespace to the type directory and stops there. An import under the module's source folder loads no file. Include it before first use. A fatal inside a hook silently kills the rest of that hook body.

> **An unqualified core class name resolves into your namespace**
> 
> It is looked up under the module namespace and not found, at runtime rather than at lint time. The file appears to work, because the classes you did import are fine.

> **Do not build an API client in the constructor**
> 
> Configuration and language are ready there, but the service, product, owner and options are bound afterwards. A client built from service data at that moment reads empty values. It then fails against the provider with a message that points nowhere near the cause. Build it lazily on first use.

> **Some base methods are orchestrators, not empty slots**
> 
> A name the core calls is not automatically the name you implement. On a registrar the base already defines `create()`. It chooses between registration and transfer, runs the gate and action hooks, and turns a submitted transfer into a pending state. You write `register()` and `transfer()`. Check the base before overriding.

> **Two directory names mean the same thing**
> 
> The names `configuration` and `settings` are also tried for each other. Pick one of each and stay with it.

## Related Articles

- [The Module System](https://dev.wisecp.com/en/the-module-system)
- [Your First Module](https://dev.wisecp.com/en/your-first-module)
- [Module Configuration](https://dev.wisecp.com/en/module-configuration)
- [Module Language Files](https://dev.wisecp.com/en/module-language-files)
- [Module Assets and Logo](https://dev.wisecp.com/en/module-assets-and-logo)
- [Module Lifecycle](https://dev.wisecp.com/en/module-lifecycle)

# Your First Module

https://dev.wisecp.com/en/your-first-module

Build a working module from an empty directory. The class, its configuration, its language files, and the moment it appears in the admin panel.

## Overview

A module is a directory named after the module. It holds a class file of the same name. It also holds the two files every module has: a configuration array and a language folder. Nothing has to be registered anywhere. The module list reads the directory, so a correctly named directory is a module the moment it exists.

This walkthrough builds a currency rate module, because that is the smallest type with a real job: it answers three methods. The same skeleton is what every other type starts from, and each type adds its own required methods on top of it.

## Prerequisites

- A development installation you can edit and reload. Do not build against an installation you cannot break.
- Write access to `coremio/modules`.
- The conventions the codebase holds itself to, because a module is read and reviewed like core code.

## Structure

Four paths, and the names are not free: the directory, the class file and the class all carry the same name.

```bash
coremio/modules/Currency/AcmeRates/
├── AcmeRates.php      # the class, named after the directory
├── config.php         # returns an array; the module rewrites it when settings are saved
└── lang/
    ├── en.php         # returns an array; 'name' and 'description' are what the panel shows
    └── tr.php
```

- **Module type**: The directory one level up (`Currency` here). It decides which contract the class has to satisfy and where the panel lists it.
- **Module name**: The directory name. Directory, file and class must match exactly, including case.
- **Namespace**: Always `WISECP\Modules\{Type}`. Core classes are then reached with a leading backslash.

## Walkthrough

### Create the Directory

1. Create `coremio/modules/Currency/AcmeRates/`.
2. Create `lang/` inside it.
3. The panel will not list the module yet: it has no class to load.

### Write the Class

1. Create `AcmeRates.php` with the namespace, the class and the three properties every module declares.
2. Load the configuration and the language pack in the constructor, so the rest of the class can read them.
3. Implement the methods the type requires, listed in the contract below.
4. Reload the modules screen; the module is listed with the name from its language file.

### Add Configuration

1. Create `config.php` returning an array of the settings your module needs, with empty or safe defaults.
2. Implement `save_config()` so it merges the submitted values into the array and writes the file back.
3. Write the file through the file manager rather than with a raw write. A configuration file is PHP, and a stale compiled copy keeps serving the old values after a save.
4. Save a setting in the panel and reload; the new value is the one you read back.

### Add Language Files

1. Create `lang/en.php` and `lang/tr.php`, each returning an array.
2. Give both a `name` and a `description`: those two keys are what the module list prints.
3. Reload the list; the module now shows its own name instead of its directory name.

## Reference

### What a Currency Module Must Implement

There is no base class to extend. The contract is the set of methods the core actually calls, and each type has its own. For `Currency` it is three:

```php
// Rate fetch. Called by Money::get_exchange_rates() and by the panel's connection test,
// which checks for it with method_exists first. $to holds the target codes, uppercase.
// Return a map of CODE => rate; anything falsy is treated as a failure.
public function exchange_rates(string $from = '', array $to = []): array|false;

// Settings persistence. The currency settings operation calls it with the submitted
// values for this module only, i.e. $_POST['module_data']['AcmeRates'].
public function save_config(array $data = []): bool;

// The settings markup shown on the currency screen. Called unconditionally on the
// instance, so it must exist even if it returns an empty string.
public function page_settings(): string;
```

- **Field names in page_settings()**: Inputs must be named `module_data[{ModuleName}][{key}]`; that is the shape `save_config()` receives. A differently named field never reaches the module.
- **config['help-link']**: Read by the currency screen and printed next to the module's settings as a link to the provider's own page. Leave it out and no link is shown.
- **lang['name'] · lang['description']**: What the module list prints. Without `name` the list falls back to `config['name']` and then to the directory name.

### Core Calls a Module Makes

```php
// Modules : the factory and the two loaders. All static.
public static function getInstance(string $type, string $name, array $params = []): ?object;
public static function Config($type, $module);                          // cached; requires a prior load
public static function Lang($type, $module, $lang = '');                // loads the file itself
public static function Load($type = '', $name = '', $nominc = false, $status = '');
public static function getName(string $type, string $module): string;
public static function save_log($type = '', $module = '', $action = '', $request = '', $response = '', $processed = '');

// Utility : request and JSON helpers.
public static function HttpRequest($url = '', $params = [], $retry = 0);   // array first argument, see below
public static function jdecode($string = '', $mode = false);              // $mode = true for an associative array
public static function jencode($string = '', $flags = 0): string|false;
public static function array_export($array = [], $options = []);          // ['pwith' => true] wraps it as a PHP file

// FileManager : the write that invalidates the compiled copy.
public static function file_write($file, $data = null, $mode = 'w', $flags = 0);
```

`HttpRequest()` has two shapes; pass an array as the first argument and the rest is ignored:

- **url**: Full address. Build query strings with `urlencode()`; nothing escapes it for you.
- **type**: Method, defaulting to `'GET'`. A body is only sent when the method is not GET.
- **data**: The body. An array is sent as form fields; a string is sent as-is, which is how you post JSON.
- **header**: List of raw header lines: `['Authorization: Bearer ' . $key, 'Content-Type: application/json']`.
- **timeout · connect_timeout · ssl_verify · allow_ipv6**: Defaults are 30s, 10s, verification on and IPv6 off. Leave the last two alone unless the provider forces you.

## Example

The complete class, then the two files it reads. It declares what every module declares and answers the three methods its type asks for. It reports failure the way the rest of the platform does.

```php
namespace WISECP\Modules\Currency;

class AcmeRates
{
    public string $name   = "AcmeRates";
    public ?array $config = null;
    public ?array $lang   = null;

    public function __construct()
    {
        $this->config = \Modules::Config("Currency", $this->name);
        $this->lang   = \Modules::Lang("Currency", $this->name);
    }

    public function save_config(array $data = []): bool
    {
        $merged = array_replace_recursive($this->config ?: [], $data);

        return (bool) \FileManager::file_write(__DIR__ . DS . "config.php", \Utility::array_export($merged, ['pwith' => true]));
    }

    public function exchange_rates(string $from = '', array $to = []): array|false
    {
        $key = (string) ($this->config['apiKey'] ?? '');
        if ($key === '')
            throw new \Exception($this->lang['error-no-key'] ?? 'API key is not set.');

        $response = \Utility::HttpRequest([
            'url'    => 'https://api.example.com/rates?base=' . urlencode($from),
            'type'   => 'GET',
            'header' => ['Authorization: Bearer ' . $key],
        ]);

        // Every call is recorded, so a failing provider can be diagnosed from the panel.
        \Modules::save_log("Currency", $this->name, "exchange", ['from' => $from, 'to' => $to], $response);

        $data = \Utility::jdecode((string) $response, true);
        if (!isset($data['rates']))
            throw new \Exception('Unexpected response from the rate provider.');

        $out = [];
        foreach ($to as $code) $out[$code] = (float) ($data['rates'][$code] ?? 0);

        return $out;
    }

    public function page_settings(): string
    {
        $key = htmlspecialchars((string) ($this->config['apiKey'] ?? ''), ENT_QUOTES);

        // The field name is the contract: this is exactly what save_config() receives.
        return '<div class="row mb-0 align-items-center">'
             . '<label class="col-sm-3 col-form-label fw-semibold">API Key</label>'
             . '<div class="col-sm-9"><input type="text" class="form-control" '
             . 'name="module_data[AcmeRates][apiKey]" value="' . $key . '"></div></div>';
    }
}
```

```php
// config.php : the keys page_settings() prints and save_config() writes back.
return [
    'apiKey'    => '',
    'help-link' => 'https://api.example.com/docs',
];

// lang/en.php : 'name' and 'description' are what the module list shows.
return [
    'name'         => 'Acme Rates',
    'description'  => 'Exchange rates from the Acme provider. An API key is required.',
    'error-no-key' => 'API key is not set.',
];
```

The other half: how the platform reaches the module. Never construct one with `new`, because the factory is what loads the class, fills the configuration and caches the object.

```php
$module = Modules::getInstance("Currency", "AcmeRates");

// Guard on the contract, not on the type: a module is a plain class and may predate a method.
if ($module && method_exists($module, "exchange_rates"))
    $rates = $module->exchange_rates("USD", ["EUR", "TRY"]);

// Only the configuration and the language pack, without instantiating anything.
Modules::Load("Currency", "AcmeRates", true);
$config = Modules::Config("Currency", "AcmeRates");
$label  = Modules::getName("Currency", "AcmeRates");
```

## Pitfalls

> **Report failure by throwing**
> 
> A module signals a problem the same way an operation does: throw, with a message a person can read. The caller catches it and surfaces that message. An `$error` property set alongside `return false` is a leftover from the previous major version. You will see it in older ports, but new code does not use it.

> **A configuration file is compiled PHP**
> 
> Write it through the file manager. A raw write leaves the previously compiled copy in place. The panel then shows the old value after a save that actually succeeded.

> **Inside the module namespace an unqualified core class does not resolve**
> 
> `Utility::jdecode(...)` written in a module file is looked up as `WISECP\Modules\Currency\Utility` and fails at runtime, not at lint time. Prefix core classes with a backslash or import them.

> **Your own helper classes are not modules**
> 
> A class you put under the module's own source folder is ordinary PHP and is constructed with `new`. It is also not autoloaded, so include it before use. The factory is only for the module types the platform knows about.

## Related Articles

- [Module Anatomy](https://dev.wisecp.com/en/module-anatomy)
- [Module Configuration](https://dev.wisecp.com/en/module-configuration)
- [Module Language Files](https://dev.wisecp.com/en/module-language-files)
- [Writing a Currency Module](https://dev.wisecp.com/en/writing-a-currency-module)


# Module Lifecycle

https://dev.wisecp.com/en/module-lifecycle

What happens to a module between arriving on disk and being deleted. When it is loaded, when it is built, where its enabled flag lives, and which of your methods the platform calls.

## Overview

There is no install routine. Copying the directory into place is the installation, and the module is listed on the next request. Everything after that is a sequence of small, separately triggered steps. Almost all of the methods involved are optional. The platform checks whether your class has them and moves on when it does not.

Two ideas are easy to confuse. **Present** means the directory exists. That is enough for it to be listed, to have its hook file executed and to be instantiated by anyone who asks. **Enabled** means an operator picked it, and where that decision is stored depends entirely on the type. Nothing about being disabled stops your code from being loaded.

## Structure

### The Stages

| Stage | What makes it happen | What runs in your module | How it is undone |
| --- | --- | --- | --- |
| On disk | The directory is copied, extracted from an archive, or fetched by the panel | Nothing. No code is executed by the act of arriving | Deleting the directory |
| Hooks registered | Every request, for every module directory. No status is checked | `hooks.php` from top to bottom | Only by gating the body yourself, or removing the file |
| Loaded | Something asks the registry for this module or for its whole type | Nothing. `config.php` and the language file are read into a static cache | Nothing to undo; the cache lives for one request |
| Instantiated | The factory is asked for an object | The base constructor, then yours if you wrote one | Nothing to undo. The object is cached for one request |
| Bound | A caller binds a service, an order or a product | Your `set_service` override, when you have one | Binding something else onto the same object |
| Enabled | An operator turns it on, or an import is told to activate it | `activate()` then `enable()`, both optional, both able to refuse | Disabling |
| Disabled | An operator turns it off | `deactivate()` then `disable()`, both optional | Enabling again, which reruns the enable methods |
| Deleted | The delete action on the module list | `uninstall()`, before any file is touched | Restoring the files. Nothing restores data you dropped |

## Reference

### Where the Enabled Flag Lives

Four types keep a `status` key in their own configuration file. The rest are selected somewhere else, and two have no flag at all.

| Type | Where the decision is stored | Written by |
| --- | --- | --- |
| Addons | `status` in the module's own `config.php` | The toggle on the addon list, through the base status method |
| Product | `status` in the module's own `config.php` | The settings screen for that module group, writing every module's file in one pass |
| Fraud | `status` in the module's own `config.php` | The fraud settings screen |
| SocialAuth | `status` in the module's own `config.php` | The social login settings |
| Mail, SMS, IP, Currency | One module name in the platform's module configuration | The settings screen for that group. Only one can be active at a time |
| Payment | A list of module names in the platform's module configuration | The payment settings screen, plus a separate entry naming the card storage gateway |
| Authentication | A list of module names in the platform's module configuration | The security settings |
| Captcha | The chosen type in the options configuration, alongside an on and off flag | The security settings |
| Servers | No flag anywhere | A server record naming the module is what puts it in use |
| Registrars | No flag anywhere | A top level domain extension pointing at the module is what puts it in use |
| Storage, Pipe, Imports | Chosen at the point of use | The backup destination, the ticket mailbox and the import run respectively |

### Lifecycle Methods You Can Declare

The first five are pure convention. No base class declares them; each is looked up with `method_exists` and skipped when absent. A falsy return stops the step it belongs to. The last two are different, and the difference matters. `change_addon_status` is already implemented on the addon base. `testConnection` is declared abstract on the social login base, which makes it mandatory there rather than optional.

```php
// Enable path, in this order. activate() runs first, then enable().
// Returning false leaves the module disabled and nothing is written.
public function activate(): bool;
public function enable(): bool;

// Disable path, in this order.
public function deactivate(): bool;
public function disable(): bool;

// Runs before the module directory is removed. Returning false aborts the delete.
// This no-argument boolean form is the ADDON one.
public function uninstall(): bool;

// The Authentication type reuses the name with a different shape: the stored
// enrolment data is passed in, and an array is returned. Only 'error' aborts.
public function uninstall(array $data = []): array;   // ['status' => 'successful']

// The Test button. Two shapes, and the caller decides which one you get:
// social login providers are called with NO argument (abstract on the base),
// registrars are called WITH the merged configuration as the only argument.
public function testConnection(): bool;                  // SocialAuth
public function testConnection($config = []): bool;      // Registrars

// Already implemented on the addon base: it runs the four methods above and then
// writes the new status into config.php. Override only to replace that behaviour.
public function change_addon_status($arg = '');
```

> **Two of these names are reused with a different signature**
> 
> Declaring the no-argument `testConnection()` on a registrar is the trap. The base passes the merged configuration array as the first argument. Your method needs that value to test the credentials the operator typed, and it threw it away. Every shipped registrar declares `$config = []`. The same applies to `uninstall`: the addon form takes nothing, the authentication form takes the stored enrolment array.

### The Addon Status Chain

This is the sequence in full. Reading it is the fastest way to see why a failing `enable()` leaves the module exactly as it was.

```php
public function change_addon_status($arg = '')
{
    $status = $arg == "enable";
    $apply  = true;

    if ($status && method_exists($this, 'activate'))    $apply = $this->activate();
    if ($status && method_exists($this, 'enable'))      $apply = $this->enable();
    if (!$status && method_exists($this, 'deactivate')) $apply = $this->deactivate();
    if (!$status && method_exists($this, 'disable'))    $apply = $this->disable();

    // The flag is written LAST, and only when the module agreed.
    if ($apply) {
        $config           = $this->config;
        $config["status"] = $status;
        $this->save_config($config);
    }

    return $apply;
}
```

The operation that calls it, so you can see how a refusal reaches the operator and where the two arguments come from:

```php
$key    = (string) Filter::init("POST/module", "route");
$status = (int) Filter::init("POST/status", "rnumbers");

$instance = Modules::getInstance("Addons", $key);

if (!method_exists($instance, "change_addon_status"))
    throw new Exception("Module class does not have a method named change_addon_status.");

$status = $status ? "enable" : "disable";

// A thrown Exception travels straight out as the error message. A false return is
// the older path: the caller then reads the legacy error property for a reason.
$result = $instance->change_addon_status($status);
if (!$result) throw new Exception($instance->error ?: "Unknown error");

User::addAction($adata["id"], "alteration", "change-addon-status-" . $status, ['module' => $key]);

Hook::run('action:addon.status_changed', $key, $status);
```

### Lifecycle Hooks

Gates can veto, actions only observe. A gate returning a non-empty string turns that string into the error the operator sees.

- **gate:module.activate**: Runs before any module group activation is written, with the group and the list of newly activated names. Return a non-empty string, or an array carrying a message, to refuse.
- **action:module.activated**: After the group settings were saved, with the names that went from off to on. Only the difference is reported, not the whole selection. The return value is ignored.
- **action:module.deactivated**: The mirror of the previous one, with the names that went from on to off. The return value is ignored.
- **action:addon.status_changed**: After an addon was enabled or disabled, with the module key and the literal word that was applied. The return value is ignored.
- **gate:module.addon_install**: Before an uploaded archive is unpacked, with the upload entry and whether it is to be activated immediately.
- **action:addon.installed**: After extraction, with the module key and the activation flag. This is where a marketplace record or an update manifest is written.
- **gate:module.addon_delete**: Before an addon is deleted, with its key. Use it to refuse while the module still owns live data.
- **action:addon.deleted**: After the directory is gone. The module's own class no longer exists at this point, so listen from somewhere else.
- **gate:module.delete**: The same veto for a non-addon module, with the type and the key.
- **action:module.config_saved**: After a module's configuration file was rewritten, with the type and the key. Useful for clearing a cache your module keeps.

## Example

An enable that creates its own schema, a disable that deliberately keeps the data, and an uninstall that finally removes it. The whole point of the three is that only the last one is destructive.

```php
public function enable(): bool
{
    // Idempotent on purpose: enable() runs again on every re-enable and after an update.
    $this->check_database();

    return true;
}

/** Only ever adds. Never drops, never rewrites an existing column. */
private function check_database(): void
{
    if (!\WDB::hasTable("Acme_events"))
        \WDB::exec('CREATE TABLE `Acme_events` ('
            . '`id` INT(11) UNSIGNED NOT NULL AUTO_INCREMENT,'
            . '`service_id` INT(11) UNSIGNED NOT NULL DEFAULT 0,'
            . '`payload` TEXT NULL DEFAULT NULL,'
            . 'PRIMARY KEY (`id`)'
            . ') ENGINE = InnoDB CHARSET=utf8mb4 COLLATE utf8mb4_unicode_ci;');

    // Columns added after the table first shipped are checked one by one.
    $col = \WDB::query("SHOW COLUMNS FROM `Acme_events` LIKE 'created_at'");
    if (!($col ? \WDB::getAssoc($col) : false))
        \WDB::exec("ALTER TABLE `Acme_events` ADD `created_at` INT(11) UNSIGNED NOT NULL DEFAULT '0'");
}

public function disable(): bool
{
    // Nothing is dropped here. Disabling is reversible, and an operator who turns the
    // module off for an afternoon must not lose a year of rows.
    return true;
}

public function uninstall(): bool
{
    // Refuse rather than destroy silently when there is still something to lose.
    // select() returns the BUILDER, so build() and getAssoc() are called on it.
    // Handing the builder to WDB::getAssoc() as an argument is a fatal: that
    // parameter expects a PDOStatement, which is what WDB::query() gives back.
    $stmt = \WDB::select('COUNT(id) AS total')->from('Acme_events');
    $rows = $stmt->build() ? (int) (($stmt->getAssoc() ?: [])['total'] ?? 0) : 0;

    if ($rows > 0 && !(int) \Filter::init("POST/purge", "rnumbers"))
        throw new \Exception($this->lang['error-uninstall-has-data'] ?? 'The module still holds records.');

    \WDB::exec("DROP TABLE IF EXISTS `Acme_events`");

    return true;
}
```

The other side lives in a listener, not in the module. A delete listener has to outlive the class it reacts to:

```php
// Runs on EVERY request, for this module, enabled or not. Keep it to registrations.
Hook::add('gate:module.addon_delete', 1, function ($key) {
    if ($key !== 'Acme') return '';

    $stmt = WDB::select('COUNT(id) AS cnt')->from('Acme_events');
    $stmt->where('processed', '=', 0);

    $open = $stmt->build() ? (int) (($stmt->getAssoc() ?: [])['cnt'] ?? 0) : 0;

    // A non-empty string is the refusal, and it is what the operator reads.
    return $open > 0 ? 'Acme still has ' . $open . ' unprocessed events.' : '';
});

Hook::add('action:addon.status_changed', 1, function ($key, $status) {
    if ($key !== 'Acme') return;

    Cache::getInstance()->clear(['acme']);
});
```

## Pitfalls

> **Enable runs more than once, so it has to be idempotent**
> 
> It runs on the first activation, on every re-activation, and it is the usual place an update rebuilds a schema. Check before you create, add columns one at a time, and never let it overwrite a row an operator edited. Seeding is safe only behind a count check on an empty table.

> **Disabling is not uninstalling, and deleting is not either**
> 
> Disable must leave every table and every row alone; it is a switch, not a cleanup. Delete removes the directory and nothing else, so anything your module wrote to the database survives it. If the data should go, drop it from the uninstall method, which is the only step that runs before the files disappear.

> **A disabled module still registers its hooks**
> 
> Hook files are collected by walking the modules directory, with no reference to any status. Your listeners fire while the module is off unless the body checks the module's own flag first. Treat the hook file as a registration list and put the decision inside each listener.

> **Refuse by throwing, with a sentence a person can act on**
> 
> A thrown exception becomes the message on screen. Returning false without a message produces the words "Unknown error". The caller then falls back to a legacy error property that new code does not set, and the operator learns nothing.

> **Only the addon delete action asks your class first**
> 
> The uninstall method is invoked by the addon delete action, and by the authentication type when an enrolment is removed. Deleting a module of any other type removes the files after its gate hook has had a chance to refuse. No method on your class is called at all. Put anything that must run at removal time behind the gate rather than in a method nobody invokes.

## Related Articles

- [The Module System](https://dev.wisecp.com/en/the-module-system)
- [Module Anatomy](https://dev.wisecp.com/en/module-anatomy)
- [Module Configuration](https://dev.wisecp.com/en/module-configuration)
- [Registering Hooks from a Module](https://dev.wisecp.com/en/registering-hooks-from-a-module)
- [Writing an Addon Module](https://dev.wisecp.com/en/writing-an-addon-module)
- [Changing the Database Schema](https://dev.wisecp.com/en/changing-the-database-schema)


# Module Configuration

https://dev.wisecp.com/en/module-configuration

Settings an operator can change: the array a module ships with, the field descriptors that become a form, and the write that saves the answers.

## Overview

A module's configuration is one PHP file that returns an array. It holds both the defaults you ship and the operator's saved answers, because saving rewrites that same file. There is no settings table and no migration step: the file is the state.

Two properties of that file drive most of the mistakes here: it is compiled, and it is readable source.

## Prerequisites

- A module that already loads. If you have none, start from [Your First Module](https://dev.wisecp.com/en/your-first-module).
- Write permission on the module directory for the web server user; without it every save fails silently at the file layer.
- How a form field's name becomes a request key, from [The Admin Form Builder](https://dev.wisecp.com/en/the-admin-form-builder).

## Structure

### The Shape of the File

The top level is yours apart from a handful of keys the platform reads, which have to be spelled exactly.

| Key | Read by | What it holds |
| --- | --- | --- |
| `meta.name` | The module list | A display name used when the language file has none |
| `meta.version` | You, and the update machinery | Your own version string |
| `meta.logo` | Logo resolution | A file name inside the module directory, or an absolute address |
| `settings` | Your code, and the settings save path | The operator's answers; every declared field lands here under its own key |
| `status` | The registry, on status-filtered loads | Whether the module is enabled, for the four types that store it here |
| `fields` | Server modules, on the product and service screens | Field descriptors shown on the product configuration form |
| `access_ps` | The addon settings screen | The privilege selection saved alongside the settings |
| `show_on_adminArea`, `show_on_clientArea` | The addon page router | Whether the addon opens a panel page, a customer page, or both |

```php
return [
    'meta' => [
        'name'    => 'AcmeDomains',
        'version' => '1.0',
        'logo'    => 'logo.png',
    ],

    // Ship every key your code reads, with a safe default. A key that only appears
    // after the first save is a key your code has to guard on every read.
    'settings' => [
        'username'      => '',
        'apiKey'        => '',
        'test-mode'     => 0,
        'nameservers'   => ['ns1.example.com', 'ns2.example.com'],
        'cost-currency' => 4,
    ],
];
```

### Declaring the Settings Fields

You do not write the form: you return an array of descriptors and the admin form builder turns it into one. The array key is the field name; the `name` entry inside it is the label. That pair is the most common thing to get backwards.

Which method you declare, and whether it is handed anything, depends on the type.

| Type | Method you declare | What the screen passes | Where the saved value comes from |
| --- | --- | --- | --- |
| Registrars | `config_fields($settings = [])` | The settings block of the current configuration | The argument |
| Payment | `config_fields()` | Nothing | `$this->config['settings']` |
| Addons | `fields()` | Nothing | `$this->config['settings']` |

```php
// The registrar form. The screen calls this with the saved settings block, so
// $data is populated. On a payment gateway or an addon the same method is called
// with NO argument, and $data would silently stay empty: read the property there.
public function config_fields($data = []): array
{
    return [
        // KEY is the field name. 'name' is the LABEL.
        'username' => [
            'name'        => $this->lang['username'] ?? 'Username',
            'description' => $this->lang['username-desc'] ?? '',
            'type'        => 'text',
            'value'       => $data['username'] ?? '',
            'placeholder' => 'api-user',
        ],

        'apiKey' => [
            'name'  => $this->lang['api-key'] ?? 'API Key',
            'type'  => 'password',
            'value' => $data['apiKey'] ?? '',
        ],

        // A checkbox. 'checked' is the current state, not the submitted value.
        'test-mode' => [
            'name'    => $this->lang['test-mode'] ?? 'Test Mode',
            'type'    => 'approval',
            'checked' => (bool) ($data['test-mode'] ?? false),
        ],

        // Shown only while the checkbox above is ticked.
        'test-endpoint' => [
            'name'         => $this->lang['test-endpoint'] ?? 'Test Endpoint',
            'type'         => 'text',
            'value'        => $data['test-endpoint'] ?? '',
            'parent'       => 'test-mode',
            'parentEffect' => 'hide',
        ],

        'mode' => [
            'name'    => $this->lang['mode'] ?? 'Mode',
            'type'    => 'dropdown',
            'value'   => $data['mode'] ?? 'live',
            'options' => ['live' => 'Live', 'sandbox' => 'Sandbox'],
        ],
    ];
}
```

## Walkthrough

### Ship the Defaults

1. Create `config.php` returning an array with a `meta` block and a `settings` block.
2. Put every key your code reads into `settings`, with an empty or harmless default. Never ship a real credential.
3. Reload the module list; the settings screen now has something to show.

### Declare the Form

1. Add `config_fields($data = [])` to your class, or `fields()` if you are writing an addon.
2. Return one descriptor per setting, keyed by the setting name, with the current value read from the source your type provides.
3. Open the module's settings page. The generic template finds your method and builds the form, submit button and action address included.
4. Change a value and save. The answers arrive under a single request key, `fields`, keyed by your field names.

### Write It Back

1. Merge the posted values into the loaded array rather than replacing it: the write is a full-file write, so anything you drop is gone.
2. Encrypt secrets on the way in, and keep the stored value when the field arrives masked or empty.
3. Write through the file manager, which invalidates the compiled copy.
4. Reload. If you read back the old value, the write went through but the compiled copy did not.

## Reference

### Writing the File

```php
// The shared trait, used by server, registrar, product and social login modules.
// $auto_status = true turns the module on when the array carries a non-empty settings block.
protected function save_config($data = [], $auto_status = true);

// Server modules narrow it: no auto-status flag, and a strict boolean return.
public function save_config($data = []): bool;

// Addons narrow it the same way, and also assign the array to $this->config.
public function save_config($data = []): bool;

// What all of them call underneath. It invalidates the compiled copy of any .php target.
public static function file_write($file, $data = null, $mode = 'w', $flags = 0);

// Array to source. ['pwith' => true] wraps it as a complete PHP file.
public static function array_export($array = [], $options = []);

// Platform configuration, not module configuration. Slash paths into the files
// under the configuration directory.
public static function get($arg = null);
public static function set($key, $values, $merge = false): array|false;
public static function save($name = '', $data = []): bool;

// Database-backed settings, keyed by name. A module admin area uses these
// instead of its own file.
public static function getd($name = '');
public static function setd($name = '', $content = '');
```

### Field Descriptor Keys

- **type**: One of `text` (the default), `password`, `textarea`, `dropdown`, `radio`, `switch`, `approval`, `file`, `output` and `javascript`. An output field prints free markup and is never saved.
- **name**: The label shown beside the field. Not the field name: that is the array key.
- **value**: The current value for text-like and dropdown fields, and the submitted value for a switch. Checkboxes use `checked` for their state instead.
- **options**: A `value => label` map for a dropdown or a radio group. A comma-separated string is accepted and expanded into a map with identical keys and labels.
- **description · description_pos · is_tooltip**: Help text, on the right by default and beside the label with `'L'`. With `is_tooltip` it collapses into a question-mark icon.
- **parent · parentEffect · parentValue**: Show or disable this field based on another one. The effect is `hide`, `disable` or `collapse`; with a radio parent, `parentValue` lists which options reveal it. The parent must be declared before the child.
- **width · wrap_width**: Percentages for the input itself and for its row. A wrap width of one hundred is treated as unset.
- **advanced_selector · multiple · rows · disabled**: A searchable dropdown, a multi-value field, the height of a textarea, and a read-only field. The searchable dropdown is the one that can load its options from one of your own methods.
- **fieldOptions · rowOptions**: Passed straight through to the form builder for that field and its row. This is the escape hatch for attributes the descriptor has no key for.

## Example

The full round trip: what arrives, what is written, and what your code reads back. The saving half is an override of the settings controller, so the merge is visible.

```php
public function controller_settings($extraFields = []): array
{
    // Everything the form posted, under one key, named after your descriptor keys.
    $fields = \Filter::POST("fields") ?: [];

    // Start from what is already on disk: the write below replaces the whole file.
    $config = $this->config;

    $config['settings']['username']  = \Filter::html_clear((string) ($fields['username'] ?? ''));
    $config['settings']['mode']      = in_array($fields['mode'] ?? '', ['live', 'sandbox'], true)
        ? $fields['mode'] : 'live';

    // An unticked checkbox is ABSENT from the post, so absence is the value "off".
    $config['settings']['test-mode'] = (int) ($fields['test-mode'] ?? 0) === 1 ? 1 : 0;

    // Secrets: encrypt on the way in, and keep the stored value when the field came
    // back masked or empty, which is what the screen sends for an unchanged secret.
    $posted = (string) ($fields['apiKey'] ?? '');
    if ($posted !== '' && !str_starts_with($posted, '*'))
        $config['settings']['apiKey'] = $this->encode_str($posted);

    // Full-file write, through the manager that invalidates the compiled copy.
    \FileManager::file_write($this->dir . 'config.php', \Utility::array_export($config, ['pwith' => true]));

    return ['status' => "successful", 'message' => \Language::gc("admin/ac-settings/successful1")];
}
```

```php
private function credentials(): array
{
    $settings = $this->config['settings'] ?? [];

    return [
        // Null-safe on every read: a key can be missing on an installation that
        // upgraded from an older version of your module.
        'username' => (string) ($settings['username'] ?? ''),
        'apiKey'   => $this->decode_str((string) ($settings['apiKey'] ?? '')),
        'sandbox'  => (int) ($settings['test-mode'] ?? 0) === 1,
    ];
}
```

The same values read without building the module, which is what a listing screen or a hook does:

```php
// Config() reads the static cache only, so the load has to happen first.
// The third argument keeps the class file out of it.
$record = Modules::Load("Registrars", "AcmeDomains", true);

$mode = $record["config"]["settings"]["mode"] ?? 'live';

// The secret is NOT readable from here: decoding is a method on the instance.
// If you need the plain value, build the module and ask it.
```

## Pitfalls

> **A configuration file is compiled PHP**
> 
> It is read back with an include, so the compiled copy is served until something invalidates it. The file manager does; a raw write or a rename does not. The operator saves, reloads, sees the old value, and no log explains it.

> **Saving replaces the whole file**
> 
> Build the new array from the one already loaded, change the keys you own and leave the rest alone. Passing only the settings block wipes the metadata, the status and everything else in one save.

> **A field that was not posted can be written as false**
> 
> The addon settings path walks your declared fields and stores `false` for every one the post did not contain. An unticked checkbox sends nothing, so that is right for checkboxes and wrong for anything shown conditionally. Derive the fields you declare from the same source you read, never a hand-written list.

> **Encrypt secrets, and never paste one in by hand**
> 
> An API key belongs in the array encrypted, through the module's own helper, so the sub-key bound to this installation is used. A value typed straight into the file cannot be decrypted and reads as garbage; enter it through the settings screen.

> **Module configuration and platform configuration are different things**
> 
> The module file is yours and travels with the module. The platform configuration files hold installation-wide settings, including which module of each single-choice type is active. A module writes only to its own file, and reads the platform files.

## Related Articles

- [Module Anatomy](https://dev.wisecp.com/en/module-anatomy)
- [Module Language Files](https://dev.wisecp.com/en/module-language-files)
- [Reading and Writing Configuration](https://dev.wisecp.com/en/reading-and-writing-configuration)
- [The Admin Form Builder](https://dev.wisecp.com/en/the-admin-form-builder)
- [Module Lifecycle](https://dev.wisecp.com/en/module-lifecycle)
- [Filtering User Input](https://dev.wisecp.com/en/filtering-user-input)


# Module Language Files

https://dev.wisecp.com/en/module-language-files

Give a module its own translatable strings. One file per language, how the right one is chosen, and why a missing key does not fall back to English.

## Overview

A module carries its strings in a `lang` directory: one PHP file per language code, each returning a flat array. The loader picks exactly one file and hands the whole array to your class as a property.

The fallback is per file, not per key. If the active language has a file, that file is all you get, and any key it does not define is absent. Keeping every file on the same key list is what makes the model safe. Measured here: of 297 modules that ship an English and a Turkish file, 295 pairs carry identical top level keys. The other two are Turkish files holding keys English never defines, which is the failure this article is about.

## Prerequisites

- A module directory that already loads; the language file is read on every load, class or not.
- An English file: the last resort for every language with no file of its own.
- Knowing which strings are yours and which belong to the platform, covered in [Translations and Language Files](https://dev.wisecp.com/en/translations-and-language-files).

## Structure

### The Directory

```bash
coremio/modules/{Type}/{Name}/lang/
├── en.php     # the fallback for every language without a file of its own
├── tr.php     # same keys, translated
└── de.php     # add a file per language you support; the code is the file name
```

Every file returns a flat array. Two keys are read by the platform; the rest are yours to name.

```php
return [
    // Read by the platform: the label and the blurb in the module list.
    'name'        => 'Acme Domains',
    'description' => 'Domain registration through the Acme API. An API key is required.',

    // Yours. Group them with a prefix so a long file stays navigable.
    'username'      => 'Username',
    'username-desc' => 'The API account this installation connects with.',
    'api-key'       => 'API Key',
    'test-mode'     => 'Test Mode',

    'error-no-domain' => 'No domain name is bound to this service.',
    'error-refused'   => 'The provider refused the request.',

    // Placeholders are positional, and the caller passes the values.
    'error-locked' => 'The domain %s is locked and cannot be transferred.',
];
```

### How the File Is Chosen

The language is resolved first, then the file. The two steps are separate, and only the second has a fallback.

| Order | What decides the language | Applies when |
| --- | --- | --- |
| 1 | The language argument you passed | You named one explicitly |
| 2 | The registry's shared language marker | A module was already loaded in some language this request |
| 3 | The active interface language | The marker is still empty |
| 4 | The installation's default locale | No interface language is selected |
| 5 | English | Nothing else answered |

```php
// Exactly one file is included. There is no per-key merge with English.
if (file_exists($path . 'lang' . DS . $lang . '.php'))
    $strings = include $path . 'lang' . DS . $lang . '.php';
elseif (file_exists($path . 'lang' . DS . 'en.php'))
    $strings = include $path . 'lang' . DS . 'en.php';

// Neither present: the property is an empty array, and every read falls to its default.
```

## Walkthrough

### Add the Files

1. Create `lang/en.php` returning an array with `name` and `description`.
2. Copy it to `lang/tr.php` and translate the values, leaving every key as it was.
3. Reload the module list. The module now shows its own label instead of its directory name.

### Read a String

1. Inside the class, read from the language property with a null-safe default. The key may be missing on an installation running an older translation.
2. For a string with a value in it, keep the placeholder in the language file. Format at the call site, so translators see the whole sentence.
3. Change the panel language and reload. The same code now gives the other file's value.

### Label a Configuration Entry

1. In a settings field descriptor, put the language value into the `name` and `description` entries. That array is built at runtime and can read the property.
2. Where a server module's configuration file holds static data rather than code, write the placeholder form `{lang.key}` instead. The module resolves it against the same array.
3. Open the settings screen in both languages and confirm each label follows.

## Reference

### The Loader

```php
// Module strings. Loads the file itself, so no prior load call is needed.
// $lang is a language code such as 'en' or 'tr'; empty means "resolve it".
public static function Lang($type, $module, $lang = '');

// Module configuration. Reads the static cache ONLY: returns null when nothing loaded it.
// This asymmetry with Lang() is the single most surprising thing about the pair.
public static function Config($type, $module);

// The display label: lang['name'], then config['name'], then the directory name.
// It performs the load for you.
public static function getName(string $type, string $module): string;

// Platform strings, NOT module strings. A module uses these for shared wording only.
public static function g($key = '', $replaces = [], $slang = ''): array|string|int|bool;
public static function gc($name = '', $replaces = [], $slang = ''): array|string|int|bool;
public static function selected(): string;
```

- **Lang()**: Returns the array, or an empty array when neither the requested file nor the English file exists. It never returns null, so reading the result is safe.
- **Config()**: Returns null when the module has not been loaded, because it only reads the cache. Unlike the language loader, it does not go to disk.
- **The shared language marker**: One static value for the whole registry, set to the last language anyone asked for. Passing an explicit language changes it for every later call that omits one.
- **When a reload happens**: Only when the requested language differs from the marker, or the module has no cached strings yet. A module already cached is not re-read when the marker moves.

### Keys the Platform Reads

- **name**: The label in the module list, and everywhere a module is named. Without it: the configuration's name entry, then the directory name.
- **description**: The sentence under the label in the module list. One line in the operator's language: what it connects to and what it needs.
- **{lang.key}**: Accepted where a server module's configuration file holds a label as static data: a card item, an add-on parameter. Resolved against this module's own array; an unknown key appears as written, not as an empty string.
- **Sorting side effect**: A type listing is ordered by the resolved display name, so translating `name` also moves the module in that language's list.

## Example

Both files, then the two places the strings are read: the class, and a configuration entry that cannot call PHP.

```php
// lang/en.php
return [
    'name'            => 'Acme Domains',
    'description'     => 'Domain registration through the Acme API.',
    'api-key'         => 'API Key',
    'api-key-desc'    => 'Found under Account, API in the provider panel.',
    'error-locked'    => 'The domain %s is locked and cannot be transferred.',
    'addon-privacy'   => 'WHOIS Privacy',
];

// lang/tr.php : the SAME keys, in the same order, values translated.
return [
    'name'            => 'Acme Alan Adları',
    'description'     => 'Acme API üzerinden alan adı kaydı.',
    'api-key'         => 'API Anahtarı',
    'api-key-desc'    => 'Sağlayıcı panelinde Hesap, API altında bulunur.',
    'error-locked'    => '%s alan adı kilitli ve transfer edilemez.',
    'addon-privacy'   => 'WHOIS Gizliliği',
];
```

```php
public function config_fields($data = []): array
{
    return [
        'apiKey' => [
            // Always with a default: a translation shipped before this key existed
            // would otherwise render an empty label.
            'name'        => $this->lang['api-key'] ?? 'API Key',
            'description' => $this->lang['api-key-desc'] ?? '',
            'type'        => 'password',
            'value'       => $data['apiKey'] ?? '',
        ],
    ];
}

public function transfer(): array|bool
{
    if ($this->is_locked()) {
        // The placeholder lives in the language file; the value is applied here,
        // so a translator sees the whole sentence rather than two fragments.
        $message = sprintf(
            $this->lang['error-locked'] ?? 'The domain %s is locked.',
            (string) ($this->service['domain'] ?? ''),
        );

        throw new \Exception($message);
    }

    // A platform string, not a module string: the wording is shared with the rest
    // of the panel and does not belong in this module's files.
    if (!$this->credentials()['apiKey'])
        throw new \Exception(\Language::gc("admin/modules/error-missing-credentials"));

    return ['status' => 'SUCCESS'];
}
```

```php
return [
    'addon-params' => [
        // Static data, resolved against lang/ when the screen renders it.
        'whois_privacy' => [
            'label'       => '{lang.addon-privacy}',
            'description' => '{lang.addon-privacy-desc}',
            'type'        => 'toggle',
        ],
    ],
];
```

## Pitfalls

> **A missing key does not fall back to English**
> 
> A key present in English and absent from the active language is absent, full stop. It shows whatever default your read supplied. Add a key to every language file in the same change, and always read with a default.

> **Asking for a specific language moves a shared marker**
> 
> The registry keeps one language marker for the whole request, and requesting a module in another language sets it. Measured: a module already cached kept returning its first language after the marker moved. Do not request a specific language on a display path unless the whole page is in it.

> **Module strings and platform strings are different systems**
> 
> Your own wording lives in the module's files, read from the language property. Wording shared with the panel comes from the platform's translation helpers. A module cannot add keys to those, so anything you invent lives in your own files.

> **Do not build a sentence out of fragments**
> 
> Concatenating two keys around a value gives word order that works in one language only. Keep the whole sentence in one key with a positional placeholder, and apply the value at the call site.

> **The placeholder form only works where it is resolved**
> 
> Writing a language placeholder into an arbitrary configuration value does nothing. It is expanded only where the resolver is called: a server module's card item and add-on parameter labels. Anywhere else it reaches the screen as literal text.

## Related Articles

- [Module Anatomy](https://dev.wisecp.com/en/module-anatomy)
- [Module Configuration](https://dev.wisecp.com/en/module-configuration)
- [Translations and Language Files](https://dev.wisecp.com/en/translations-and-language-files)
- [The Module System](https://dev.wisecp.com/en/the-module-system)
- [Your First Module](https://dev.wisecp.com/en/your-first-module)
- [Translating a Theme](https://dev.wisecp.com/en/translating-a-theme)


# Module Assets and Logo

https://dev.wisecp.com/en/module-assets-and-logo

Ship stylesheets, scripts and images inside your module, link them without hard-coding a path, and give it a panel icon.

## Overview

A module's static files live in the module directory and travel with it. Nothing is copied or registered. The directory is reachable over the web, and the base class hands you its address.

Two properties do all the work and are easy to mix up: a filesystem path and a public URL. The logo is separate, with its own resolution order, and the one asset the platform finds by itself.

## Prerequisites

- A module extending one of the base classes, so the directory and URL properties are populated. Plain-class types build them themselves.
- A hook file, if the asset must reach a page the module does not own. See [Registering Hooks from a Module](https://dev.wisecp.com/en/registering-hooks-from-a-module).
- No build step, no manifest, no asset pipeline.

## Structure

### The Assets Directory

```bash
coremio/modules/{Type}/{Name}/
├── logo.svg            # the panel icon. Module ROOT, not assets/
└── assets/
    ├── style/          # css
    ├── js/             # javascript
    └── images/         # images used INSIDE your interface, never the logo
```

Do not repeat the module name in file names. The directory already says which module a file belongs to, so `assets/js/app.js` is the convention. A relative address such as `url(../images/icon.svg)` resolves in a stylesheet, because the file is served from its real location.

### The Two Paths

- **$this->dir**: Absolute filesystem path to the module directory, ending in a separator. Read files, check existence, or take a modification time for cache busting.
- **$this->url**: Absolute public URL of the same directory, ending in a slash. Everything in markup is built from it: `$this->url . 'assets/style/app.css'`.
- **Never a literal path**: Both are set by the base constructor, before your first method runs. A literal path works on your machine only.

## Walkthrough

### Add a Stylesheet

1. Create `assets/style/app.css` in the module directory.
2. Prefix class names with something specific to the module. The panel's stylesheet is on the same page, and a generic name collides silently.
3. Reference images with a relative address, and put them in `assets/images`.

### Load It on the Right Page

1. Decide where it belongs. A page your module builds returns its own styles and scripts; anything else goes through a head hook.
2. In the hook body, return an empty string unless the page needs the asset.
3. Build the address from the URL property, with a cache-busting value from the file's modification time.
4. Confirm it appears in the network panel on that page, and not on an unrelated one.

### Add a Logo

1. Put an image called `logo` in the module root, with an `svg`, `webp`, `png`, `jpg`, `jpeg` or `gif` extension. It is found by name.
2. For another file name, or one in a subdirectory, name it in the configuration under `meta.logo`.
3. Reload the module list; the icon appears next to the name.

## Reference

### Logo Resolution

```php
// On the instance: resolves for this module's own name and type.
public function logo(): string;

// Statically, when you have no instance. Note the type DEFAULT: a call that omits it
// looks the module up as a server module.
public static function logo(string $name = '', $type = 'Servers'): string;
```

| Order | Source | How it is turned into an address |
| --- | --- | --- |
| 1 | `meta.logo` in the configuration, or a top level `logo` entry | Used unchanged when it starts with a protocol. Otherwise resolved against the module directory, so `images/logo.png` works |
| 2 | A file called `logo` in the module root with a supported extension | Found by pattern, resolved against the module directory. Keep one file |
| 3 | A file named after the module, lowercased, in the shared admin logo directory | The last resort. A module with no image still shows a brand icon |
| 4 | Nothing matched | An empty string; the caller shows a placeholder |

### Asset Injection Points

- **ui:admin.head.css**: Return a complete stylesheet tag, or an empty string. It appears in the panel's head, once per listener.
- **ui:admin.head.js**: The same, for scripts. Several tags in one string is normal: a configuration object and the script that reads it.
- **ui:client.head.css**: The customer-facing equivalent. Keep them apart: a panel stylesheet on a public page leaks your interface into the theme.
- **ui:client.head.js**: Scripts for the customer side. Anything on a public page must survive a signed-out visitor.
- **page_styles · page_scripts**: Keys on the array a module admin page returns. Preferred over a hook for a page your module owns: the gating is implicit.
- **Cache busting**: Append the file's modification time as a query value, falling back to `meta.version` in the configuration. The instance has no version property, so read it from `$this->config`. Without it an operator gets the old file after an update and reports an unreproducible bug.

## Example

The whole pattern in one hook file. A page gate keeps the asset off the rest of the panel. The configuration is handed to the script, not scraped from the markup.

```php
// This file runs on EVERY request, for this module, enabled or not. Only register here.
$acme_on_page = fn () => in_array(Controllers::$cname ?? '', ['tickets', 'services'], true);

Hook::add('ui:admin.head.css', 1, function () use ($acme_on_page) {
    if (!$acme_on_page()) return '';

    $m = Modules::getInstance('Addons', 'Acme');

    // dir for the file on disk, url for the address in the markup.
    $v = @filemtime($m->dir . 'assets' . DS . 'style' . DS . 'app.css') ?: ($m->config['meta']['version'] ?? '1.0');

    return '<link rel="stylesheet" href="' . $m->url . 'assets/style/app.css?v=' . $v . '">';
});

Hook::add('ui:admin.head.js', 1, function () use ($acme_on_page) {
    if (!$acme_on_page()) return '';

    $m    = Modules::getInstance('Addons', 'Acme');
    $lang = $m->lang;

    // Everything the script needs, encoded once. Escaping flags matter: this string
    // is printed inside a script tag in an HTML document.
    $config = Utility::jencode([
        'endpoint' => Controllers::$init->ControllerURI(),
        'i18n'     => [
            'run'    => $lang['btn-run'] ?? 'Run',
            'failed' => $lang['err-failed'] ?? 'Request failed',
        ],
    ], JSON_HEX_TAG | JSON_HEX_AMP | JSON_HEX_APOS | JSON_HEX_QUOT);

    $v = @filemtime($m->dir . 'assets' . DS . 'js' . DS . 'app.js') ?: ($m->config['meta']['version'] ?? '1.0');

    return '<script>window.Acme = ' . $config . ';</script>'
         . '<script src="' . $m->url . 'assets/js/app.js?v=' . $v . '"></script>';
});
```

The reading side, which never guesses an address and never re-derives a label:

```javascript
(function () {
    var cfg = window.Acme;
    if (!cfg) return;                     // asset loaded on a page it was not meant for

    document.querySelectorAll('[data-acme-run]').forEach(function (btn) {
        btn.textContent = cfg.i18n.run;
        btn.addEventListener('click', () => WcpRequest(cfg.endpoint, {
            data: { operation: 'use_addon_method', method: 'run', id: btn.dataset.acmeRun },
        }));
    });
})();
```

The logo, declared only when the file is not called `logo` at the module root:

```php
return [
    'meta' => [
        'name'    => 'Acme',
        'version' => '1.0',

        // Resolved against the module directory. A subpath is allowed, and an
        // address that starts with a protocol is used exactly as written.
        'logo'    => 'assets/images/brand.svg',
    ],
];
```

## Pitfalls

> **The logo cache is keyed by module name alone, not by type**
> 
> Two modules of different types that share a name share one resolved logo per request. The winner is whichever was asked for first. Give a module a name no other type uses.

> **The logo helper does not load the module**
> 
> It reads the configuration from the cache, so a configured logo name counts only when something already loaded that module. Called cold it falls through to a file called `logo` — which is why that name is safe. From an instance it is always safe.

> **A hard-coded modules path works only on your machine**
> 
> The same file has a different address on every installation. The application may live in a subdirectory, on another host, or behind another protocol. Both properties account for that; a literal string does not, and it fails as a missing stylesheet elsewhere.

> **An ungated asset is loaded on every screen**
> 
> Hook files run for every module directory on every request, enabled or not. A head listener with no page condition adds them to every panel page. Gate on the controller, and have the script return early when its configuration is absent.

> **The logo is not an interface image**
> 
> The panel icon stays at the module root, where the resolver looks for it. Images used inside your screens belong in the assets directory. Apart, re-branding is one file, not a hunt through the interface.

## Related Articles

- [Module Anatomy](https://dev.wisecp.com/en/module-anatomy)
- [Registering Hooks from a Module](https://dev.wisecp.com/en/registering-hooks-from-a-module)
- [Adding an Admin Page](https://dev.wisecp.com/en/adding-an-admin-page)
- [Module Configuration](https://dev.wisecp.com/en/module-configuration)
- [Admin JavaScript Library](https://dev.wisecp.com/en/admin-javascript-library)
- [Theme Assets](https://dev.wisecp.com/en/theme-assets)


# Adding an Admin Page

https://dev.wisecp.com/en/adding-an-admin-page

Give a module its own page in the admin panel with two files. An area class returns the HTML; one line in `router.php` registers it.

## Overview

A module that manages its own records or runs a bulk job needs a real page. That page does not belong in the core: you declare a class and register it, and the route, menu entry, privilege check, breadcrumb and theme shell come for free. New to modules? See [Your First Module](https://dev.wisecp.com/en/your-first-module).

The dispatcher is `coremio/controllers/admin/module-page.php`; slug routes resolve to it and the result goes to the addon theme wrapper.

## Structure

- **classes/ModuleAdminArea.php**: The base class and registry: `register()`, route wiring, menu hook, instance helpers.
- **admin/module-page.php**: The internal dispatcher: resolves `page_*` and `op_*`, invisible in the URL.
- **tools/addons-area.php**: The theme wrapper: title bar, buttons, plugins, styles, scripts, modals.
- **{module}/AdminArea.php + router.php**: The two files you write. `Router::loadModuleRouters()` reads the router file before any route is matched.

## Walkthrough

### Declare the Area Class

1. Create `AdminArea.php` in your module directory, namespaced `WISECP\Modules\{Type}\{Name}`.
2. Extend `\ModuleAdminArea` and return a manifest from the static `manifest()` method.
3. Override `available()` when the page only makes sense under a condition; it guards the menu entry too.

### Register It

1. Create `router.php` next to it: include the class file, then call `\ModuleAdminArea::register(AdminArea::class)`.
2. Nothing else belongs there. Registration adds three routes, stores the manifest and hooks the menu entry.

### Write a Page Method

1. `page_home()` answers the area root; every method takes the trailing URL segments as an array.
2. Build the HTML with heredoc; read your strings from `$this->lang`.
3. Point links at yourself with `$this->link()`, never a hand-written path.

### Write an Operation

1. Name the method `op_{name}`; only that prefix is callable.
2. Post to the area URL with an `operation` parameter; a thrown exception becomes the standard JSON error.
3. Start with `$operation->demo()`, read input through `Filter::init()`, finish with `$operation->output()`.

### Store Area Settings

1. Use `settings()`, `setting()` and `save_settings()`, not a file of your own.
2. `save_settings()` merges into the current values, so a partial save keeps the rest, then fires `action:module.area_settings_saved`.

## Reference

### Base Class API

```php
class ModuleAdminArea
{
    public string $module_type = '';   // resolved from the namespace
    public string $module_name = '';   // resolved from the namespace
    public array  $lang        = [];   // {module}/lang/{selected}.php, falling back to en.php

    public function __construct();

    // You override this one. Anything else with a default is optional.
    public static function manifest(): array;

    public static function register(string $areaClass): void;
    public static function get(string $type, string $name): ?array;

    public function available(): bool;                                  // default true
    public function slug(): string;
    public function link(array $params = []): string;
    public function module_dir(): string;                               // filesystem path, trailing separator
    public function module_url(): string;                               // public URL of the module directory

    public function settings(): array;
    public function setting(string $key, mixed $default = null): mixed;
    public function save_settings(array $values): void;

    protected function license_state_badge(string $slug): string;
}
```

> **Eleven module types, not sixteen**
> 
> The accepted types are Servers, Payment, Registrars, Product, Addons, SMS, Mail, Authentication, Pipe, Imports and Fraud. For anything else `register()` does nothing, silently.

### What the Manifest Accepts

- **title**: Page and browser title; the menu label when `menu.name` is absent.
- **slug**: The URL segment, default the lowercased module name. `Filter::route()` keeps `a-zA-Z0-9`, hyphen, underscore and dot.
- **privileges**: Privilege keys, checked once for the whole area. Empty means no check.
- **menu**: `['path' => ['PRODUCTS', 'GROUP_HOSTING_SERVER'], 'name' => 'Hetzner Cloud']`. The path walks down the tree; omit it and no entry is created.
- **type, name, class**: Written by `register()` from the namespace and the argument; do not set them.

### URL Scheme and Method Resolution

```php
// Three routes, most specific first. $target is module-page/{Type}/{Name}.
$router->add($slug . '-2', $slug . '/(?)/(?)', $target . '/(1)/(2)');
$router->add($slug . '-1', $slug . '/(?)',     $target . '/(1)');
$router->add($slug,        $slug,              $target);

// Which is why link() derives the route key from the parameter count:
//   $this->link()                     -> /{admin}/{slug}
//   $this->link(['configuration'])    -> /{admin}/{slug}/configuration
//   $this->link(['logs', 'archive'])  -> /{admin}/{slug}/logs/archive
```

| Request | Method called | Argument |
| --- | --- | --- |
| `GET /{admin}/{slug}` | `page_home()` | empty array |
| `GET /{admin}/{slug}/configuration` | `page_configuration()` | `['configuration']` |
| `GET /{admin}/{slug}/logs/archive` | `page_logs_archive()`, else `page_logs()` | `['logs', 'archive']` |
| `POST` with `operation=sync_prices` | `op_sync_prices(Operation $operation)` | the `Operation` |
| no matching method | 404 page | nothing appears |

Hyphens, dots and commas in a segment become underscores: `/{slug}/price-list` resolves to `page_price_list()`. Operation names too.

### What a Page Method May Return

A string becomes the page body; an array passes the keys below to the wrapper.

| Key | Type | Effect |
| --- | --- | --- |
| `content` | string | The page body. Empty shows a `No module area content is available.` alert. |
| `page_title` | string | Overrides the manifest title. |
| `page_title_buttons` | array | A list of button descriptors; plain HTML shows nothing. |
| `page_title_after` | string | Free HTML after the title and buttons. |
| `page_title_logo` | string | An image URL, or raw HTML when it contains `<`. |
| `content_layout` | string | `panel` by default; `plain` drops the panel. |
| `content_data_class` | string | Extra class on the content wrapper (plain layout). |
| `breadcrumbs` | array | Appended to the Dashboard + area title trail. |
| `plugins` | array | Merged onto the wrapper defaults. |
| `page_styles`, `page_scripts`, `modals` | string | Head, footer and modal area. |

### The Signed-in Administrator

```php
public static function LoginData($type = 'member', $isRemembered = false, $recheck = false);
```

```php
$adminId = (int) (\UserManager::LoginData('admin')['id'] ?? 0);   // 0 outside the panel (CLI, cron)

// The operator picker for an assignment field.
$staff = \Admin::list();          // [id => ['id' => 1, 'full_name' => 'Jane Doe'], ...]
```

## Example

A two-file area for an imaginary `Acme` module.

```php
<?php
namespace WISECP\Modules\Servers\Acme;

use AdminFormBuilder;
use Exception;
use Filter;
use Operation;
use User;
use UserManager;
use WDB;

class AdminArea extends \ModuleAdminArea
{
    public static function manifest(): array
    {
        return [
            'title'      => 'Acme',
            // 'slug'    => 'acme',                  // default: strtolower(module name)
            'privileges' => ['PRODUCTS_OPERATION'],
            'menu'       => ['path' => ['PRODUCTS', 'GROUP_HOSTING_SERVER'], 'name' => 'Acme'],
        ];
    }

    /** Hides the menu entry and 404s the page while no Acme server exists. */
    public function available(): bool
    {
        static $available = null;

        if ($available === null)
            $available = (bool) WDB::select('id')->from('servers')
                ->where('type', '=', 'Acme', '&&')
                ->where('status', '=', 'active')
                ->build();

        return $available;
    }

    public function page_home(array $params): array
    {
        $L    = $this->lang;
        $conf = $this->settings();

        $form = new AdminFormBuilder('acmeAreaForm', $this->link(), ['disableStickySubmit' => true]);
        $form->addHidden('operation', 'save_settings');
        $form->addAmount('profit_rate', $L['profit-rate'] ?? '', (string) ($conf['profit_rate'] ?? 25));
        $form->addSwitch('auto_sync', $L['auto-sync'] ?? '', $L['auto-sync-desc'] ?? '', '1', (int) ($conf['auto_sync'] ?? 0) === 1);

        return [
            'content'          => $form->render(),
            'page_title'       => $L['area-title'] ?? 'Acme',
            'page_title_after' => $this->license_state_badge('acme'),
        ];
    }

    public function op_save_settings(Operation $operation): bool
    {
        $operation->demo();

        $values = [
            'profit_rate' => (float) Filter::init('POST/profit_rate', 'amount'),
            'auto_sync'   => (int) Filter::init('POST/auto_sync', 'rnumbers') === 1,
        ];

        if ($values['profit_rate'] < 0)
            throw new Exception($this->lang['err-negative-rate'] ?? 'The profit rate cannot be negative.');

        $this->save_settings($values);

        $adminId = (int) (UserManager::LoginData('admin')['id'] ?? 0);
        User::addAction($adminId, 'update', 'acme-settings-updated', ['changes' => $values]);

        return $operation->output([
            'status'   => 'successful',
            'redirect' => 'reload',
        ]);
    }
}
```

```php
<?php
namespace WISECP\Modules\Servers\Acme;

include_once __DIR__ . DS . 'AdminArea.php';

\ModuleAdminArea::register(AdminArea::class);
```

The dispatcher decides whether your method is reached at all:

```php
$area_info = ModuleAdminArea::get($type, $name);
if (!$area_info) return $this->page_404();

if (($area_info['privileges'] ?? []) && !Admin::isPrivilege($area_info['privileges'])) return 'Access Denied';

$area = new $area_info['class']();
if (!$area->available()) return $this->page_404();

if ($operation = Filter::init('REQUEST/operation')) return $this->area_operation($area, $operation);

$page    = Filter::route($this->params[2] ?? '') ?: 'home';
$subpage = Filter::route($this->params[3] ?? '');

$filter_method = fn ($param) => str_replace(['-', '.', ','], '_', $param);
$method        = $filter_method('page_' . $page);            // page_logs
$method2       = $filter_method($method . '_' . $subpage);   // page_logs_archive

$page_params = array_slice($this->params, 2);

if ($subpage && method_exists($area, $method2)) $result = $area->$method2($page_params);
elseif (method_exists($area, $method))          $result = $area->$method($page_params);
else return $this->page_404();

if (!is_array($result)) $result = ['content' => (string) $result];
```

## Pitfalls

> **A page method that ends silently is a fatal, not a routing problem**
> 
> An undefined property or a forgotten import produces no output, and `try/catch` on `Exception` will not catch it because an `Error` is not an `Exception`. The commonest case is `Admin::$data['id']`: that static property does not exist, so read the operator with `UserManager::LoginData('admin')`.

> **A slug that collides with a core route breaks link generation**
> 
> Real controller files win at dispatch, so an area registered as `services` is unreachable and overwrites that key for every link. Pick a slug no core route uses.

> **A missing parent hides the menu entry without a word**
> 
> The menu hook walks down the tree and returns as soon as a step is absent. If the parent was trimmed by the administrator's privileges your entry is not added, while the page stays reachable by URL.

> **Operations are blocked while the licence is not active**
> 
> The area dispatcher checks the licence before it looks for your method and answers with a JSON error. Pages are affected too, at the view layer.

> **Area settings live under the module name**
> 
> The row key in the configurations table is the module name as the directory spells it, so renaming the directory orphans the saved settings. Migrate by overriding `settings()`.

## Related Articles

- [Module Anatomy](https://dev.wisecp.com/en/module-anatomy)
- [Registering Hooks from a Module](https://dev.wisecp.com/en/registering-hooks-from-a-module)
- [Exposing API Endpoints](https://dev.wisecp.com/en/exposing-api-endpoints)
- [The Admin Form Builder](https://dev.wisecp.com/en/the-admin-form-builder)
- [Operations](https://dev.wisecp.com/en/operations)
- [Building Links and Routes](https://dev.wisecp.com/en/building-links-and-routes)


# Exposing API Endpoints

https://dev.wisecp.com/en/exposing-api-endpoints

Publish your module's capabilities as REST endpoints by adding route entries to one hook, with no core edit.

## Overview

Authentication, per-endpoint permissions, rate limiting, CORS, idempotency and request logging already exist. You declare which address does what, and the Kernel does the rest before it calls your method.

What the installation owner sees is a checkbox: each endpoint becomes a line on the API credentials screen, and a key only reaches the endpoints its owner ticked.

```text
Settings -> API credentials -> Create
  [ ] GET    /admin/mymodule/items
  [x] POST   /admin/mymodule/items/{id}/rebuild      <- only this one was granted
  [ ] DELETE /admin/mymodule/items/{id}
```

## Prerequisites

- A module with a `hooks.php` file; see [Registering Hooks from a Module](https://dev.wisecp.com/en/registering-hooks-from-a-module).
- For write endpoints, an existing panel operation to bridge to rather than a second implementation.
- Module `src/` classes are not autoloaded; `include_once` anything `hooks.php` references.

## Structure

- **filter:api.routes**: The single connection point, fired once per audience with the route list by reference. Append; the return is ignored.
- **Routes / ClientRoutes / ModuleRoutes**: The three registries; each runs the hook with its own `$audience` (see the surface table).
- **Kernel::dispatch()**: Sees a group beginning with `Module:` and hands the request to your module instance.
- **Config::set()**: Publishes your checkboxes into the in-memory permission catalogue.

## Walkthrough

### Declare the Surface in One Place

1. Create `src/ApiSurface.php` with a `GROUP` constant and a `map()` listing every endpoint: verb, path, action, target.
2. Derive the routes and the permission catalogue from that list, so route, scope and checkbox cannot drift.
3. Write literal paths before their parametric twins. The router takes the first match at a given segment count.

### Register the Routes

1. In `hooks.php`, add a listener on `filter:api.routes` and return when the audience is not yours.
2. Append your tuples to `$routes`. Both parameters arrive by reference, so append, do not return.
3. Publish the permission catalogue from the **body** of `hooks.php`, outside every listener.

### Write the Handler Methods

1. Add one `api_{action}` method per endpoint. A hard `method_exists` check means `__call` will not answer.
2. Give every method the same one-line body forwarding to your bridge, and generate them from the declaration.
3. Return a `Response`, or a legacy envelope array that the Kernel normalises.
4. For a list, keep the envelope identical to the core resources: `page`, `limit` and `search` in, `meta.total`, `meta.page`, `meta.limit` and `meta.next_page` out.

### Bridge Writes to the Panel Operation

1. Mirror the superglobals, call `op_{name}`, capture what it prints, then restore them in a `finally` block.
2. Translate the printed envelope: drop `data.html`, move `message` into `meta`, and turn a thrown exception into a validation error.
3. Let the operation's privilege gate step aside during an API call; the Kernel applied a narrower check.

## Reference

### The Hook and the Route Tuple

```php
// The listener. Both arguments are references; append to $routes and return nothing.
Hook::add('filter:api.routes', 20, function (array &$routes, string &$audience): void {
    if ($audience !== 'admin') return;               // 'admin' | 'client' | 'module'
    // ...
});

// One entry. Index 6 exists only on the free surface.
// [0] string  $method    GET | POST | PUT | PATCH | DELETE
// [1] string  $pattern   full path after the surface prefix; {x} captures a path parameter
// [2] string  $group     'Module:{Type}/{Name}' routes to your module instance
// [3] string  $action    the permission name AND the api_{action} method suffix
// [4] bool    $public    false = a credential is required (default false)
// [5] bool    $authOnly  false = ALSO enforce the scope "Group/Action"
//                        default false on admin/client, TRUE on the free surface
// [6] string  $audience  free surface only: 'admin' (default) | 'client' | 'any'
$routes[] = ['POST', 'mymodule/items/{id}/rebuild', 'Module:Addons/MyModule', 'item_rebuild', false, false];
```

| Surface | Audience value | Address the entry answers on |
| --- | --- | --- |
| credentialed admin | `admin` | `/api/v1/admin/{pattern}` |
| customer | `client` | `/api/v1/client/{pattern}` |
| free | `module` | `/api/v1/{pattern}` |

The scope a route enforces is always `{group}/{action}`, so the entry above is spent against `Module:Addons/MyModule/item_rebuild`. Appending never shadows a core address: the core entries are registered first and the router returns the first match. To take over one, edit its tuple in place.

### The Handler Contract

```php
public function api_item_rebuild(Request $request, array $match): Response;

// $request, every property public and already parsed:
//   string  $method          'GET', 'POST', ...
//   string  $audience        'admin' | 'client' | ''
//   array   $segments        the path split on '/'
//   string  $resource        the first segment
//   array   $query           the query string
//   array   $body            the decoded JSON body (or the form body)
//   array   $headers         lower-cased header names
//   string  $ip              the resolved client address
//   ?string $token           the raw credential, when one was sent
//   ?string $idempotencyKey
//   string  $rawBody

// $match, produced by the router:
//   'group'    => 'Module:Addons/MyModule'
//   'action'   => 'item_rebuild'
//   'params'   => ['id' => '42']       the {x} captures, keyed by name
//   'scope'    => 'Module:Addons/MyModule/item_rebuild'
//   'public'   => false
//   'authOnly' => false
//   'audience' => 'admin'
```

### Building the Response

```php
class Response
{
    public function __construct(int $status = 200, array $payload = []);

    public static function success($data = null, int $status = 200, array $meta = []): self;
    public static function error(string $code, string $message, int $status = 400, array $details = []): self;
    public static function fromLegacy(array $ret): self;          // {status, message, data} envelope

    public function withHeader(string $name, string $value): self;
    public function withHeaders(array $headers): self;
    public function getStatus(): int;
    public function getPayload(): array;
    public function send(): void;                                 // the Kernel calls this, you do not
}

// Failures are thrown, not returned. Each factory carries its own HTTP status.
class ApiException extends Exception
{
    public static function badRequest(string $message, string $code = 'bad_request', array $details = []);
    public static function unauthorized(string $message = 'Authentication required.', string $code = 'unauthorized');
    public static function forbidden(string $message = 'Insufficient scope.', string $code = 'forbidden');
    public static function notFound(string $message = 'Resource not found.', string $code = 'not_found');
    public static function methodNotAllowed(string $message = 'Method not allowed.', string $code = 'method_not_allowed');
    public static function validation(string $message, array $details = [], string $code = 'validation_failed');
    public static function rateLimited(string $message = 'Too many requests.', string $code = 'rate_limited');
    public static function server(string $message = 'Internal server error.', string $code = 'server_error');
}
```

### Publishing the Checkboxes

```php
public static function set($key, $values, $merge = false): array|false;
```

```php
// The third argument switches the merge from array_replace_recursive to array_merge, so your
// action list is written whole instead of being blended index by index into a same-named list.
// The catalogue's other groups survive either way.
Config::set('api-actions', ['Module:Addons/MyModule' => ['items_list', 'item_rebuild']], true);
```

Only the in-memory copy is written; `Config::save()` is never called here, so `coremio/configuration/api-actions.php` stays as the core shipped it.

### How Far a Granted Scope Reaches

| Granted on the key | What it opens | Safe to recommend |
| --- | --- | --- |
| `Module:Addons/MyModule/item_rebuild` | that one endpoint | yes |
| `Module:Addons/*` | **every Addons module on the installation** | no |
| `*` | the whole admin API | no |

> **There is no wildcard for one module**
> 
> The gate cuts the required scope at the *first* slash and your group already contains one. So the group of `Module:Addons/MyModule/x` is `Module:Addons`, and the only wildcard matching it also matches every other Addons module.

## Example

A minimal admin surface: declaration, registration, one handler and the bridge.

```php
<?php
namespace WISECP\Modules\Addons\MyModule\Src;

use Config;

final class ApiSurface
{
    public const GROUP = 'Module:Addons/MyModule';

    /** [method, path, action, target], literal paths BEFORE their {id} twins. */
    public static function map(): array
    {
        return [
            ['GET',    'mymodule/items',              'items_list',   'read:items_list'],
            ['GET',    'mymodule/items/export',       'items_export', 'read:items_export'],
            ['GET',    'mymodule/items/{id}',         'item_detail',  'read:item_detail'],
            ['POST',   'mymodule/items',              'item_save',    'op:save_item'],
            ['POST',   'mymodule/items/{id}/rebuild', 'item_rebuild', 'op:rebuild_item'],
            ['DELETE', 'mymodule/items/{id}',         'item_delete',  'op:delete_item'],
        ];
    }

    public static function routes(): array
    {
        $routes = [];
        foreach (self::map() as [$method, $path, $action])
            $routes[] = [$method, $path, self::GROUP, $action, false, false];

        return $routes;
    }

    public static function publish_permission_catalog(): void
    {
        $actions = array_map(static fn (array $e): string => $e[2], self::map());
        if (!$actions) return;

        Config::set('api-actions', [self::GROUP => $actions], true);
    }

    /** @return array{0:string,1:string}|null [kind, name] for an action, e.g. ['op', 'rebuild_item'] */
    public static function target(string $action): ?array
    {
        foreach (self::map() as $entry)
            if ($entry[2] === $action) return array_pad(explode(':', $entry[3], 2), 2, '');

        return null;
    }
}
```

```php
<?php
use WISECP\Modules\Addons\MyModule\Src\ApiSurface;

include_once __DIR__ . DS . 'src' . DS . 'ApiSurface.php';
include_once __DIR__ . DS . 'src' . DS . 'ApiBridge.php';

// The addresses: admin surface, credential required, scope enforced per endpoint.
Hook::add('filter:api.routes', 20, function (&$routes, &$audience) {
    if ($audience !== 'admin') return;

    foreach (ApiSurface::routes() as $route) $routes[] = $route;
});

// The checkboxes: in the FILE BODY, never inside the listener above.
ApiSurface::publish_permission_catalog();
```

```php
use WISECP\Api\Core\Request;
use WISECP\Api\Core\Response;
use WISECP\Modules\Addons\MyModule\Src\ApiBridge;

class MyModule extends AddonModule
{
    public function api_items_list(Request $request, array $match): Response
    {
        return ApiBridge::dispatch('items_list', $request, $match);
    }

    public function api_item_rebuild(Request $request, array $match): Response
    {
        return ApiBridge::dispatch('item_rebuild', $request, $match);
    }

    // ... one per declared action; generate them from ApiSurface::map()
}
```

```php
public static function op(string $name, Request $request, array $match): Response
{
    $area  = self::area();                         // licence gate + the AdminArea instance
    $input = array_merge($request->query, $request->body, $match['params'] ?? []);

    $snapshot = [
        'post'    => $_POST,
        'get'     => $_GET,
        'request' => $_REQUEST,
        'method'  => $_SERVER['REQUEST_METHOD'] ?? '',
    ];

    $_POST = $_REQUEST = $input;
    $_GET  = [];
    $_SERVER['REQUEST_METHOD'] = 'POST';           // operations assume a form POST

    self::$in_api = true;
    $failure = null;

    ob_start();
    try     { $area->{'op_' . $name}(new Operation($name)); }
    catch   (Exception $e) { $failure = $e; }
    finally {
        $printed = (string) ob_get_clean();
        self::$in_api = false;

        $_POST    = $snapshot['post'];             // hand the borrowed request back clean
        $_GET     = $snapshot['get'];
        $_REQUEST = $snapshot['request'];
        $_SERVER['REQUEST_METHOD'] = $snapshot['method'];
    }

    if ($failure) throw ApiException::validation($failure->getMessage(), [], 'operation_failed');

    $envelope = Utility::jdecode($printed, true) ?: [];
    $data     = $envelope['data'] ?? null;

    if (is_array($data)) unset($data['html']);     // the panel's modal body is not API payload

    return Response::success($data, 200, ['message' => $envelope['message'] ?? '']);
}
```

The operation's own privilege gate has to step aside for the bridge, and only for the bridge:

```php
private function require_operation(): void
{
    // No signed-in operator exists during an API call, and the Kernel already checked a
    // narrower permission: this exact endpoint's scope.
    if (ApiBridge::in_api()) return;

    if (!\Admin::isPrivilege(['MY_MODULE_OPERATION']))
        throw new Exception($this->lang['err-no-privilege'] ?? 'You do not have permission for this action.');
}
```

## Pitfalls

> **A parametric path declared first swallows its literal twin**
> 
> With `items/{id}` above `items/export`, a request for the export answers **200** from the detail handler with `id = "export"`. Nothing is logged and nothing fails. Feed a concrete value through the router and check the action.

> **Publishing the catalogue inside the listener hides every checkbox**
> 
> The settings screen reads the catalogue before it asks for the scope map, so a listener that has not fired publishes nothing. The routes still work and a hand-written key still fails the scope check, which looks like a permissions bug. Call it from the file body.

> **A magic __call will not answer**
> 
> The dispatcher tests `method_exists` before calling and answers `404 action_not_implemented` when it fails. That is deliberate: one real method per endpoint is what makes each endpoint separately grantable.

> **Kernel::internal cannot reach these endpoints**
> 
> It splits its argument at the first slash and your group already has one, so the lookup fails with `endpoint_not_found`. Code inside the installation calls your classes directly.

> **The category label prints the raw group name**
> 
> The tab label comes from a core translation key a module cannot add, so the screen shows `Module:Addons/MyModule` verbatim. If you rewrite the visible text, never touch the checkbox values: they are the scope strings the gate compares.

> **A licensed module has to repeat its own licence gate**
> 
> The panel dispatcher checks the licence before every operation and an API request does not pass through it. Put the same check at the top of your bridge and answer `403`. On a local machine that gate is closed permanently.

## Related Articles

- [Registering Hooks from a Module](https://dev.wisecp.com/en/registering-hooks-from-a-module)
- [Adding an Admin Page](https://dev.wisecp.com/en/adding-an-admin-page)
- [API Authentication and Permissions](https://dev.wisecp.com/en/api-authentication-and-permissions)
- [Request and Response Format](https://dev.wisecp.com/en/request-and-response-format)
- [Operations](https://dev.wisecp.com/en/operations)
- [The WISECP API](https://dev.wisecp.com/en/the-wisecp-api)


# The Client Area Bridge

https://dev.wisecp.com/en/the-client-area-bridge

Call a method on your addon module from the customer's browser. The operation already exists: no controller, no route, no endpoint of your own.

## Overview

An addon page in the client area is HTML your module produced. Then it needs to do something: save a preference, fetch a status, start a job. One operation is already wired on the website, and it calls any method whose name starts with `use_`.

The admin panel has the same bridge behind the tools controller. What differs is who is allowed through, and that difference is the security part of this article.

## Prerequisites

- An Addon module with `status` true in its configuration; a disabled addon is invisible to the bridge.
- A web face: the client area opt-in (`show_on_clientArea` plus `clientArea()`) or a public `main()` page. Without one the bridge refuses every call.
- `coremio/modules/Addons/SampleAddon`, a working demonstration of both faces.

## Structure

- **controllers/website/addon.php**: The website face: resolves `/addon/{Name}`, produces the page and dispatches the bridge operation.
- **operations/ClientAddon.php**: The trait holding `use_addon_method`, plus the gate that decides whether your addon has a web face.
- **operations/AdminTools.php**: The admin twin: same operation name and prefix rule, behind an administrator session.
- **AddonModule**: Your base class. Supplies `$area_link`, `$error`, `$config`, `$lang`, `$dir`, `$url` and `view()`.

```text
browser  POST /addon/MyAddon   operation=use_addon_method & method=save-preference
   |
   +-- addon controller  ->  addon_ctx()     enabled config + module instance + web face?
   +-- ClientAddon       ->  member session required when the face is the client area
   +-- name normalised   ->  'save-preference'  becomes  use_save_preference
   +-- method_exists     ->  refuse when absent
   +-- $module->use_save_preference()          called with NO arguments
   +-- falsy return      ->  Exception($module->error)
   '-- array or string   ->  JSON body
```

## Walkthrough

### Give the Addon a Web Face

1. Set `show_on_clientArea` to true in `config.php` and add a `clientArea()` method returning `['page_title' => …, 'breadcrumbs' => …, 'content' => …]`. That makes `/addon/{Name}` the customer page and requires a signed-in member.
2. Or add a `main()` method for a public page, which carries no session guarantee at all.
3. Set `meta.slug` for a pretty address; the controller rewrites `$area_link` to it.

### Write the use_ Method

1. Name it `use_{something}`; nothing else is callable, and that prefix is the whole boundary at the dispatch layer.
2. Take no parameters. Read your input yourself with `Filter::init("POST/…")`, as an operation does.
3. Return a non-empty array or string; a falsy return counts as failure and raises an exception carrying `$this->error`.
4. For a real failure, throw: the caller converts it into the standard error envelope.

### Call It from the Page

1. Post to `$this->area_link`, which the controller already points at the correct address. Never hand-write the path.
2. Send `operation=use_addon_method` and `method={name without the prefix}`, plus whatever else your method reads.
3. On the website use plain `fetch` with the `X-Requested-With` header; in the admin panel use `WcpRequest`.

## Reference

### The Two Endpoints

| Face | Address | Who gets through |
| --- | --- | --- |
| client area | `/addon/{Name}` or the configured slug | a signed-in member, if the addon opted in |
| public page | `/addon/{Name}` | **anyone**, with no session |
| legacy alias | `/addon/{Name}/client` | as the client area; 404 without the opt-in |
| admin panel | the tools addons address | an administrator with the tools privilege |
| admin-only addon called from the website | any of the above | nobody: the operation throws `Addon not found.` |

> **The web face check is a real security boundary, added after a real hole**
> 
> Before it existed, an admin-only addon's `use_*` methods were reachable from the website with no session. Settings were overwritten, ticket data read, paid API credit burned. Treat it as the outer wall, not the only one.

### The Operation and the Name Rule

```php
public function use_addon_method(Operation $operation): bool;
```

```php
$method = (string) Filter::init("REQUEST/method", "route");        // keeps a-zA-Z0-9 - _ .
$method = "use_" . str_replace([' ', '-', '.'], '_', $method);     // spaces, hyphens, dots -> underscore

if ($method === "use_" || !method_exists($module, $method))
    throw new \Exception("Undefined addon method.");

$result = $module->{$method}();                                     // NO arguments
if (!$result) throw new \Exception((string) (($module->error ?? '') ?: 'An error occurred'));

return $operation->output($result);
```

So `method=save-preference`, `save.preference` and `save_preference` all reach `use_save_preference()`. The `route` filter runs first, keeping only `a-zA-Z0-9`, hyphen, underscore and dot, so a name cannot escape into another class.

### What Your Method Must Return

```php
public function use_sample_method(): array|string;
```

| You return | The customer receives | Use it for |
| --- | --- | --- |
| a non-empty array | that array, JSON encoded | the normal case; keep the standard keys |
| a non-empty string | the string, written as is | a ready HTML fragment for the DOM |
| `[]`, `''`, `false` or `null` | `{"status":"error","message":"…"}` built from `$this->error` | nothing: a legacy path, do not design for it |
| a thrown exception | `{"status":"error","message":"your message"}` | every real failure |

The envelope to aim for matches the rest of the panel:

```php
return [
    'status'  => 'successful',
    'message' => $this->lang['saved'] ?? 'Saved.',
    'data'    => ['preference' => $value, 'updated_at' => time()],
];
```

### What the Base Class Hands You

```php
class AddonModule
{
    public string|bool $error = '';        // read by the bridge when your method returns falsy
    public array  $config    = [];         // config.php merged with the saved settings
    public array  $lang      = [];         // lang/{selected}.php
    public string $area_link = '';         // the address to post to; REWRITTEN per face
    public string $_name     = '';         // the directory name
    public array  $user      = [];         // the signed-in member, when there is one
    public array  $admin     = [];         // the signed-in administrator, when there is one
    public string $url       = CORE_FOLDER . DS . MODULES_FOLDER . DS . 'Addons' . DS;
    public string $dir;                    // filesystem path of the module directory
    // The constructor appends {Name} to $url and resolves it into a public URL, fills $dir,
    // $config, $lang, $user and $admin, and points $area_link at the current face.

    public function __construct();
    protected function view($file = '', $variables = []): string;
    public function privileges();
    public function save_settings($pFields, $accessPs): bool;
    public function change_addon_status($arg = '');
    public function save_config($data = []): bool;
    public function use_default_settings($formElements = null);
    public function isEnabled();
}
```

> **area_link is not one fixed value**
> 
> In the panel it points at the tools addons address. On the website the addon controller overwrites it with the public address. Print it rather than building a path.

### Reading Core Data

```php
// $groupAction is "Group/Action" from the route registry; prefix it with "client:" for the
// customer registry, in which case $input must also carry owner_id.
public static function internal(string $groupAction, array $input = [], array $query = []): array;
```

```php
use WISECP\Api\Kernel;

// In process: no HTTP, no authentication, no rate limit, but the SAME envelope the
// external API returns. $input fills the path parameters first, then the body.
$resp    = Kernel::internal('Tickets/GetTicketMessages', ['id' => $ticketId], ['limit' => 50]);
$replies = $resp['data'] ?? [];
```

Prefer this over calling a helper directly: the resource layer hands back decrypted, allow-listed and normalised rows.

## Example

A client page that stores one customer preference.

```php
<?php
namespace WISECP\Modules\Addons\MyAddon;

use Exception;
use Filter;
use Language;
use User;
use UserManager;

class MyAddon extends \AddonModule
{
    /** Rendered at /addon/MyAddon once show_on_clientArea is true in config.php. */
    public function clientArea(): array
    {
        $member = UserManager::LoginData('member');

        return [
            'page_title'  => $this->lang['meta']['name'] ?? 'My Addon',
            'breadcrumbs' => [['link' => '', 'title' => $this->lang['meta']['name'] ?? 'My Addon']],
            'content'     => $this->view('client.php', [
                'link'       => $this->area_link,                                  // post here
                'preference' => (string) (User::getInfo((int) $member['id'], ['my_addon_pref'])['my_addon_pref'] ?? ''),
            ]),
        ];
    }

    /**
     * Reached as method=save-preference. No parameters: it reads its own input, exactly
     * like an operation, and it re-reads the account from the session rather than trusting
     * anything the request sent.
     *
     * @throws Exception
     */
    public function use_save_preference(): array
    {
        $member = UserManager::LoginData('member');
        if (!$member) throw new Exception(Language::gc('website/index/addon-login-required') ?: 'Login required.');

        $value = Filter::init('POST/preference', 'route');
        if ($value === '' || !in_array($value, ['daily', 'weekly', 'never'], true))
            throw new Exception($this->lang['err-bad-preference'] ?? 'Choose one of the offered options.');

        User::AddInfo((int) $member['id'], ['my_addon_pref' => $value]);

        return [
            'status'  => 'successful',
            'message' => $this->lang['saved'] ?? 'Saved.',
            'data'    => ['preference' => $value],
        ];
    }
}
```

```html
<select id="ma-pref" class="form-select">
    <option value="daily">Daily</option>
    <option value="weekly">Weekly</option>
    <option value="never">Never</option>
</select>
<button type="button" id="ma-save" class="btn btn-primary mt-2">Save</button>

<script>
(function () {
    var link = ;
    var btn  = document.getElementById('ma-save');
    var sel  = document.getElementById('ma-pref');
    if (!btn || !sel) return;

    btn.addEventListener('click', function () {
        btn.disabled = true;

        var body = new URLSearchParams({
            operation:  'use_addon_method',
            method:     'save-preference',   // hyphen is fine, it becomes use_save_preference
            preference: sel.value
        });

        fetch(link, {
            method:  'POST',
            headers: { 'X-Requested-With': 'XMLHttpRequest', 'Content-Type': 'application/x-www-form-urlencoded' },
            body:    body
        })
        .then(function (r) { return r.json(); })
        .then(function (r) {
            if (r.status === 'successful') window.WCPTheme && window.WCPTheme.toast(r.message, 'success');
            else window.WCPTheme && window.WCPTheme.toast(r.message, 'error');
        })
        .finally(function () { btn.disabled = false; });
    });
})();
</script>
```

The same call from an admin page:

```javascript
WcpRequest(AREA_LINK, {
    method: 'POST',
    data:   { operation: 'use_addon_method', method: 'save-preference', preference: 'weekly' },
    button: runBtn,
    buttonLoader: window.saving_loader,
    done: function (response) {           // omit `done` and the standard handling applies
        output.textContent = JSON.stringify(response, null, 2);
    }
});
```

## Pitfalls

> **A public page makes every use_ method public too**
> 
> An addon whose web face is `main()` is reachable with no session by design, for compatibility with older modules. Ask, for each method, who should be able to call it. Write that check in the method.

> **Do not trust an account id that arrives in the request**
> 
> The bridge authenticates the visitor, not the record. A method that reads a customer id from the body serves another customer's data to whoever asks. Read the account from the member session.

> **An empty successful result reads as a failure**
> 
> Returning `[]` after a legitimately empty query produces an error envelope carrying whatever is in the error property, often an empty message. Return an array whose data field is the empty list.

> **Saving through the settings operation overwrites status and access privileges**
> 
> The standard settings save writes the status flag and access privilege list along with your fields, so a custom save that reuses it can disable your addon. Give the custom save its own `use_` method.

> **A disabled addon answers nothing**
> 
> The context resolver checks the enabled flag before it builds the instance, so a call against a disabled addon fails with `Addon not found.`, not a method error.

## Related Articles

- [Writing an Addon Module](https://dev.wisecp.com/en/writing-an-addon-module)
- [Exposing API Endpoints](https://dev.wisecp.com/en/exposing-api-endpoints)
- [Adding an Admin Page](https://dev.wisecp.com/en/adding-an-admin-page)
- [Operations](https://dev.wisecp.com/en/operations)
- [Filtering User Input](https://dev.wisecp.com/en/filtering-user-input)
- [The Client Area](https://dev.wisecp.com/en/the-client-area)

# Registering Hooks from a Module

https://dev.wisecp.com/en/registering-hooks-from-a-module

Put a `hooks.php` in your module directory: that is how a module reaches into the core without editing it.

## Overview

Hooks are the whole extension surface. From one file it owns, a module catches an event, changes a value, injects markup, registers a capability or refuses an operation. Nothing in `coremio/` knows your module exists.

The catalogue holds **979** hook points in five categories: `ui` 343, `action` 298, `filter` 196, `gate` 131, `register` 11. Browse it rather than guessing a name.

## Prerequisites

- A module directory at `coremio/modules/{Type}/{Name}/`; any of the sixteen types works.
- The hook name, copied from `hooks/INDEX.md`. A name that does not exist fails silently.
- Its page under `hooks/{domain}/`: parameters, which are references, and the return contract.

## Structure

- **classes/Hook.php**: The engine: registration, priority ordering, argument binding, and the loader that finds your file.
- **{module}/hooks.php**: Your listeners. Found by a glob over `coremio/modules/*/*/hooks.php`: the name and location are fixed.
- **coremio/hooks/**: The core's own listener files, loaded in the same pass, before the module files.
- **hooks/INDEX.md**: The generated catalogue: every hook point, its domain, and the file and line that fires it.
- **{module}/router.php**: A second, earlier entry point: loaded while the router is built, before hook files.

## Walkthrough

### Create the File

1. Add `hooks.php` at the root of your module directory, beside `{Name}.php` and `config.php`.
2. It is included; write plain statements at the top level.
3. Classes under your own `src/` are not autoloaded, so `include_once` them before you name them.

### Gate It on Your Own State

1. Load your configuration without instantiating the module: pass `true` as the loader's third argument.
2. Wrap every listener in a check on your enabled flag, and on your licence when the module is licensed.
3. Keep the file cheap; it runs on every request that touches a hook.

### Register a Listener

1. Call the add method with the hook name, a priority and either a closure or a descriptor array.
2. Declare only the parameters you need; the engine matches them positionally.
3. To change a value, declare that parameter by reference; only the reference variant carries it back.

### Choose a Priority

1. Lower runs first. There is no default, so pass a number deliberately.
2. Use a low number to shadow a later entry, as route overrides do, and a high one to append last.

## Reference

### Registering

```php
class Hook
{
    // $properties is EITHER a callable OR a descriptor array (see the three forms below).
    public static function add($name, $priority, $properties = []): void;

    public static function run($name, ...$args): array;        // by value
    public static function runRefs($name, &...$args): array;   // EVERY argument by reference
    public static function runDetailed($name, ...$args): array; // per-listener telemetry
}
```

| Run method | Returns | On a listener exception |
| --- | --- | --- |
| `run` | every non-null listener return, in priority order | logged, then the next listener runs |
| `runRefs` | the same, plus your reference edits reach the caller | logged, then the next listener runs |
| `runDetailed` | `[['source' => ['type','class','method','file','line'], 'value' => …, 'error' => ?string], …]`, nulls kept | captured in `error` |
| any, unknown name | an empty array | nothing happens; a typo is silent |

### The Three Registration Forms

```php
// 1. Closure. Anything callable goes in directly.
Hook::add('action:service.created', 10, function ($id, $data) {
    // ...
});

// 2. Instance method. The class is constructed ONCE with NO arguments and cached for the
//    whole request, so its constructor must work without parameters.
Hook::add('filter:invoice.totals', 10, [
    'class'  => 'MyAddonHooks',
    'method' => 'adjustTotals',
]);

// 3. Static method. Nothing is constructed.
Hook::add('ui:admin.service_detail.bottom', 10, [
    'class'          => 'MyAddonHooks',
    'method::static' => 'renderPanel',
]);
```

A missing class or method makes the listener return null and the request continues. That looks exactly like a hook that was never fired. Prefer a closure.

### How Arguments Reach Your Listener

```php
// For each parameter your listener declares, at position $i:
//   declared by reference AND an argument exists at $i  -> passed by reference
//   an argument exists at $i                            -> passed by value
//   no argument at $i                                   -> the parameter is dropped,
//                                                          so your default applies
$callArgs = [];
foreach ($reflection->getParameters() as $i => $param) {
    if ($param->isPassedByReference() && isset($args[$i])) $callArgs[] = &$args[$i];
    else if (isset($args[$i])) $callArgs[] = $args[$i];
}
```

Declaring fewer parameters than the hook supplies is safe. A reference carries your change back only when the hook was fired with the reference variant. On a by-value hook, `&$value` silently changes nothing.

### The Five Categories and What Each Expects Back

| Prefix | Your listener does | Your return value |
| --- | --- | --- |
| `action:` | reacts to an event | ignored |
| `filter:` | changes a value before it is used | the edit goes through the reference parameter |
| `ui:` | injects markup, styles or scripts | the string that gets printed |
| `register:` | registers a capability: cron task, route, menu entry, widget | the registration, or `false` to register nothing |
| `gate:` | vetoes an operation | **a non-empty value blocks**, `null` lets it continue |

The name is `category:domain.subject.action`, lower case, snake case parts. A `ui:` name ends in a placement word: its listener gets no context and infers the target from the name.

### Reading Your Own Configuration in the Hook File

```php
// $nominc = true loads the config and the language file WITHOUT including the class.
public static function Load($type = '', $name = '', $nominc = false, $status = '');
public static function Config($type, $module);
public static function getInstance(string $type, string $name, array $params = []): ?object;
```

```php
Modules::Load('Addons', 'MyAddon', true);              // config only, no class
$my_config = Modules::Config('Addons', 'MyAddon') ?: [];

if (($my_config['status'] ?? false) && License::valid_addon('my-addon')) {
    // ... register listeners here
}
```

Build the instance inside a listener. Constructing the module on every request only to decide whether to register is the most common reason a module slows the panel.

## Example

A hook file touching four surfaces: an event, a value, a page and a scheduled task.

```php
<?php

// src/ classes are not autoloaded: include what this file names, before it names it.
include_once __DIR__ . DS . 'src' . DS . 'Notifier.php';

use WISECP\Modules\Addons\MyAddon\Src\Notifier;

// Config without the class. One cheap call decides whether anything below registers.
Modules::Load('Addons', 'MyAddon', true);
$my_config = Modules::Config('Addons', 'MyAddon') ?: [];

if (!($my_config['status'] ?? false)) return;

/* An event. This hook is fired as ($id, $data): the new users_products id and the row that
   was inserted. The return value is ignored, so do the work and say nothing. */
Hook::add('action:service.created', 10, function ($id, $data) {
    Notifier::service_created((int) $id, is_array($data) ? $data : []);
});

/* A value. $totals is declared BY REFERENCE because this hook is fired with runRefs;
   $invoice and $items are read-only context. */
Hook::add('filter:invoice.totals', 10, function (&$totals, $invoice, $items) {
    if ((int) ($invoice['legal'] ?? 0) !== 1) return;

    $fee = (float) ($GLOBALS['my_addon_fee'] ?? 0);
    if ($fee <= 0) return;

    $totals['total'] = round((float) $totals['total'] + $fee, 4);
});

/* Markup. Return the string; the surface prints it. Build the instance HERE, not above. */
Hook::add('ui:admin.service_detail.bottom', 20, fn ($service) =>
    Modules::getInstance('Addons', 'MyAddon')->render_service_panel(is_array($service) ? $service : []));

/* A capability. Registration hooks run during bootstrap of the thing they feed. */
Hook::add('register:cronjobs', 1, function () {
    include_once __DIR__ . DS . 'cronjobs' . DS . 'SyncTask.php';

    CronJobQueue::register(
        \WISECP\Modules\Addons\MyAddon\CronJobs\SyncTask::TYPE,
        \WISECP\Modules\Addons\MyAddon\CronJobs\SyncTask::class
    );
});
```

The reading side: the core call that fires the value hook above.

```php
// coremio/helpers/Invoices.php, inside recalculate_totals(): after the figures are built
// and before they are written to the row.
Hook::runRefs('filter:invoice.totals', $totals, $invoice, $items);

// $totals is now whatever the listeners left in it, and $persist writes that.
```

The same care applies when your own module publishes an extension point:

```php
// By value: an event other modules may observe.
Hook::run('action:myaddon.sync_finished', $summary, $startedAt);

// By reference: EVERY argument is by reference here, including the context ones, so
// each of them must be a plain variable first.
$context = ['id' => $recordId, 'lang' => Language::selected()];
Hook::runRefs('filter:myaddon.payload', $payload, $context);

// A gate: a non-empty return from any listener stops the operation.
$veto = Hook::run('gate:myaddon.export', $recordId);
if (array_filter($veto)) throw new Exception((string) current(array_filter($veto)));
```

## Pitfalls

> **Every argument of the reference variant is a reference, context included**
> 
> The signature is variadic by reference. A literal, a cast, a function return or a null-coalescing expression in *any* position is a fatal error. Move each one into a plain variable first; the by-value variant is unaffected.

> **An exception inside a listener is swallowed**
> 
> The engine catches every throwable, logs it and moves on. Your integration does not happen, the page appears normally and nothing says so; a missing `include_once` is the usual cause. When a listener seems inert, read the error log first.

> **Work that must run before the first hook does not belong here**
> 
> Hook files are read lazily, on the first run call, and not at all while the installation date is the zero date. Anything needed earlier (a route, a permission catalogue) goes in `router.php`, loaded while the router is built.

> **The file body runs on every request that touches a hook**
> 
> Instantiating your module, querying the database or calling a service at the top level costs that every time, customer page views included. Read the configuration with the class-free loader and build the instance inside the listener.

> **A priority collision is resolved, not reported**
> 
> Registering two listeners at the same number gives the second the next free slot, so order follows registration order, which follows the alphabetical order of module directories. Pick a number far from the crowd.

## Related Articles

- [How Hooks Work](https://dev.wisecp.com/en/how-hooks-work)
- [The Hook Catalog](https://dev.wisecp.com/en/hook-domains)
- [Writing a Hook Listener](https://dev.wisecp.com/en/writing-a-hook-listener)
- [Exposing API Endpoints](https://dev.wisecp.com/en/exposing-api-endpoints)
- [Adding a Dashboard Widget](https://dev.wisecp.com/en/adding-a-dashboard-widget)
- [Module Anatomy](https://dev.wisecp.com/en/module-anatomy)

# Adding a Dashboard Widget

https://dev.wisecp.com/en/adding-a-dashboard-widget

Add a card to the admin dashboard by returning a descriptor from one registration hook, with no core file touched.

## Overview

**Dashboard widgets are not a module type:** there is no widget module and no base class. `get_widgets()` builds them out of privilege checks, one template shows them, hooks extend them.

You return an array from `register:admin.dashboard_widgets`. The core gives it a rank, applies the operator's saved layout, and hands it to the shared card shell. Four more hooks reach the rest.

## Prerequisites

- A module with a working `hooks.php`; see [Registering Hooks from a Module](https://dev.wisecp.com/en/registering-hooks-from-a-module).
- Your content as an HTML string; the body is echoed as is.
- A privilege key if the card should be restricted.

## Structure

- **controllers/admin/index.php**: `get_widgets()` builds the list and merges your hook's return; `get_statistics()` builds the figures strip.
- **templates/admin/index.php**: The dashboard shell: fires the injection hooks and the widget filter, then loops the list into cards.
- **inc/template-widget-item.php**: The card shell. An unknown name falls through to your content.
- **operations/AdminIndex.php**: `get_widget_content`, behind the refresh button.
- **js/home.js**: Lays the grid out with Packery: rank is only the order in the HTML, not where a card lands.

## Walkthrough

### Register the Card

1. Listen on `register:admin.dashboard_widgets` from your `hooks.php` and return one descriptor array.
2. Give it at least `name`, `title` and `content`; everything else has a default.
3. Check the privilege inside the listener: return `false` when it fails, or set `allowed` to the result.

### Build the Body

1. Produce the HTML with heredoc and read your strings from the module's language file.
2. Build it inside the listener, not in the file body, so a dashboard without your card costs nothing.
3. Reserve the height of anything that fills in later.

### Control Placement and Size

1. Set `rank` for the order in the markup; left out, the core appends yours last.
2. Set `size` to `wide` for a full-width card; any other value gives the standard half-width one.
3. For the strip, the surrounding markup or someone else's cards, use the four other hooks below.

## Reference

### The Five Dashboard Hooks

| Hook | Call site | Your return |
| --- | --- | --- |
| `register:admin.dashboard_widgets` | `Hook::run`, no arguments | one descriptor becomes one card; `false` registers nothing |
| `filter:admin.dashboard.widgets` | `Hook::runRefs`, `$widgets` by reference | edit in place: reorder, drop, retitle |
| `filter:admin.dashboard.statistics` | `Hook::run`, `$result` | a non-empty return **replaces the whole array** |
| `ui:admin.dashboard.top` | `Hook::run`, no arguments | a string above the figures strip |
| `ui:admin.dashboard.statistics.after` and `ui:admin.dashboard.bottom` | same, no arguments | a string after the strip, and after the grid |

> **The statistics filter replaces, it does not merge**
> 
> The call site keeps the last non-empty listener return and assigns it over the whole result. Change the key you care about and return the whole array; returning only your own key wipes the strip, which still looks plausible.

### The Descriptor Array

- **name**: The identity: `data-id`, the saved-layout key, and the refresh argument. Default `wt{rank}`, never rely on it.
- **title**: The heading, shown as raw HTML. Default `Untitled Widget`; wrapped in a link when you set `link`.
- **content**: The card body, echoed as is. Empty shows the widget name instead, the symptom of forgetting it.
- **icon**: Bootstrap icon class for the disc beside the title. Default `bi bi-box`.
- **allowed**: Boolean gate, default true. False removes the card before it appears.
- **status**: `open` or `close`, default open. A closed card shows its header only.
- **rank**: Integer order in the markup. Default: one past the last registered widget.
- **size**: `wide` adds the full-width class; any other value keeps the standard width.
- **hidden**: Boolean. The card is produced but starts with display none, how the close button remembers itself.
- **link**: Turns the title into a link. Build it with the link generator, never a literal path.
- **buttons**: Header buttons: `['create' => ['name' => …, 'link' => …, 'icon' => …]]`. `create` gets a plus icon; other keys use `icon`, or the name as text.
- **header_buttons**: Raw HTML before the refresh, collapse and close buttons.

### What the Core Fills In

```php
$hook = Hook::run("register:admin.dashboard_widgets");
if ($hook)
{
    $last = end($widgets);
    $rank = $last["rank"] ?? 10;

    foreach ($hook as $h)
    {
        $rank++;
        $wn = "wt" . $rank;

        if (isset($h["name"]) && $h["name"]) $wn = $h["name"];
        if (!isset($h["allowed"])) $h["allowed"] = true;
        if (!isset($h["status"])) $h["status"] = "open";
        if (!isset($h["rank"]))   $h["rank"]   = $rank;

        $widgets[$wn] = $h;                 // your name is the key: it can REPLACE a built-in card
    }
}
```

### The Refresh Button

```php
public function get_widget_content(Operation $operation): bool;
```

```php
// GET {dashboard}?operation=get_widget_content&name={your name}
$widgets = $this->get_widgets();                 // the WHOLE list is rebuilt, hook included
if (!isset($widgets[$wn])) throw new Exception("Invalid widget");

// One card re-rendered through the same shell, returned as an HTML string.
return $operation->output($this->view->chose("admin")->render("inc" . DS . "template-widget-item", [
    'widget' => $widgets[$wn],
], true));
```

### Putting a Table in a Card

```php
$t = new \WISECP\Components\Table("myAddonWidget", [
    'preset'      => 'invoiceList',      // row rendering comes from the MAIN list
    'hideActions' => true,
    'perPage'     => false,
    'search'      => false,
    'info'        => false,
    'pagination'  => false,
]);

foreach ($t->getColumns() as $k => $v) $t->setColumn($k, ['sortable' => false]);
$t->deleteColumn("selection");
$t->setRows($rows);

$html = $t->build();
```

> **Rows must carry every key the preset reads**
> 
> A widget table borrows the main list's row builder, which reads far more keys than the columns you kept. A missing key shows an empty cell instead of failing. Derive the list from the preset file; a change there reaches your card too.

## Example

A module's own queue: registered, gated, ranked and refreshable.

```php
<?php

Modules::Load('Addons', 'MyAddon', true);
$my_config = Modules::Config('Addons', 'MyAddon') ?: [];

if (!($my_config['status'] ?? false)) return;

/*
 * One card on the dashboard. The listener takes no arguments and returns ONE descriptor;
 * it runs on every dashboard render and on every refresh of this card, so it stays cheap
 * and builds the module instance only after the privilege check has passed.
 */
Hook::add('register:admin.dashboard_widgets', 10, function () {
    if (!Admin::isPrivilege('TOOLS_ADDONS')) return false;

    $module = Modules::getInstance('Addons', 'MyAddon');

    return [
        'name'    => 'myaddon_queue',
        'title'   => $module->lang['widget-title'] ?? 'My Addon Queue',
        'icon'    => 'bi bi-list-check',
        'rank'    => 6,
        'status'  => 'open',
        'link'    => LinkGenerator::admin('tools-2', ['addons', 'MyAddon']),
        'buttons' => [
            'create' => [
                'name' => Language::gc('admin/index/button-create-a-new'),
                'link' => LinkGenerator::wQS(LinkGenerator::admin('tools-2', ['addons', 'MyAddon']), ['trigger' => 'create']),
            ],
        ],
        'content' => $module->render_dashboard_widget(),
    ];
});

/* A figure on the strip. Read what you were given, change one key, return ALL of it. */
Hook::add('filter:admin.dashboard.statistics', 10, function ($result) {
    if (!is_array($result)) return false;

    $result['static_blocks']['myaddon_pending'] = [
        'title' => 'Pending syncs',
        'value' => (int) WDB::select('COUNT(id) AS total')->from('MyAddon_queue')
            ->where('status', '=', 'pending')->build() ? (int) (WDB::getAssoc()['total'] ?? 0) : 0,
    ];

    return $result;
});
```

```php
public function render_dashboard_widget(): string
{
    $rows = '';

    foreach ($this->queue_preview(5) as $row) {
        $label = htmlspecialchars((string) ($row['label'] ?? ''), ENT_QUOTES);
        $state = htmlspecialchars((string) ($row['status'] ?? ''), ENT_QUOTES);

        $rows .= <<<HTML
        <li class="list-group-item d-flex justify-content-between align-items-center px-0">
            <span class="text-truncate">{$label}</span>
            <span class="badge text-bg-light">{$state}</span>
        </li>
        HTML;
    }

    if ($rows === '')
        $rows = '<li class="list-group-item px-0 text-body-secondary">' . ($this->lang['widget-empty'] ?? 'Nothing queued.') . '</li>';

    // The class carries the reserved height (see the stylesheet below), which is what stops
    // the whole grid from repacking once the list fills in.
    return <<<HTML
    <ul class="list-group list-group-flush myaddon-queue">{$rows}</ul>
    HTML;
}
```

```css
/* The final height, declared before the content exists. Ship it through
   ui:admin.head.css from the same hooks.php that registers the card. */
.myaddon-queue { min-block-size: 220px; }
```

The reading side is the card shell: the fallback for a missing `content`, and why the body is not escaped.

```php
$w_name    = $widget["name"]  ?? "widget" . $w_rank;
$w_title   = ($widget["title"] ?? '') ?: 'Untitled Widget';
$w_icon    = ($widget["icon"]  ?? '') ?: 'bi bi-box';
$w_content = $widget["content"] ?? '';

if ($w_name == "orders_chart") {
    // ... a long chain of built-in names, one branch each
}
else
    echo $w_content ?: $w_name;      // your HTML, unescaped, or the name when you forgot it
```

## Pitfalls

> **A card that grows after the first paint repacks the grid**
> 
> The layout is masonry and re-measures every card on each pass. A body that fills in later changes its card's height, and unrelated cards slide across the screen. Measured: a chart box growing 54 pixels moved two other cards 733 pixels.

> **The operator's saved arrangement outranks your descriptor**
> 
> A stored rank, collapsed state or hidden flag for your widget name beats the descriptor, and it travels in a cookie: a card registered open can be collapsed for one administrator, open for a colleague. A collapsed card's content is in the page, hidden by CSS.

> **Your listener runs on every dashboard load, not once**
> 
> It also runs for each refresh of any card, because that operation rebuilds the entire list. That is what makes the refresh button work, and it means a query there is paid every time. Do the privilege check first and cache anything expensive.

> **Registering under an existing name replaces that card**
> 
> The merge keys by name, so `notes` or `tasks` as your widget name takes over the built-in card. Prefix the name with your module.

## Related Articles

- [Registering Hooks from a Module](https://dev.wisecp.com/en/registering-hooks-from-a-module)
- [Adding an Admin Page](https://dev.wisecp.com/en/adding-an-admin-page)
- [The Hook Catalog](https://dev.wisecp.com/en/hook-domains)
- [Interface Components](https://dev.wisecp.com/en/interface-components)
- [The Module System](https://dev.wisecp.com/en/the-module-system)
- [Building Links and Routes](https://dev.wisecp.com/en/building-links-and-routes)

# Writing a Server Module

https://dev.wisecp.com/en/writing-a-server-module

A server module turns a paid order into a real account on a hosting panel or a cloud provider. The core calls your lifecycle methods by name; what you return is written back onto the service.

## Overview

Servers is the largest module type: 50 modules, three of them sandbox archetypes for shared hosting panels, dedicated machines and virtualization.

Your class extends `ServerModule`, which brings in `ModuleBaseTrait`. Between them they already hold the server, the service, the product, the buyer, the resolved limits, the addon answers and the tool machinery.

The core never constructs your class: it resolves an instance through the factory and calls the verb by name. A verb you did not implement is skipped, and that is the opt-in mechanism of the whole type.

- **Services::run_module()**: The single door: builds the instance, resolves aliases, runs the method, applies the result.
- **Services::instance_module()**: Picks the module type from the service, then calls the factory and fills the service and the order. Hosting and server go to Servers.
- **Modules::getInstance()**: The canonical factory: config, language, instance cache. `new` is never used for a module.
- **ModuleQueue**: The retrying background runner: same door, then the status the action implies.

## Prerequisites

- [Module Anatomy](https://dev.wisecp.com/en/module-anatomy) and [Module Configuration](https://dev.wisecp.com/en/module-configuration) first; this article covers only the Servers type.
- A provider account with API access and a server record that passes the connection test.
- A test product bound to that server, otherwise `create()` has no plan to read.
- Failure is reported by throwing, not by returning `false`.

## Structure

One directory per module, named exactly like the class inside it.

```bash
coremio/modules/Servers/Acme/
├── Acme.php            the class: extends ServerModule
├── ApiClient.php       your HTTP wrapper, plain new, not a WISECP module
├── config.php          metadata, server form fields, supported cards and tools
├── logo.png            shown in the module picker
├── lang/en.php         $this->lang, one file per language
└── pages/              optional templates rendered with get_page()
```

```php
namespace WISECP\Modules\Servers;

use Exception;
use Language;
use ServerModule;

class Acme extends ServerModule
{
    private ApiClient $api;

    // Called by set_server(), which the constructor and set_service() both run,
    // once $this->server is filled and its secrets are decoded.
    protected function define_server_info(array $server = []): void
    {
        include_once __DIR__ . DS . 'ApiClient.php';
        $this->api = new ApiClient($server);
    }
}
```

| Surface | Entry point in your class | Reached from |
| --- | --- | --- |
| Server settings form | `config.php` fields, then `test_connect()` | Admin, Servers, Manage Server |
| Product settings | `product_configuration()`, `save_product_configuration()` | Product detail, Module tab |
| Provisioning | `create()`, `suspend()`, `unsuspend()`, `cancel()` | Order approval, admin actions, the queue |
| Client area dashboard | `dashboard_data()` | Service detail in the client area |
| Client area tools | `tool_data()`, `tool_action()` | The tool sidebar, see the tools article |
| Metered billing | `metrics_usage()` or `metrics_usage_bulk()` | The usage collection cron |
| Import | `list()` | Admin, import accounts from the panel |

## Walkthrough

### Write the Layers in This Order

Lifecycle first is wasted work. `create()` reads the buyer's choices out of the service options, and those options exist only once the product form is defined.

1. `define_server_info()`, then `test_connect()`: add the server and prove the credentials.
2. `product_configuration()` and `save_product_configuration()`, plus the callable that fills the plan dropdown.
3. `create()`, then `suspend()`, `unsuspend()`, `cancel()`.
4. `change_password()` and `upgrade()`.
5. `list()`, `dashboard_data()`, the single sign on methods, metrics.
6. `configure_features()` and the tools.

Verify each layer against the live provider before moving on: a first failure at the end has six possible causes.

### Connecting to the Provider

Credentials come from the server record. `config.php` decides which fields the form shows, and the base class hands them to you decoded.

```php
return [
    'name'                => 'Acme Cloud',

    // 'hosting' = domain based accounts, 'server' = VPS or dedicated machines.
    'type'                => 'hosting',

    // A long API token belongs in the access hash field, not in the password field:
    // servers.password is a varchar and an encrypted long token overflows it.
    'use-access-hash'     => true,
    'require-access-hash' => true,

    'use-test-connection' => true,
    'use-port'            => true,
    'not-secure-port'     => 2082,
    'secure-port'         => 2083,

    // Extra fields land in $server['fields']. 'crypt' stores the value encrypted.
    'fields' => [
        'region'    => ['type' => 'text', 'name' => '{lang.region}', 'col_class' => 'col-md-6'],
        'sub_token' => ['type' => 'password', 'name' => '{lang.sub_token}', 'crypt' => true],
    ],

    // Which service option identifies the account on the provider side.
    'service-relationship' => 'domain',
];
```

> **The password is already decrypted**
> 
> The base class round trips `$server['password']` in one place, so it always reaches `define_server_info()` in plain text. Decrypting it again gives the API a mangled secret.

### Product Fields and the Plan Dropdown

`product_configuration()` returns a field descriptor map. What the admin picks is saved as the product's module data, and it reaches your provisioning code as `$this->options['creation_info']`.

Make the option value the plan **name**, not the provider's numeric id, and resolve it to an id inside `create()`.

### The Lifecycle Body

Every provisioning verb has the same three beats: read what the buyer chose, call the provider, return what to store. Returning an array is how you persist.

1. Read the plan from `creation_info`, the limits through `get_limit()`, the addon answers from `addon_params`, the order form answers from `requirement_params`.
2. Call the provider. Let the client throw on an API error, or throw yourself with a translated message.
3. Return `['config' => [...]]`; the core merges it into the service options and every later verb reads the account back from there.

## Reference

### Lifecycle Signatures

None is declared abstract on the base class. The core probes each with `method_exists()` and skips what is absent, so the signature is the contract.

```php
// Setup. define_server_info runs from set_server(), before any other verb.
protected function define_server_info(array $server = []): void;
public function test_connect(): array|bool;
public function configure_features(): void;

// Product and service forms. Both save methods take the values BY REFERENCE.
public function product_configuration(array $data = []): array;
public function save_product_configuration(array &$values): void;
public function service_configuration(): array;
public function save_service_configuration(array &$values): void;

// Provisioning.
public function create(): array|bool;
public function suspend(): bool;
public function unsuspend(): bool;
public function cancel(): bool;
public function renew(): bool;
public function upgrade(array $new_product = []): bool;
public function change_password(string $password): bool;
public function change_limits(array $limits): bool;
public function reset_limits(): array|bool;

// Addons. $addon is one row of the service's addon table.
public function addon_create(array $addon = []): array|bool;
public function addon_suspend(array $addon = []): array|bool;
public function addon_unsuspend(array $addon = []): array|bool;
public function addon_cancel(array $addon = []): array|bool;
public function addon_upgrade(array $addon, array $new_addon): array|bool;

// Client area.
public function dashboard_data(): array;
public function tool_data(string $tool, string $action = 'index', array $params = []): array;
public function tool_action(string $tool, string $action, array $data = []): array;
public function sso_panel_login(): string;
public function sso_root_panel_login(): string;

// Metered billing and import.
public function metrics_usage(): array;
public function metrics_usage_bulk(array $services = []): array;
public function metric_enable(array $metric): void;
public function metric_disable(array $metric): void;
public function list(bool $rCount = false, array $filters = [], array $orders = [], int $start = 0, int $end = -1): array|int;
```

| Method | Required | Called when | Status afterwards |
| --- | --- | --- | --- |
| define_server_info | yes | Every instantiation | none |
| test_connect | yes | Test Connection on the server form | none |
| product_configuration | yes | Product detail, Module tab | none |
| create | yes | Order approved, or Recreate in admin | active |
| suspend | yes | Overdue invoice, or manual suspend | suspended |
| unsuspend | yes | Payment received, or manual unsuspend | active |
| cancel | yes | Cancellation processed | cancelled |
| change_password | yes | Account password changed | unchanged |
| upgrade | yes | Plan change on the same module | unchanged |
| renew | optional | Renewal invoice paid | unchanged |
| change_limits, reset_limits | optional | Provider allows per account limit overrides | unchanged |
| addon_* | optional | An addon on the service changes state | addon only |
| list | optional | Admin opens Import Accounts; implementing it makes that screen appear | none |
| metric_enable, metric_disable | optional | Metered billing with provider side limit overrides | none |

### What create() May Return

Return `true` for a bare success, or an array. Every key is an instruction; the reserved ones are merged into the service options recursively.

- **config**: Merged under `config`: the account identity, so `user`, the encrypted `password`, `home_dir`, the provider side id.
- **login**: Merged under `login`. Credentials the client area shows or the single sign on uses.
- **creation_info**: Merged under `creation_info`, the same bag the product form writes into. Record what was actually provisioned.
- **options**: Merged into the option root; the escape hatch for anything else.
- **status**: Consumed by the queue and never stored. `'inprocess'` or `'waiting'` keeps the service out of active state while asynchronous provisioning finishes.
- **any other key**: Written to the option root as is: `hostname`, `ip`, `ftp_info`.

### The State You Already Have

All of it is filled before your method runs. Do not query for any of it.

- **$this->server**: ip, hostname, username, the decrypted password, the access hash and `fields` from your config.
- **$this->service, $this->product, $this->user, $this->order**: Plain arrays: the service row, its product, the buyer, the order.
- **$this->options and save_options()**: A live array. Mutate it and call `save_options()` to persist mid method; returning an array does the same at the end.
- **get_limit(string $key): mixed**: One resolved limit, service level overriding product level. Keys: `disk_limit`, `bandwidth_limit`, `email_limit`, `database_limit`, `addons_limit`, `subdomain_limit`, `ftp_limit`, `park_limit`, `max_email_per_hour`.
- **$this->addon_params, $this->addon_params_by_id**: Merged totals of the active addons, and the same split per addon row id. Keys from `addon-params`.
- **$this->requirement_params**: The buyer's order form answers, keyed by `requirement-params`.
- **encode_str(), decode_str()**: Encrypt before storing a secret, decrypt before sending it. Never store a panel password in clear.
- **username_generator(string|int|null $domain): string**: Static. Derives a panel safe username from the domain; an empty domain yields an empty one, which the provider rejects.

### The Core Side of the Call

```php
public static function run_module(int|array $service, string $action, array $params = []): mixed;
public static function instance_module(int|array $service): ?object;
```

```php
Services::run_module($id, 'create');                       // no arguments
Services::run_module($id, 'change_password', [$password]); // one positional string
Services::run_module($service, 'upgrade', [$newProduct]);  // the new product array
Services::run_module($service, 'addon_create', [$addon]);  // one addon row
```

- **returns null**: Your class has no such method. The queue records "module method not found" and the action fails.
- **returns false**: The module refused; the queue marks the item failed and retries.
- **terminate resolves to cancel**: If `terminate()` is missing the core tries `cancel()`, then `cancelled()`.
- **gate:service.module_action**: Runs before your method. A listener returning a non empty string vetoes the action with that message.
- **filter:service.module_result**: Runs on the result before it is applied, by reference, so a listener can rewrite what is stored.
- **action:service.module_ran**: Fires after the result is applied, with the service, the instance, the action, the result and the error.

## Example

A complete provisioning method and the core code that reads the return value back. The keys you return are the keys the core merges.

```php
public function create(): array|bool
{
    $domain = $this->options['domain'] ?? '';
    if (!$domain) throw new Exception($this->lang['error-domain-required']);

    // Re-provision: reuse the identity from the previous run instead of minting a new one.
    $username = $this->options['config']['user'] ?? '';
    if (!$username) $username = self::username_generator($domain);

    $password = ($this->options['config']['password'] ?? '') !== ''
        ? $this->decode_str($this->options['config']['password'])
        : Utility::generate_hash(12);

    $creation = $this->options['creation_info'] ?? [];

    $parameters = [
        'username'  => $username,
        'password'  => $password,
        'domain'    => $domain,

        // Option values carry the plan NAME, so resolve it on this server.
        'plan_id'   => $this->resolve_plan_id($creation['plan'] ?? ''),

        'disk'      => $this->get_limit('disk_limit'),
        'bandwidth' => $this->get_limit('bandwidth_limit'),

        // A switch posts "0" or "1" as a string; empty("0") is true, so cast instead.
        'shell'     => (int) ($creation['shell_access'] ?? 0) === 1,
    ];

    // Order form answers and addon totals, keyed by the names declared in config.php.
    foreach ($this->requirement_params as $key => $value) $parameters[$key] = $value;
    foreach ($this->addon_params as $key => $value) $parameters[$key] = $value;

    // Retry safety: the queue re-runs this method after a failure.
    if (!$this->api->account_exists($username))
        $this->api->call('accounts', $parameters, 'POST');

    return [
        'config' => [
            'user'     => $username,
            'password' => $this->encode_str($password),
            'home_dir' => '/home/' . $username,
        ],
        // Not a reserved key, so it is written to the option root.
        'ip' => $this->server['ip'] ?? '',
    ];
}
```

```php
// Services::apply_module_result, simplified to the part that matters to you.
if (is_array($result)) {
    if (isset($result['config']) && is_array($result['config']))
        $options['config'] = array_replace_recursive($options['config'] ?? [], $result['config']);

    // Anything outside the reserved set lands on the option root.
    $reserved = ['config', 'login', 'creation_info', 'options', 'status'];
    foreach ($result as $rKey => $rVal)
        if (!in_array($rKey, $reserved, true)) $options[$rKey] = $rVal;
}

// ModuleQueue then decides the new service status.
$moduleStatus = is_array($result) ? ($result['status'] ?? null) : null;
$targetStatus = $moduleStatus ?: match ($action) {
    'create', 'unsuspend', 'register' => 'active',
    'suspend'                          => 'suspended',
    'cancel'                           => 'cancelled',
    default                            => null,
};

// And this is what every later verb reads:
$username = $this->options['config']['user'] ?? '';
$password = $this->decode_str($this->options['config']['password'] ?? '');
```

## Pitfalls

> **Report failure by throwing, not by returning false**
> 
> Assigning to the error property and returning `false` is a leftover of the previous generation. cPanel throws in 41 places and assigns in none. Throw a translated exception and the caller surfaces the message.

> **The queue will run create() twice**
> 
> A failure after the account exists is retried from the top, and a provider that rejects duplicates then fails forever. Check for the account first, and persist the credentials with `save_options()` as soon as you have them.

> **Store the plan name, not the provider id**
> 
> In a load balanced group the order lands on whichever machine has room, and the same plan carries a different id there. The name is stable: save the name, resolve it at call time.

> **Never test a switch value with empty()**
> 
> Approval and switch fields post the strings `"0"` and `"1"`, and `empty("0")` is true. Write `(int) ($data['key'] ?? 0) === 1` instead.

> **Do not decrypt the server password yourself**
> 
> The base class already did it, in one place. A second decryption corrupts the plain text path, and the failure only shows up against a real provider.

## Related Articles

- [Module Anatomy](https://dev.wisecp.com/en/module-anatomy)
- [Module Configuration](https://dev.wisecp.com/en/module-configuration)
- [Module Lifecycle](https://dev.wisecp.com/en/module-lifecycle)
- [Server Module Tools](https://dev.wisecp.com/en/server-module-tools)
- [Writing a Product Module](https://dev.wisecp.com/en/writing-a-product-module)
- [Domain Helpers](https://dev.wisecp.com/en/domain-helpers)

# Server Module Tools

https://dev.wisecp.com/en/server-module-tools

Tools are the pages a customer gets inside a service: file manager, databases, DNS, cron jobs. The base class owns everything around them; you supply two methods and a template.

## Overview

`ServerModule` ships a catalogue of tool descriptors, all switched off. Turning one on means declaring it supported, then answering two questions: what the page shows, and what a button does.

Everything between the browser and those two answers belongs to the base class.

- **get_tool_data()**: The read path: validates the descriptor, calls your `tool_data()`, normalizes rows, runs the filter hook, caches per request.
- **handle_tool_action()**: The write path: sanitizes and validates each field, checks the capability, calls your `tool_action()`, writes the service history with secrets masked.
- **create_tool_table()**: Builds the listing table from the declared columns and the row callback.
- **capability**: One verb the tool offers: permission gate and interface flag.

## Prerequisites

- A working server module ([Writing a Server Module](https://dev.wisecp.com/en/writing-a-server-module)). Tools open only after `create()` stores the account identity.
- Confirm on the live provider that the main entity supports both create and delete.
- If the tool creates something the provider bills for, plan the quota.

## Structure

Six layers, three of them yours.

| Layer | Where | What it holds |
| --- | --- | --- |
| Catalogue | `coremio/classes/ServerModule.php` | Descriptor, filters, rules, columns, row callbacks |
| Enabling the tool | Your `config.php` or `configure_features()` | Supported tools and capabilities |
| Read | Your module, `tool_data()` | One case per tool, then a fetch method |
| Write | Your module, `tool_action()` | One case per tool, then an action router |
| Interface | `templates/system/module/service/hosting/tools` | Shared template per tool slug |
| Text | `coremio/locale/en/cm/system/module.php` | Shared labels and messages |

## Walkthrough

### Enable the Tool

Declaring the slug switches the tool on with its default capabilities.

```php
// config.php: the declarative half.
return [
    // ... the rest of the module configuration
    'supported' => [
        'tools' => ['file-manager', 'ftp-accounts', 'databases', 'cron-jobs'],

        // Per tool option overrides, the same keys set_tool() would write.
        'tool_options' => [
            'file-manager' => ['allow_upload_overwrite' => true, 'allow_chmod_recursive' => true],
        ],
    ],
];
```

```php
public function configure_features(): void
{
    // Same effect as the config list, useful when the decision depends on live state.
    $this->support_tools(['cron-jobs' => ['list', 'create', 'delete']]);

    // The provider has no per account quota and no e-mail field on cron jobs,
    // so remove the capabilities we cannot serve, plus the column they feed.
    $this->remove_tool_capability('ftp-accounts', ['quota']);
    $this->remove_tool_column('ftp-accounts', ['quota']);

    $this->set_tool_order('databases', 5);
}
```

### Read Path

`tool_data()` is a single dispatcher; keep the per tool work in private fetch methods.

1. Return `['items' => [...]]` keyed by column name, plus any extra key the template reads.

### Write Path

`tool_action()` has the same shape, with a router matching on the action verb.

1. Throw on refusal; the message reaches the customer as a red alert.
2. Return `[]`, or a payload when the interface needs a value back.

### Columns and Text

Column titles, labels and messages come from the shared language file. The template is shared, so a field name is a contract. Provider specific strings go in your module language file.

## Reference

### Tool Descriptor

```php
$this->tools['ftp-accounts'] = [
    'name'         => 'FTP Accounts',
    // Group names depend on the module type declared in config.php:
    //   hosting: files, databases, domains, email, software, security
    //   server:  power, system, network, storage
    'group'        => 'files',
    'order'        => 20,               // position inside the group
    'icon'         => 'bi bi-hdd-network',   // a Bootstrap icon, or 'img:<url>' for a remote glyph
    'page'         => 'ftp-accounts',   // template file name under tools/
    'type'         => 'page-loader',    // 'page-loader' renders a page, 'action' is a single button
    'supported'    => false,            // your module flips this on
    'capabilities' => ['list', 'create', 'edit', 'delete', 'quota', 'directory'],
];
```

- **supported**: Ships as `false`. A request for an unsupported tool is refused before your code runs.
- **capabilities**: Both the permission list and the interface flag; checked before a button appears.
- **type**: `'page-loader'` loads the template named by `page`. `'action'` is a single button in the tool grid, optionally with `'confirm' => true`.
- **options**: Per tool switches the template reads. Set from the configuration or at runtime.
- **capability aliases**: `get_content` and `save_content` count as `edit`; `update_email` counts as `email`.

### Module Signatures

```php
// The two the base class calls. $params carries the sanitized query parameters,
// $data carries the sanitized POST body.
public function tool_data(string $tool, string $action = 'index', array $params = []): array;
public function tool_action(string $tool, string $action, array $data = []): array;

// Optional: adjust the catalogue for this module. Runs once the server record is
// bound (from the constructor, through set_server) and again from set_service.
public function configure_features(): void;

// Optional: only needed when the reset-password tool must let the panel invent one.
protected function panel_generated_password(): string;
```

### Base Class Helpers

```php
public function support_tools(array $tools): static;
public function add_tool(string $key, array $config): static;
public function set_tool(string $key, array $config): static;
public function remove_tool(string $key): static;
public function set_tool_order(string $key, int $order): static;

public function add_tool_capability(string $key, array|string $capabilities): static;
public function remove_tool_capability(string $key, array|string $capabilities): static;

public function add_tool_column(string $tool, string $column, array $config): static;
public function remove_tool_column(string $tool, array|string $columns): static;

public function add_tool_group(string $key, array $config): static;
public function set_tool_group_order(string $key, int $order): static;

public function get_tool(string $key): ?array;
public function get_tools(): array;
public function get_effective_tools(): array;
public function get_disabled_features(): array;
```

> **support_tools() reads its argument two ways**
> 
> A plain list element turns the tool on with its default capabilities. A key pointing at an array **replaces** that list with what you passed.

### Validation Layers

All three are declared in the base class and run before your action method.

```php
// 1. Sanitizing. Any field not listed falls back to 'hclear'.
protected function get_tool_action_data_filters(): array;
// The shipped rule for this tool, verbatim. Note that an FTP user name is filtered
// as an e-mail, because panels accept the user@domain form:
// 'ftp-accounts' => ['username' => 'email', 'password' => null,
//                    'directory' => 'path', 'quota' => 'numeric']
// Filters: hclear, numeric, route, identifier, domain, subdomain, hostname,
//          email, email_list, ip, url, path, json_filenames, null (pass through)

// 2. Required fields, per action. An empty string after trimming throws.
protected function get_tool_action_rules(): array;
// 'ftp-accounts' => ['create' => ['username', 'password'],
//                    'edit' => ['username'], 'delete' => ['username']]

// 3. Format checks, run after the required check and skipped for empty values.
protected function get_tool_action_field_validations(): array;
// 'mx-entry' => ['create' => ['domain' => ['domain'], 'priority' => ['numeric']]]
// Rules: url, email, email_list, email_local, ip, ipv4, ipv6, ip_or_wildcard,
//        domain, dns_label, numeric, cron_field, enum:a,b,c
```

> **A password field must be declared with the null filter**
> 
> Every other filter strips the characters that make a password strong: the account then gets a secret the customer never typed.

### tool_action() Returns

- **[]**: Ordinary success: the base class looks up `action-success-{tool}-{action}`, then `action-success-{action}`, then a generated label.
- **['status' => 'successful', 'message' => '...']**: Success with your own wording, returned as is.
- **['stream' => ...]**: A file download; the base class streams it and the request ends there.
- **['redirect' => ...]**: Reserved: immediate full page navigation, same tab.
- **throw**: Failure. The message reaches the customer, so write it translated.

### Template Variables

```php
/** @var ServerModule $module      the live instance, so $module->service is reachable */
/** @var string       $tool        the slug */
/** @var array        $tool_config the descriptor, including capabilities and options */
/** @var string       $tool_label  the translated tool name */
/** @var string       $action      'index' unless the page was opened on a sub action */
/** @var array        $data        exactly what your fetch method returned */
/** @var mixed        $table       the prepared listing table, or null when the tool has no columns */
/** @var array        $tables      sub tables, keyed by their own slug */
/** @var bool         $admin_view  true in the admin panel, false in the client area */
/** @var string|null  $error       set when the fetch failed */
```

| Browser function | Purpose |
| --- | --- |
| `request_tool_action(tool, action, data, options)` | Posts one action, with spinner and toast |
| `reload_module_content(tool)` | Reloads the tool page after a change |
| `open_modal(id, {title, body, footer})` | Builds and opens a dialog |
| `confirmDeleteModal({message, description, buttonText, onConfirm})` | The standard delete confirmation |
| `watchRequired(selector)` | Enables submit once required inputs are filled |
| `passwordInput(id, placeholder, options)` | Password field with generate, reveal and copy |

## Example

One tool end to end: fetch, router, and the base class that consumes both.

```php
public function tool_data(string $tool, string $action = 'index', array $params = []): array
{
    $username = $this->options['config']['user'] ?? '';
    $domain   = $this->options['domain'] ?? '';

    return match ($tool) {
        'ftp-accounts' => $this->fetch_ftp_accounts($username, $domain),
        'databases'    => $this->fetch_databases($username),
        default        => [],
    };
}

private function fetch_ftp_accounts(string $username, string $domain): array
{
    $response = $this->api->call('ftp/list', ['username' => $username]);

    $items = [];
    foreach ($response['data'] ?? [] as $row)
        $items[] = [
            'username'  => $row['user'] ?? '',
            'directory' => $row['dir'] ?? '/',
            'quota'     => (int) ($row['quota'] ?? 0),
        ];

    // 'items' feeds the table; anything else is read by the template.
    return ['items' => $items, 'domain' => $domain];
}

public function tool_action(string $tool, string $action, array $data = []): array
{
    $username = $this->options['config']['user'] ?? '';

    return match ($tool) {
        'ftp-accounts' => $this->action_ftp_accounts($action, $data, $username),
        default        => throw new Exception($this->lang['err-tool-unknown']),
    };
}

private function action_ftp_accounts(string $action, array $data, string $username): array
{
    return match ($action) {
        'create' => $this->ftp_create($data, $username),
        'delete' => $this->ftp_delete($data, $username),
        default  => throw new Exception($this->lang['err-action-unknown']),
    };
}

private function ftp_create(array $data, string $username): array
{
    $this->api->call('ftp/create', [
        'account'   => $username,

        // Already sanitized by the declared filters, and already checked for presence.
        'user'      => $data['username'] ?? '',
        'password'  => $data['password'] ?? '',
        'directory' => $data['directory'] ?? '/',
    ], 'POST');

    // Empty array: the base class writes the translated success message.
    return [];
}
```

```php
// ServerModule::handle_tool_action, reduced to the sequence that matters.
$tool   = Filter::init("REQUEST/tool", "route");
$action = Filter::init("REQUEST/action", "route") ?: 'index';
$data   = !empty($_POST) ? $_POST : $_GET;
unset($data['operation'], $data['method'], $data['tool'], $data['action']);

$data = $this->sanitize_tool_action_data($tool, $data);

$tool_config = $this->get_tool($tool);
if (!$tool_config) throw new \Exception('Tool not found');
if (empty($tool_config['supported'])) throw new \Exception('Tool not supported');

$this->check_tool_capability($tool_config, $action);
$this->validate_tool_action($tool, $action, $data);

$result = $this->tool_action($tool, $action, $data);

// Secrets are masked before the action reaches the service history.
foreach ($data as $k => $v)
    if (is_string($v) && $v !== '' && preg_match('/pass(word)?|secret|token/i', (string) $k))
        $data[$k] = '***';

if (empty($result)) return $this->tool_action_success_response($tool, $action);

return $result;
```

```javascript
var TOOL = 'ftp-accounts';

window.ftpCreateSubmit = function (btn) {
    request_tool_action(TOOL, 'create', {
        username:  document.getElementById('ftpUser').value,
        password:  document.getElementById('ftpPass').value,
        directory: document.getElementById('ftpDir').value,
    }, {
        button: btn,
        buttonLoader: creating_loader,
        successToast: true,
        afterDone: function () {
            close_modal(document.querySelector('.modal.show'));
            reload_module_content(TOOL);
        },
    });
};
```

## Pitfalls

> **An unimplemented capability is a button that throws**
> 
> The interface draws a control for every default capability. Remove what your router does not handle, or the customer clicks Edit and gets "Action not available".

> **A billable resource needs a quota gate**
> 
> An open create button turns curiosity into your invoice. Tie the resource to an addon and refuse the action once the allowance is used up.

> **Never name a form field action, operation, method or tool**
> 
> The helper puts the tool action in the body, then merges your data over it. A field called `action` replaces the verb and the dispatcher throws. Prefix it (`task_action`) in the template, the required rules and your action method.

> **redirect navigates away immediately**
> 
> The helper handles that key in the same tab, before your callback finishes. Use a neutral name such as `console_url` for a URL you want in a dialog or a new tab.

> **A listing only tool is not shipped**
> 
> A tool whose main entity the provider cannot both create and delete is not shipped. Take it out of the configuration and delete the dead methods.

## Related Articles

- [Writing a Server Module](https://dev.wisecp.com/en/writing-a-server-module)
- [Interface Components](https://dev.wisecp.com/en/interface-components)
- [Filtering User Input](https://dev.wisecp.com/en/filtering-user-input)
- [The Client Area Bridge](https://dev.wisecp.com/en/the-client-area-bridge)
- [Translations and Language Files](https://dev.wisecp.com/en/translations-and-language-files)
- [Product Module Client Management](https://dev.wisecp.com/en/product-module-client-management)

# Writing a Product Module

https://dev.wisecp.com/en/writing-a-product-module

A product module provisions what is neither a hosting account nor a domain: a licence, a subscription, a certificate. No server record behind it.

## Overview

A service that is neither hosting nor server nor domain resolves to the Product type: `special` and `software` services, and SSL certificates.

Four of the seven Product modules are SSL products, and SSL has its own base class beneath the generic one.

- **ProductModule**: The generic base: module trait, client area contract, service importer. Not abstract.
- **SslProductModule**: Abstract: certificate dashboard, product fields, validation parsing, seven customer actions.
- **Services::module_type()**: Hosting and server go to Servers, domain to Registrars, everything else to Product.
- **Services::run_module()**: The same single door; an unimplemented verb is skipped.

## Prerequisites

- [Module Anatomy](https://dev.wisecp.com/en/module-anatomy) and [Module Configuration](https://dev.wisecp.com/en/module-configuration).
- Certificates extend the SSL base, everything else the generic one.
- Provider API credentials, read from `$this->config`.
- A product of type `special` or `software` bound to the module.

## Structure

The same shape as every module type, minus the server.

```bash
coremio/modules/Product/Acme/
├── Acme.php            extends ProductModule, or SslProductModule for certificates
├── ApiClient.php       your HTTP wrapper
├── config.php          metadata and the settings form
├── logo.png
├── lang/en.php
└── pages/
    ├── configuration.php   the module settings screen in the admin panel
    └── dashboard.php       the management surface; its presence opens the client tab
```

| Difference | Server module | Product module |
| --- | --- | --- |
| Credentials | Server record, base class decrypts | Module configuration, you decrypt |
| Connection test | `test_connect()` on the server form | `controller_test_connection()` |
| Settings screen | From the configuration fields | Your `page_configuration()` + save controller |
| Resource limits | `get_limit()`, base class resolves | From the product module data |
| Client tools | Tool catalogue and shared templates | Your dashboard page + callable actions |
| Client tab | Always, with a dashboard | Opt in: ship `pages/dashboard.php` |

## Walkthrough

### Pick the Base Class

The SSL base is abstract: seven methods to implement, the certificate surface inherited.

```php
namespace WISECP\Modules\Product;

use Exception;
use ProductModule;

// A licence, a subscription, an application tenant.
class Acme extends ProductModule
{
    public function __construct()
    {
        parent::__construct();       // required: it runs initModule('Product')
    }
}
```

```php
namespace WISECP\Modules\Product;

use SslProductModule;

class AcmeSSL extends SslProductModule
{
    // Seven abstract methods must be implemented; the rest of the surface is inherited.
    protected function initApi(): void { /* ... */ }
    public function fetchRemoteStatus(): array { return []; }
    protected function sslProductOptions(): array { return []; }
    protected function apiReissue(string $csr, string $dcv_method, string $approver_email): array|bool { return true; }
    protected function apiResendValidation(string $domain = ''): array|bool { return true; }
    protected function apiRevalidate(string $domain = ''): array|bool { return true; }
    protected function apiChangeValidationMethod(string $domain, string $method, string $approver = 'admin'): array|bool { return true; }
}
```

### Settings Screen

The module owns its settings page. Methods prefixed `controller_` are reachable from it, hyphens turned into underscores.

1. `page_configuration()` builds the screen, usually with the form builder.
2. `controller_save()` writes the posted fields with `save_config()`. Encrypt every secret first.
3. `controller_test_connection()` proves the credentials.

### Product Fields

`product_configuration()` returns the field descriptors the admin fills per product. They become the product's module data and reach provisioning as `$this->options['creation_info']`.

### Lifecycle

Same shape as a server module: read the choices, call the provider, hand back what to store.

1. Build the API client lazily inside the method, not in the constructor.
2. Make `create()` retry safe: the queue re-runs it, so check for an existing account and persist credentials early.

### Client Surface

Nothing is exposed to the customer by default; see [Product Module Client Management](https://dev.wisecp.com/en/product-module-client-management).

## Reference

### The Generic Base

```php
class ProductModule
{
    public bool   $client_area = false;              // true only while rendering in the client area
    public string $area_link   = '';                 // client controller link carrying the service id
    public array  $client_callable_methods = [];     // handle_* names the customer may run
    public array  $client_readonly_methods = [];     // the subset served over GET, no CSRF token

    public function __construct();                   // calls initModule('Product')
    public function get_page($page_file = '', $vars = []): string;
    public function use_controller($param = '');
    public function service_management_page(): string;
    public function has_client_management(): bool;
    public function client_overview_data(): array;
    public function client_quick_actions(int $limit = 8): array;
    protected function import_service(array $data): int;
}
```

- **use_controller($param)**: Dispatches to `controller_{param}`, hyphens to underscores. An absent method returns nothing, so an unknown page fails quietly.
- **get_page($page_file, $vars)**: Loads a template from your `pages/` directory, injecting `$module`; falls back to the shared special product templates.
- **has_client_management()**: True when `pages/dashboard.php` exists or `page_dashboard()` is defined; override to `false` for admin only.
- **import_service(array $data): int**: Creates a service row for an account that already exists at the provider. Returns the id, or 0 when owner or product is missing.

### Lifecycle Signatures

None exists on the base and none is abstract; the core probes each with `method_exists()`, so the signature is the contract.

```php
// Settings screen. Reached through use_controller().
public function page_configuration(): string;
public function controller_save(): array;
public function controller_test_connection(): array;

// Product and service forms. Both save methods take their values BY REFERENCE.
public function product_configuration(array $data = []): array;
public function save_product_configuration(array &$values): void;
public function service_configuration(): array;
public function save_service_configuration(array &$values): void;

// Provisioning. Note the return type is array|bool here, wider than the server type.
public function create(): array|bool;
public function renew(): array|bool;
public function suspend(): array|bool;
public function unsuspend(): array|bool;
public function cancel(): array|bool;
public function upgrade(): array|bool;
public function change_password(string $password): bool;

// Addons, identical in shape to the server type.
public function addon_create(array $addon = []): array|bool;
public function addon_suspend(array $addon = []): array|bool;
public function addon_unsuspend(array $addon = []): array|bool;
public function addon_cancel(array $addon = []): array|bool;
public function addon_upgrade(array $addon, array $new_addon): array|bool;

// Live state and the admin dashboard.
public function fetchRemoteStatus(): array;
public function getDashboardData(): array;

// Metered billing.
public function metrics_usage(): array;
public function metrics_usage_bulk(array $services): array;
public function metric_enable(array $metric): void;
public function metric_disable(array $metric): void;

// Customer actions. The name here is the declared name with handle_ in front.
public function handle_reset_usage(): array;
```

> **upgrade() takes no argument on this type**
> 
> A server module receives the new product as `upgrade(array $new_product = [])`. Product modules declare `upgrade(): array|bool` and read the new state from the service.

### The SSL Contract

```php
abstract protected function initApi(): void;
abstract public function fetchRemoteStatus(): array;
abstract protected function sslProductOptions(): array;
abstract protected function apiReissue(string $csr, string $dcv_method, string $approver_email): array|bool;
abstract protected function apiResendValidation(string $domain = ''): array|bool;
abstract protected function apiRevalidate(string $domain = ''): array|bool;
abstract protected function apiChangeValidationMethod(string $domain, string $method, string $approver = 'admin'): array|bool;
```

```php
// Suspension has no meaning for a certificate, so both are answered for you.
public function suspend(): array|bool;
public function unsuspend(): array|bool;

// Product fields: the certificate dropdown plus the included SAN count.
public function product_configuration(array $data = []): array;
public function save_product_configuration(array &$values): void;

// The seven customer actions, each already owner, CSRF and active-service guarded.
public function handle_reissue(): array;
public function handle_resend_validation(): array;
public function handle_revalidate(): array;
public function handle_change_validation_method(): array;
public function handle_add_san(): array;
public function handle_remove_san(): array;
public function handle_download_certificate(): string;

// Helpers around the certificate state.
public function stagedSans(): array;
public function certificateId(): string;
public static function collectExpiringServices(string $module, int $maxDays = 30): array;
```

- **client_callable_methods**: All seven action names, already filled by the base.
- **client_readonly_methods**: Only `download_certificate`: streamed over GET, so no token and no POST.
- **dcv_methods**: `email`, `http`, `https`, `dns`. Anything else normalises back to e-mail.
- **shared certificate strings**: Merged into `$this->lang` at construction; your file wins on a clash.
- **fetchRemoteStatus() shape**: Reads `status`, `domain`, `ssl_type`, `sans`, `sans_included`, `sans_addon`, `sans_max`, `issued_at`, `expires_at`, `validation_method`, `approver_email`, `dcv_file`, `dcv_dns`, `serial_number`, `signature_algo`, `key_size`, `issuer`, `crt_code` and `ca_code`.

### import_service() Keys

- **owner_id, product_id**: Both required integers; a missing or unknown one returns 0 without writing.
- **cycle**: A key such as `monthly`: resolves period, duration and price, priced in the buyer's currency first.
- **period, period_time, amount, amount_cid**: Explicit overrides: `period` skips the cycle lookup, `amount` skips the price lookup.
- **options**: Merged into the service options. `established` is forced true; the product's module data goes to `creation_info`.
- **name, status, cdate, duedate, renewaldate**: Name defaults to the product name, status to `active`, the three dates to now.

## Example

A licence module: settings save, provisioning call, reading the result back.

```php
public function controller_save(): array
{
    $endpoint = Filter::init("POST/api_endpoint", "hclear");
    if (!$endpoint) throw new Exception($this->lang['err-endpoint-required']);

    $this->save_config([
        'api_endpoint' => $endpoint,

        // Pass-through: any other filter strips what makes a key strong.
        'api_key'      => $this->encode_str(Filter::init("POST/api_key", "password")),
        'mode'         => Filter::init("POST/mode", "letters"),
    ]);

    return ['status' => 'successful'];
}

public function controller_test_connection(): array
{
    $this->initApi();
    $this->api->call('ping');

    return ['status' => 'successful', 'message' => $this->lang['connection-ok']];
}

private function initApi(): void
{
    // Lazy: the instance is built in contexts that never reach the network.
    if (isset($this->api)) return;

    include_once __DIR__ . DS . 'ApiClient.php';
    $this->api = new ApiClient(
        $this->config['settings']['api_endpoint'] ?? '',
        $this->decode_str($this->config['settings']['api_key'] ?? ''),
    );
}
```

```php
public function create(): array|bool
{
    // What the admin configured on the product, with the order-time copy preferred.
    $module_data = ($this->options['creation_info'] ?? []) ?: ($this->product['module_data'] ?? []);

    $plan  = $module_data['plan'] ?? 'starter';
    $seats = (int) ($module_data['seats'] ?? 1);

    // Addons add to the base allowance; requirements are what the buyer typed.
    $seats += (int) ($this->addon_params['extra_seats'] ?? 0);
    $company = $this->requirement_params['company_name'] ?? '';

    $this->initApi();

    // Retry safety: the queue re-runs this method after a failure.
    $existing = $this->options['config']['id'] ?? '';
    if ($existing) return true;

    $result = $this->api->call('licences', [
        'plan'    => $plan,
        'seats'   => $seats,
        'company' => $company,
        'email'   => $this->user['email'] ?? '',
    ], 'POST');

    return [
        'config' => [
            'id'  => $result['licence_id'] ?? '',
            'key' => $this->encode_str($result['licence_key'] ?? ''),
        ],
        'login' => [
            'username' => $result['username'] ?? '',
            'password' => $this->encode_str($result['password'] ?? ''),
        ],
    ];
}
```

```php
// Every later verb starts from what create() returned. The core merged
// 'config' and 'login' into the service options before this ran.
public function cancel(): array|bool
{
    $licenceId = $this->options['config']['id'] ?? '';
    if (!$licenceId) return true;             // nothing was ever provisioned

    $this->initApi();
    $this->api->call('licences/' . $licenceId, [], 'DELETE');

    return true;
}

// Live provider state for the admin dashboard and the client overview.
public function fetchRemoteStatus(): array
{
    $licenceId = $this->options['config']['id'] ?? '';
    if (!$licenceId) return [];

    $this->initApi();
    $remote = $this->api->call('licences/' . $licenceId);

    return [
        'status'     => $remote['state'] ?? 'unknown',
        'seats_used' => (int) ($remote['seats_used'] ?? 0),

        // Format before returning: the client area prints these values as they are.
        'expires_at' => DateManager::format(Config::get("options/date-format"), $remote['expires'] ?? ''),
    ];
}
```

## Pitfalls

> **Do not copy a server module and delete the server parts**
> 
> `upgrade()` has a different signature, and there is no resolved limit helper and no tool catalogue. Start from a Product sandbox archetype.

> **Build the API client lazily, never in the constructor**
> 
> The instance is built on pages that never call the provider.

> **A secret in the module configuration must be encrypted**
> 
> No server record does it for you: encrypt before saving, decrypt before use, and read the posted value with the pass-through filter.

> **Report failure by throwing**
> 
> Setting an error string and returning `false` is a leftover. Throw a translated exception; the queue records it and retries.

> **Format dates and numbers before returning them**
> 
> The client area prints values exactly as given, a raw provider timestamp included.

## Related Articles

- [Product Module Client Management](https://dev.wisecp.com/en/product-module-client-management)
- [Writing a Server Module](https://dev.wisecp.com/en/writing-a-server-module)
- [Module Anatomy](https://dev.wisecp.com/en/module-anatomy)
- [Module Configuration](https://dev.wisecp.com/en/module-configuration)
- [Module Lifecycle](https://dev.wisecp.com/en/module-lifecycle)
- [The Admin Form Builder](https://dev.wisecp.com/en/the-admin-form-builder)

# Product Module Client Management

https://dev.wisecp.com/en/product-module-client-management

Five opt-in members decide what a customer sees and may do on a product service. They are the tab, the gauges, the shortcuts, the actions and the view flag.

## Overview

A product module exposes nothing to the customer by default. The five members below are opt in and type agnostic, so filling them needs no core change.

The **overview** tab shows usage rings and account rows from `client_overview_data()`; the **management** tab shows your own dashboard page. Both can carry gauges, so the module has to know which one it is filling.

- **has_client_management()**: The tab gate.
- **client_overview_data()**: Gauges and account rows for the overview tab.
- **client_quick_actions()**: Shortcut buttons on the overview.
- **client_callable_methods**: The allowlist of customer-runnable names.
- **$client_area**: True only in the customer view of the dashboard, false in the admin view.

## Prerequisites

- A working product module: [Writing a Product Module](https://dev.wisecp.com/en/writing-a-product-module).
- A service of type `special` or `software`, `active`, with a module bound. Every other state, suspended included, is refused first.
- Test as the customer; the admin view never sets the flag.

## Structure

| What you want | What the module does | Default |
| --- | --- | --- |
| No customer screen at all | Nothing | The default |
| A management tab | Ship `pages/dashboard.php` or define `page_dashboard()` | No tab |
| No tab, dashboard for admins only | Override `has_client_management()` to return false | Tab appears |
| Usage gauges on the overview | Fill `client_overview_data()` | Empty, no gauges |
| Shortcuts on the overview | Fill `client_quick_actions()` | Empty, no shortcuts |
| A runnable action | Declare it in `$client_callable_methods`, write `handle_{name}()` | Refused |
| An action over GET (a download) | Also list it in `$client_readonly_methods` | Token and POST |
| Hide a dashboard block from customers | Check `$client_area` around it | Shown to both |

```bash
Management tab clicked
  └─ GET use_module_method-less request  ->  ClientServices::get_management_details
       ├─ owner + type + active-state check
       ├─ client_module_instance()        ->  $client_area = true, $area_link set
       ├─ ProductModule::service_management_page()  ->  get_page('dashboard')
       └─ inline script tags split out, evaluated separately by the theme bridge

Overview tab
  └─ the services controller, for special and software services
       ├─ client_overview_data()   ->  usage rings + account rows
       └─ client_quick_actions()   ->  shortcut buttons

A shortcut or a dashboard button
  └─ POST use_module_method  ->  allowlist  ->  handle_{name}()
```

## Walkthrough

### Open the Management Tab

Shipping a dashboard page is the opt in: the gate looks for the file.

1. Create `pages/dashboard.php`. Inside it, `$module` is the live instance.
2. For an admin-only dashboard, override the gate. The admin service page still shows it.
3. The theme bridge strips inline scripts out of the HTML and evaluates them separately.

### Fill the Overview

`client_overview_data()` returns three lists. Return an empty array while nothing is provisioned: the overview then shows no gauges.

1. `gauges`: one entry per limited resource, zero or below meaning unlimited.
2. `resources`: optional counters — the used and total shape, or a single `value` that is not a ratio.
3. `account`: identity rows. The service password is injected for you.

### Expose an Action

Two things are required, neither alone: the name in the allowlist and a `handle_` prefixed method.

1. Declare `public array $client_callable_methods = ['reset_usage'];`, without the prefix.
2. Write `public function handle_reset_usage(): array` — no arguments; read from the request and the service.
3. Build the API client inside the handler: a customer call has nothing set up for it.

### Avoid Showing Everything Twice

The same file also serves the admin service page, which has no overview tab. Hide duplicated blocks behind `$client_area` rather than deleting them.

## Reference

### The Five Members

```php
// Declared on ProductModule; override what you need.
public bool  $client_area            = false;   // set to true only by the client renderer
public array $client_callable_methods = [];     // names WITHOUT the handle_ prefix
public array $client_readonly_methods = [];     // a subset of the above, GET-safe

public function has_client_management(): bool;
public function client_overview_data(): array;
public function client_quick_actions(int $limit = 8): array;

// Your handler for a declared callable. No parameters; returns the JSON payload.
public function handle_reset_usage(): array;
```

### client_overview_data() Shape

```php
return [
    'gauges' => [
        [
            'key'   => 'storage',
            'label' => $this->lang['storage-usage'],
            'icon'  => 'bi bi-hdd',
            'used'  => 4.5,
            'total' => 20,          // 0 or below means unlimited; pass -1 for clarity
            'unit'  => 'GB',
        ],
    ],
    'resources' => [
        // Ratio form: same keys as a gauge.
        ['key' => 'seats', 'label' => 'Seats', 'icon' => 'bi bi-people', 'used' => 3, 'total' => 10, 'unit' => ''],
        // Value form: a single reading with no limit.
        ['key' => 'region', 'label' => 'Region', 'icon' => 'bi bi-globe', 'value' => 'eu-west'],
    ],
    'account' => [
        ['key' => 'account_id', 'label' => 'Account ID', 'type' => 'text', 'copyable' => true, 'value' => 'ac_1042'],
        ['key' => 'username',   'label' => 'Username',   'type' => 'text', 'copyable' => true, 'value' => 'acme'],
        ['key' => 'status',     'label' => 'Status',     'type' => 'badge', 'badge_color' => 'success', 'value' => 'Active'],
        ['key' => 'console',    'label' => 'Console',    'type' => 'link',  'value' => 'https://panel.example.com'],
    ],
];
```

- **gauges: total**: Zero or below is unlimited: the gauge moves to the specification list with the infinity sign.
- **gauges: no percentage**: Send raw numbers. The theme computes the ratio and the colour tier; a pre-computed percentage is ignored.
- **resources: total of zero**: Zero is a real limit here, unlike a gauge. Only a negative total is unlimited.
- **account: type**: `text`, `link`, `badge` or `password`. `copyable` adds a copy button; a link row takes the full width.
- **account: badge_color**: `success`, `warning`, `danger` or the default. The icon comes from the colour.
- **account: the password row**: Injected after the row keyed `username`, or appended when there is no such row.
- **every value is shown as given**: Format dates and numbers first: a raw provider timestamp reaches the customer as one.

### client_quick_actions() Shape

```php
public function client_quick_actions(int $limit = 8): array
{
    $actions = [
        // action => true: runs the handler in place, through the callable allowlist.
        ['key' => 'reset_usage', 'label' => $this->lang['action-reset-usage'],
         'icon' => 'bi bi-arrow-counterclockwise', 'action' => true, 'method' => 'reset_usage'],

        // action => false: navigates to a page inside the management tab instead.
        ['key' => 'backups', 'label' => $this->lang['action-backups'],
         'icon' => 'bi bi-archive', 'action' => false, 'method' => 'backups'],
    ];

    // Honour the limit: the caller decides how many fit.
    return array_slice($actions, 0, $limit);
}
```

### Before Your Handler Runs

Most "my action does nothing" reports are one of these refusals.

```php
// ClientServices::use_module_method, reduced to the decisions.
$service = $this->owned_managed_service($uid);      // owner, type, module and active-state check
$method  = (string) Filter::init("REQUEST/method", "route");

$module = $this->client_module_instance($service);  // sets $client_area and $area_link

$handleMethod = 'handle_' . str_replace('-', '_', $method);
$isTool       = in_array($method, ['tool_action', 'tool_table', 'sso_panel_login'], true);

// BOTH conditions: declared in the allowlist AND the handler actually exists.
$isClientCallable = !$isTool
    && in_array($method, $module->client_callable_methods ?? [], true)
    && method_exists($module, $handleMethod);

if (!$isTool && !$isClientCallable)
    throw new \Exception(Language::gc("website/services/err-invalid"));

// Read-only callables skip the token and the active-service requirement.
$isReadonly = $isClientCallable && in_array($method, $module->client_readonly_methods ?? [], true);
$mutates    = ($method === 'tool_action' || $isClientCallable) && !$isReadonly;

if ($mutates && !\Validation::verify_csrf_token((string) Filter::init("POST/token", "hclear"), "services"))
    throw new \Exception(Language::g("needs/csrf-failed"));

if ($mutates && ($service['status'] ?? '') !== 'active')
    throw new \Exception(Language::gc("website/services/err-invalid"));
```

- **ownership, type and module**: It must belong to the signed-in account, be hosting, server, special or software, and name a module other than `none`. A service marked with Restrict Service Details is also not found.
- **the allowlist is not optional**: An undeclared handler is refused even when it exists, and silently, so it reads like a broken button. That is what keeps `handle_create` unreachable.
- **token and live service**: The owner lookup already demands `active`, so a suspended service is refused for reads too. Mutations also need a valid token.
- **gate:service.client_tool**: Runs immediately before your handler. A listener returning a non empty string vetoes the call with that message.
- **action:service.client_tool_ran**: Fires afterwards with the service, the requested method, the resolved method name and the result.
- **filter:client.service_management_content**: Filters the finished dashboard HTML by reference, before the inline scripts are split out.

## Example

The module half and the dashboard half; the second makes the `$client_area` decision.

```php
// Declared without the handle_ prefix. Anything absent here is refused.
public array $client_callable_methods = ['reset_usage', 'download_report'];

// Served over a GET link, so exempt from the token and the POST requirement.
public array $client_readonly_methods = ['download_report'];

public function client_overview_data(): array
{
    $remote = $this->fetchRemoteStatus();
    if (!$remote) return [];                       // nothing provisioned yet: no gauges

    $gauges = [];
    if (isset($remote['storage_limit']))
        $gauges[] = [
            'key'   => 'storage',
            'label' => $this->lang['storage-usage'],
            'icon'  => 'bi bi-hdd',
            'used'  => (float) ($remote['used_storage'] ?? 0),
            'total' => (float) $remote['storage_limit'] > 0 ? (float) $remote['storage_limit'] : -1,
            'unit'  => 'GB',
        ];

    $account = [];
    $username = (string) ($this->options['login']['username'] ?? '');

    // The controller injects the password row right after this one.
    if ($username !== '')
        $account[] = ['key' => 'username', 'label' => $this->lang['username'], 'type' => 'text', 'copyable' => true, 'value' => $username];

    $status = (string) ($remote['status'] ?? '');
    if ($status !== '')
        $account[] = [
            'key'         => 'status',
            'label'       => $this->lang['account-status'],
            'type'        => 'badge',
            'badge_color' => $status === 'active' ? 'success' : 'secondary',
            'value'       => $this->lang['status-' . $status] ?? ucfirst($status),
        ];

    if (!$gauges && !$account) return [];

    return ['gauges' => $gauges, 'resources' => [], 'account' => $account];
}

public function handle_reset_usage(): array
{
    // Client-triggered: nothing has prepared the API client for you.
    $this->initApi();

    $accountId = $this->options['config']['id'] ?? '';
    if (!$accountId) throw new Exception($this->lang['err-not-provisioned']);

    $this->api->call('accounts/' . $accountId . '/usage/reset', [], 'POST');

    return ['status' => 'successful', 'message' => $this->lang['usage-reset-ok']];
}
```

```php
<?php
/** @var ProductModule $module */

// Admin has no Overview tab, so it must keep these blocks. The customer already
// sees the same numbers there, so hide them here rather than deleting them.
if (!($module->client_area ?? false)):
?>
    <div class="row">
        <!-- gauge widgets and the account card -->
    </div>
<?php endif; ?>

<!-- Actions stay in both views: this is the management surface. -->
<button type="button" class="btn btn-primary" id="acmeResetUsage">Reset Usage</button>

<script>
(function () {
    // Null-guard every lookup: a block hidden by $client_area leaves its ids missing,
    // and an uncaught error here takes the whole panel down, not just this button.
    var btn = document.getElementById('acmeResetUsage');
    if (!btn) return;

    btn.addEventListener('click', function () {
        run_module_method('reset_usage');
    });
})();
</script>
```

## Pitfalls

> **One unguarded element lookup kills the whole panel**
> 
> A block hidden by `$client_area` takes its element ids with it, and inline script that calls `addEventListener` on the missing node throws. The script bridge throws with it, so the customer gets a generic "could not load" message. Guard every lookup.

> **A shortcut is a shortcut, not the home of an action**
> 
> Put the action on the dashboard and let the shortcut point at it.

> **Empty page under the scheduler**
> 
> Template output is skipped in the scheduled-task context. A command line probe reports a zero length page while the real request returns the full panel. Check over real HTTP or impersonation.

## Related Articles

- [Writing a Product Module](https://dev.wisecp.com/en/writing-a-product-module)
- [The Client Area Bridge](https://dev.wisecp.com/en/the-client-area-bridge)
- [Server Module Tools](https://dev.wisecp.com/en/server-module-tools)
- [The Client Area](https://dev.wisecp.com/en/the-client-area)
- [Registering Hooks from a Module](https://dev.wisecp.com/en/registering-hooks-from-a-module)
- [Security Practices](https://dev.wisecp.com/en/security-practices)

# Writing an Addon Module

https://dev.wisecp.com/en/writing-an-addon-module

An addon is the module type with no fixed job. It gets a settings form, an admin page, a client page and a hook file. From there it reaches anywhere in the product, without a core edit.

## Overview

Every other module type answers a defined question. A server module provisions accounts, a payment module takes money, a registrar registers domains. An addon answers none in particular, which makes it the broadest type and the one most often misused.

The nine modules in `coremio/modules/Addons` share a base class and nothing else. They are a ticket assistant, a chat widget, identity verification, two accounting bridges, a virus scanner, a translation tool, a licence manager and the sandbox archetype.

> **An addon module is not a product add-on**
> 
> The words collide. A **product add-on** is a billing concept: an extra a customer buys alongside a service, invoiced and activated through the purchase flow. An **addon module** is a plugin you install. Nothing in this article concerns the first one.

- **AddonModule**: The base class: configuration, language, encryption, the settings save path, the enable and disable switch. Nothing is abstract or required.
- **hooks.php**: A file next to the class, loaded on every request, registering the listeners that put the addon into core flows.
- **adminArea()**: An optional page in the admin panel, routed under the addon's own address with no route registration.
- **clientArea()**: An optional page in the client area, with a menu entry and an optional pretty address.
- **use_ methods**: The request bridge. A method whose name starts with `use_` is callable over the panel's own dispatcher; nothing else is.

## Prerequisites

- [Module Anatomy](https://dev.wisecp.com/en/module-anatomy) and [Module Configuration](https://dev.wisecp.com/en/module-configuration); this article covers only the addon specific parts.
- Know what you are extending. A new screen needs the admin page; a changed behaviour needs a hook that already exists.
- Decide the audience early. An administrative addon must not expose a client page, because that also opens its request bridge to unauthenticated callers.

## Structure

```bash
coremio/modules/Addons/Acme/
├── Acme.php        the class: extends AddonModule
├── config.php      meta, status, access privileges, saved settings
├── hooks.php       optional: listeners registered on every request
├── logo.png
├── lang/en.php     $this->lang, including the meta block
├── views/          templates rendered by $this->view()
│   ├── index.php       the admin overview
│   └── client.php      the client page
└── src/            your own helper classes, built with plain new
```

```php
return [
    'created_at' => 1561714288,
    'meta' => [
        'name'         => 'Acme',
        'version'      => '1.0',
        'author'       => 'Your name goes here',
        'opening-type' => 'normal',

        // Client-area menu icon. 'font' means icon is a class, 'image' means a path or URL.
        'icon_type'    => 'font',
        'icon'         => 'bi bi-puzzle',

        // Optional pretty address: the page also opens at /{slug}, next to /addon/Acme.
        'slug'         => 'acme',
    ],
    'show_on_adminArea'  => true,     // draw the admin page
    'show_on_clientArea' => true,     // draw the client page and its menu entry
    'status'             => true,     // enabled; the panel switch writes this back
    'access_ps'          => [],       // admin privileges allowed to open it
    'settings'           => [],       // written by the settings form, read by your code
];
```

| Reach | How | A module that does it |
| --- | --- | --- |
| A page in the admin panel | `adminArea()` plus a view | Every addon with a screen |
| A page in the client area | `clientArea()` plus the configuration flag | The sandbox archetype |
| Markup injected into a core screen | A `ui:` hook | The AI assistant, on the ticket reply editor |
| A scheduled job | `register:cronjobs` plus a queue handler class | The accounting bridge, polling invoice state |
| Reacting to a core event | An `action:` hook | The accounting bridge, on invoice formalization |
| Its own API endpoints | `filter:api.routes` plus handler methods | The live chat widget |
| An extra admin menu entry | `register:admin.menu` | The chat and the virus scanner |
| An extra operation on a core controller | `register:admin.operations` | The virus scanner and the accounting bridge |

## Walkthrough

### Write the Class

The constructor does the work. It derives the name from the class, resolves the directory and the public URL, then loads the configuration and the language file. It also fills `$this->admin` and `$this->user`.

1. Name the class exactly like the directory. The base derives the name by reflection, so a mismatch breaks every path it builds.
2. Define `fields()` to get a settings form, built by the same field engine as the other module types.
3. Define `enable()` if installing needs to do anything: create a table, seed a row, check a requirement.

### Ship the Settings Form

You build neither the form nor the save path. `fields()` returns descriptors, the panel draws them, and the base class writes the values back under `settings` in your configuration file.

1. `fields()` returns the descriptor map, each entry reading its current value from `$this->config['settings']`.
2. `save_fields()` is optional: it receives the posted values, validates them, and returns the array to store. Encrypt secrets here.
3. `settings_notice()` is optional: return HTML and it appears as a banner above every field.

### Add the Pages

Both page methods return the same descriptor: a title, breadcrumbs and content. Routing is automatic, so there is nothing to register. Sub-pages are driven by a request parameter naming a view file, with a fallback when the file is missing.

### Reach Into the Core

This is what makes the type useful, and where the discipline lives. Put listeners in `hooks.php` and gate them on the addon being enabled. A hook point must never know your module's name.

1. Load the configuration cheaply at the top and wrap the listeners in a status check. Registering queue handlers is the deliberate exception and stays outside the check.
2. Register the listeners. Anything that produces markup goes in a closure, so nothing is built for a page that will not show it.
3. If the point you need does not exist, open it properly rather than editing the core in place. A generic hook plus your condition in `hooks.php` is the supported shape.

## Reference

### What the Base Class Provides

```php
class AddonModule
{
    public string|bool $error     = '';   // legacy; new code throws instead
    public array       $config    = [];   // config.php, already parsed
    public array       $lang      = [];   // lang/{lang}.php for the active language
    public string      $area_link = '';   // the addon's own page address
    public string      $_name     = '';   // the directory and class name
    public array       $user      = [];   // the signed-in customer, when there is one
    public array       $admin     = [];   // the signed-in administrator, when there is one
    public string      $url;              // public URL of the module directory
    public string      $dir;              // filesystem path of the module directory

    protected string $cryptKey = 'system';

    public function __construct();
    protected function view($file = '', $variables = []): string;
    public function privileges();
    public function save_settings($pFields, $accessPs): bool;
    public function change_addon_status($arg = '');
    public function save_config($data = []): bool;
    protected function encode_str(string $str = '', string $key = ''): string;
    protected function decode_str(string $str = '', string $key = ''): string;
    public function use_default_settings($formElements = null);
    public function isEnabled();
}
```

- **view($file, $variables)**: Loads `views/{$file}` from your module directory with the given variables extracted. Pass the file name including the extension.
- **save_config(array $data): bool**: Replaces the whole configuration file: read it, change what you need, write the result. It goes through the managed file writer, which handles the compiled-file cache.
- **encode_str(), decode_str()**: Encrypt and decrypt with the installation-bound key named by `$cryptKey`, the system key by default. Empty stays empty; a failed decryption returns an empty string, not the ciphertext.
- **isEnabled()**: Reads the status flag out of the configuration. Every hook listener should consult it before doing anything.
- **privileges()**: The full privilege list, for a settings screen where the operator picks which roles may open the addon.
- **use_default_settings($formElements)**: Wraps the standard settings screen around your fields: status switch, privilege picker and save button.

### Optional Methods

None exists on the base class. Each is probed with `method_exists()` and skipped when absent, so the signature is the contract.

```php
// Settings form.
public function fields(): array;
public function save_fields($fields = []): array|bool;   // return the array to store, or throw
public function settings_notice(): string;               // HTML banner above the fields
public function edit_settings_tab(\WISECP\Components\Tab $tab): void;   // add a tab to the settings page

// Install lifecycle. Returning false aborts the state change.
public function enable(): bool;
public function disable(): bool;
public function uninstall(): bool;

// Pages. Each returns a page descriptor.
public function adminArea(): array;
public function clientArea(): array;
public function main(): string;      // a public page for visitors who are not signed in

// Request bridge: only a use_-prefixed method is reachable.
public function use_sample_method(): array|string;
```

| Method | Runs when | Returning false means |
| --- | --- | --- |
| fields | The settings screen opens, and again on save | not applicable |
| save_fields | Settings are saved, before anything is written | Abort, with the message from the error property |
| enable | The operator switches the addon on | It stays off |
| disable | The operator switches it off | It stays on |
| uninstall | The addon is removed | Removal is refused |
| adminArea | The admin opens the addon page | not applicable |
| clientArea | A signed-in customer opens the addon page | not applicable |
| main | A visitor opens the public page | not applicable |

### The Page Descriptor

```php
return [
    'page_title'  => 'Acme',
    'breadcrumbs' => [
        ['link' => $this->area_link, 'title' => 'Acme'],
        ['link' => '', 'title' => 'Reports'],       // an empty link marks the current page
    ],

    // Admin only. Buttons drawn next to the page title.
    'page_title_buttons' => [
        [
            'outerHTML'  => '',                     // bypasses the three keys below when set
            'element'    => 'button',
            'attributes' => ['class' => 'btn btn-primary', 'onclick' => "acmeRefresh();"],
            'content'    => 'Refresh',
        ],
    ],

    'content' => $this->view('index.php', $variables),
];
```

### The Request Bridge

Two dispatchers reach an addon: one behind the admin session, one on the public site. Both apply the same rule, the requested name is prefixed with `use_` and nothing else is callable.

```php
$method = (string) Filter::init("REQUEST/method", "route");

// Spaces, hyphens and dots all normalise to an underscore, then the prefix is added.
$method = "use_" . str_replace([' ', '-', '.'], '_', $method);

if (!method_exists($instance, $method))
    throw new Exception("Module does not have a method named {$method}.");

$result = $instance->$method();

// A falsy return is treated as failure, so never return an empty array on success.
if (!$result) throw new Exception($instance->error ?: "Unknown error");
```

- **the admin bridge**: Behind the admin session and the addon privilege; the one an admin-only addon uses.
- **the website bridge**: Open only to an addon with a web face. A client page requires a signed-in customer; a public page does not.
- **no web face, no website bridge**: Without a client page or a public page the website dispatcher cannot reach the addon at all.
- **a falsy return is an error**: Return a non empty array or a non empty string. `true`, `[]` and `''` are all read as failure and turn into an exception.
- **no arguments**: The method takes none. Read the request yourself, through the input filter, exactly as an operation would.

### Hook Points Addons Actually Use

- **register:cronjobs**: Register queue handler classes. It runs on every installation, because a disabled addon never has a job dispatched to it. Register from inside the listener; the return is ignored.
- **filter:api.routes**: Append the addon's own endpoints. Check the audience argument first, then push route tuples pointing at your own methods.
- **register:admin.menu**: Add an entry to the admin navigation, so an operator can find the addon.
- **register:admin.operations**: Attach an extra operation to a core controller, so the addon can answer a request on a screen it does not own.
- **action: hooks**: React to something that happened. This one fires when an invoice is formalized, where an accounting bridge starts.
- **ui: hooks**: Inject markup into a core screen. The ticket detail page carries several, and that is where the assistant and the scanner appear.
- **action:module.addon_settings_saved**: Fires after any addon's settings are written, with the module name and the new configuration.

## Example

A minimal but complete addon: the settings, the hook file that makes it do something, the code that reads the settings back. A setting nobody reads is the most common defect in this type.

```php
namespace WISECP\Modules\Addons;

use AddonModule;
use Exception;
use Filter;

class Acme extends AddonModule
{
    public string $version = '1.0';

    public function fields(): array
    {
        $settings = $this->config['settings'] ?? [];

        return [
            'api_key' => [
                'name'        => $this->lang['api-key'],
                'description' => $this->lang['api-key-desc'],
                'type'        => 'password',
                'wrap_width'  => 100,

                // Show the stored value only as a mask; the real key stays encrypted.
                'value'       => ($settings['api_key'] ?? '') !== '' ? '********' : '',
            ],
            'notify' => [
                'name'       => $this->lang['notify'],
                'type'       => 'switch',
                'wrap_width' => 100,

                // A switch reads 'checked', not 'value'.
                'checked'    => (int) ($settings['notify'] ?? 0) === 1,
            ],
            'threshold' => [
                'name'         => $this->lang['threshold'],
                'type'         => 'text',
                'wrap_width'   => 100,
                'value'        => $settings['threshold'] ?? '100',

                // Only shown while the switch above is on.
                'parent'       => 'notify',
                'parentEffect' => 'hide',
            ],
        ];
    }

    public function save_fields($fields = []): array|bool
    {
        // The mask means "unchanged": keep whatever is already stored.
        if (($fields['api_key'] ?? '') === '********')
            $fields['api_key'] = $this->config['settings']['api_key'] ?? '';
        elseif (($fields['api_key'] ?? '') !== '')
            $fields['api_key'] = $this->encode_str($fields['api_key']);

        if ((int) ($fields['notify'] ?? 0) === 1 && (int) ($fields['threshold'] ?? 0) <= 0)
            throw new Exception($this->lang['err-threshold']);

        return $fields;
    }

    public function enable(): bool
    {
        // Anything installation needs. Returning false leaves the addon off.
        return true;
    }

    public function adminArea(): array
    {
        $action = Filter::init("REQUEST/action", "route") ?: 'index';
        if (!is_file($this->dir . 'views' . DS . $action . '.php')) $action = 'index';

        return [
            'page_title'  => $this->lang['meta']['name'],
            'breadcrumbs' => [['link' => '', 'title' => $this->lang['meta']['name']]],
            'content'     => $this->view($action . '.php', [
                'link'    => $this->area_link,
                'name'    => $this->lang['meta']['name'],
                'version' => $this->config['meta']['version'],
            ]),
        ];
    }

    // Reachable as ?operation=use_addon_method&method=refresh
    public function use_refresh(): array
    {
        $id = (int) Filter::init("POST/id", "rnumbers");
        if (!$id) throw new Exception($this->lang['err-id-required']);

        // Never return an empty array on success: the bridge reads falsy as failure.
        return ['status' => 'successful', 'id' => $id];
    }
}
```

```php
<?php

// Cheap load: the config only, without including the class. This file runs on
// every single request, so it must cost almost nothing when the addon is off.
Modules::Load('Addons', 'Acme', true);
$acme_config = Modules::Config('Addons', 'Acme') ?: [];

// Queue handlers register OUTSIDE the status check, on every installation: a job
// already sitting in the queue must still find its handler class after the addon
// is switched off. Being disabled just means no new job of this type is dispatched.
Hook::add('register:cronjobs', 1, function () {
    require_once __DIR__ . DS . 'cronjobs' . DS . 'AcmeSync.php';
    CronJobQueue::register(AcmeSync::TYPE, AcmeSync::class);
});

if ($acme_config['status'] ?? false) {

    // React to a core event. The instance is built inside the listener, not outside,
    // so an addon that is never triggered never pays for it.
    Hook::add('action:invoice.formalized', 1, function ($invoice, $userId = 0) {
        $m = Modules::getInstance('Addons', 'Acme');
        if (!$m) return;
        $m->queue_invoice((int) ($invoice['id'] ?? 0));
    });

    // Inject markup into a core screen.
    Hook::add('ui:admin.tickets_detail.bottom', 1, function ($ticket) {
        $m = Modules::getInstance('Addons', 'Acme');

        return $m ? $m->ticket_panel($ticket) : '';
    });
}
```

```php
public function queue_invoice(int $invoiceId): void
{
    if (!$invoiceId) return;

    $settings = $this->config['settings'] ?? [];

    // Never empty(): a stored "0" would read as absent and silently flip the switch on.
    if ((int) ($settings['notify'] ?? 0) !== 1) return;

    // Decrypt only at the point of use, never into a property.
    $apiKey = $this->decode_str($settings['api_key'] ?? '');
    if ($apiKey === '') throw new Exception($this->lang['err-not-configured']);

    $threshold = (int) ($settings['threshold'] ?? 0);
    // ... hand the invoice to the provider
}
```

## Pitfalls

> **The hook file runs on every request, including the ones that ignore you**
> 
> Load the configuration with the lightweight call that skips including the class. Build the instance inside each listener, not once at the top: an addon that constructs an API client at file scope taxes every page.

> **save_config() replaces the whole file**
> 
> It does not merge. Read the current configuration, change your keys, pass the complete array back. Passing only what you changed drops the meta block, the status flag and the privilege list.

> **A client page also opens the website bridge**
> 
> An addon with only an admin page cannot be reached from the public dispatcher. Adding a client page or a public page changes that for every one of its `use_` methods, not only the intended ones.

> **A bridge method must not return something falsy**
> 
> The dispatcher reads any falsy return as failure and raises an exception. A method with nothing to report still returns a non empty payload, a status key for example.

> **The core must never know your module's name**
> 
> When the behaviour you need is not reachable, open a generic hook point and put your condition in your own hook file. A core file that names your module, your flag or your table is a change the next upgrade overwrites.

> **Throw, and do not copy the error property from the sandbox**
> 
> The archetype still shows the old pattern of assigning to the error property and returning false. It is a leftover of the previous generation; production addons throw instead.

## Related Articles

- [Module Anatomy](https://dev.wisecp.com/en/module-anatomy)
- [Adding an Admin Page](https://dev.wisecp.com/en/adding-an-admin-page)
- [Registering Hooks from a Module](https://dev.wisecp.com/en/registering-hooks-from-a-module)
- [Exposing API Endpoints](https://dev.wisecp.com/en/exposing-api-endpoints)
- [Adding a Scheduled Task](https://dev.wisecp.com/en/adding-a-scheduled-task)
- [Working Without Touching the Core](https://dev.wisecp.com/en/working-without-touching-the-core)

