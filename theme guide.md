# The Theme Engine

https://dev.wisecp.com/en/the-theme-engine

A website theme owns the whole public surface: the markup, styling and wording of every page a visitor sees.

## Overview

The admin panel and the website are built by different systems. The panel is fixed: plain PHP templates. The website is themed: a directory of views the platform hands its data to.

A theme is not a skin: no markup underneath to fall back on. No cart view means no cart, so every theme implements the same list of surfaces.

## Structure

### Template Engines

Three engines are supported; a theme commits to one in its manifest. The same data reaches the view either way.

- **smarty**: Tag syntax with its own filters and functions, views ending in `.tpl`.
- **twig**: The other tag engine, views ending in `.twig`.
- **php**: Plain `.php` templates, no tag layer. Full language access, full responsibility for escaping.

### What a Theme Holds

A theme directory sits under `templates/website`. The directory is the theme; the manifest is the first file read.

```bash
templates/website/{Theme}/
├── theme.php     # the manifest: engine, meta, status, settings schema. Written by the author
├── config.php    # the saved setting VALUES. Written at runtime, never by hand
├── hooks.php     # the theme's listeners: variables, output filters, routing additions
├── cover.png     # the catalogue thumbnail meta['image'] points at
├── layouts/      # page shells a view extends: default, auth, checkout, invoice
├── partials/     # the pieces a layout assembles: header, footer, topbar, drawer, popup
├── components/   # markup two or more views share: plan grid, payment methods, panels
├── views/        # the surfaces, grouped by area: account/ auth/ checkout/ content/ page/ products/
├── tables/       # column presets for the client area list tables
├── assets/       # css, js, images, favicon the theme ships
├── locale/       # the theme's own wording, per language and per view scope
└── content/      # operator edits of that wording, written from the panel
```

- **theme.php**: The only required file: engine, catalogue entry, settings schema. Without it the directory never appears in the panel.
- **config.php**: The saved values. The platform writes it; a fresh theme ships without one.
- **layouts/**: Page shells: public, checkout, invoice, plus an optional sign-in shell. A view extends one and fills its blocks.
- **partials/**: Pieces a layout assembles rather than a view: header, footer, topbar, drawer, popup.
- **components/**: Markup two or more views share: plan grid, payment methods, dashboard panel.
- **views/**: The surfaces, grouped by area. A controller asks for `account/dashboard`; the engine adds directory and extension.
- **tables/**: Column presets for the client area lists. A missing preset is not an error: raw columns are shown.
- **assets/**: Everything the browser fetches: `css/`, `js/`, `images/`, favicon, libraries. One function addresses them.
- **locale/ and content/**: `locale/` holds the author's defaults, per language and view scope; `content/` holds the operator's edits.
- **hooks.php**: Optional, included once before the first page. Where a theme reaches the platform without editing a controller.

### How Data Arrives

Controllers do the work and hand the result to the view as named variables. A theme asks through a hook rather than querying the database itself.

Most variables belong to one surface, catalogued in [Template Variables](https://dev.wisecp.com/en/template-variables). A smaller set is injected on **every** themed view.

- **$setting**: Array. Manifest settings keys merged with saved values. A multilang field is already the active language's string. A colour field appears twice: `brand` as hex, `brand_rgb` as `"0, 149, 149"`.
- **$ui_lang · $ui_dir**: Strings. Active language key (`en`) and `ltr`/`rtl`. They belong on the `<html>` element; writing either by hand breaks the right to left packs.
- **$badress · $sadress · $tadress**: Strings with a trailing slash: installation root, shared `resources/`, this theme's directory. Use `$tadress` only outside `assets/`.
- **$template_dir**: String. The theme's **filesystem** path, not a URL. Printing it into markup leaks a server path.
- **$cookie_domain**: String, empty unless the installation shares cookies across subdomains. The theme's cookies must carry this scope.
- **$demo_mode**: Boolean, true only on a demonstration installation.

`$company_name` and `$current_year` are on every client page. Check anything else with `|default:` before printing.

## Reference

### The Manifest: theme.php

A plain array, no class. Five top-level keys; only `engine` changes how the theme is built. Every key is in [Theme Anatomy](https://dev.wisecp.com/en/theme-anatomy).

- **engine**: `'smarty'`, `'twig'` or `'php'`. The only place the engine is declared; it decides the view extension.
- **status**: `'ready'` or `'development'`. A development theme can be previewed but not activated. Omitting the key means ready.
- **update-url**: Where the version check posts. Empty means never checked.
- **meta**: Catalogue entry: `name`, `version`, `author`, `website`, `image`, `description`. Read from here, never from saved values.
- **meta['disabled_routes']**: Route keys the theme does not serve. Those addresses answer 404 and leave the sitemap.
- **settings**: The settings **schema**, below. The admin form is generated from it; values land in `config.php`.

### The Settings Schema

`settings` holds two maps: `groups` is `key => ['label', 'icon']`, `fields` is `key => descriptor`:

- **type**: `switch` · `checkbox` · `color` · `number` · `select` · `textarea`. Anything else appears as a text field.
- **group**: Key of the group the field belongs to. An undeclared group still appears, ungrouped.
- **label · desc · placeholder**: Not literal text: **keys into the theme's own locale file**. An unresolved key prints as itself.
- **default**: Value used until the operator saves one, so the views must be able to print it.
- **options**: Select only, as `value => label key`. Its presence makes the field a dropdown.
- **depends**: Map of `other field => required value`. The row collapses until every condition matches.
- **multilang · rows**: Text and textarea only. `multilang` gives one tab per active language and stores a `lang => value` map. `rows` sizes a textarea, default 4.

### The Theme Object

```php
public static function active(): self;                 // the installation's theme, falling back to Basic
public static function installed(): array;
public static function manifest(string $themeName): array;

public function getName(): string;
public function engine(): string;
public function exists(): bool;
public function dir(): string;                          // filesystem path, trailing separator
public function assetUrl(string $path = ''): string;    // address of a file under assets/
public function viewExists(string $view): bool;         // 'account/dashboard', no extension
public function render(string $view, array $data = []): string;
public function lang(string $key, array $vars = []): string;

public function settingsSchema(): array;                // theme.php → settings
public function savedConfig(): array;                   // config.php → the saved values
public function setting(string $key): mixed;            // saved value, else the schema default
public function allSettings(): array;                   // what the views receive as $setting
public function boot(): void;                            // includes hooks.php, once, before the first render
```

## Example

The two halves of a setting: the schema declares it, a view reads it back under `$setting`.

```php
return [
    'engine' => 'smarty',
    'meta'   => [
        'name'    => 'Acme',
        'version' => '1.0.0',
        'author'  => 'Acme Ltd',
        'image'   => 'cover.png',
    ],
    'settings' => [
        'groups' => [
            'topbar' => ['label' => 'grp_topbar', 'icon' => 'bi-megaphone'],
        ],
        'fields' => [
            'topbar_enabled' => [
                'type'    => 'switch',
                'group'   => 'topbar',
                'label'   => 'set_topbar_enabled',   // a key in locale/{lang}.php
                'default' => false,
            ],
            'topbar_text' => [
                'type'      => 'textarea',
                'group'     => 'topbar',
                'label'     => 'set_topbar_text',
                'default'   => '',
                'rows'      => 3,
                'multilang' => true,
                // The row stays collapsed until the switch above is on.
                'depends'   => ['topbar_enabled' => true],
            ],
        ],
    ],
];
```

```smarty
{if $setting.topbar_enabled}
    <div class="topbar">{$setting.topbar_text nofilter}</div>
{/if}
```

Data a theme needs on every page goes through its own hooks file. The listener gets the template path and the data, and it **must return the data**.

```php
// ONE handler per theme: only the last registration's return value survives.
Hook::add("filter:template.variables", 1, function ($template, $data) {

    // Runs on every website page, so anything with a query goes through the cache.
    $data["footer_groups"] = Cache::remember('website', 'acme_footer_' . Language::selected(), 3600,
        fn (): array => Products::groups());

    return $data;   // returning nothing drops every key the platform collected
});

// Markup, not data: the layout's hook points take a string and print it as-is.
Hook::add("ui:client.head.css", 1, fn () => '<link rel="stylesheet" href="' . Theme::active()->assetUrl('css/extra.css') . '">');
```

## Pitfalls

> **A fix belongs in every theme, not only in yours**
> 
> Themes are siblings, not forks: a defect in one is almost always in the others. Apply the correction across the set. When a core release changes a shipped theme, automatic updating stops on an installation whose active theme is custom. The updates page shows the warning and the operator takes the release by hand. Port the same change into your theme right after.

> **Form protection is the theme's job to wire, not to invent**
> 
> The protections exist in the platform, but a form only gets them if the theme includes them.

> **A template is not a place to keep a secret**
> 
> Both tag engines compile views to plain PHP on disk, literals and conditions included. Keep values and decisions in PHP.

> **A theme registers one variables listener, not several**
> 
> Only the last registration's return value survives, so a second one silently discards the first one's data.

## Related Articles

- [Theme Anatomy](https://dev.wisecp.com/en/theme-anatomy)
- [Your First Theme](https://dev.wisecp.com/en/your-first-theme)
- [Template Variables](https://dev.wisecp.com/en/template-variables)
- [Securing Theme Forms](https://dev.wisecp.com/en/securing-theme-forms)


# Theme Anatomy

https://dev.wisecp.com/en/theme-anatomy

Every key a theme can declare in `theme.php`, and which directory holds what.

## Overview

A theme is a directory and a manifest: a plain PHP array returned from `theme.php`.

`theme.php` is the **schema**, `config.php` the **values**. An upgrade replaces the first and never touches the second.

## Structure

### The Directories

Counts are from the shipped `WStyle` theme.

| Directory | Holds | Required |
| --- | --- | --- |
| `theme.php` | The manifest array | Yes |
| `layouts/` | 4 shells: default, auth, checkout, invoice | `default.*` only |
| `views/` | 63 surfaces in 6 areas, plus 6 loose files | `home.*` only |
| `partials/` | 16 pieces the layouts assemble | No |
| `components/` | 33 shared blocks, plus `home/` with 10 sections | No |
| `tables/` | 7 column presets, always `.php` | No; raw columns without it |
| `assets/` | `css/`, `js/`, `images/`, favicon, libraries | No |
| `locale/` | Author defaults, per language and scope | No |
| `content/` | Operator edits, written from the panel | No. Created on first save |
| `hooks.php` | The theme's listeners, included at boot | No |
| `config.php` | Saved setting values | No. Created on first save |

`tables/` holds plain PHP even in a Smarty or Twig theme: a preset is not a view. The list component includes it with a `$table` variable in scope. The file name is the preset the controller asked for.

### Inside views/

Views are grouped by area. A controller asks for a path without prefix or extension; the engine adds both.

```bash
views/
├── account/     25   the client area: dashboard, services, invoices, tickets, settings
├── auth/         6   sign in, sign up, forgotten password, reset, activate, accept invite
├── checkout/    14   configure, cart, checkout, pay, order complete
├── content/     14   knowledge base, news, contact, legal
├── products/     3   catalog surfaces
├── page/         1   file based pages: views/page/{slug} answers /{slug}
├── home.tpl          the one view apply_theme insists on
├── 404.tpl           theme's own not-found surface
├── maintenance.tpl   rendered while the site is closed
└── domain.tpl        the public domain search

render("account/dashboard")  ->  views/account/dashboard.tpl   (smarty)
                             ->  views/account/dashboard.twig  (twig)
                             ->  views/account/dashboard.php   (php)
```

## Reference

### Top Level Keys

| Key | Type | What it decides |
| --- | --- | --- |
| `engine` | string | `smarty`, `twig` or `php`. Sets the view extension and the sandbox; missing means `php` |
| `meta` | array | The catalogue entry plus the behaviour flags core reads |
| `status` | string | `ready` or `development`. Previewable but not activatable. Missing means ready |
| `update-url` | string | Where the version check posts. Empty, or no `meta.version`, means never checked |
| `settings` | array | The schema: `groups` and `fields`. The admin form is generated from it |

### The meta Block

Read by the theme list, the detail panel and the update check.

- **name**: Display name; the locale file wins.
- **version**: Installed version, sent as `installed-version`. Missing disables the check.
- **description**: One sentence for the catalogue card; the locale file wins.
- **image**: Card thumbnail, resolved **relative to the theme directory**, not `assets/`.
- **author, provider**: Two names for one field; `provider` wins.
- **website, providerUrl**: Author's address. Here `website` wins over `providerUrl`.
- **commercial, premium, price, period**: Either flag makes the card paid; `price` and `period` label it.
- **official, license, support, updates, features**: Detail panel rows; `license` defaults to `Open Source`.
- **docsUrl, supportUrl, demoUrl, purchaseUrl, learnMoreUrl**: Detail panel buttons, printed only when set.

### Behaviour Flags in meta

Core reads these to decide how it behaves.

- **signup_minimal**: Boolean. Set it when the register view asks for the core fields only; requirement settings for the omitted fields are then not enforced.
- **dashboard_due_soon_alert**: Boolean. Off, the dashboard reminder strip carries the overdue invoice only. On, also the next due invoice.
- **disabled_routes**: Route **keys** the theme does not serve: those addresses answer 404 and leave the sitemap.

### settings.groups

Group key to a two key descriptor. Groups only sort the admin form.

```php
$settings = [
    'groups' => [
        // group key      label = a key in the theme's locale file, NOT literal text
        'appearance' => ['label' => 'grp_appearance', 'icon' => 'bi-palette'],
        'checkout'   => ['label' => 'grp_checkout',   'icon' => 'bi-cart3'],
    ],
];
```

### settings.fields

Setting key to a descriptor: the POST name, the config key and the name views read.

- **type**: One of the six below; anything else is a text field saving a string.
- **group**: Key of the group. An undeclared group still appears, ungrouped.
- **label, desc, placeholder**: Keys into the theme's own locale file, with an English fallback.
- **default**: Used until the operator saves one, and when a submission is invalid.
- **options**: Select only, as `value => label key`. Also the allowlist: unknown values fall back to the default.
- **depends**: `other field => required value`. The row stays collapsed until every condition matches. A hidden field is still saved.
- **multilang**: Text and textarea only. One tab per active language, saved as `lang => value`.
- **rows**: Textarea height, default 4.
- **html, allowed_tags**: Text and textarea only. Without `html => true` every tag is stripped on save; with it the value is sanitized against `allowed_tags`.
- **from_logo**: Color only, an integer index. Groups the field into the brand colour pair and adds the "pick from logo" action.

### Field Types and What They Save

| type | Control | Saved value |
| --- | --- | --- |
| `switch`, `checkbox` | Checkbox, description as label | Real boolean `true` / `false` |
| `color` | Swatch plus a typable hex box | `'#rrggbb'`. 3-8 hex digits, else the default |
| `number` | Number input | Integer, cast |
| `select` | Dropdown built from `options` | The chosen key, or the default |
| `textarea` | Textarea, or language tabs with `multilang` | String, or a `lang => string` map |
| anything else | Text input, or language tabs with `multilang` | String, or a `lang => string` map |

### Reading the Manifest

```php
// The whole manifest of ANY theme, by folder name. Cached per name, empty array when absent.
public static function manifest(string $themeName): array;

// The running theme. Falls back to Basic when the configured folder is gone.
public static function active(): self;

public function meta(): array;                    // manifest['meta']
public function engine(): string;                 // lowercased, 'php' when unset
public function settings(): array;                // manifest['settings']
public function settingsSchema(): array;          // the SAME array as settings()
public function exists(): bool;                   // manifest is non-empty AND the directory is there

// Values.
public function savedConfig(): array;             // config.php, cached per instance
public function setting(string $key): mixed;      // saved value, else the field's default, else null
public function allSettings(): array;             // every schema field, merged, as views receive it

public static function is_shipped(?string $name = null): bool;   // one of the shipped themes (Basic, WStyle)
public static function engineLabel(string $engine): string;      // 'smarty' -> 'Smarty', for the panel
```

`allSettings()` is what views get as `$setting`: a multilang field as the active language's string, a colour field twice (`primary_color` plus `primary_color_rgb`).

## Example

### One Field, Declared

A promotional strip: a switch that gates a rich text field.

```php
return [
    'meta' => [
        'name'        => 'Acme',
        'version'     => '1.0.0',
        'author'      => 'Acme Ltd',
        'website'     => 'https://acme.example',
        'image'       => 'cover.png',      // beside theme.php, NOT under assets/
        'description' => '',               // locale/en.php wins, so it is left empty here

        // Behaviour flags: read by core, not by the catalogue.
        'signup_minimal'           => false,
        'dashboard_due_soon_alert' => false,
        'disabled_routes'          => ['references', 'references_detail'],
    ],
    'update-url' => '',                    // empty: never checked for updates
    'engine'     => 'smarty',
    'status'     => 'ready',
    'settings'   => [
        'groups' => [
            'topbar' => ['label' => 'grp_topbar', 'icon' => 'bi-megaphone'],
        ],
        'fields' => [
            'topbar_enabled' => [
                'type'    => 'switch',
                'group'   => 'topbar',
                'label'   => 'set_topbar_enabled',      // a key in locale/{lang}.php
                'desc'    => 'set_topbar_enabled_desc',
                'default' => false,
            ],
            'topbar_text' => [
                'type'      => 'textarea',
                'group'     => 'topbar',
                'label'     => 'set_topbar_text',
                'default'   => '',
                'rows'      => 3,
                'multilang' => true,        // one tab per active language
                'html'      => true,        // otherwise every tag is stripped on save
                'depends'   => ['topbar_enabled' => true],
            ],
        ],
    ],
];
```

```php
return [
    // The catalogue card reads these two from here, not from meta.
    'name'        => 'Acme',
    'description' => 'A compact storefront theme with a promotional strip.',

    'grp_topbar'              => 'Promotional Strip',
    'set_topbar_enabled'      => 'Show the strip',
    'set_topbar_enabled_desc' => 'Prints a single line above the header on every public page.',
    'set_topbar_text'         => 'Strip content',
];
```

### The Same Field, Read Back

The panel writes the values; the theme reads them.

```php
return [
    'topbar_enabled' => true,
    'topbar_text'    => [
        'en' => '<strong>Launch week</strong> 20% off every plan.',
        'tr' => '<strong>Lansman haftası</strong> tüm planlarda %20 indirim.',
    ],
];
```

```smarty
{* $setting.topbar_text is already the active language's string, not the map. *}
{if $setting.topbar_enabled}
    <div class="topbar">{$setting.topbar_text nofilter}</div>
{/if}
```

```php
// setting() returns the RAW stored value: a multilang field is still the lang => value map here.
$strip = Theme::active()->setting('topbar_text');
$lang  = Language::selected() ?: 'en';
$text  = is_array($strip) ? ($strip[$lang] ?? $strip['en'] ?? '') : (string) $strip;

// allSettings() is the resolved form, which is why the templates never do the above.
$resolved = Theme::active()->allSettings();
$text     = (string) ($resolved['topbar_text'] ?? '');
```

## Pitfalls

> **config.php is not yours to write**
> 
> Generated on save and left alone by an upgrade. Hand edits survive only until the operator presses save.

> **A value the schema does not declare never reaches a view**
> 
> Merged settings are built from the schema fields, not the saved file, so a stray key never reaches the templates.

> **Labels are locale keys**
> 
> A settings screen showing `set_topbar_enabled` means the key is missing from the locale file.

> **settings() and settingsSchema() are the same array**
> 
> Both return the manifest's settings block. The processed form is allSettings().

> **meta.image is relative to the theme, not to assets/**
> 
> The shipped value is a bare `cover.png` next to the manifest. A leading slash produces a broken card.

## Related Articles

- [The Theme Engine](https://dev.wisecp.com/en/the-theme-engine)
- [Your First Theme](https://dev.wisecp.com/en/your-first-theme)
- [Theme Settings](https://dev.wisecp.com/en/theme-settings)
- [Translating a Theme](https://dev.wisecp.com/en/translating-a-theme)
- [Theme Hooks and Output Filters](https://dev.wisecp.com/en/theme-hooks-and-output-filters)


# Your First Theme

https://dev.wisecp.com/en/your-first-theme

Four files, one directory and one button: a theme the installation will actually serve.

## Overview

A theme is not registered anywhere. Create a directory under `templates/website`, put a manifest in it, and the panel finds it.

Two gates decide whether the operator can switch to it: the manifest, and a default layout plus a home view.

## Prerequisites

- Write access to `templates/website` and `temp`, where compiled templates land.
- An installation you may switch themes on: activation replaces its public site.
- [The Theme Engine](https://dev.wisecp.com/en/the-theme-engine) read once, for the manifest-versus-values split.
- `developer` turned on in `coremio/configuration/debug.php`, or a template error comes back as an empty page.

## Structure

### What the Gates Actually Require

The theme screen gates nothing: one card per directory, manifest or not. Everything is decided on Activate, and the first failing check is the message.

| Requirement | Checked when | If it is not met |
| --- | --- | --- |
| `theme.php` exists | Activating, and by the card | The card appears with no engine and an Unknown author; activation refuses it |
| `status` is not `'development'` | Activating, **first** | Refused with the development message; preview still works |
| `layouts/default.*` | Activating, after the status check | Refused with the incomplete package message |
| `views/home.*` | Activating, after the status check | Same: the two files are checked as a pair |
| `locale/{lang}.php` | Never | Optional; without it labels print as raw keys |

The `*` is the extension your `engine` chose. The examples below use Smarty.

## Step by Step

### 1. Create the Directory

1. Create `templates/website/Acme/`. The folder name is the theme's identity.
2. Create `layouts/`, `views/`, `locale/` and `assets/css/` in it.

Reload the theme screen. The card shows an empty folder: placeholder cover, folder name, Unknown author.

### 2. Write the Manifest

1. Create `templates/website/Acme/theme.php` returning the array below.
2. Set `engine` deliberately: it decides every view's extension. Leaving it out means plain PHP.
3. Leave `update-url` empty while you build: the theme is then never checked for updates.

Reload the screen. The card now carries the name, version, author and engine. Activate is refused: `status` is `'development'`, checked first.

### 3. Add the Page Shell

1. Create `layouts/default.tpl`: the document, and the blocks a view fills.
2. Declare five blocks: `title`, `head`, `content`, `scripts`, `body_end`. The shipped themes use these names.
3. Use `{asset}` for every file and `{lang}` for every string.

Half the gate is satisfied; activation stays refused until the second file.

### 4. Add the Home View

1. Create `views/home.tpl`, extending the layout and filling `content`.
2. Print one real value: `{$company_name}` is on every client page.

Both halves of the integrity gate are in place; only `status` is left.

### 5. Add the Theme's Own Wording

1. Create `locale/en.php`: a flat map of key to text.
2. Put `name` and `description` in it; the card reads these before the manifest.
3. Add a key for every string the home view prints, referenced with `{lang key='...'}`.

The card now shows your name and description.

### 6. Activate It

1. Change `status` to `'ready'` in `theme.php`. While it says `'development'` the button is refused.
2. Press Activate on the Acme card in `{admin}/settings/theme?group=theme`.
3. Open the site root in another tab.

The home page is your markup, stored as a single key in `coremio/configuration/theme.php`.

## Reference

### What a View Can Call

Both tag engines run in a sandbox with an empty class allowlist. Nine functions are the entire bridge: Smarty named, Twig positional.

| Function | Smarty | Twig | Returns |
| --- | --- | --- | --- |
| `link` | `{link route='x' p1='a' p2='b'}`
`{link page='pages/1'}` | `link('x', null, 'a', 'b')` | A client URL for a route key, or a stored page's target |
| `lang` | `{lang key='k' foo='bar'}` | `lang('k')` | The theme's translation of `k`. Extra Smarty parameters fill `{foo}`; **Twig takes the key** |
| `asset` | `{asset path='css/x.css'}` | `asset('css/x.css')` | URL under the theme's `assets/`, versioned for css and js |
| `config` | `{config key='favicon'}` | `config('favicon')` | One value from the **theme** configuration. A key with a slash returns an empty string |
| `money` | `{money amount=$v currency=$c}` | `money(v, c)` | The amount with its symbol; currency defaults to the visitor's |
| `hook` | `{hook name='ui:client.head.css'}` | `hook('ui:client.head.css')` | Every listener's string return |
| `captcha` | `{captcha area='a' tray='t' class='' force=false}` | `captcha('a', 't', '', false)` | The active provider's widget, or empty when captcha is off there |
| `csrf` | `{csrf form='key'}` | `csrf('key')` | The hidden token input for that form key |
| `content` | `{content var=$page.content}` | `content(page.content)` | Operator authored HTML, printed raw. Template syntax inside it is compiled |

The Smarty sandbox also permits twenty plain PHP functions. Twig has no PHP access at all: twelve tags and seventeen filters. Anything else raises a sandbox error.

```php
// Smarty: the only PHP functions a view may call
count  sizeof  nl2br  number_format  htmlspecialchars  strip_tags
strlen  mb_strlen  substr  mb_substr  str_contains  str_starts_with
str_ends_with  ucfirst  date  time  implode  explode  in_array  is_array

// Twig: the only tags
if  for  set  block  with  apply  autoescape  verbatim  extends  include  use  embed
```

Escaping is on by default: Smarty unless you append `nofilter`, Twig unless you pipe through `raw`.

## Example

The complete theme, four files.

```php
return [
    'meta' => [
        'name'    => 'Acme',
        'version' => '1.0.0',
        'author'  => 'Acme Ltd',
        'website' => 'https://acme.example',
        'image'   => 'cover.png',   // beside this file, not under assets/
    ],
    'update-url' => '',             // empty while you build: no version check at all
    'engine'     => 'smarty',       // decides the extension of EVERY view
    'status'     => 'development',  // preview only; set to 'ready' when it can be activated
    'settings'   => [
        'groups' => [],
        'fields' => [],
    ],
];
```

```smarty
<!doctype html>
{* $ui_lang and $ui_dir arrive on every render; writing them by hand breaks RTL packs. *}
<html lang="{$ui_lang}" dir="{$ui_dir}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{block name=title}{$page_title|default:$company_name}{/block}</title>

    <link rel="icon" href="{asset path='favicon.svg'}">
    <link rel="stylesheet" href="{asset path='css/default.css'}">

    {* Page specific CSS goes here. Page specific SCRIPTS do not: see below. *}
    {block name=head}{/block}
    {hook name='ui:client.head.css'}
</head>
<body>
    {hook name='ui:client.body.begin'}

    <header class="site-header">
        <a href="{link route='home'}">{$company_name}</a>
    </header>

    <main>{block name=content}{/block}</main>

    <footer>&copy; {$current_year} {$company_name}</footer>

    {* Core scripts first, then the page's own: defer executes in document order. *}
    <script src="{asset path='js/default.js'}" defer></script>
    {block name=scripts}{/block}

    {* Modals live here, outside <main>, so a fixed overlay is not trapped in its stacking context. *}
    {block name=body_end}{/block}
    {hook name='ui:client.body.end'}
</body>
</html>
```

```smarty
{extends file='layouts/default.tpl'}

{block name=title}{lang key='home_title'} | {$company_name}{/block}

{block name=head}
    <link rel="stylesheet" href="{asset path='css/home.css'}">
{/block}

{block name=content}
    <section class="hero">
        <h1>{lang key='home_headline' brand=$company_name}</h1>
        <p>{lang key='home_lead'}</p>
        <a class="btn" href="{link route='sign-in'}">{lang key='home_cta'}</a>
    </section>
{/block}
```

```php
return [
    // Read by the theme card in the panel, ahead of meta.name / meta.description.
    'name'        => 'Acme',
    'description' => 'A minimal starter theme.',

    // The home view's strings. {brand} is filled by the tag: {lang key='home_headline' brand=$company_name}
    'home_title'    => 'Home',
    'home_headline' => 'Everything {brand} runs, in one place.',
    'home_lead'     => 'Hosting, domains and licences from a single account.',
    'home_cta'      => 'Sign in',
];
```

Set `status` to `'ready'` when Activate should work. Until then the theme is previewable but not activatable.

## Pitfalls

> **A missing view is a blank page, not an error**
> 
> The platform catches everything a template engine throws and returns an empty string. A typo or an unclosed block produces an empty page. Turn `developer` on to see it.

> **Layouts and partials are referenced from the theme root**
> 
> `{extends file='layouts/default.tpl'}`, not a path relative to the view: the template directory is the theme directory.

> **The view has nine functions and no classes**
> 
> Every platform class is unreachable from a view. A theme that needs the whole language declares `engine => 'php'`.

> **Do not build the second surface by copying the first**
> 
> Markup that appears twice belongs in `components/`, shell pieces in `partials/`.

> **A view never queries anything**
> 
> Controllers prepare the data as named variables. When a surface needs more, add a listener in the theme's hooks file.

## Related Articles

- [The Theme Engine](https://dev.wisecp.com/en/the-theme-engine)
- [Theme Anatomy](https://dev.wisecp.com/en/theme-anatomy)
- [Theme Assets](https://dev.wisecp.com/en/theme-assets)
- [Template Variables](https://dev.wisecp.com/en/template-variables)
- [Translating a Theme](https://dev.wisecp.com/en/translating-a-theme)


# Theme Assets

https://dev.wisecp.com/en/theme-assets

Where a theme's stylesheets, scripts, images and fonts live, and why only two of the four get a version query.

## Overview

Everything the browser fetches from a theme sits under its `assets/` directory, addressed by one function. Nothing is registered, bundled or compiled: the file is where you put it.

Stylesheets and scripts come back with a cache busting query built from the file's modification time. Fonts and images come back clean, deliberately.

## Structure

### Inside assets/

The layout is a convention: the function takes any path under `assets/`. Counts are from the shipped `WStyle` theme.

```bash
assets/
├── css/                    50 stylesheets: default.css, theme.css, then one per surface
│   └── libs/               5 third party bundles: bootstrap-icons, fontawesome, fonts, prism, wcp-table
├── js/                     55 scripts: default.js, money.js, then one per surface
│   └── libs/               6 third party bundles: tom-select, intl-tel-input, jspdf, ...
├── images/                 27 entries, grouped: hero/, logo/, banks/, avatars/, addons/
├── videos/                 anything heavier than an image
├── favicon.svg             addressed like any other asset
└── component-showcase.html a live catalogue of the theme's own primitives
```

`default.css` and `default.js` load on every page, so every visitor pays for them. A stylesheet named after a surface is linked from that view and by nothing else.

`component-showcase.html` is the theme's own component catalogue. Open it in a browser before writing new markup.

## Step by Step

### 1. Put the File in Place

1. Drop the file under `assets/`, in `css/`, `js/` or `images/`.
2. Name a surface file after its surface: `css/balance.css`, `js/balance.js`.
3. Third party bundles go unmodified in `css/libs/` or `js/libs/`, so upgrading one is a directory swap.

Nothing watches the directory: the file is reachable but unreferenced.

### 2. Link It from a View

1. Site wide files belong in the layout's head, once.
2. A surface's own stylesheet goes in that view's `{block name=head}`.
3. A surface's own script goes in `{block name=scripts}`, never in `head`. Core scripts print before that block, so a head script runs too early and fails silently.
4. Write the path relative to `assets/`. The function adds the rest.

Reload and read the markup. A link ending in `?v=1753974812` is your file, found on disk. A link with no query is the diagnostic below.

### 3. Reference It from PHP

1. In `hooks.php`, or any other PHP, call the same function on the active theme.
2. Inject markup through a layout hook point instead of editing the layout. An optional stylesheet ships without a second head.

The injected tag now appears wherever the hook point fires, with the same version query.

### 4. Add a Font

1. Put the `woff2` files and their `@font-face` stylesheet together under `css/libs/fonts/`.
2. Inside that stylesheet, reference the font files **relatively**. Browsers resolve `url()` against the stylesheet, not the page.
3. Link the stylesheet with the normal tag, and preload the font file the first paint needs.
4. If the font already ships with a hashed query in its stylesheet, repeat that exact query on the preload.

The network panel shows one request per font file. Two requests mean the preload and the stylesheet disagree.

## Reference

### The Addressing Function

```php
public function assetUrl(string $path = ''): string;

// $path  relative to the theme's assets/ directory. A leading slash is trimmed,
//        so 'css/default.css' and '/css/default.css' are the same request.
//        An empty string returns the assets directory itself.
//
// returns  {APP_URI}/templates/website/{Theme}/assets/{$path}
//          plus ?v={mtime} when the extension is css or js AND the file exists on disk.
```

| Argument | Returned URL | Version query |
| --- | --- | --- |
| `'css/default.css'` | `.../assets/css/default.css?v=1753974812` | Yes, the file's modification time |
| `'js/home.js'` | `.../assets/js/home.js?v=1753902114` | Yes |
| `'images/hero/banner.webp'` | `.../assets/images/hero/banner.webp` | No, deliberately |
| `'css/libs/fonts/x.woff2'` | `.../assets/css/libs/fonts/x.woff2` | No, deliberately |
| `'css/typo.css'` (no such file) | `.../assets/css/typo.css` | None. The URL is still returned, and 404s |
| `''` | `.../assets/` | None |

A link with no version query says the file is not on disk. The function stats it only to read a modification time.

### Calling It from a View

```html
<!-- Smarty: one named parameter, 'path' -->
<link rel="stylesheet" href="{asset path='css/balance.css'}">
<script src="{asset path='js/balance.js'}" defer></script>

<!-- Twig: one positional argument -->
<link rel="stylesheet" href="{{ asset('css/balance.css') }}">

<!-- Plain PHP theme, and any PHP outside a view -->
<link rel="stylesheet" href="<?= Theme::active()->assetUrl('css/balance.css') ?>">
```

- **{asset path='...'}**: Smarty. One named parameter, and only one. Mistype it and the tag falls back to an empty path: the link points at the bare assets directory instead of failing.
- **asset('...')**: Twig. Positional, and present in the sandbox's function allowlist. Same resolution, same string.
- **Theme::active()->assetUrl()**: PHP. What both tags call underneath. Use it from `hooks.php` and from any markup built outside a template.
- **$tadress**: Every page also receives the theme directory's URL as a plain variable. It is the theme root, not `assets/`, and carries no version query.

## Example

### Site Wide Assets, Once, in the Layout

The head of a shipped layout, cut to the asset lines. Order matters: the icon font is preloaded before the stylesheet that declares it.

```html
<link rel="icon" type="image/svg+xml" href="{asset path='favicon.svg'}">

<!-- The query is the hash already present in bootstrap-icons.min.css's own src.
     Get it wrong and the browser fetches the font twice. -->
<link rel="preload" href="{asset path='css/libs/bootstrap-icons/fonts/bootstrap-icons.woff2'}?e34853135f9e39acf64315236852cd5a"
      as="font" type="font/woff2" crossorigin>

<link rel="stylesheet" href="{asset path='css/libs/bootstrap-icons/bootstrap-icons.min.css'}">
<link rel="stylesheet" href="{asset path='css/libs/fonts/urbanist.css'}">
<link rel="stylesheet" href="{asset path='css/theme.css'}">
<link rel="stylesheet" href="{asset path='css/default.css'}">
<link rel="stylesheet" href="{asset path='css/default-dark.css'}">

<!-- Loaded only where it is needed, decided by a variable, not by a second layout. -->
{if $is_client_area}<link rel="stylesheet" href="{asset path='css/client-nav.css'}">{/if}

<!-- Core scripts, before the page's own block. -->
<script src="{asset path='js/bootstrap.bundle.min.js'}" defer></script>
<script src="{asset path='js/money.js'}" defer></script>
<script src="{asset path='js/default.js'}" defer></script>
```

### Surface Assets, in the View That Needs Them

A real view's two asset blocks. Stylesheets go in `head`, scripts in `scripts`. The library the page script needs is loaded in the same block, above it.

```smarty
{extends file='layouts/default.tpl'}

{block name=head}
    <link rel="stylesheet" href="{asset path='css/account-settings.css'}">
    <link rel="stylesheet" href="{asset path='css/libs/wcp-table/table.css'}">
    <link rel="stylesheet" href="{asset path='css/balance.css'}">
{/block}

{block name=scripts}
    {* The library first, then the page script that uses it: same block, document order. *}
    <script src="{asset path='js/libs/wcp-table/table.js'}" defer></script>
    <script src="{asset path='js/balance.js'}" defer></script>
{/block}

{block name=content}
    {* ... *}
{/block}
```

```php
// A hook point takes a string and prints it as is, so build the tag here and
// leave the layout alone. Same function, same version query.
Hook::add("ui:client.head.css", 1, function (): string {
    $href = Theme::active()->assetUrl('css/extra.css');

    return '<link rel="stylesheet" href="' . $href . '">';
});
```

## Pitfalls

> **No version query means the file is not there**
> 
> A URL is versioned only when the function finds the file on disk. A link that comes back as `css/balance.css` with nothing after it is a wrong path, and it is about to 404.

> **Fonts and images are unversioned on purpose. Do not "fix" it**
> 
> A font is requested twice: once by your preload tag, once by the `url()` inside the font stylesheet. The function never touches that second one. Version the preload and the two URLs stop matching. The preload is wasted, and a face declared `font-display: optional` misses first paint and keeps the fallback metrics.

> **A preload's query must match the stylesheet's query character for character**
> 
> Bundles like the icon font ship their own hashed query inside `src:`. Repeat it exactly on the preload, or the browser downloads the font twice. Read the hash from the bundle's own stylesheet, not from another theme.

> **A page script in the head is a script that runs too early**
> 
> Deferred scripts execute in document order. A page script moved into `{block name=head}` runs before the theme's core script. It cannot see the global object, and fails without an error. One exception: a library the core script itself consumes loads in the head, above it.

> **A url() inside CSS is resolved against the CSS file**
> 
> The addressing function is for markup. A background image, font file or SVG mask referenced from inside a stylesheet resolves against that stylesheet, never through the function. Keep those references relative and keep the files beside the stylesheet that names them.

## Related Articles

- [The Theme Engine](https://dev.wisecp.com/en/the-theme-engine)
- [Theme Anatomy](https://dev.wisecp.com/en/theme-anatomy)
- [Your First Theme](https://dev.wisecp.com/en/your-first-theme)
- [Theme Performance and Caching](https://dev.wisecp.com/en/theme-performance-and-caching)
- [Theme Hooks and Output Filters](https://dev.wisecp.com/en/theme-hooks-and-output-filters)

# Page Surfaces

https://dev.wisecp.com/en/page-surfaces

Every public and client page resolves to one view file inside your theme. This is the complete map from address to file.

## Overview

A controller never names a template file. It names a *view*, a slash-separated path with no extension. The engine turns that into a file inside your theme: `render("account/services")` becomes `views/account/services.tpl` for a Smarty theme and `views/account/services.twig` for a Twig one. The theme decides the extension through its manifest; the controller never learns which engine is running.

That single rule is what makes a theme portable. Ship the 69 view files a full theme needs and every surface answers. Ship fewer and the controller answers 404 for the ones you left out, because almost every website controller checks the view first.

## Structure

### The Views Tree

Views are grouped by area, never dumped flat into one folder. The counts below are the shipped themes measured file by file. **Basic** and **WStyle** carry 69 views each.

| Folder | Basic / WStyle | What lives there |
| --- | --- | --- |
| `views/` (root) | 6 | Standalone surfaces with no natural family: home, domain search, the SMS landing page, licence verification, 404, maintenance |
| `views/products/` | 3 | Catalog: category page, software detail page, software store |
| `views/checkout/` | 14 | Configure, cart, checkout, payment and their shared fragments |
| `views/account/` | 25 | The signed-in client area, from the dashboard to sub accounts |
| `views/auth/` | 6 | Login, register, password reset, activation, invitation |
| `views/content/` | 14 | Editorial surfaces: blog, news, references, knowledge base, CMS pages, contact |
| `views/page/` | 1 | File-based pages: a view here publishes a URL with no controller and no route |

### Layouts, Partials and Components

Views hold page bodies only. The shell around them lives in three sibling folders, referenced from the theme root, never relative to the view.

- **layouts/**: Full documents a view extends. WStyle ships four: `default` (public and client shell), `auth` (split stage and form), `checkout` (focused funnel shell), `invoice` (print-friendly).
- **partials/**: Chrome the layout includes, in three sets. Public: `header`, `topbar`, `drawer`, `footer`, `announcements`, `cookie-notice`. Client area: `client-topbar`, `client-sidebar`, `client-footer`. Funnel: `checkout-header`, `checkout-footer`.
- **components/**: Reusable blocks a view includes with arguments: plan grids, dashboard panels, payment method lists, configure sub cards.
- **tables/**: Plain PHP table presets for the client lists: services, domains, invoices, tickets. The list engine reads them, not a template.

## Reference

### The Resolution API

Three methods decide whether a surface exists and how it is produced. All three are on the theme instance returned by `Theme::active()`.

```php
public static function active(): self;

// True when views/<view>.<ext> is a real file. $view is slash separated, no extension.
// '' and any path containing '..' return false before the disk is touched.
public function viewExists(string $view): bool;

// Render one view to a string, independent of request context (cron, mail, PDF).
public function render(string $view, array $data = []): string;

// Read one theme setting: config.php value, or the schema default from theme.php.
public function setting(string $key): mixed;

// The manifest's meta block, including the theme flags the core reads.
public function meta(): array;

// Absolute URL under the theme's assets/ folder. Backs the {asset} template function.
public function assetUrl(string $path = ''): string;
```

- **Theme::viewExists()**: The extension comes from the manifest engine: `twig` gives `.twig`, `php` gives `.php`, anything else gives `.tpl`. Website controllers alone call it 41 times, and every one of those guards a page.
- **Theme::render()**: The context free entry point. Use it from cron, mail and PDF code; `View::chose("website")` silently skips the theme when `CRON` or `ADMINISTRATOR` is defined.
- **View::render()**: The in-request path used by every controller. It adds the address variables, runs `filter:template.variables`, loads the view's content scope and finally filters the finished markup.
- **TemplateEngine::render_file()**: The engine dispatcher underneath both. It builds a fresh Smarty or Twig instance per call and prefixes the view with `views/`.

### Root Views

| View | Produced by | Address |
| --- | --- | --- |
| `home` | `controllers/website/index.php` | route `home`, the site root |
| `domain` | `controllers/website/domain.php` | route `domain`, `/domain` |
| `sms-introduction` | `controllers/website/sms.php` | route `international-sms`, `/international-sms` |
| `license-verification` | `controllers/website/license.php` | route `license`, `/license-verify` |
| `404` | `Controllers::page_404` | no route: any unmatched address, and every failed view guard |
| `maintenance` | `controllers/system/maintenance.php` | no route: every address while maintenance mode is on |

### Catalog Views

| View | Produced by | Address |
| --- | --- | --- |
| `products/category` | `controllers/website/products.php` | routes `products` and `products-2`, `/{category}` and `/{kind}/{category}` |
| `products/software` | `controllers/website/softwares.php` | routes `softwares` and `softwares_cat`, `/softwares` and `/softwares/{category}` |
| `products/detail` | `controllers/website/product-detail.php` | route `software_detail`, `/software/{slug}` |

### Checkout Views

Four of the fourteen are fragments. They carry no layout, and another template pulls them in or an AJAX response returns them.

| View | Produced by | Address |
| --- | --- | --- |
| `checkout/configure` | `controllers/website/configure.php` | routes `configure`, `configure-p`, `/configure/{type}/{id}` |
| `checkout/configure-addon` | `configure.php`, `page_addon()` | the addon branch of the same configure address |
| `checkout/configure-domain` | `configure.php`, `page_edit_item()` | route `configure-edit`, `/configure/edit/{key}` |
| `checkout/cart` | `controllers/website/cart.php` | routes `cart` and `basket`, `/cart` |
| `checkout/checkout` | `controllers/website/checkout.php` | route `checkout`, `/checkout` |
| `checkout/order-complete` | `checkout.php`, `page_complete()` | route `order-complete`, `/checkout/complete/{id}` |
| `checkout/invoice-complete` | `invoices.php`, `page_complete()` | route `invoice-complete`, `/invoices/complete/{id}` |
| `checkout/pay` | `controllers/website/pay.php` | route `pay`, `/pay/{id}` |
| `checkout/pay-result` | `controllers/website/payment.php` | routes `pay-successful` and `pay-failed` |
| `checkout/pay-choices` | operations `ClientCheckout`, `ClientInvoicePay`, `BalanceOps` | fragment: AJAX response, three callers share it |
| `checkout/section-account` | `checkout/checkout.tpl` | fragment: included by the checkout body |
| `checkout/section-billing` | `checkout/checkout.tpl` | fragment: included by the checkout body |
| `checkout/section-payment` | `checkout/checkout.tpl` | fragment: included by the checkout body |
| `checkout/section-rail-items` | `checkout/checkout.tpl` | fragment: included by the order summary rail |

### Client Area Views

| View | Produced by | Address |
| --- | --- | --- |
| `account/dashboard-hero` | `controllers/website/dashboard.php` | route `my-account`, `/dashboard`, hero layout variant |
| `account/dashboard-standard` | `dashboard.php` | same address, standard layout variant |
| `account/settings` | `controllers/website/account.php` | route `info`, `/account/info` |
| `account/services` | `services.php`, `page_list()` | route `services`, `/services` |
| `account/service-detail` | `services.php`, `page_detail()` | route `service-detail`, `/services/detail/{id}` |
| `account/service-transfer-approve` | `services.php`, `page_transfer_approve()` | route `service-transfer-approve`, token in the path |
| `account/domains` | `domains.php`, `page_list()` | route `domains`, `/domains` |
| `account/domain-detail` | `domains.php`, `page_detail()` | route `domain-detail`, `/domain-detail/{id}` |
| `account/invoices` | `invoices.php`, `page_list()` | route `invoices`, `/invoices` |
| `account/invoice-detail` | `invoices.php`, `page_detail()` | route `invoice-detail`, `/invoices/detail/{id}` |
| `account/subscriptions` | `invoices.php`, `page_subscriptions()` | route `invoice-subscriptions`, `/invoices/subscriptions` |
| `account/bulk-pay` | `invoices.php`, `page_bulk_pay()` | route `bulk-pay`, `/invoices/bulk-pay` |
| `account/balance` | `controllers/website/balance.php` | route `balance`, `/balance` |
| `account/tickets` | `tickets.php`, `page_list()` | route `tickets`, `/tickets` |
| `account/ticket-detail` | `tickets.php`, `page_detail()` and `page_guest_view()` | routes `ticket-detail` and `ticket-guest` |
| `account/ticket-create` | `tickets.php`, `page_create()` | route `ticket-create`, `/tickets/create` |
| `account/ticket-msg` | `tickets.php` | fragment: one reply, returned to the AJAX poller |
| `account/sms` | `sms.php`, `page_panel()` | route `sms`, `/sms` |
| `account/sub-accounts` | `controllers/website/sub-accounts.php` | route `sub-accounts`, `/sub-accounts` |
| `account/api-credentials` | `controllers/website/api.php` | route `api`, `/api-credentials` |
| `account/affiliate` | `controllers/website/affiliate.php` | route `affiliate`, `/affiliate` |
| `account/reseller-program` | `reseller.php`, `page_program()` | route `reseller`, guest and non reseller view |
| `account/reseller` | `reseller.php`, `page_dashboard()` | route `reseller`, the reseller's own panel |
| `account/license-transfer-verify` | `verify-license-transfer.php` | route `verify-license-transfer`, token in the path |
| `account/access-denied` | `Controllers`, the permission guard | no route: any client page a sub account may not open |

### Auth Views

All six go through one private helper on the sign controller. That is why they share a guard, a canonical link and a page title convention.

| View | Produced by | Address |
| --- | --- | --- |
| `auth/login` | `sign.php`, `page_in()` | route `sign-in`, `/login` |
| `auth/register` | `sign.php`, `page_up()` | route `sign-up`, `/register` |
| `auth/forget-password` | `sign.php`, `page_forget()` | route `sign-forget`, `/forget-password` |
| `auth/reset-password` | `sign.php`, `page_reset()` | route `sign-reset`, `/reset-password` |
| `auth/activate` | `sign.php`, `page_activate()` | route `account-activation`, `/account-activation` |
| `auth/accept-invite` | `sign.php`, `page_invite()` | route `accept-invite`, `/invitation/{token}` |

### Content Views

| View | Produced by | Address |
| --- | --- | --- |
| `content/blog` | `controllers/website/articles.php` | route `articles`, `/articles` |
| `content/blog-detail` | `page-detail.php` | route `articles_detail`, a bare slug |
| `content/blog-comment-list` | operation `ClientBlogComments` | fragment: the comment list, reloaded over AJAX |
| `content/news` | `controllers/website/news.php` | route `news`, `/news` |
| `content/news-detail` | `page-detail.php` | route `news_detail`, a bare slug |
| `content/references` | `controllers/website/references.php` | route `references`, `/references` |
| `content/references-detail` | `page-detail.php` | route `references_detail`, a bare slug |
| `content/page-detail` | `page-detail.php` | routes `normal_detail` and `contract_detail`, every CMS page and contract |
| `content/knowledgebase` | `knowledgebase.php` | route `kbase`, `/knowledgebase` |
| `content/knowledgebase-category` | `knowledgebase.php` | route `kbase_category`, `/knowledgebase/category/{slug}` |
| `content/knowledgebase-article` | `knowledgebase.php` and `knowledgebase_detail.php` | route `kbase_detail`, `/knowledgebase/article/{slug}` |
| `content/contact` | `controllers/website/contact.php` | route `contact`, `/contact` |
| `content/newsletter-unsubscribe` | `controllers/website/newsletter.php` | no route key: the first URL segment resolves to the controller |
| `content/addon` | `controllers/website/addon.php` | no route key: `/addon/{ModuleName}`, an addon module's own page |

### File-Based Pages

A view under `views/page/` publishes an address on its own. No controller, no route entry, no core edit. A routing filter runs last, after every registered route has failed to match, and hands the request to a generic controller when the file is there.

- **views/page/about.tpl**: Published at `/about`. The slug is restricted to `a-z 0-9 / _ -` and any `..` is rejected, checked twice: once in the routing filter, once in the controller.
- **filter:routing.match**: The router's last fallback. It never shadows a registered route, so adding a file-based page can never break an existing address.
- **locale scope**: The `page/` prefix does not reach the locale: `views/page/about.tpl` reads `locale/{lang}/about.php`, because the scope mirrors the public URL, not the folder.

### What Happens When a View Is Missing

Three different things, depending on who asked. Knowing which one you are looking at saves an hour of hunting for a template error that was never raised.

| Caller | Behaviour | What you see |
| --- | --- | --- |
| A controller that guards with `viewExists()` | Returns the 404 page instead | HTTP 404 with your theme's `404` view, no error at all |
| A view with no guard | The engine throws, the dispatcher catches it and returns an empty string | A blank page. A warning goes to the log, and in development the message appears inside a preformatted block |
| An optional feature view | The feature turns itself off | No error, a different code path runs (see the PDF pitfall below) |

## Example

One surface end to end: the controller line that names the view, and the view file that answers it. The two halves are shown together because the view path is the only contract between them.

```php
// controllers/website/services.php, page_list()

// Guard first: a theme that ships no services list answers 404 rather than a blank page.
if (!Theme::active()->viewExists("account/services")) return $this->page_404("website");

$this->addData("page_title", Language::gc("website/services/meta/title"));
$this->addData("canonical_link", LinkGenerator::client("services"));
$this->addData("meta_robots", "noindex, nofollow");

// Turns on the client shell (sidebar, subnav, notification bell) and marks the active tab.
$this->addData("show_client_subnav", true);
$this->addData("subnav_active", "services");

// Logo, company name, contract links, currency list, the client chrome data.
$this->set_predefined_data("client");

// No extension, no engine: "account/services" is resolved against the ACTIVE theme.
return $this->view->chose("website")->render("account/services", $this->data, true);
```

```smarty
{extends file='layouts/default.tpl'}

{* Stylesheets and libraries the core JS itself consumes go in head. *}
{block name=head}
    <script src="{asset path='js/list.js'}" defer></script>
{/block}

{* Page scripts go in scripts, AFTER the core bundle, or window.WStyle is not ready yet. *}
{block name=scripts}
    <script src="{asset path='js/services.js'}" defer></script>
{/block}

{block name=content}
<section class="pt-4 pb-4">
    <div class="container list-page" data-services-page>
        {csrf form='services'}

        <nav aria-label="{lang key='website/services/breadcrumb-aria'}">
            <ol class="breadcrumb mb-1">
                <li class="breadcrumb-item"><a href="{link route='my-account'}">{lang key='website/index/subnav-dashboard'}</a></li>
                <li class="breadcrumb-item active" aria-current="page">{lang key='website/services/title'}</li>
            </ol>
        </nav>
    </div>
</section>
{/block}
```

## Pitfalls

> **A missing view is a blank page, not an exception**
> 
> The engine dispatcher catches every throwable and returns an empty string in production. If a surface comes out as nothing at all, look for a view file before you look at your data. In development the caught message appears on the page instead, which is the fastest way to confirm it.

> **Background output must not go through the request path**
> 
> `View::chose("website")` only resolves the theme when neither `CRON` nor `ADMINISTRATOR` is defined. From a scheduled task it silently falls back to plain PHP templates. It then fails to find your template file and returns an empty string. Use `Theme::active()->render()` for cron, mail attachments and PDF generation.

> **The dashboard is one view or two, and the theme decides**
> 
> The controller first asks for `account/dashboard`. If that file exists the theme has one dashboard and the layout setting is ignored entirely. Only when it is absent does the controller fall back to `account/dashboard-hero` or `account/dashboard-standard`, picked by the theme setting. Basic and WStyle ship the pair; a theme may ship the single view instead.

> **An optional view can switch a whole engine on**
> 
> Invoice PDF generation picks headless Chrome only when a Chrome binary resolves *and* the theme ships `account/invoice-pdf`. None of the three bundled themes ships it, so the built in generator is what actually runs today. Adding that one file changes the output engine for every invoice.

> **The maintenance view is a whole document, not a page body**
> 
> It is the one view that does not extend a layout. The site is closed, so no navigation, cart or account chrome may link back into it. It opens with its own doctype, its own head and its own first-paint guard. A theme that ships no maintenance view falls back to the legacy system template, which is not skinned.

## Related Articles

- [Theme Anatomy](https://dev.wisecp.com/en/theme-anatomy)
- [Template Variables](https://dev.wisecp.com/en/template-variables)
- [Catalog and Product Pages](https://dev.wisecp.com/en/catalog-and-product-pages)
- [Cart and Checkout](https://dev.wisecp.com/en/cart-and-checkout)
- [The Client Area](https://dev.wisecp.com/en/the-client-area)
- [Login and Registration](https://dev.wisecp.com/en/login-and-registration)
- [System Pages](https://dev.wisecp.com/en/system-pages)

# Catalog and Product Pages

https://dev.wisecp.com/en/catalog-and-product-pages

Three views carry the whole public catalog. The hard part is printing a price the JavaScript will not immediately change.

## Overview

The catalog is the only part of a theme where the same number is produced twice: the server prints it for the first paint, the browser again on every billing toggle. One cent of disagreement and the page flickers.

Amounts arrive converted and promotion resolved, so no pricing logic belongs in a view.

## Structure

Three views, one price mechanism. The category page carries two arrangements and picks between them at runtime. Until a category has priced products, every branch below is skipped.

| View | Answers | Main data |
| --- | --- | --- |
| `views/products/category` | Every category address, hub and leaf | `$mode`, `$tabs` or `$cards` plus `$plans`, `$billing_cycles`, `$price_mode` |
| `views/products/software` | The store root and each store category | `$products`, `$filters`, `$page_size`, `$total_count` |
| `views/products/detail` | One software product's own page | `$product`, `$prices`, `$gallery`, `$included`, `$versions`, `$faq` |

- **components/plan-grid.tpl**: Vertical cards. Expects `$plans` and reads `$billing_cycles` from the parent, so pass both.
- **components/plan-rows.tpl**: Horizontal rows. Expects `specs` and `chips` on each plan instead of `features`.
- **Products::catalog_plans()**: Builds both shapes; the third argument picks one.

## Step by Step

### Handle Both Category Modes

Branch on `$mode` first thing inside the content block.

1. **tabs**: every child is a leaf, so the page shows sub family tabs and a grid per pane. Each `$tabs` entry carries its own title, meta, hero background and plans.
2. **drilldown**: the category has sub categories *and* its own plans — `$cards` and `$plans`.
3. Print the billing toggle only when there is something to toggle: `{if $billing_cycles|@count > 1}`.

### Print the Plan, Not the Price Logic

1. Give the card the class `plan-card`, or `plan-row` in the horizontal layout.
2. Add the `data-*` attributes from the contract below.
3. Use `$plan.in_stock` to swap the call to action for a sold out state; do not hide the card.
4. Use `$plan.link` as it comes.

### Print the Price Twice, Identically

The server prints so the page is never blank, the script overwrites on load, and both must produce the same string.

1. Print the server value into `<span data-role="price-now">{$plan.price_now}</span>`.
2. Print the per month suffix beside it, hidden on a period total: `{if $plan.is_period}d-none{/if}`.
3. Leave the derived figures empty — they are the script's.
4. Put the page context on the wrapper.

### Respect the Category's Layout Choice

A category can ask for horizontal rows instead of the grid; the choice arrives as `$layout`.

1. Branch once, at the include: rows or grid, never a half converted card.
2. A grid plan has `features`; a rows plan has `specs` and `chips` and no `features`.
3. Feature text with no `value|label` pairs turns every line into a chip and leaves the specification columns empty. That is data, not a bug.

## Reference

### The Shape of One Plan

```php
public static function catalog_plans(int $categoryId, string $type, string $layout = 'grid'): array;

// $categoryId  the category whose products are wanted
// $type        the product kind, the same segment the configure address uses
// $layout      'grid' or 'rows'; it changes the SHAPE of every returned plan
```

```php
$plan = [
    'id'          => 18,                 // product id
    'title'       => 'Starter',
    'tagline'     => 'For a first site',
    'popular'     => true,               // from the product's own options
    'in_stock'    => true,               // '' stock means unlimited, 0 means sold out
    'prices'      => [                   // ONLY the cycles that actually carry a price
        'monthly' => 4.90,
        'annual'  => 49.00,
    ],
    'currency'    => 'USD',              // display code, for the data-currency attribute
    'currency_id' => 2,                  // display currency id, used by the formatter
    'link'        => '/configure/hosting/18',
    'features'    => ['10 GB disk', '1 domain'],
];

// With $layout = 'rows' the last key is replaced by two:
//   'specs' => [['value' => '10 GB', 'label' => 'Disk'], ...]   the comparable columns
//   'chips' => ['Free migration', ...]                          the plain lines
```

### How the First-Paint Price Is Produced

```php
public static function seed_plan_prices(array &$plans, string $cycle, bool $periodMode): void;

// $cycle       the default billing cycle for this page
// $periodMode  true prints the full period total, false prints the monthly equivalent
//
// Adds two keys to every plan:
//   price_now  the formatted string the template prints
//   is_period  whether that string is a period total (drives the "/mo" suffix)
//
// A plan with no price for $cycle gets price_now = '' rather than a zero.
// A plan with no monthly price is forced into period mode even when $periodMode is false.

public static function plan_cycles(array $tabs): array;
// [['key' => 'monthly', 'label' => 'Monthly'], ['key' => 'annual', 'label' => 'Annually']]
// Only cycles that at least one plan on the page actually prices. Fixed display order.

public static function price_monthly_equivalent(): bool;
// The GLOBAL setting behind $periodMode. It lives in the installation's theme
// configuration, NOT in your theme's own settings schema, so a theme cannot force it.
```

### The data-* Contract

Rename one and the price silently stops updating. The script takes each `data-billing` element as a group, then each `.plan-card` and `.plan-row` inside it as a card.

- **data-billing**: On the wrapper: the opening cycle from `$default_cycle`, and the group selector; the toggle rewrites it.
- **.plan-card and .plan-row**: The card selector: grid uses the first, rows the second. Add classes, do not replace these.
- **data-price-mode**: On the wrapper: `monthly` or `period` from `$price_mode`. A page value, not a theme setting.
- **data-currency**: On the wrapper and each card; the card wins, so mixed currencies format correctly.
- **data-monthly, data-annual, and so on**: On the card: raw decimals, one per cycle, with `|default:''` so an unpriced cycle stays empty rather than zero.
- **data-role**: The nodes the script writes into: `price-now`, `price-suffix`, `price-was`, `savings`, `cycle-total`, `cycle-mo`, `currency-label`.
- **data-save-text**: On the wrapper: the savings badge label, from the locale because the script has no translator.

### The Billing Toggle Contract

The button carries its cycle in `data-cycle`, never `data-billing`: the handler walks up to the nearest `data-billing` ancestor, so a button carrying it becomes its own group.

- **data-billing-toggle**: On the button group. Moves the active class between its buttons, so several toggles can coexist.
- **data-action="set-billing"**: On each button. The theme's delegated click handler dispatches on it; no listener of your own.
- **data-cycle**: On each button: `monthly`, `quarterly`, `semiannually`, `annual`, `biennial` or `triennial`. Print it from `$c.key`, the label from `$c.label`.

## Example

```php
// controllers/website/products.php, the drilldown branch

$plans = Products::catalog_plans($categoryId, $kind, $layout);

// Global setting, inverted: monthly-equivalent ON means period mode OFF.
$period_mode = !Theme::price_monthly_equivalent();

// Writes price_now + is_period onto every plan, in the page's default cycle.
Products::seed_plan_prices($plans, $default_cycle, $period_mode);

$this->addData("mode",           "drilldown");
$this->addData("plans",          $plans);
$this->addData("cards",          $cards);
$this->addData("layout",         $layout);
$this->addData("billing_cycles", Products::plan_cycles([['plans' => $plans]]));
$this->addData("default_cycle",  $default_cycle);
$this->addData("price_mode",     $period_mode ? "period" : "monthly");
```

```smarty
{* The section wrapper carries the page-level context the price script reads. *}
<section data-billing="{$default_cycle}" data-currency="{$selected_currency_code}" data-price-mode="{$price_mode}">

    {if $layout == 'rows'}
        {include file='components/plan-rows.tpl' plans=$plans billing_cycles=$billing_cycles}
    {else}
        {include file='components/plan-grid.tpl' plans=$plans billing_cycles=$billing_cycles}
    {/if}

</section>

{* components/plan-grid.tpl, the price band of one card *}
<div class="card plan-card{if !$plan.in_stock} plan-soldout{/if}"
     data-currency="{$plan.currency}"
     data-monthly="{$plan.prices.monthly|default:''}"
     data-annual="{$plan.prices.annual|default:''}"
     data-biennial="{$plan.prices.biennial|default:''}">

  <div class="plan-price price-band">
    {* Filled by the script. Empty on the server on purpose. *}
    <del class="price-was num-tabular d-none" data-role="price-was"></del>
    <span class="badge d-none" data-role="savings"></span>

    {* Printed by the server so the first paint is never blank, then rewritten
       with the SAME string by the script on load. *}
    <span class="price-now num-tabular" data-role="price-now">{$plan.price_now|default:''}</span>
    <span class="{if $plan.is_period}d-none{/if}" data-role="price-suffix">{lang key='website/products/category-per-month'}</span>

    {* One row per cycle; the totals are the script's job. *}
    {foreach $billing_cycles as $c}
      <tr data-cycle="{$c.key}">
        <td>{$c.label}</td>
        <td data-role="cycle-total"></td>
        <td data-role="cycle-mo"></td>
      </tr>
    {/foreach}
  </div>
</div>
```

## Pitfalls

> **One cent of disagreement is a visible flicker**
> 
> `toFixed(2)` rounds the stored double uncorrected, the server formatter rounds half to even, and PHP's round corrects floating point first. All three disagree on values ending in five: 237.905 becomes two strings and the page jumps. The seeding helper pre rounds with `sprintf('%.2f')`; do the same.

> **The horizontal layout has two keys behind it**
> 
> The panel writes the choice into `list_template` (2 means flat); the page reads `layout` and expects `rows`. Nothing maps between them, so a flat category still comes out as a grid; the homepage rack reads both.

> **Monthly equivalent is not your theme's setting**
> 
> An installation wide option, deliberately kept out of the theme settings schema. Read it through `$price_mode` and add no lookalike switch of your own.

> **Pasted embeds can delay every derived figure**
> 
> A category's rich text body goes out raw, so a synchronous third party tag in it blocks parsing. The main price survives; the derived figures wait.

> **Stock is live, not cached with the plan**
> 
> Plan lists are cached; the in stock flag is refreshed after. Treat `$plan.in_stock` as current and the rest as up to an hour old, and never cache a fragment mixing the two.

## Related Articles

- [Page Surfaces](https://dev.wisecp.com/en/page-surfaces)
- [Cart and Checkout](https://dev.wisecp.com/en/cart-and-checkout)
- [Template Variables](https://dev.wisecp.com/en/template-variables)
- [Theme Performance and Caching](https://dev.wisecp.com/en/theme-performance-and-caching)
- [Theme Settings](https://dev.wisecp.com/en/theme-settings)


# Cart and Checkout

https://dev.wisecp.com/en/cart-and-checkout

Fourteen views make up the purchase funnel and share one focused shell. The order summary is portalled out of the document at wide sizes.

## Overview

Configure, cart and checkout drop the public header and footer for a stripped shell with a step indicator. A visitor mid purchase should not be offered the way out.

Two things behave unlike anything else in a theme. The order summary is moved out of its container at desktop widths. The sidebar arrangement is a theme setting, not per page.

## Structure

Four addresses, numbered by `$checkout_step`, plus what follows an order.

| Step | View | What the visitor does |
| --- | --- | --- |
| 1 | `checkout/configure` | Cycle, domain, requirements, add-ons |
| 1 | `checkout/configure-addon` | An add-on for an owned service |
| 1 | `checkout/configure-domain` | Edits a domain line in the cart |
| 2 | `checkout/cart` | Reviews lines, applies a coupon, removes items |
| 3 | `checkout/checkout` | Account, billing and payment in one page |
| 3 | `checkout/pay` | An invoice, outside the cart |
| 4 | `checkout/order-complete` | Order confirmation |
| 4 | `checkout/invoice-complete` | Directly paid invoice confirmation |
| 4 | `checkout/pay-result` | Return from a redirect gateway |

The remaining five carry no layout.

- **section-account, section-billing, section-payment**: Included by `checkout/checkout.tpl`, one per card.
- **section-rail-items**: The summary's line items, separate because the summary appears twice and must match.
- **checkout/pay-choices**: Not included, returned as AJAX HTML by three operations.

## Step by Step

### Wire the Funnel Shell

1. Extend `layouts/checkout.tpl`: step indicator, thin footer, shell script, no nav.
2. Do not compute the step in the view; the controller sets it.
3. Put page scripts in `{block name=scripts}`: the core bundle loads before it, the shell after.
4. Put modals in `{block name=body_end}`: inside `<main>` a stacking context breaks a fixed overlay.

### Build the Order Summary Correctly

Count the nesting here; indentation in a design file misleads.

1. The aside must be a **direct child** of the split container and a **sibling** of the main column.
2. Give the form an `id` and attach outside submit buttons with the `form` attribute. The stack action bar and the portalled aside both land outside.
3. Build the summary from the shape the cart operations return; two shapes, two totals.

### Offer the Sidebar Variants

1. Declare `checkout_sidebar` as a select: rail, card, stack.
2. Stamp the choice on the root element from the layout. Rail is the default, with no class.
3. The shell script reads that class and skips the portal in card and stack mode.

### Host the Payment Pane

1. Give the payment section an empty pane the operation fills with returned HTML.
2. A gateway may return nothing embeddable: the response reports a fallback and your footer button takes the visitor to the pay page.
3. Never build your own gateway markup — the shared partial turns a redirect into one button.

## Reference

### Funnel Variables

Only two names are shared by the funnel. `$cart_items` in the checkout body prints and reports nothing.

- **$checkout_step**: On all four: 1 configure, 2 cart, 3 checkout or pay, 4 complete. Read by the header partial.
- **$checkout_legal**: On configure, cart and checkout: purchase time contracts, flagged per page. Not the footer links.

| Cart page | Checkout page | Holds |
| --- | --- | --- |
| `$cart_items` | `$checkout_items` | The lines, priced and formatted |
| `$cart_summary` | `$checkout_summary` | Discount groups, stacked taxes and savings |
| `$cart_subtotal`, `$cart_total` | read from the summary | Figures printed outside the summary |
| `$cart_count` | `$cart_count` | Recomputed from the lines; overrides the badge |
| `$coupon_enabled`, `$cart_has_unconfigured` | not set | Cart only: coupons on, and unconfigured lines |
| not set | `$payment_methods`, `$payment_default`, `$payment_locked` | Checkout only: gateways, preselection, no-choice flag |
| not set | `$is_member`, `$billing_profiles`, `$countries` | Checkout only: account and billing cards |

- **$domain_section**: Configure only. Always carries `visible` and `mode`. Branch on `visible` first: a suppressed card arrives as `["visible" => false, "mode" => ""]`, not absent. `license` mode adds `need_domain`, `need_ip`, `can_change`; `chooser` adds `tabs`, `tab_count`, `default_tab`, `subdomains`, `nameservers`, `check_url`, `free_json`.

### Functions the Funnel Needs

```php
// {csrf form='<key>'} prints a hidden token input, scoped by key.
// $input = false returns the bare token instead of an input element.
public static function get_csrf_token($form_index = '', $input = true);

// {money amount=$x currency=$cid} formats one amount.
// $currency defaults to the visitor's selected currency when omitted.
public static function formatter_symbol($amount = 0, $currency = 0, $exchange = false, $info = false): string;

// {link route='cart'} and {link route='configure' p1=$type p2=$id}
// p1..p5 are COLLECTED in numeric order into one list, not indexed into it:
// skipping p1 does not leave a hole, it moves p2 into the first slot, so a link
// built with p2 alone silently addresses the wrong segment.
public static function client($route = '', $params = [], $lang = '');

// {captcha area='<area>' tray='<id>' class='mb-3' force=true} renders the active
// provider, or '' when the operator has not enabled captcha for that form area.
// $opts accepts exactly three keys, and the template function passes only these:
//   tray   id of a collapse wrapper; the slot is rendered closed inside it
//   class  wrapper classes; defaults to 'mt-2' with a tray and 'mb-3' without
//   force  render unconditionally and visibly, ignoring the per-area toggle
public static function widget(string $area = '', array $opts = []): string;
```

### Gateway Pane Modes

| Mode | What the gateway returned | What reaches your pane |
| --- | --- | --- |
| `html` | Its own markup, a card form or hosted fields | That markup, embedded as is |
| `choices` | Options, say instalments or bank accounts | The shared pay-choices partial |
| `redirect` | A single destination address | The same partial with one choice: one button |
| `none` | Nothing embeddable, a legacy full page form | Nothing; a fallback is reported, the footer button takes over |

Your script sees two of these. The first three arrive as `mode: "html"` with an `html` string, the fourth as `mode: "fallback"` with no markup.

### What pay-choices Receives

Built from an explicit two key payload: layout, cart and account variables are out of scope.

```php
$this->view->chose("website")->render("checkout/pay-choices", [

    // One entry per button. A redirect gateway arrives as a list of exactly one.
    'pay_choices' => [
        [
            'url'   => 'https://gateway.example.com/session/abc',
            'label' => 'Pay now',   // the module's own button label
            'image' => '',          // when set, print the image INSTEAD of the label
        ],
    ],

    // Optional heading above the buttons. Print nothing when it is empty.
    'pay_choices_note' => '',

], true);

// The pay page (views/checkout/pay) sets the same two names itself, plus
// pay_mode, pay_html, pay_total_fmt, pay_stored_cards, pay_can_store,
// pay_can_autopay, pay_has_installments, pay_capture_url and pay_error.
```

## Example

```smarty
{extends file='layouts/checkout.tpl'}

{block name=scripts}
    <script src="{asset path='js/checkout.js'}" defer></script>
{/block}

{block name=content}
<section>
  <div class="container">

    {* The form WRAPS the split, aside included. Give it an id even so: the
       action bar below sits outside it and needs the explicit association. *}
    <form method="post" novalidate data-checkout id="checkout-form">

      {* Depth matters: the aside is a CHILD of checkout-split and a SIBLING of
         checkout-split-main. One level deeper it collapses under the content in
         card and stack mode, where the script does not portal it away. *}
      <div class="checkout-split">

        <div class="checkout-split-main">
          <div class="checkout-split-main-inner">
            {csrf form='checkout'}
            {include file='views/checkout/section-account.tpl'}
            {include file='views/checkout/section-billing.tpl'}
            {include file='views/checkout/section-payment.tpl'}
          </div>
        </div>

        {* On >=lg the shell script MOVES this node to <body>. Anything inside it
           that must submit needs form="checkout-form" from that moment on. *}
        <aside class="checkout-split-aside">
          <div class="checkout-split-aside-inner">
            {include file='views/checkout/section-rail-items.tpl'}
          </div>
        </aside>

      </div>
    </form>

    {* Stack-mode action bar: OUTSIDE the form on purpose, so form="" is the only
       thing that makes it submit. Same rule the portalled aside falls under. *}
    <div class="checkout-actionbar{if $payment_locked} d-none{/if}" data-checkout-actionbar>
      <span class="checkout-actionbar-value num-tabular" data-role="actionbar-total">{$checkout_summary.total_fmt}</span>
      <button type="submit" form="checkout-form" class="btn btn-primary">
        {lang key='website/checkout/place-order'}
      </button>
    </div>

  </div>
</section>
{/block}

{* Modals live OUTSIDE main: the content block is wrapped in a stacking context
   that breaks a fixed-position backdrop. *}
{block name=body_end}
  <div class="modal" id="billingProfileModal" tabindex="-1"></div>
{/block}
```

```php
// templates/website/{Theme}/theme.php returns the whole manifest.

return [
    'meta'   => ['name' => 'Acme', 'version' => '1.0.0', 'author' => 'Acme'],
    'engine' => 'smarty',
    'status' => 'ready',

    'settings' => [
    'groups' => [
        'checkout' => ['label' => 'grp_checkout', 'icon' => 'bi-cart3'],
    ],
    'fields' => [
        // Read by layouts/checkout.tpl, which stamps a class on the root element.
        // It is a THEME setting, not a per-page one: configure, cart and checkout
        // all change together, which is what keeps the funnel visually coherent.
        'checkout_sidebar' => [
            'type'    => 'select',
            'group'   => 'checkout',
            'label'   => 'set_checkout_sidebar',
            'desc'    => 'set_checkout_sidebar_desc',
            'options' => [
                'rail'  => 'opt_checkout_rail',    // portalled full-height rail (default)
                'card'  => 'opt_checkout_card',    // plain card beside the content
                'stack' => 'opt_checkout_stack',   // stacked under the content
            ],
            'default' => 'rail',
        ],
    ],
    ],
];
```

## Pitfalls

> **A misplaced aside is invisible in rail mode**
> 
> The script moves the summary to the body, so its position never shows. In card or stack the same markup drops it under the content.

> **A delegated handler will not see a modal button**
> 
> Modals sit outside the page root your script scopes to, so a handler checking that container silently misses them. Allow the closest modal too.

> **Use the theme's collapse, not the framework's**
> 
> The framework collapse snaps in this shell, so the shipped themes use their own everywhere, configure's domain and nameserver panels included.

> **Collecting a field is not persisting it**
> 
> Configure validates field groups the collect chain reads, not the template. An input with a plausible name arrives nowhere; follow each new field to its operation.

> **The domain card reuses the public search endpoint**
> 
> Configure has no availability endpoint: it posts to the domain controller's check operation with the public search's token form key.

## Related Articles

- [Page Surfaces](https://dev.wisecp.com/en/page-surfaces)
- [Catalog and Product Pages](https://dev.wisecp.com/en/catalog-and-product-pages)
- [Securing Theme Forms](https://dev.wisecp.com/en/securing-theme-forms)
- [Theme Settings](https://dev.wisecp.com/en/theme-settings)
- [Writing a Payment Gateway](https://dev.wisecp.com/en/writing-a-payment-gateway)

# The Client Area

https://dev.wisecp.com/en/the-client-area

Twenty five views share one shell that a single controller flag turns on. Every one can appear for an account other than the person logged in.

## Overview

This is the only family where the same page answers two questions: what is on it, and whose it is. A member can act on another account, so a list may belong to someone else. A tab may be missing because a permission was withheld, not because the data is empty.

The views extend the public layout, and a flag turns it into the client shell.

## Structure

The twenty five views group into six jobs.

| Group | Views | Notes |
| --- | --- | --- |
| Dashboard | `dashboard-hero`, `dashboard-standard` | Two variants of one address, or a single `dashboard` view |
| Assets | `services`, `service-detail`, `domains`, `domain-detail`, `service-transfer-approve`, `license-transfer-verify` | Lists use a preset; detail pages are multi tab and deep linkable |
| Money | `invoices`, `invoice-detail`, `subscriptions`, `bulk-pay`, `balance` | Two extend the invoice layout |
| Support | `tickets`, `ticket-detail`, `ticket-create`, `ticket-msg` | `ticket-msg` is a fragment: one reply for the poller |
| Account | `settings`, `sub-accounts`, `api-credentials`, `sms` | Avatar menu pages; they mark no tab |
| Programs | `affiliate`, `reseller`, `reseller-program`, `access-denied` | One reseller address, two views |

## Step by Step

### Turn On the Client Shell

1. Compute the flag once at the top of the layout: signed in *and* sub navigation requested.
2. Use it to choose the chrome: client topbar, sidebar and footer, no marketing bands.
3. Mark the active tab from `$subnav_active`. Controllers fix its values: `dashboard`, `services`, `domains`, `billing`, `support`, `sms`. Avatar menu pages carry an empty string.
4. Do not invent tab names: an unrecognised value shows no active tab, silently.

### Build for the Right Account

1. Never print the login identity as the owner: one is who signed in, the other is whose data is on screen.
2. Expect a panel to be absent rather than empty; a withheld permission means the query never ran.
3. Keep the account switcher visible; a member acting for another needs a way back.

### Build a List Surface

The four big lists print rows twice: on the server for the first page, through the table engine after.

1. Put the row markup in the theme's table preset, under `tables/`.
2. Have the view's loop print the same markup, so first paint and an AJAX page are byte identical.
3. Read page values from the table options, not globals: the preset sees no controller data.

### Make Detail Tabs Linkable

1. Give each tab button a stable key attribute, separate from the pane id, so the page does not jump.
2. On load, open the tab named in the query string; it must override the remembered tab.
3. When a tab is shown, write its key back and drop parameters from the tab you left.
4. For a lazily loaded tab, check on load and again on the next tick: the remembered tab is restored at two moments.

## Reference

### Shell Variables

- **$show_client_subnav**: Turns the default layout into the client shell. Set true **before** the predefined client data.
- **$subnav_active**: The current sub navigation tab; empty string is deliberate.
- **$notification_count**: The bell badge, always set even at zero, with a preformatted `$notification_count_text`. Core chrome.
- **$account_info**: The identity block, always the **signed-in member**. Keys: `name`, `surname`, `full_name`, `email`, `avatar`, `initials`, `balance` (formatted), `support_pin`, `two_factor`, `is_reseller`, `dealership`, `last_login`, `last_login_date`, `last_login_ip`, `last_login_country`, `last_login_city`. Fall back avatar → initials → icon.
- **$account_info.last_login_date**: Empty when the account has never signed in. Formatting an absent date stamps the current time and presents now as the last sign in — print a dash.
- **$currency_formats_json and siblings**: Sample strings, currency ids and rates, so the money script matches the server.

### The Account Context API

```php
// Null when there is no member session. Never mutates the login identity.
public static function activeAccount(): ?array;

// Own account is always allowed; a switched account only when the permission was granted.
public static function accountCan(string $permission): bool;

// Only a still-valid membership (or self) is accepted; a client-supplied id is never trusted.
public static function switchAccount(int $ownerId): bool;

// The bell count for the signed-in member. 0 means "no member" as well as "nothing unread".
public static function notification_count(int $user_id = 0): int;
```

```php
$ctx = [
    'login_id'    => 41,      // who authenticated; NEVER changes while switched
    'owner_id'    => 88,      // whose data this page shows
    'is_self'     => false,   // true when the two ids are the same
    'permissions' => ['view_services', 'view_invoices'],   // re-read every request
];

// The switched account is re-validated against the membership table on EVERY request,
// so a revoked membership loses access immediately rather than at the next login.
```

### Dashboard Data

| Variable | Holds | Behaviour when unpermitted |
| --- | --- | --- |
| `$dash_services`, `$dash_domains`, `$dash_invoices`, `$dash_tickets` | The panel lists | Empty; the query never runs |
| `$dash_stats` | The figure strip | Built from permitted reads |
| `$dash_alerts` | Attention items, through a filter hook modules extend | Fewer entries, never missing |
| `$dash_health`, `$dash_feed`, `$dash_balance` | Health line, feed, balance | Present and empty — "not yours to see", not zero |
| `$dash_greeting`, `$dash_first_name`, `$dash_today` | The welcome line; names the signed-in member | Always present |
| `$dash_l10n` | Strings the dashboard scripts label with | Always present |
| `$show_activity` | Whether the activity feed may appear | False when switched: activity is personal |

### Client Area Theme Flags

- **dashboard_layout**: A settings field, `hero` or `standard`, selecting `views/account/dashboard-{value}`. Ignored when a single `account/dashboard` ships.
- **meta.dashboard_due_soon_alert**: A manifest flag, not a setting. False alerts only on overdue invoices; true also on the next.
- **meta.disabled_routes**: Route keys your theme does not serve, answered with 404 rather than a half finished inherited view. Templates see the list too.

## Example

```smarty
{* Computed ONCE, at the very top, before the doctype. Both halves are required:
   a signed-in visitor on a marketing page must still get the public chrome. *}
{$is_client_area = $is_logged_in && !empty($show_client_subnav)}<!DOCTYPE html>
<html lang="{$ui_lang|default:'en'}" dir="{$ui_dir|default:'ltr'}">
<head>
    {* Shell-only stylesheet: a public page never pays for it. *}
    {if $is_client_area}<link rel="stylesheet" href="{asset path='css/client-nav.css'}">{/if}
    {block name=head}{/block}
    {hook name='ui:client.head.css'}
</head>

<body class="{block name=body_class}{/block}"{if $is_client_area} data-client-nav="sidebar"{/if}>
{hook name='ui:client.body.begin'}

{if $is_client_area}
    {include file='partials/client-topbar.tpl'}
    {include file='partials/client-sidebar.tpl'}
{else}
    {include file='partials/header.tpl'}
{/if}

<main class="client-main">
{block name=content}{/block}
</main>

{* Marketing bands belong to the public site only. *}
{if !$is_client_area}{block name=bands}{/block}{/if}

{if $is_client_area}
    {include file='partials/client-footer.tpl'}
{else}
    {include file='partials/footer.tpl'}
{/if}

{block name=body_end}{/block}
{hook name='ui:client.body.end'}
```

```php
// The identity is never the owner. Resolve both, then ask about permissions.
$ctx  = UserManager::activeAccount();
$uid  = (int) $ctx["owner_id"];
$self = (bool) $ctx["is_self"];

// Own account bypasses the gate entirely; a switched account is checked per area.
$can = fn (string $p): bool => $self || in_array($p, $ctx["permissions"], true);

if (!Theme::active()->viewExists("account/services")) return $this->page_404("website");

// MUST come before set_predefined_data("client"): the notification count is read
// behind this flag, and a controller that sets it afterwards gets a zero badge.
$this->addData("show_client_subnav", true);
$this->addData("subnav_active", "services");

// An unpermitted area contributes nothing rather than an empty-looking panel.
$this->addData("service_list", $can("view_services") ? $this->model->services($uid) : []);

$this->set_predefined_data("client");

return $this->view->chose("website")->render("account/services", $this->data, true);
```

## Pitfalls

> **Set the shell flag before the client data, not after**
> 
> The notification count is computed behind `show_client_subnav`. Set the flag afterwards and the badge is always zero.

> **Hiding a link is not access control**
> 
> Omitting a tab for an unpermitted area is presentation. The boundary is the guard behind the access denied view, which runs regardless.

> **A table preset may be included twice**
> 
> Declare its helpers as variables holding closures: a named function fatals on the second include, which an AJAX page request returning a summary produces.

> **Set globals unconditionally, hide them conditionally**
> 
> A counter the shell reads is always assigned, even at zero. A variable set inside an `if` is undefined wherever the branch is skipped.

> **Localise the cancel button too**
> 
> The confirmation dialog defaults its cancel label to English, so passing only the action label leaves a stray English word in other languages.

## Related Articles

- [Page Surfaces](https://dev.wisecp.com/en/page-surfaces)
- [Login and Registration](https://dev.wisecp.com/en/login-and-registration)
- [Template Variables](https://dev.wisecp.com/en/template-variables)
- [Theme Hooks and Output Filters](https://dev.wisecp.com/en/theme-hooks-and-output-filters)
- [Interface Components](https://dev.wisecp.com/en/interface-components)

# Login and Registration

https://dev.wisecp.com/en/login-and-registration

Six views, one split layout, one step machine. The auth surface is built once, then rearranged in the browser.

## Overview

The login page is one document holding seven states. Nothing reloads, so every string, field and consent must be present in the markup the server produced.

Registration is the opposite problem. Almost every field is optional, so the form is a set of conditionals.

## Structure

All six go through one private helper on the sign controller. They share a guard, a canonical link and a title convention.

| View | Guest only | Notes |
| --- | --- | --- |
| `auth/login` | Yes | Seven states, one document |
| `auth/register` | Yes, unless a verification is pending | Countries, custom fields, contracts, gating |
| `auth/forget-password` | Yes | Always reports success, existing address or not |
| `auth/reset-password` | Yes | The token resolves first: the form, or an expired notice |
| `auth/activate` | Yes | The email confirmation landing page |
| `auth/accept-invite` | No | Guests and members; the state says which of four cases |

- **stage_title, stage_text, stage_foot**: The left column. The foot holds the cross link, wrapped in the matching switch.
- **split_class, inner_class**: Shape modifiers for a wider form; register uses both.
- **layouts/auth.tpl**: Carries head, scripts, body_class, content, body_end, plus the five above.

## Step by Step

### Build the Login State Machine

1. Print every state in one document, one visible and the rest hidden. Each carries a step attribute: `login`, `password`, `totp`, `sms`, `email`, `code-login`, `restricted`.
2. Gate the machine behind the login switch, with a notice in the other branch.
3. Put the passwordless call to action before the social buttons, each behind its own switch and divider.
4. Use one shared one time code component for all four code states. It binds to the input class, not the state, so a second one gets no paste or auto verify.

### Gate the Registration Fields

1. Read visibility from the gating map, never from a configuration key.
2. Mark required fields with a data attribute, not the native one alone, so the validator skips hidden ones.
3. Print custom fields and contracts from the arrays given; neither has a fixed set.
4. The server is the authority — every browser check is repeated on submit.

### Decide Whether Your Form Is Minimal

1. Set the manifest flag true **only** when your register view prints none of phone, landline and national id.
2. Leave it false when you print them behind the visibility switches, as two of three shipped themes do.
3. It is read from the manifest, not the request, so a submission cannot claim it.

### Wrap Every Auth Link in Its Switch

1. Wrap a sign up link in the registration switch, a sign in link in the login switch. The account menu and cart use the combined one.
2. Do not rely on hiding — the server refuses a closed registration or login itself.

## Reference

### Auth Core Signatures

```php
// The single entry point for creating a member. See the input keys below.
public static function register(array $input): array;

// Password login. $type is 'member' or 'admin'; the member master switch is checked here.
public static function attemptLogin(string $type, string $email, string $password, bool $remember = false): array;

// The ONE place a session is established, password or not. Every passwordless flow
// (code login, social, post-registration, email verification) ends here.
public static function completeLogin(string $type, int $userId, object $user, bool $remember = false, array $sso = []): string;

// Passwordless: issue a one-time code, then exchange it for a session.
// issueLoginCode reports success whether or not the address exists.
public static function jetpassEnabled(): bool;
public static function issueLoginCode(string $email): array;
public static function jetpassLogin(string $email, string $code, bool $remember = false): array;

// Password reset. issueReset is also enumeration-safe.
public static function issueReset(string $type, string $email): array;
public static function verifyResetToken(string $rawKey, string $type): object|false;
public static function resetPassword(string $type, int $userId, string $password): void;

// Providers to offer, for a given mode and audience.
public static function activeProviders(string $mode, string $context): array;
```

### What register() Accepts

```php
$input = [
    // Identity
    'first_name'     => 'Ada',
    'last_name'      => 'Lovelace',
    'email'          => 'ada@example.com',
    'password'       => 'read with the pass-through filter, never a text filter',

    // Account type and the corporate fields it unlocks
    'account_type'   => 'individual',   // or 'corporate'
    'company_name'   => '',
    'tax_number'     => '',
    'tax_office'     => '',

    // Optional contact fields, each behind its own visibility switch
    'phone'          => '',
    'landline_phone' => '',
    'national_id'    => '',

    // Billing address
    'address1'       => '',
    'state'          => '',             // id when picked from the list, free text otherwise
    'city'           => '',             // same rule as state
    'postal_code'    => '',
    'country'        => 0,              // resolved to an id before it gets here

    // Consents
    'marketing_email'    => 0,
    'marketing_sms'      => 0,
    'contract'           => 1,          // the terms checkbox
    'contracts_required' => true,       // whether any contract is actually configured

    // Operator-defined extra fields, collected by the operation
    'custom_fields'      => [],

    // Declared by the THEME MANIFEST, never by the request. True skips the required
    // checks for phone, landline and national id, and nothing else.
    'minimal_fields'     => false,
];
```

### The Field Gating Map

| Template variable | Default | Controls |
| --- | --- | --- |
| `$registration.account_type_visible` | on | The individual/corporate chooser |
| `$registration.company_name_required` | on | Company name, corporate branch |
| `$registration.tax_number_required`, `tax_office_required` | off | The corporate tax fields |
| `$registration.phone_visible`, `phone_required` | on, on | Mobile number |
| `$registration.landline_visible`, `landline_required` | off, off | Landline number |
| `$registration.national_id_visible`, `national_id_required` | off, off | National identity, individual branch |
| `$registration.password_min_length` | 12 | Enforced on both sides |

### What Each View Receives

| View | Variable | Holds |
| --- | --- | --- |
| `auth/register` | `$countries` | `id`, `a2_iso`, `name`. States and cities cascade over AJAX |
| `auth/register` | `$custom_fields` | Operator-defined extra fields, localised. No fixed set |
| `auth/register` | `$contracts` | Contract pages for sign up; each carries a `link` |
| `auth/register` | `$contract_names` | Those titles joined with commas, for a naming label |
| `auth/register` | `$registration` | The gating map above |
| `auth/register` | `$verify_email`, `$verify_name` | Filled when `$verify_pending` is true, so the step names the member |
| `auth/reset-password` | `$reset_valid` | Whether the token resolved; false means the expired notice |
| `auth/reset-password` | `$verify_key` | The raw token, posted back with the new password |
| `auth/login` | `$social_providers`, `$jetpass_enabled` | Set by the login page; register sets its own |

### Page and Feature Switches

- **$registration_enabled**: Whether to show a standalone sign up link: the master switch minus purchase first mode. It can be off while the cart still registers.
- **$login_enabled**: The member login master switch; administrator login is separate.
- **$account_actions_enabled**: Whether an account is possible at all: the registration *master*, not the link switch.
- **$jetpass_enabled**: The passwordless code flow, off by default. Gate its call to action.
- **$social_providers**: The providers for this page; empty means no divider and no buttons.
- **$verify_pending**: Set when a signed-in member has an unverified address; the register view shows the verification step.

## Example

```smarty
{extends file='layouts/auth.tpl'}

{block name=stage_title}{lang key='auth_stage_login_title'}{/block}
{block name=stage_text}{lang key='auth_stage_login_text'}{/block}

{* The cross link is gated: offering sign-up on a closed installation is a dead end. *}
{block name=stage_foot}
  {if $registration_enabled}
    {lang key='auth_stage_login_foot'} <a href="{link route='sign-up'}">{lang key='auth_stage_login_foot_link'}</a>
  {/if}
{/block}

{block name=content}
{if $login_enabled}

  {* State 1: email. Visible; every other state ships hidden in the SAME document. *}
  <div data-auth-step="login">
    <form data-auth-form="login">
      {csrf form='sign-in'}
      <input type="email" name="email" class="form-control" required>
      {captcha area='sign-in' tray='login-captcha'}
      <button type="submit" class="btn">{lang key='website/sign/continue'}</button>
    </form>

    {* Passwordless BEFORE social, each behind its own switch and divider. *}
    {if $jetpass_enabled}
      <div class="auth-divider">{lang key='website/sign/or'}</div>
      <button type="button" class="btn" data-auth-action="auth-code-login">{lang key='website/sign/jetpass-cta'}</button>
    {/if}
    {if $social_providers}
      <div class="auth-divider">{lang key='website/sign/or'}</div>
      {foreach $social_providers as $p}
        <a class="btn" href="{$p.url}"><i class="bi {$p.icon}"></i>{$p.label}</a>
      {/foreach}
    {/if}
  </div>

  <div class="d-none" data-auth-step="password">{* ... *}</div>

  {* All four code states reuse ONE component: the script binds to .otp-input,
     not to the state, so a second implementation gets none of its behaviour. *}
  <div class="d-none" data-auth-step="code-login">
    <div class="otp-group" data-otp>
      <input class="otp-input" inputmode="numeric" maxlength="1">
    </div>
  </div>

  <div class="d-none" data-auth-step="restricted">{* ... *}</div>

{else}
  <p>{lang key='website/sign/login-disabled'}</p>
{/if}
{/block}
```

```php
// templates/website/{Theme}/theme.php

return [
    'meta' => [
        'name'    => 'Acme',
        'version' => '1.0.0',
        'author'  => 'Acme',

        // TRUE only when the register view renders NONE of phone, landline and
        // national id. Auth::register then skips the operator's required checks
        // for exactly those three fields; every other validation still runs.
        //
        // FALSE when the view renders them behind $registration.*_visible, which
        // is what two of the three bundled themes do.
        //
        // Setting it true while still printing the fields is the one wrong answer:
        // the requirement is skipped even though the visitor saw an empty input.
        'signup_minimal' => false,
    ],
    'engine' => 'smarty',
    'status' => 'ready',
];
```

## Pitfalls

> **Never reveal whether an account exists**
> 
> Password reset and the passwordless code report success for an unknown address on purpose. Your script must advance in every case.

> **A password is read with the pass-through filter**
> 
> Every text filter strips the characters that make a password strong. The stored value then differs from what was typed, with no error.

> **The minimal flag is a claim about your markup**
> 
> It is read from your manifest and trusted, so it must describe what your form prints. Declare it while printing the optional fields and an empty submission passes unreported.

> **Every auth form needs its token and captcha area**
> 
> One action name is the token form key, the captcha area and the rate limit bucket at once. A new name opts out of all three. The passwordless request and its verification share the login action.

> **A closed login still lets a new registration in**
> 
> Post registration sign in calls the session establishing method directly, not the password path where the switch is checked.

## Related Articles

- [Page Surfaces](https://dev.wisecp.com/en/page-surfaces)
- [The Client Area](https://dev.wisecp.com/en/the-client-area)
- [Securing Theme Forms](https://dev.wisecp.com/en/securing-theme-forms)
- [Writing a Social Login Provider](https://dev.wisecp.com/en/writing-a-social-login-provider)
- [Writing a Captcha Module](https://dev.wisecp.com/en/writing-a-captcha-module)

# System Pages

https://dev.wisecp.com/en/system-pages

Some pages have to appear while the application is broken, blocked or switched off. Exactly two of them can still come from your theme.

## Overview

A missing page and a closed site are ordinary, and your design belongs on screen. A fatal error or a blocked address is not: the page builder may be what failed, so those carry their own shell.

Your theme owns `404` and `maintenance`. The second is unlike every other view you write: a complete document, not a page body.

## Structure

| Surface | Comes from | When it appears |
| --- | --- | --- |
| Not found | Your theme, `views/404` | Any unmatched address, and every failed view guard |
| Maintenance | Your theme, `views/maintenance` | Every address in maintenance mode, unless an administrator is recognised |
| Maintenance fallback | Core, `templates/system/maintenance.php` | Only when the active theme ships no maintenance view |
| Application error | Core, `templates/system/application-error.php` | A fatal error on a non AJAX request, answered with HTTP 500 |
| Blocked address | Core, `templates/system/blocked-ip.php` | The address blocker rejected the request before routing |

The three core pages share one shell, so a style fix lands in all three. Each uses its helpers and nothing else.

- **templates/system/inc/shell.php**: Required by each page; returns the helper map below.
- **lang, dir**: For the root element. Never hardcode a language code.
- **e, l**: The escaper and the translator, which falls back to the string you pass.
- **head, mark, brand**: The head links, an inline icon by shape, and the product signature.
- **mask**: Replaces the administrator directory. Every printed string passes through it.

## Step by Step

### Build the Not Found Page

1. Extend the default layout: this page comes through the normal request path.
2. Offer the two escape routes behind their switches: knowledge base when enabled, support or contact form by ticket system.
3. Never print the address that was not found. It is attacker supplied text.
4. Every failed view guard lands here too — this is what a half installed theme shows.

### Build the Maintenance Document

The one view that does not extend a layout. Turn the maintenance switch on, or you cannot see it.

1. Open with your own doctype, root element and head, taking language and direction from the template variables.
2. Copy your first paint guard in verbatim, final fallback value included.
3. Print your skin tokens from the theme settings.
4. Include only the logo, the message and the switcher — no navigation, cart or account menu.
5. Keep the four injection points, so a module can still reach a closed site.

### Touch the Core Pages Correctly

1. Require the shared shell and use its helpers; never copy its style block.
2. Take text from the locale files: the English strings in the PHP are only the translator's fallback.

## Reference

### The Shell Helpers

```php
$sys = require __DIR__ . DIRECTORY_SEPARATOR . 'inc' . DIRECTORY_SEPARATOR . 'shell.php';

// Values
$sys['lang'];                       // 'en', 'tr', ... from the language package
$sys['dir'];                        // 'ltr' or 'rtl'
$sys['app'];                        // the installation's base address

// Callables
$sys['e']($text);                   // htmlspecialchars, quotes included, UTF-8
$sys['l']($file, $key, $fallback);  // system/{file}/{key}, or $fallback when unavailable
$sys['head']();                     // local font link + the shared stylesheet
$sys['mark']($shape);               // inline icon: 'alert', 'shield' or 'tools'
$sys['brand']();                    // product signature + the version file
$sys['mask']($text);                // administrator directory replaced by a placeholder

// Typical use, with the per-page locale file bound once:
$e = $sys['e'];
$L = static fn (string $k, string $fallback): string => $sys['l']('error', $k, $fallback);
```

### What the Error Page May Show

| Situation | Message | Technical block |
| --- | --- | --- |
| An engine error: a type or parse failure | Generic: the original text was written for a developer | Development only |
| An exception raised on purpose | Shown: the operator needs to read it | Development only |
| A fatal caught at shutdown, so no class is known | Generic, as an engine error | Development only |
| A source excerpt around the failing line | Not applicable | Development only; readable, unreadable or encoded, and too large handled |
| The request data | Not applicable | Development only, passwords and tokens redacted first |

### Theme Side Contract

```php
// True when the manifest loaded AND the theme directory is really there.
public function exists(): bool;

// True when views/<view>.<ext> is a real file. The extension follows the manifest engine.
public function viewExists(string $view): bool;

// Rendering a system surface outside a request (a CLI health check, a probe):
// chose("website") would skip the theme, so call the theme directly.
public function render(string $view, array $data = []): string;
```

- **the maintenance gate**: The controller asks both, then falls back to the core template. Skip the second and the call returns an empty string: a blank page.
- **ui:client.maintenance.body**: Injected before the closing body tag once the page is built, so it reaches theme view and core fallback alike. Return the HTML to add; strings are joined.
- **views/404**: Its own four: `$page_title`, `$meta_robots`, and the switches `$support_enabled` and `$kbase_enabled` that pick the escape routes. The full client data package *is* prepared here.

### What the Maintenance View Receives

The whole list. The client data package is not prepared: menus, cart, announcements and account variables are absent; reading one shows nothing and logs a warning.

- **$ui_lang, $ui_dir**: The language code and `ltr` or `rtl` for your root element. Added by the template engine, so they survive a missing data package.
- **$light_logo_link, $dark_logo_link, $company_name**: The two logo variants and the operator's name. Print both and let your stylesheet pick: there is no server side detection.
- **$lang_list, $lang_count, $selected_lang_key**: Each entry carries `key`, `name`, `link`, `selected` and `flag-img`. Print the control only when the count is above one.
- **$current_year, $setting**: The year for a copyright line, and your theme's settings. `$setting` reaches every themed view, so a closed site keeps operator colours.

## Example

```smarty
<!DOCTYPE html>
{* NOT layouts/default.tpl: the site is closed, so no nav, cart or account chrome
   may link back into it. The language switcher is the only control, and it
   re-renders THIS page in the chosen language. *}
<html lang="{$ui_lang|default:'en'}" dir="{$ui_dir|default:'ltr'}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="robots" content="noindex, nofollow">
    <title>{lang key='system/maintenance/meta'}</title>

    {* Copy the guard VERBATIM from the main layout. Its final fallback value must
       match the theme script's, or the page paints in one mode and flips to the
       other one frame later. The bug only shows on a visitor with no stored value,
       which is why a developer's own browser never reproduces it. *}
    <script>
      var t = localStorage.getItem('wstyle-theme') || 'light';
      document.documentElement.setAttribute('data-bs-theme', t);
    </script>

    <link rel="stylesheet" href="{asset path='css/theme.css'}">
    <link rel="stylesheet" href="{asset path='css/maintenance.css'}">

    {* Operator colours still apply on a closed site. *}
    <style>:root { --brand: {$setting.primary_color}; }</style>

    {hook name='ui:client.head.css'}
    {hook name='ui:client.head.js'}
</head>
<body class="maintenance-page">
{hook name='ui:client.body.begin'}

<header>
    <a href="{link route='home'}"><span>{$company_name}</span></a>

    {* The ONLY control on the page. *}
    {if $lang_count > 1}
      {foreach $lang_list as $l}
        <a href="{$l.link}">{$l.name}</a>
      {/foreach}
    {/if}
</header>

<main id="maintenance-content">
    <p>{lang key='system/maintenance/text'}</p>
</main>

{hook name='ui:client.body.end'}
</body>
</html>
```

```php
public function main(): void
{
    // Minimal on purpose: a closed page needs no menus, cart or announcements.
    $this->takeDatas(["language", "website_logos", "company_name", "lang_list"]);
    $this->addData("current_year", date("Y"));

    // BOTH checks are required. Without viewExists a theme that ships no
    // maintenance view falls through to the plain-PHP path, finds no file
    // and returns an empty string: a blank page with no error anywhere.
    $theme = \Theme::active();
    if ($theme->exists() && $theme->viewExists("maintenance"))
        $html = $this->view->chose("website")->render("maintenance", $this->data, true);
    else
        $html = $this->view->chose("system")->render("maintenance", $this->data, true);

    // Injected after rendering, so it reaches the theme view and the fallback alike.
    $injection = implode('', \Hook::run('ui:client.maintenance.body', $this->data));
    if ($injection) $html = preg_replace('/<\/body>/i', $injection . '</body>', $html, 1);

    echo $html;
}
```

## Pitfalls

> **The first paint guard must match the theme script**
> 
> Both read the same stored value and need the same final fallback. A mismatch paints one mode and flips a frame later, seen only by a visitor with no stored preference.

> **The administrator directory must not reach the markup**
> 
> It reaches the request address, the dumped request data and a retry button's target, so a screenshot leaks it. Mask everything; the retry link stays empty.

> **No external request on a core system page**
> 
> They appear when the server is unwell, so a content network buys nothing. Icons are inline, the typeface is local with a system fallback, and there is no framework.

> **Hiding the message and hiding the detail are different**
> 
> An engine failure carries developer text, so a generic sentence replaces it. An exception raised on purpose is shown: the operator has to read it.

> **Nothing on a system page folds away**
> 
> Every section arrives open: the reader solves the problem here. Centre with automatic margins; centring alignment crops the top edge once content overflows.

## Related Articles

- [Page Surfaces](https://dev.wisecp.com/en/page-surfaces)
- [Theme Assets](https://dev.wisecp.com/en/theme-assets)
- [Error Handling](https://dev.wisecp.com/en/error-handling)
- [Theme Hooks and Output Filters](https://dev.wisecp.com/en/theme-hooks-and-output-filters)
- [Debugging and Logs](https://dev.wisecp.com/en/debugging-and-logs)

# Template Variables

https://dev.wisecp.com/en/template-variables

Everything a view can print arrives as a named variable. Each is written by one of three layers: the engine, the client page pack, or the theme's hooks file.

## Overview

A theme does not query. The platform prepares the data and the view prints what it was given. Knowing which layer wrote a variable tells you where to look when it is missing, and whether a theme may change it.

The layers always run in the same order. The engine writes the display environment on every page. Then `set_predefined_data("client")` writes what every client page needs, and the theme's `hooks.php` adds the rest, the only layer the theme owns.

## Structure

### The Three Layers

| Layer | Written by | What belongs there |
| --- | --- | --- |
| Display environment | `View::render()`, on every page including admin | Language, writing direction, asset addresses, theme settings |
| Client content | `set_predefined_data("client")`, from every client controller | Branding, menus, currency, cart, session state, legal links |
| Theme data | `hooks.php`, through `filter:template.variables` | Anything only this theme needs, derived from core values |

Only the theme layer answers with a value, and that value replaces the whole set instead of merging into it. A listener returns the array it was handed, with its own keys added; a return that is not an array leaves the set untouched.

Page data sits on top of all three. The controller adds it with `addData()`, and only the page that asked for it gets it.

## Reference

### What the Engine Writes on Every Page

Assigned in the themed branch of the view layer, after the hook has run. A listener cannot remove them. The list is narrower on the plain PHP engine. There, `$cookie_domain`, `$demo_mode` and `$demo_themes` are written only on the tag-engine path. `$template_dir` resolves to a path built from the theme's name. A theme on that engine asks the theme object for its directory.

- **$template_dir**: Filesystem path of the active theme directory, with a trailing separator. Not an address: use it only to read files.
- **$tadress**: Address of the theme directory. Prefer `{asset path='...'}`, which points inside `assets/` and adds a cache-busting stamp to CSS and JS.
- **$badress**: Base address of the installation, with a trailing slash. On a bound subdomain this is that host, not the main site.
- **$sadress**: Address of the shared resources directory (uploads, flags, plugin assets), owned by no theme.
- **$ui_lang**: Active language code, resolved when the page is built, so a late language switch is reflected.
- **$ui_dir**: `ltr` or `rtl`, from the language pack. Print it on the html element; do not hardcode a direction.
- **$cookie_domain**: The host scope this installation's cookies use, empty on a single-host install. Theme JavaScript writing a cookie uses the same scope.
- **$demo_mode · $demo_themes**: Demo chrome only. The theme list is computed only while demo mode is on.
- **$setting**: Every field of this theme's settings schema merged with its saved value, from `Theme::allSettings()`. Colour fields also expose a `_rgb` twin.

### What Every Client Page Adds

- **$company_name**: Trading name, from the language pack's constants, falling back to the company information block.
- **$light_logo_link · $dark_logo_link**: Site logo per colour mode. The client panel and the invoice document carry their own pair (`$client_logo_light_link`, `$invoice_logo_light_link` and their dark twins). Each falls back to the site logo.
- **$favicon_link**: Square brand mark, for places a wordmark does not fit (a collapsed rail, an app icon).
- **$current_year**: Current year as a string, for the footer copyright line. Do not compute a date in a view.
- **$lang_list · $lang_count · $selected_lang_key**: Active languages with a ready link each, their number, and the selected code in upper case. The layout also prints them as alternate language links.
- **$currencies · $currencies_count · $selected_currency · $selected_currency_code**: Currencies offered to the visitor (hidden ones filtered out) and the active one. Switching is a server round trip: link to the same page with a `currency` parameter.
- **$currency_formats_json · $currency_ids_json · $currency_rates_json · $currency_default**: Ready JSON for the theme's money script. Each format is a sample string the script parses to learn that currency's separators.
- **$header_menu · $footer_menu · $mobile_menu**: The three operator-managed menu trees. The mobile tree falls back to the header tree when it is empty.
- **$social_links · $contact_email · $contact_phone**: Social profiles as a list, plus the first configured address and number as plain strings, for the header and footer strips.
- **$visibility_cart · $cart_count**: Whether the shop is on at all, and the live item count. The count is always set, so a badge can be revealed without a reload.
- **$login_enabled · $registration_enabled · $account_actions_enabled**: The operator's account switches. The third is false when neither signing in nor signing up is possible. That hides the account menu and the cart.
- **$affiliate_enabled · $reseller_enabled · $only_client_panel**: Program switches for the account menu, and portal-only mode, in which a theme drops its links back to the public website.
- **$is_logged_in · $client_id**: The canonical session flag and the signed-in identity. Every "signed in or guest" branch tests this flag.
- **$privacy_contract_link · $terms_contract_link · $cookie_contract_link**: Legal pages the operator mapped, empty when unmapped. Hide the link when it is empty instead of printing a dead one.
- **$cookie_notice**: The consent prompt as a finished payload, or an empty array when consent is off. Decided server side: the cookies are not readable from the browser.
- **$notification_count · $notification_count_text**: Bell counter and its pill text (capped beyond two digits). Always set, but only queried for a page that asked for the client shell.
- **$password_min_length · $password_special_chars · $default_country**: The operator's password rules for the suggestion button, and the site's default country as a last-resort fallback for phone and address fields.
- **$links · $meta · $breadcrumb · $local_l · $show_powered_by**: Page frame: the controller's link map, page metadata, the trail, the installation's default language, and whether the footer prints the platform credit.

### What a Signed-In Page Adds

These exist only while a member is signed in. A public page must not read them without guarding on `$is_logged_in`.

- **$account_info · $user_info**: Display identity for the chrome (name, avatar, initials, formatted balance, support PIN, previous sign-in) and the raw account row behind it.
- **$announcements**: Operator notices for the account shell, already filtered by language and by the active account's country.
- **$show_services · $show_domains · $show_invoices · $show_support · $show_sms**: Section visibility: the granted sub-user permission AND the operator's feature switch. A section the visitor cannot open is never linked.
- **$is_self_account · $switch_accounts**: Whether the active account is the member's own, and the accounts they may switch to.
- **$client_badges · $client_badges_text**: Attention counters per section as raw numbers, plus their pill text. The raw numbers stay for accessible names, which say the real figure.
- **$order_service_link**: Where "buy a service" goes, empty when the sub-user may not order. A theme that disables the catalogue overrides this in its hooks file.
- **$dashboard_modal · $twofa_required · $password_required**: The dashboard gate chain. Only one modal opens per load, and the chosen one is named here rather than decided in the view.
- **$client_addon_links**: Client-area pages contributed by enabled addon modules, each with a name, an address and an icon.
- **$verification_required · $verify_link**: Set while the member owes identity verification. The banner belongs to the layout; the redirect for gated pages already happened before the view ran.

### The Shape of the Composite Values

```php
// $lang_list : one row per ACTIVE language, ranked, the installation default first.
$lang_list = [
    ['rank' => 1, 'local' => 1, 'selected' => true, 'key' => 'en', 'name' => 'English',
     'global-name' => 'English', 'link' => 'https://example.com/home',
     'cc' => 'gb', 'cname' => 'United Kingdom', 'pc' => '44', 'flag-img' => 'https://example.com/resources/assets/images/flags/gb.svg'],
];

// $currencies : active, non-hidden currency rows.
$currencies = [
    ['id' => 1, 'code' => 'USD', 'name' => 'US Dollar', 'prefix' => '$', 'suffix' => '',
     'rate' => '1.00000000', 'local' => 1, 'hidden' => 0],
];

// $social_links : one entry per configured profile.
$social_links = [
    ['name' => 'X', 'url' => 'https://x.com/example', 'icon' => 'bi bi-twitter-x'],
];

// $account_info : display identity, already formatted. `balance` is a STRING with its symbol.
$account_info = [
    'name' => 'Ada', 'surname' => 'Lovelace', 'full_name' => 'Ada Lovelace',
    'email' => 'ada@example.com', 'avatar' => '', 'initials' => 'AL',
    'balance' => '$120.00', 'support_pin' => '481625', 'two_factor' => true,
    'is_reseller' => false, 'last_login' => [], 'last_login_date' => '02/08/2026 - 11:40',
    'last_login_ip' => '203.0.113.9', 'last_login_country' => 'GB', 'last_login_city' => 'London',
    'dealership' => [],
];

// $client_badges : raw counts; $client_badges_text holds the same keys as pill strings.
$client_badges = ['invoices' => 2, 'support' => 1, 'domains' => 0, 'services' => 0];

// $cookie_notice : empty array when consent is off.
$cookie_notice = [
    'needs_prompt' => true, 'text' => 'We use cookies…', 'policy_link' => 'https://example.com/cookie-policy',
    'policy_label' => 'Cookie policy', 'accept' => 'Accept', 'reject' => 'Reject',
    'prefs' => 'Preferences', 'save' => 'Save', 'prefs_title' => 'Cookie preferences',
    'categories' => [
        ['key' => 'necessary', 'label' => 'Necessary', 'desc' => '…', 'locked' => true, 'granted' => true],
    ],
    'url' => 'https://example.com/cookie-consent',
];
```

### Signatures

```php
// Controllers : the page and the pack.
public function set_predefined_data(string $type = 'client', array $meta = [], array $breadcrumbs = [], array $links = []): void;
public function addData($k = '', $v = ''): void;
public function getData($key);

// View : the render itself. $return_output = true returns the HTML instead of printing it.
public function chose($dir, $noTemplate = false): self;
public function render($_name = null, $data = [], $return_output = false, $source = false): mixed;

// Theme : the settings map behind $setting, and the context-free render used off-request.
public function allSettings(): array;
public function render(string $view, array $data = []): string;
```

## Example

A client page and the view that reads it back. The controller adds only what belongs to this page.

```php
public function page_overview(&$links, &$meta, &$breadcrumbs): string
{
    // Read INSIDE set_predefined_data, so it has to be set before the call.
    $this->addData("show_client_subnav", true);

    $this->set_predefined_data("client", $meta, $breadcrumbs, $links);

    // Page data on top of the pack. Set unconditionally: a key the view reads has to
    // exist on every render, empty or not, or the view falls into an undefined key.
    $this->addData("recent_orders", $this->model->recent_orders((int) $this->getData("client_id")));

    echo $this->view->chose("website")->render("account/overview", $this->data, true);

    return '';
}
```

```smarty
{* Environment layer: direction and language come from the engine, never hardcoded. *}
<section dir="{$ui_dir}" lang="{$ui_lang}">

    {* Client layer: guard on the session flag before touching a signed-in-only value. *}
    {if $is_logged_in}
        <p>{$account_info.full_name} · {$account_info.balance}</p>
        {if $client_badges.invoices > 0}
        <span class="badge">{$client_badges_text.invoices}</span>
        {/if}
    {/if}

    {* Theme layer: a setting this theme declared, read straight off $setting. *}
    {if $setting.topbar_enabled}<div class="topbar">{$setting.topbar_text nofilter}</div>{/if}

    {* Page layer: default:[] so an empty page never costs a warning per row. *}
    {foreach $recent_orders|default:[] as $order}
        <a href="{link route='invoice-detail' p1=$order.id}">{$order.number}</a>
    {/foreach}
</section>
```

## Pitfalls

> **A variable set inside a condition is missing on the pages that skipped it**
> 
> Set every global unconditionally and decide visibility in the view. Absence is not "false": the view falls into an undefined key, and every one costs a log write. Inside a loop that is a measurable slowdown.

> **The variables hook replaces the array, it does not merge into it**
> 
> A listener that returns something other than the array it was handed drops every key the platform collected. The layout then breaks on the first missing address. Start from the incoming array, add to it, return all of it.

> **A background page gets the environment, not the client pack**
> 
> A view built from a scheduled task or a document generator goes through the theme object directly. No controller ran to fill the pack. Only the environment layer and the theme settings are there; anything else is passed as page data.

> **Addresses are built in the view, not passed as variables**
> 
> The controller does not hand out ready links for pages the theme can address itself. Build them with the link function and a route key. A theme that renames a page then needs no controller change.

## Related Articles

- [The Theme Engine](https://dev.wisecp.com/en/the-theme-engine)
- [Theme Hooks and Output Filters](https://dev.wisecp.com/en/theme-hooks-and-output-filters)
- [Theme Settings](https://dev.wisecp.com/en/theme-settings)
- [Menus and Navigation](https://dev.wisecp.com/en/menus-and-navigation)
- [Controllers and Routing](https://dev.wisecp.com/en/controllers-and-routing)

# Theme Hooks and Output Filters

https://dev.wisecp.com/en/theme-hooks-and-output-filters

A theme changes the data it receives and the markup it emits from one file of its own. No controller or core template is edited.

## Overview

Every theme may ship a `hooks.php`, included once before its first page is built. It travels and is deleted with the theme.

**Data** listeners change what reaches the view: a variable this theme needs on every page, a menu tree, the finished HTML. **Markup** points work the other way: the theme opens them, a module injects into them.

## Structure

### Where It Lives

- **templates/website/{Theme}/hooks.php**: Optional. Plain PHP, no class, no return value. Registers listeners and the theme's helper functions.
- **Theme::boot()**: Included once per request, before the first page is built. Called by the view layer and the context-free off-request path.
- **Scope**: Website requests only. The admin panel and scheduled tasks never resolve a theme.
- **Cost**: Every listener runs on every page view.

### The Two Families

| Family | The theme is | Mechanism | Return |
| --- | --- | --- | --- |
| Data (filter) | the listener | `Hook::add()` in the hooks file | Replaced by the return, or changed by reference |
| Markup (injection) | the publisher | `{hook name='...'}` in the layout and partials | The listeners' strings are concatenated, printed raw |

## Reference

### The Hook API

```php
// $priority: lower runs first; a clash is resolved by bumping, never by dropping a listener.
// $properties: a closure, or ['class' => 'X', 'method' => 'y'], or ['class' => 'X', 'method::static' => 'y'].
public static function add($name, $priority, $properties = []): void;

// By value. Returns one entry per listener; a markup point concatenates them.
public static function run($name, ...$args): array;

// By reference. EVERY argument is passed by reference, so every one must be a plain variable.
public static function runRefs($name, &...$args): array;
```

### The Variables Filter

The single point through which a theme adds a variable to every page. It runs for every template, so a listener wanting only some tests the path.

- **filter:template.variables**: Fires in the view layer once the data set is complete, before the template is included.
- **($template, $data)**: The full path of the template processed, and the variable array. Neither is by reference.
- **Return**: An array REPLACES the data set entirely. Anything else is ignored and the set is left as it was.
- **One listener per theme**: Only the LAST returned array survives; everything a theme adds belongs in one body.

### The Output Filter

The finished HTML, after the engine built it and before it reaches the browser. A rewrite that would otherwise touch every payload happens here.

- **filter:client.page.output**: Fires in the themed branch of the view layer, on both the printed and returned paths.
- **(&$output, $view, $engine)**: The whole HTML by reference, the view name that produced it, and the active engine. Changed in place; the return is unused.
- **Scope**: Tag engines only. A theme declaring the plain PHP engine prints through include and is never captured.
- **Fragments too**: Section fragments answered over AJAX come through the same branch; tell them apart by `$view`.

### Markup Points

Written as `{hook name='ui:client.head.css'}` in the layout. The shipped themes open more than a hundred and fifty; those every theme should carry are below.

| Point | Where the layout puts it | What belongs there | Return |
| --- | --- | --- | --- |
| `ui:client.head.css` | Head, end | Stylesheets, style blocks | Return `<link>` or `<style>` only, no visible markup |
| `ui:client.head.js` | Head, end | A library the theme's script uses | Return `<script>` only, inline or with a `src` |
| `ui:client.body.begin` | Body, start | Tag manager frames, top banners | Return any HTML string |
| `ui:client.body.end` | Body, end | Deferred scripts, chat widgets, modals | Return any HTML string |
| `ui:client.nav.items` | Header nav list, end | An extra top-level item | Return `<li>`, never a bare `<a>` |
| `ui:client.header.actions` | Header action cluster | An icon button beside the cart | Return an `<a>` or `<button>` carrying `.btn.btn-soft.header-icon-btn` |
| `ui:client.user_menu.items` | Account dropdown | A row in the signed-in menu | Return an `<a>` carrying `.dropdown-item` |
| `ui:client.drawer.items` | Mobile drawer | Mobile twin of a nav item | Return a bare `<a>` carrying `.drawer-link`, no `<li>` |
| `ui:client.content.top` | Above page content | A site-wide notice strip | Return any HTML string |
| `ui:client.footer.columns` | After footer columns | An extra link column | Return one grid column `<div>`, same shape as its siblings, no `<ul>`/`<li>` |
| `ui:client.footer.bottom` | Footer bottom strip | A badge, a legal line | Return any HTML string |

### Other Hooks

| Hook | When | What a theme does with it |
| --- | --- | --- |
| `filter:client.theme` | While the theme name resolves | Serve a different theme per host, segment or preview |
| `filter:client.menu` | After a menu tree is built | Add, drop or reorder a node the panel does not manage |
| `filter:client.breadcrumb` | Before the trail is shortened | Rename or reroot the first crumb |
| `filter:client.predefined_data` | End of the client data pack | Adjust a pack value while knowing the route |
| `filter:client.routes` | Before routes are matched | A short address for a page this theme publishes |
| `filter:routing.match` | Last routing fallback | Answer a slug no real route claims |
| `gate:client.page_access` | Before the controller is built | Send a page to its canonical host, or refuse it |
| `filter:sitemap.links` | While the sitemap is collected | Publish the address a page declares canonical |

## Example

A complete hooks file: one variables listener, one output filter, one markup injection.

```php
/*
 * ONE variables listener for the whole theme: the view layer keeps only the LAST
 * returned array, so a second registration would drop everything this one writes.
 */
Hook::add("filter:template.variables", 1, function ($template, $data) {

    // Derived from core config, which a template cannot read for itself.
    $data["support_enabled"] = (int) (Config::get("options/ticket-system") ?? 0) === 1;

    // Routes this theme answers 404 for, as a keyed map: the sandbox has no in_array(),
    // so the template asks {if !$route_off.affiliate} instead.
    $data["route_off"] = array_fill_keys(Theme::active()->meta()["disabled_routes"] ?? [], true);

    // Runs on EVERY page, so anything with a query goes through the cache. The key
    // carries the currency AND the language, because the output depends on both;
    // leave either out and one visitor is served the other one's copy.
    $data["footer_groups"] = Cache::remember('website',
        'acme_footer_' . Money::getUCID() . '_' . Language::selected(), 3600,
        fn (): array => Products::groups());

    // Page-specific work stays behind a path test rather than a second registration.
    if (str_ends_with(str_replace('\\', '/', (string) $template), '/page/pricing.php'))
        $data["plans"] = AcmePricing::cards();

    return $data;   // returning anything else drops every key the platform collected
});

/*
 * The finished page. Cheaper and safer than rewriting every payload that produced a
 * link: one place, and it cannot miss one.
 */
Hook::add('filter:client.page.output', 1, function (&$output, $view, $engine) {
    if (str_contains($view, '/')) return;   // fragments are not documents

    $output = str_replace('</body>', '<script src="/assets/acme-widget.js" defer></script></body>', $output);
});

/*
 * Markup, not data: the layout's points take a string and print it as-is. Registered
 * here so the theme's own extra stylesheet needs no edit to any layout file.
 */
Hook::add("ui:client.head.css", 1, fn (): string =>
    '<link rel="stylesheet" href="' . Theme::active()->assetUrl('css/extra.css') . '">');
```

```smarty
<head>
    {block name=head}{/block}
    {hook name='ui:client.head.css'}
    {hook name='ui:client.head.js'}
</head>
<body>
    {hook name='ui:client.body.begin'}

    {* A value the hooks file put there, read like any other variable. *}
    {if !$route_off.affiliate}<a href="{link route='affiliate'}">{lang key='nav_affiliate'}</a>{/if}

    {block name=content}{/block}
    {hook name='ui:client.body.end'}
</body>
```

## Pitfalls

> **A second variables listener deletes the first's work**
> 
> The failure is invisible in the file that caused it: the page that breaks reads the other listener's variable. Keep one body, add keys to it.

> **Every argument of a by-reference run is by reference**
> 
> A literal, inline array, cast, null-coalesced read or function result cannot be passed. PHP raises a fatal, not a notice. Assign the context to a variable first.

> **A listener is a per-page cost**
> 
> A bare query in a listener runs on every page view. Put database reads behind the cache, keyed by language and currency when the result depends on them.

> **A fatal inside a listener is swallowed**
> 
> The dispatcher catches it and carries on. A listener that dies leaves no error: the value it should have changed arrives unchanged. When one does nothing, suspect a fatal, not the registration.

> **Injected markup reaches the page raw**
> 
> The markup points do not escape, so the injecting side is responsible. Never concatenate visitor input into what a point returns.

## Related Articles

- [Template Variables](https://dev.wisecp.com/en/template-variables)
- [How Hooks Work](https://dev.wisecp.com/en/how-hooks-work)
- [Writing a Hook Listener](https://dev.wisecp.com/en/writing-a-hook-listener)
- [Menus and Navigation](https://dev.wisecp.com/en/menus-and-navigation)
- [Theme Performance and Caching](https://dev.wisecp.com/en/theme-performance-and-caching)

# Menus and Navigation

https://dev.wisecp.com/en/menus-and-navigation

Navigation is data the operator edits in the panel. The theme receives finished trees and prints them. Every other address comes from a route key, never from a written path.

## Overview

A theme never decides what is in the menu. The operator builds the trees in the panel. The platform resolves each node's address and hands the view ready arrays. The theme owns the markup: how a dropdown looks, where the mobile drawer lives, which node deserves an icon.

The same rule covers links a theme writes itself. A "Log in" or "Cart" link is addressed by its **route key**, not by a path. Route paths are translated per language and can be renamed. A written path breaks on the second language, and again on the first rename.

## Structure

### The Trees a Client Page Receives

- **$header_menu**: The public header. Nodes may carry children (a dropdown) or an operator-authored markup panel (a mega menu).
- **$footer_menu**: The footer. Its top level is the columns, and each node's children are that column's links. Read it one level deeper than the header.
- **$mobile_menu**: The drawer. Falls back to the header tree when the operator has not built one, so a theme can always print it.
- **Other types**: The panel also manages a client-area tree and a content-page sidebar tree. Neither is in the client pack; a view that needs one asks for it in the theme's hooks file.

### The Shape of a Node

```php
$node = [
    'id'     => 12,
    'parent' => 0,
    'icon'   => 'bi bi-hdd-rack',      // a class string, not an address
    'target' => 1,                      // 1 opens in a new tab
    'page'   => 'category/551',         // the panel's page identifier, already resolved below
    'title'  => 'Hosting',              // in the ACTIVE language
    'link'   => 'https://example.com/hosting',   // may be EMPTY: a heading-only node
    'extra'  => [
        'desc'  => 'Shared and reseller plans',   // second line inside a dropdown row
        // Badge. The two colours come from the panel's colour pickers, so they arrive as
        // hex strings with a leading hash and go straight into a style attribute.
        'tag'   => ['name' => 'New', 'color' => '#RRGGBB', 'text_color' => '#RRGGBB', 'icon' => 'bi bi-stars'],
        'mega'  => '<div class="mega">…</div>',  // operator-authored markup, links already resolved
    ],
    'children' => [ /* same shape, recursively */ ],
];
```

The address is resolved before the view sees it. A node bound to a panel page carries the finished link in `link`; no template converts an identifier itself.

## Step by Step

### Print the Header Tree

1. Loop the top level and branch on what the node carries. Children mean a dropdown, a mega panel means raw markup, neither means a plain item.
2. Guard the address. A node may be a heading with no link at all. Print the attribute only when `link` is not empty.
3. Print the mega panel unescaped. It is operator-authored HTML and the engine escapes by default, so otherwise the tags show up on the page.
4. Close the list with the navigation injection point, so a module can add an item without editing the partial. The header then survives any menu the operator builds.

### Print the Footer Tree

1. Loop the top level as columns and print each node's title as the column heading.
2. Loop that node's children as the column's links, with the same empty-address guard.
3. Follow the columns with the footer injection point. The footer then grows a column the operator adds, with no template change.

### Address the Pages the Theme Writes Itself

1. Use the link function with a **route key** for platform pages: the cart, sign-in, the account overview, a ticket form.
2. Pass positional parameters when the route takes them, in the order the pattern declares them.
3. Use the page form for a record the operator picked in the panel: a content page, a category. It takes the same identifier the menu system stores.
4. Never concatenate a query string by hand. Open the page and confirm the address survives a language switch, which is the point of a key.

### Hide Links to Pages the Theme Does Not Serve

1. List the route keys your theme does not implement in the manifest's meta block. The platform then answers those addresses with a not-found instead of half a page.
2. Expose the same list to templates as a keyed map from the theme's hooks file. The template sandbox has no array search function, so a map is what a condition can read.
3. Wrap every link that leads there in that condition. A closed route with a visible link is a dead end the visitor finds before you do.

## Reference

### The Menus Helper

```php
// $type: 'header' | 'footer' | 'mobile' | 'clientArea' | 'pages-sidebar'
// $parent: 0 for the whole tree, or a node id to start below it
// $lang: empty means the active language
public static function tree(string $type = 'header', int $parent = 0, string $lang = ''): array;

// Resolve {link page='...'} / {link route='...'} placeholders inside operator-authored
// markup. Already applied to a node's mega panel; call it for your own stored HTML.
public static function resolveLinks(string $html): string;
```

- **Caching**: One query per type and language, cached for an hour and memoised for the request. A panel save clears it; a theme never needs to.
- **filter:client.menu**: Fires once per tree, by reference, with the rows and a context of `type` and `lang`. A theme adds or drops a node the panel does not manage here. Edit the rows in place; the return is ignored.
- **Only active nodes**: Disabled rows never reach the tree, so a template needs no status check.

### The Link Function

- **{link route='cart'}**: A route key, translated to this language's path. Keys are declared in the route language files, so one key answers in every language.
- **{link route='invoice-detail' p1=$id}**: Positional parameters, up to five, filled into the route pattern in order. In the other tag engine they are the arguments after the route.
- **{link page='pages/4'}**: A panel page identifier (a content page, a category, a product group), resolved the way the menu system does.
- **The PHP side**: The same three shapes are `LinkGenerator::client()`, `LinkGenerator::convert_to_link()` and, for a query string, `LinkGenerator::wQS()`.

```php
// $lang: empty means the active language; pass one to build the same page in another.
public static function client($route = '', $params = [], $lang = ''): string;

// Panel identifier ("pages/4", "category/551", "home") to a finished address.
public static function convert_to_link(string $arg): string;

// Append a query string. Never concatenate one by hand: this one knows whether the
// address already carries a "?".
public static function wQS(string|bool|null $url, array|string $params = []): string;
```

## Example

The header partial, then the hooks file that closes a section this theme does not serve. Both halves are shown: the template's condition is meaningless without the map that feeds it.

```smarty
<ul class="nav site-nav">
    {foreach $header_menu as $item}
        {if $item.children}
    <li class="nav-item dropdown">
        <a class="nav-link"{if $item.link} href="{$item.link}"{/if}>{$item.title}</a>
        <div class="dropdown-menu">
            {foreach $item.children as $child}
            <a class="dropdown-item"{if $child.link} href="{$child.link}"{/if}{if $child.target} target="_blank" rel="noopener"{/if}>
                <i class="{$child.icon}"></i>
                <span>{$child.title}</span>
                {if $child.extra.desc ?? ''}<span class="item-desc">{$child.extra.desc}</span>{/if}
            </a>
            {/foreach}
        </div>
    </li>
        {elseif $item.extra.mega ?? ''}
    {* Operator-authored markup: printed raw, or the tags themselves show up on the page. *}
    <li class="nav-item dropdown">
        <a class="nav-link"{if $item.link} href="{$item.link}"{/if}>{$item.title}</a>
        <div class="dropdown-menu">{$item.extra.mega nofilter}</div>
    </li>
        {else}
    <li class="nav-item"><a class="nav-link"{if $item.link} href="{$item.link}"{/if}>{$item.title}</a></li>
        {/if}
    {/foreach}

    {* A section this theme closed: the map comes from hooks.php, the route from its key. *}
    {if !$route_off.affiliate}
    <li class="nav-item"><a class="nav-link" href="{link route='affiliate'}">{lang key='nav_affiliate'}</a></li>
    {/if}

    {hook name='ui:client.nav.items'}
</ul>

<a class="btn" href="{link route='cart'}">{lang key='nav_cart'}</a>
<a class="btn" href="{link route='invoice-detail' p1=$latest_invoice_id}">{lang key='nav_last_invoice'}</a>
```

```php
Hook::add("filter:template.variables", 1, function ($template, $data) {

    // Keyed map, not a list: the template sandbox has no in_array(), so a condition
    // can only ask {if !$route_off.affiliate}.
    $data["route_off"] = array_fill_keys(Theme::active()->meta()["disabled_routes"] ?? [], true);

    return $data;
});

/*
 * A node the panel does not manage. By reference: the tree is changed in place and
 * the context says which tree this is, so the footer is not touched by a header rule.
 */
Hook::add("filter:client.menu", 10, function (&$rows, $ctx) {
    if (($ctx["type"] ?? '') !== 'header') return;

    $rows[] = [
        'id'       => 0,
        'parent'   => 0,
        'icon'     => 'bi bi-life-preserver',
        'target'   => 0,
        'page'     => '',
        'title'    => Theme::active()->lang('nav_status'),
        'link'     => LinkGenerator::client('contact'),
        'extra'    => [],
        'children' => [],
    ];
});
```

## Pitfalls

> **A node's address can be empty**
> 
> A parent that only opens a dropdown, or a footer column heading, carries no address. Printing the attribute unconditionally emits an empty one. The browser resolves that to the current page, and a screen reader announces a link to nowhere.

> **A written path is a broken path in the second language**
> 
> Route paths are translated, and the operator can rename them. Address platform pages by route key and content by page identifier. The only paths a theme may write are its own file-based pages, whose slug is the address by definition.

> **Do not read the menu tables from a theme**
> 
> The trees arrive built, resolved, filtered to active rows and cached per language. A query in a template repeats work already paid for and skips the hook every other listener relies on.

> **The drawer is not always its own tree**
> 
> With no mobile menu built, the drawer receives the header tree. A drawer that assumes a flat list prints a dropdown's children as top-level rows, so handle children there as well.

## Related Articles

- [Template Variables](https://dev.wisecp.com/en/template-variables)
- [Theme Hooks and Output Filters](https://dev.wisecp.com/en/theme-hooks-and-output-filters)
- [Building Links and Routes](https://dev.wisecp.com/en/building-links-and-routes)
- [Translating a Theme](https://dev.wisecp.com/en/translating-a-theme)
- [Page Surfaces](https://dev.wisecp.com/en/page-surfaces)

# Translating a Theme

https://dev.wisecp.com/en/translating-a-theme

A theme keeps its own wording in its own locale files, one file per page. The operator can reword any of it from the panel, without editing a file.

## Overview

No visible string is written into a template. Every label, button and accessible name comes from a locale key. The theme resolves it against its own files first, the platform's second.

The **developer** writes the defaults, one file per page. The **operator** rewrites them from the content editor. Those edits land in a generated tree a theme update never overwrites.

## Structure

### Where Text Lives

- **locale/{lang}.php**: The theme's common scope: chrome, shared strings and the labels of its own settings, as a flat map of key to string. A `name` and a `description` key here override the manifest's. The panel's theme list shows them translated.
- **locale/{lang}/{scope}.php**: One file per page. Only the scope of the view in use is loaded, so a page pays for its own strings and nothing else.
- **content/{lang}/{scope}.php**: The operator's overrides, written by the panel. Generated, out of version control, and holding only the values that differ from the default. Removing the last one deletes the file.
- **The platform's own strings**: Wording every theme shares (accessibility labels, form errors, period names) stays in the platform's language files. It is reached by its path-style key. Do not copy it into a theme.

### What a Scope Is

A scope is the page's **public identity**, not its file path. For an ordinary view the two match; for a file-based marketing page they do not.

| View | Scope | Locale file |
| --- | --- | --- |
| `views/home.tpl` | `home` | `locale/en/home.php` |
| `views/account/settings.tpl` | `account/settings` | `locale/en/account/settings.php` |
| `views/page/about.tpl` | `about` | `locale/en/about.php` |

The last row matters. A file-based page is addressed at its bare slug, so the `page/` segment is stripped before the scope resolves.

## Step by Step

### Add a Shared String

1. Put the key in the common file of every installed language, not only yours. A key missing from a language falls through to English, a silent half-translation.
2. Prefix by region when the key belongs to the chrome (`hdr_`, `ftr_`). The common file is the only place a prefix helps.
3. Print it in the partial and open the page in both languages.

### Add a Page's Own Strings

1. Create the scope file, one per language, named after the page's public identity.
2. Write short keys: the file is already the scope, so `hero_cta`, not `home_hero_cta`. Two pages reusing a short key never collide, because only one view scope is loaded per request.
3. Open the page. The scope is loaded before the view runs, and the page appears in the content editor.

### Print It With Values

1. Call the language function with the key.
2. Pass a value by adding an attribute of the same name: `count` replaces `{count}` in the string. Any number of them, any names you like.
3. Keep the placeholder in the value and the sentence order in the translation. Never build a sentence by concatenating two keys around a value: word order differs per language.

### Add a Language

1. Copy the English tree, common file and every scope file, into the new language code.
2. Translate the values and leave the keys untouched. A renamed key is a missing key in that language only.
3. Check the pages that carry counts and dates. Their placeholders move, and that is where a mechanical translation breaks first.

## Reference

### Resolution Order

Every lookup walks this list and stops at the first hit. The last step makes an unresolved key visible instead of blank.

| Step | Source | Why it is there |
| --- | --- | --- |
| 1 | Operator override, active language | Panel edits win over everything, even a theme update |
| 2 | Theme locale, active language (common + loaded scope) | The developer's default wording |
| 3 | Theme locale, English | An untranslated language still works |
| 4 | The platform's own language files | Path-style keys like `website/index/logo-alt` keep working from a theme |
| 5 | The key itself | A missing string shows as its key, found in review rather than production |

### The Translation API

```php
// $vars is a replacement map with the braces INCLUDED: ['{count}' => 3].
// This is what the template function calls; use it from PHP for the same result.
public function lang(string $key, array $vars = []): string;

// Load a view's scope before rendering it. Called for you on the request path and by the
// context-free render; call it yourself only when you render a view by hand.
public function loadViewContent(string $view): void;

// Another theme's locale, without booting it as the active theme (the panel's theme list
// reads a theme's own name and description this way). Falls back to English.
public static function localeFor(string $themeName, string $lang = ''): array;

// '' when the name is not a usable scope. A scope is [a-zA-Z0-9/_-] and may not climb out.
public static function cleanScope(string $scope): string;
```

```php
// Every editable scope of a theme: 'common' first, then the page scopes, sorted.
public static function contentScopes(string $themeName): array;

// The developer's defaults of one scope in one language (string values only).
public static function scopeDefaults(string $themeName, string $lang, string $scope): array;

// The operator's overrides of the same scope. Empty when nothing was changed.
public static function scopeOverrides(string $themeName, string $lang, string $scope): array;

// Persist them. An EMPTY set deletes the file, which is how "reset to default" is stored:
// an override is an exception, never a copy of the default.
public static function writeScopeOverrides(string $themeName, string $lang, string $scope, array $values): bool;
```

### The Template Function

- **key**: The only required attribute. A theme key is a bare name; a platform key is its path form. An empty key returns an empty string.
- **Any other attribute**: Becomes a replacement: `count=$n` replaces `{count}` in the value. The braces belong to the string, not to the call.
- **Template syntax inside a value**: A returned string containing a template call (a link, a setting, a formatted amount) goes through the same sandbox. One that will not compile appears as-is.
- **Escaping**: Website output is escaped by default. A value that carries markup goes out with the unfiltered modifier, decided per call, not per file.

## Example

One page's strings in two languages, and the view that prints them. The placeholder lives in the value; sentence order differs between the files.

```php
return [
    // Short keys: the FILE is the scope, so a page prefix would only repeat it.
    'title'        => 'Services',
    'lede'         => 'Everything running on your account, with its next renewal date.',
    'count'        => '{n} services',
    'empty_title'  => 'No services yet',
    'empty_text'   => 'Your first order will show up here.',
    'renew_cta'    => 'Renew',
    'aria_list'    => 'Service list',
];
```

```php
return [
    // Same keys, translated values. A renamed key here is a missing key in Turkish only.
    'title'        => 'Hizmetler',
    'lede'         => 'Hesabınızda çalışan her şey ve bir sonraki yenileme tarihi.',
    'count'        => '{n} hizmet',
    'empty_title'  => 'Henüz hizmet yok',
    'empty_text'   => 'İlk siparişiniz burada görünecek.',
    'renew_cta'    => 'Yenile',
    'aria_list'    => 'Hizmet listesi',
];
```

```smarty
<h1>{lang key='title'}</h1>
<p>{lang key='lede'}</p>

{if $services}
    {* The attribute name is the placeholder name; the sentence order is the translator's. *}
    <p>{lang key='count' n=$services|@count}</p>

    <ul aria-label="{lang key='aria_list'}">
        {foreach $services as $service}
        <li>
            {$service.name}
            <a href="{link route='service-detail' p1=$service.id}">{lang key='renew_cta'}</a>
        </li>
        {/foreach}
    </ul>
{else}
    {* Accessible names are strings too: nothing visible OR announced is hardcoded. *}
    <h2>{lang key='empty_title'}</h2>
    <p>{lang key='empty_text'}</p>
{/if}

{* A platform key, in its path form: the same wording every theme shares. *}
<a href="{link route='home'}" aria-label="{lang key='website/index/logo-alt'}">{$company_name}</a>
```

## Pitfalls

> **A scope file under the page folder is never loaded**
> 
> Put the file at the slug's own path, not under the view's folder. Left where the view is, it exists, it lints, and all its strings appear as keys.

> **A page cannot read another page's keys**
> 
> Only the common scope and the current view's scope are loaded per request. A key borrowed from another page may resolve in development and print as its key in production. Shared wording belongs in the common file.

> **The override tree is generated, not source**
> 
> Written by the panel, kept out of version control, holding only the differences. Editing it by hand puts a value where the next panel save overwrites it. Shipping it in a theme package hands your test wording to every installation.

> **Do not copy platform wording into a theme**
> 
> Strings every theme shares already exist and are already translated. A copy inside a theme drifts, and it stops the operator's platform-wide wording change from reaching your theme.

## Related Articles

- [Translations and Language Files](https://dev.wisecp.com/en/translations-and-language-files)
- [Theme Settings](https://dev.wisecp.com/en/theme-settings)
- [Template Variables](https://dev.wisecp.com/en/template-variables)
- [Page Surfaces](https://dev.wisecp.com/en/page-surfaces)
- [Module Language Files](https://dev.wisecp.com/en/module-language-files)

# Theme Settings

https://dev.wisecp.com/en/theme-settings

Declare the settings a theme accepts in its manifest. The admin form, the validation and the saved file follow. The view reads the result as one variable.

## Overview

A theme is data, not a class: it declares **what** it can be configured with, never **how** that is edited. The panel validates each field by its declared type and writes the values into the theme's own directory.

That is the boundary of the contract. A theme adds a field and gets a control, but does not decide the form's plumbing. A value outside the schema is never stored.

## Structure

### Two Files, Two Owners

- **theme.php**: Schema, defaults, and the theme's name, version and engine. Shipped with the theme, never written at runtime.
- **config.php**: The saved values only: a flat map written by the panel, absent until the first save and kept out of version control.
- **Reading**: The saved value if there is one, otherwise the schema default.
- **The form**: Generated from the schema. A theme wanting more ships its own admin markup, and is handed the builder and the saved values.

### The Shape of the Schema

```php
$manifest['settings'] = [
    // key => the group's heading and icon. The label is a LOCALE KEY of this theme.
    'groups' => [
        'topbar' => ['label' => 'grp_topbar', 'icon' => 'bi-megaphone'],
    ],
    // key => a descriptor. The key is the POST name, the config key and the name the
    // view reads, all at once: renaming it orphans whatever was already saved.
    'fields' => [
        'topbar_enabled' => ['type' => 'switch', 'group' => 'topbar', 'label' => 'set_topbar_enabled', 'default' => false],
    ],
];
```

## Step by Step

### Declare a Group

1. Add an entry to the groups map with a heading and an icon class.
2. Put the heading in the theme's own locale file, both languages, and reference it by key. An untranslated key prints as itself.
3. A field whose group is not declared still appears, ungrouped.

### Declare a Field

1. Pick the type from the list below. It decides the control, the validation and what is stored.
2. Give it a real default, not an empty placeholder. That is the value on a fresh install and after every reset.
3. Add `desc` for a help sentence, and `depends` when the field only makes sense while another is on.
4. Reload the configure page; the control is there, no other file touched.

### Read It in a View

1. Read it off the settings variable by key. It is a variable and not a function so that it works inside a condition.
2. For a colour, read the property value and its `_rgb` twin wherever a translucent version is needed.
3. From PHP, ask the theme object for one setting by key, without loading the whole map.

### Take Over the Admin Form

1. Add an `admin-settings.php` to the theme directory; it replaces the generated form entirely.
2. Build the markup with the form builder, saved values and schema you are handed. Return the HTML as a string.
3. Keep field names identical to the schema keys. A control named anything else is posted and discarded.

## Reference

### Field Types

| Type | Control | Stored as | Validation on save |
| --- | --- | --- | --- |
| `switch`, `checkbox` | Checkbox; `desc` is the label beside it | Boolean | Set means true, absent means false |
| `select` | Dropdown built from `options` | The chosen key | Must be a key of `options`, else the default |
| `number` | Number input | Integer | Cast to integer; non-numeric becomes zero |
| `color` | Colour picker plus a typeable hex box | Hex string with the leading hash | Three to eight hex digits, else the default |
| `textarea` | Textarea sized by `rows` | String, or a language map when multilingual | Tags stripped unless `html` is on, then allowlist-filtered |
| anything else, or a missing type | Text input | String, or a language map when multilingual | Same as a textarea |

### What a Descriptor Accepts

- **type**: One of the types above; absent means a text field.
- **group**: The group this field belongs to. An undeclared group leaves it ungrouped.
- **label**: A locale key of this theme, not literal text. Resolved in the operator's language, then English, then the key.
- **desc**: A locale key for the help line. On a checkbox it becomes the label beside the box instead.
- **placeholder**: A locale key for the hint inside a text control. Not a label; the field still needs one.
- **default**: The value until the operator saves one, and the fallback when a posted value fails validation. An empty colour makes the derived rgb twin three zeroes.
- **options**: Select only, as stored value to label key: `['rail' => 'opt_rail', 'card' => 'opt_card']`. Also the allowlist the saved value is checked against.
- **depends**: A map of another field to the value it must hold: `['topbar_enabled' => true]`. The row collapses until every condition matches, on load and as the operator types.
- **multilang**: Text and textarea only. One tab per active language, stored as a language map. Views receive a plain string.
- **rows**: Height of a textarea, defaulting to four. Ignored by every other type.
- **html · allowed_tags**: Keeps markup in a text value instead of stripping it. The allowlist is narrowable per field; the default covers basic formatting and links.
- **from_logo**: Colour only. Marks the field as a brand colour and gives its palette index. The "pick colours from the logo" button fills it from the site logo.

### The Settings API

```php
// The manifest's settings block, exactly as declared (groups + fields).
public function settingsSchema(): array;

// One value: the saved one if the key was ever saved, otherwise the schema default,
// otherwise null. This is how the platform reads a single theme capability.
public function setting(string $key): mixed;

// The saved values only (config.php). Empty until the first save.
public function savedConfig(): array;

// Every schema field merged with its value : what the view receives as $setting.
// A multilingual value is reduced to the ACTIVE language here, and a colour field
// also produces a "{key}_rgb" entry.
public function allSettings(): array;

// A hex colour to the comma-separated triplet an rgba() expression needs. Short forms
// are expanded and a short value is padded, so an empty string yields three zeroes
// rather than an error. This is what produces the "{key}_rgb" entries above.
public static function hexToRgb(string $hex): string;
```

## Example

A group with four fields, the saved file it produces and the view that reads it back. One reason to show them together: the schema key, the file key and the view name are the same string.

```php
return [
    'engine' => 'smarty',
    'status' => 'ready',
    'meta'   => [
        'name'    => 'Acme',
        'version' => '1.0.0',
        'author'  => 'Acme Ltd',
        'image'   => 'cover.png',
    ],
    'settings' => [
        'groups' => [
            'brand'    => ['label' => 'grp_brand', 'icon' => 'bi-palette'],
            'topbar'   => ['label' => 'grp_topbar', 'icon' => 'bi-megaphone'],
            'checkout' => ['label' => 'grp_checkout', 'icon' => 'bi-cart3'],
        ],
        'fields' => [
            // A brand colour. `from_logo` is its index in the palette the panel extracts
            // from the site logo, so the "pick colours from the logo" button fills it.
            // The default is a hash plus six hex digits (the picker's own format) —
            // replace the placeholder with this theme's real colour. Leave it empty and
            // the derived primary_color_rgb reads as three zeroes, which paints every
            // translucent surface built from it black.
            'primary_color' => [
                'type'      => 'color',
                'group'     => 'brand',
                'label'     => 'set_primary_color',
                'from_logo' => 0,
                'default'   => '#RRGGBB',
            ],

            // The gate. Everything below it depends on this one.
            'topbar_enabled' => [
                'type'    => 'switch',
                'group'   => 'topbar',
                'label'   => 'set_topbar_enabled',
                'desc'    => 'set_topbar_enabled_desc',
                'default' => false,
            ],

            // Per-language, and allowed to carry markup: an announcement usually has a
            // link in it. The stored value is a map; the view still gets a string.
            'topbar_text' => [
                'type'      => 'textarea',
                'group'     => 'topbar',
                'label'     => 'set_topbar_text',
                'default'   => '',
                'rows'      => 3,
                'multilang' => true,
                'html'      => true,
                'depends'   => ['topbar_enabled' => true],
            ],

            // The keys are what gets stored; the values are locale keys of this theme.
            'checkout_sidebar' => [
                'type'    => 'select',
                'group'   => 'checkout',
                'label'   => 'set_checkout_sidebar',
                'options' => [
                    'rail'  => 'opt_checkout_rail',
                    'card'  => 'opt_checkout_card',
                    'stack' => 'opt_checkout_stack',
                ],
                'default' => 'rail',
            ],

            'popup_width' => [
                'type'    => 'number',
                'group'   => 'checkout',
                'label'   => 'set_popup_width',
                'default' => 500,
            ],
        ],
    ],
];
```

```php
return [
    // Stored with the leading hash, exactly as the picker posted it. Note that EVERY
    // schema field is written on every save, whether the operator touched it or not.
    'primary_color'    => '#RRGGBB',
    'topbar_enabled'   => true,
    // Multilingual: stored per language, one entry per ACTIVE language at save time.
    'topbar_text'      => [
        'en' => 'Free migration this month.',
        'tr' => 'Bu ay ücretsiz taşıma.',
    ],
    'checkout_sidebar' => 'card',
    'popup_width'      => 520,
];
```

```smarty
{* A boolean reads directly inside a condition, which is why this is a VARIABLE and
   not a function: a registered function cannot be called from {if}. *}
{if $setting.topbar_enabled}
    {* Markup was kept on save, so it is printed unfiltered here. *}
    <div class="topbar">{$setting.topbar_text nofilter}</div>
{/if}

{* A select is just its stored key: use it as a class rather than branching four times. *}
<div class="checkout-layout-{$setting.checkout_sidebar}">

{* A colour field also produces an _rgb twin, for a translucent version of the same colour. *}
<style>
    :root {
        --acme-brand: {$setting.primary_color};
        --acme-brand-rgb: {$setting.primary_color_rgb};
    }
</style>
```

## Pitfalls

> **Renaming a key orphans what was already saved**
> 
> The key is the form name, the stored key and the name the view reads. A value under the old name is never read again and never cleaned up. Every installation that had configured the theme falls back to the default.

> **The saved file is generated, not source**
> 
> It holds values and nothing else. Name and version stay in the manifest, where the theme listing and the upgrade check read them. Shipping one in a theme package hands your test configuration to everyone who installs it.

> **A multilingual value is a map in the file and a string in the view**
> 
> PHP asking for the raw setting gets the whole map. The view gets the active language resolved. Code confusing the two works in exactly one place.

> **A collapsed dependent field is still saved and still read**
> 
> The dependency hides the row; it does not disable the value. A view reading a dependent setting must check the field it depends on too. Otherwise it shows something the operator believes is off.

## Related Articles

- [The Theme Engine](https://dev.wisecp.com/en/the-theme-engine)
- [Theme Anatomy](https://dev.wisecp.com/en/theme-anatomy)
- [Template Variables](https://dev.wisecp.com/en/template-variables)
- [Translating a Theme](https://dev.wisecp.com/en/translating-a-theme)
- [The Admin Form Builder](https://dev.wisecp.com/en/the-admin-form-builder)

# Securing Theme Forms

https://dev.wisecp.com/en/securing-theme-forms

A public form is not protected by the framework. The template prints the guards and the handler enforces them.

## Overview

No middleware shields a theme form. You write the pair: the view prints a token and a challenge box, the operation checks them. The form key joins the halves.

Five layers stack on one request. Four judge *who submits*, the fifth judges *what was submitted*.

## Prerequisites

- A Smarty or Twig theme; `{csrf}` and `{captcha}` exist in both.
- An operation to receive the submit ([Operations](https://dev.wisecp.com/en/operations)).
- Thresholds are the operator's, in `coremio/configuration/options.php`. Never hard-code one.

## Structure

### The Five Layers

| Layer | What it stops | What leaving it out opens |
| --- | --- | --- |
| **CSRF** | Submits that did not come from your own form. | Any page can post to your endpoint with a signed-in visitor's session. |
| **Process restriction** | Repetition from one address, with a timed hard block. | One address can hold the endpoint open indefinitely. |
| **Bot shield** | Automation, by demanding a challenge past a threshold. | An automated client never meets a challenge; the form becomes an oracle. |
| **Captcha (static)** | Every submit on an area the operator locked. | The closed area stays open, and the panel shows the setting as on. |
| **Spam guard** | Banned words, disposable mailboxes, reputation lists. | No content rule runs, so the blocked list stays empty and looks healthy. |

Each layer is one call, and that call is the proof.

- **Validation::verify_csrf_token()**: The token layer. A false result must end the request.
- **ProcessRestriction::blocked()**: The hard block layer. Asks only; `hit()` counts the request at the end.
- **BotShield::triggered()**: The adaptive layer. True means this address must answer a challenge.
- **Captcha::enabled()**: The static layer. Reads the per-area switch; combined with the previous by OR.
- **Validation::spam_guard()**: The content layer. Returns the blocking reason and records the attempt.

> **The order is part of the protection**
> 
> Token, hard block, challenge, content, then the work. Reading input earlier lets an attacker reach your parser.

### Which Half Owns Which Piece

| Piece | Theme side | Server side |
| --- | --- | --- |
| Token | `{csrf form='contact-form'}` prints a hidden `token` input. | Verified with the same key. |
| Challenge | `{captcha area='contact-form'}` prints the provider's box, or nothing. | Read by the captcha helper; the theme never names it. |
| Request header | The fetch call sends `X-Requested-With`. | Required unless the third argument says otherwise. |
| Thresholds | Nothing; the theme reads no limit. | Read from the operator's configuration. |

## Walkthrough

### 1. Mark Up the Form

1. Pick one key string and use it everywhere: token key, captcha area, throttle action.
2. Print the token inside the form element.
3. Print the captcha next to the button; it shows nothing when that area is off.

```smarty
<form action="{link route='contact'}" method="post" novalidate data-contact-form>

    <input type="text"  class="form-control" name="name"  required>
    <input type="email" class="form-control" name="email" required>
    <textarea class="form-control" name="message" rows="6" required></textarea>

    {* Same string as the handler's verify key. Emits <input type="hidden" name="token">. *}
    {csrf form='contact-form'}

    <div class="d-flex flex-wrap align-items-center gap-3 border-top pt-3 mt-4">
        {* Renders '' when the operator has captcha off for this area, so the row still lays out. *}
        {captcha area='contact-form'}
        <button class="btn btn-primary ms-sm-auto" type="submit">{lang key='website/contact/send-button'}</button>
    </div>
</form>
```

### 2. Guard the Handler

1. Verify the token before you touch the request.
2. Ask whether this address is hard blocked, and stop if it is.
3. Decide whether a challenge is required: the area is locked *or* the shield tripped.
4. Read and validate the inputs, then run the spam guard on them.
5. Do the work, then close the window: advance the limiter, update the shield counter.

```php
// 1. Token. Before anything is read from the request.
if (!\Validation::verify_csrf_token((string) Filter::init("POST/token", "hclear"), "contact-form"))
    throw new \Exception(Language::g("needs/csrf-failed"));

// 2. Hard block. This address already crossed the limit and is serving a timeout.
if (\ProcessRestriction::blocked("contact-form"))
    throw new \Exception(Language::gc("website/contact/rate-limited"));

// 3. Challenge. Operator-locked area OR this address tripped the shield.
$needCaptcha = \Captcha::enabled("contact-form") || \BotShield::triggered("contact-form");
if ($needCaptcha && !(new \Captcha())->check()) {
    \BotShield::record("contact-form");
    return $operation->output([
        "status"  => "captcha_required",
        "message" => Language::gc("website/contact/captcha-required"),
    ]);
}

// 4. Content and sender, after validation, before the write.
if (\Validation::spam_guard($full_name, $message, $email, $phone, $ip) !== '')
    throw new \Exception(Language::g("needs/spam-blocked"));

// 5. ... the actual work ...

// 6. Close the window.
\ProcessRestriction::hit("contact-form");
if ($needCaptcha) \BotShield::clear("contact-form");
else              \BotShield::record("contact-form");
```

### 3. Send the Request

1. Build the body from the form element; `FormData` carries the token and the answer.
2. Send the AJAX header, or the token check refuses.
3. Handle the third outcome: `captcha_required` is a question, not a refusal.
4. Refresh the challenge in the finally branch. The answer is single use.

```javascript
var fd = new FormData(form);          // token + captcha answer are named inputs inside the form

fetch(endpoint, {
    method: 'POST',
    body: fd,
    headers: { 'X-Requested-With': 'XMLHttpRequest' }
})
    .then(function (r) { return r.json(); })
    .then(function (res) {
        if (res.status === 'captcha_required') {
            captchaRequired = true;               // remember it; the next empty submit is stopped locally
            captchaMsg      = res.message || captchaMsg;
            revealCaptcha();
            setAlert(captchaMsg, 'warning');
            return;
        }
        if (res.status !== 'successful') { setAlert(res.message, 'danger'); return; }
        captchaRequired = false;
        showDoneStep(res);
    })
    .finally(function () {
        if (typeof window.wcpCaptchaRefresh === 'function') window.wcpCaptchaRefresh();
    });
```

## Reference

### Template Functions

| Call | What it emits | When it emits nothing |
| --- | --- | --- |
| `{csrf form='<key>'}` | A hidden `token` input: an HMAC of the key against a per-session secret. | Never. An empty key is still a key, shared by every form using it. |
| `{captcha area='<area>' tray='<id>'}` | The provider's box in the theme slot; a tray when `tray` is given. | When captcha is off *and* the shield is not armed. Design the row without it. |

In Twig the options have an order: `captcha(area, tray, class, force)` and `csrf(form)`. Skipping one means naming it.

### Helper Signatures

Read the argument order carefully: the token functions take the key *second*.

```php
// coremio/classes/Validation.php

// $input = true returns the ready <input type="hidden" name="token">; false returns the bare token.
public static function get_csrf_token($form_index = '', $input = true);

// The KEY IS THE SECOND ARGUMENT. $nonAjax = true drops the X-Requested-With requirement.
public static function verify_csrf_token($incoming_data = '', $form_index = '', $nonAjax = false);

// '' means clean. A non-empty string is the operator-facing reason, already written to the blocked list.
public static function spam_guard(string $subject = '', string $message = '', string $email = '', string $phone = '', string $ip = '', string $domain = ''): string;
```

```php
// coremio/helpers/processrestriction.php
// $ip = null resolves the caller's address itself, proxy and CDN aware. Pass one only when
// you are judging an address other than the current visitor's.

public static function blocked(string $action, ?string $ip = null): bool;   // serving a timeout right now
public static function hit(string $action, ?string $ip = null): bool;       // count one; true = now blocked
public static function clear(string $action, ?string $ip = null): void;     // reset counter and block
```

```php
// coremio/helpers/botshield.php

public static function active(string $action): bool;                        // armed for this action at all
public static function triggered(string $action, ?string $ip = null): bool; // this address needs a challenge
public static function record(string $action, ?string $ip = null): void;    // count one uncontested attempt
public static function clear(string $action, ?string $ip = null): void;     // a challenge was solved
```

```php
// coremio/helpers/captcha.php

public static function enabled(string $area = ''): bool;                    // operator switched this area on
public static function widget(string $area = '', array $opts = []): string; // what {captcha} calls
public function check(): bool;                                              // instance method: (new Captcha())->check()
```

### Widget Options

- **tray**: The id of a collapse tray to hold the box. Sanitised to letters, digits, underscore, hyphen.
- **class**: Utility classes for the slot wrapper.
- **force**: Always visible, ignoring the per-area switch and the shield. A decision, not a default.

### The Three Response Shapes

- **successful**: The work happened; swap the form for a confirmation step.
- **captcha_required**: The server is asking, not refusing. Reveal the box and keep the typed values.
- **error**: A thrown exception as JSON. Already translated; the spam reason is not in it.

## Example

The contact form, both halves. The key `contact-form` repeats in view and handler, so one grep proves the pair.

```php
public function submit(Operation $operation): bool
{
    $operation->demo();

    if (!\Validation::verify_csrf_token((string) Filter::init("POST/token", "hclear"), "contact-form"))
        throw new \Exception(Language::g("needs/csrf-failed"));

    if (\ProcessRestriction::blocked("contact-form"))
        throw new \Exception(Language::gc("website/contact/rate-limited"));

    $needCaptcha = \Captcha::enabled("contact-form") || \BotShield::triggered("contact-form");
    if ($needCaptcha && !(new \Captcha())->check()) {
        \BotShield::record("contact-form");
        return $operation->output([
            "status"  => "captcha_required",
            "message" => Language::gc("website/contact/captcha-required"),
        ]);
    }

    $full_name = trim((string) Filter::init("POST/name", "hclear"));
    $email     = trim((string) Filter::init("POST/email", "email"));
    $phone     = trim((string) Filter::init("POST/phone", "numbers"));
    $message   = trim((string) Filter::init("POST/message", "hclear"));
    $ip        = \UserManager::GetIP();

    if (\Validation::isEmpty($full_name))
        throw new \Exception(Language::gc("website/contact/error-name"));
    if (\Validation::isEmpty($email) || !\Validation::isEmail($email))
        throw new \Exception(Language::gc("website/contact/error-email"));
    if (\Validation::isEmpty($message) || mb_strlen($message) < 5)
        throw new \Exception(Language::gc("website/contact/error-message"));

    // Content rules run on the values that are about to be stored, never on the raw request.
    if (\Validation::spam_guard($full_name, $message, $email, $phone, $ip) !== '')
        throw new \Exception(Language::g("needs/spam-blocked"));

    $message_id = $this->model->add([
        'full_name' => $full_name,
        'email'     => $email,
        'phone'     => $phone,
        'message'   => $message,
        'ip'        => $ip,
        'cdate'     => \DateManager::Now(),
    ]);

    \ProcessRestriction::hit("contact-form");
    if ($needCaptcha) \BotShield::clear("contact-form");
    else              \BotShield::record("contact-form");

    return $operation->output([
        "status"  => "successful",
        "message" => Language::gc("website/contact/success"),
        "email"   => $email,
    ]);
}
```

Two extension points: a veto hook before the write, an event hook after.

```php
// Any listener returning a non-empty string refuses the submission with that message.
foreach (\Hook::run('gate:client.contact_submit', $full_name, $email, $phone, $message, $ip) as $veto)
    if (is_string($veto) && $veto !== '') throw new \Exception($veto);

// After the write. Return values are ignored; this is an announcement, not a decision.
\Hook::run('action:client.contact_submitted', $message_id, $full_name, $email, $phone, $message, $ip);
```

- **gate:client.contact_submit**: Runs after validation, before the write. A non-empty string is a refusal.
- **action:client.contact_submitted**: Runs after the row exists, with its id first. The return is ignored.

## Pitfalls

> **A mistyped key rejects every submit, forever**
> 
> The token is an HMAC of the key. Print one key, verify another, and the visitor sees an expired session. No warning, no log entry.

> **Without the AJAX header the token check refuses by design**
> 
> Verification demands `X-Requested-With` unless the third argument disables it. A missing header looks like a wrong key.

> **Gate the client on the server's answer, not on whether the box is visible**
> 
> A tray can open because the visitor started typing. Keep a flag that only `captcha_required` sets.

> **Do not reveal the results panel in the finally branch**
> 
> Opening the results container after every request leaks the placeholder. Open it on success only.

> **Four bot layers do not imply the fifth**
> 
> The throttling layers judge an address, not content. A perfect token still stores banned words until the spam guard runs.

## Related Articles

- [Login and Registration](https://dev.wisecp.com/en/login-and-registration)
- [Theme Hooks and Output Filters](https://dev.wisecp.com/en/theme-hooks-and-output-filters)
- [Operations](https://dev.wisecp.com/en/operations)
- [Filtering User Input](https://dev.wisecp.com/en/filtering-user-input)
- [Writing a Captcha Module](https://dev.wisecp.com/en/writing-a-captcha-module)

# Theme Performance and Caching

https://dev.wisecp.com/en/theme-performance-and-caching

A theme runs on every page view, so anything it computes twice is paid twice. The cache helper stops that; the key makes it correct.

## Overview

Themes are cheap by construction: the template prints values, the controller produces them. The one place a theme can slow an installation down is `hooks.php`.

The remedy is one call: a bucket, a key, a lifetime and a producer. Two things decide whether it is correct: the key, and what you leave outside it.

## Prerequisites

- A working theme. New to `hooks.php`? Read [Theme Hooks and Output Filters](https://dev.wisecp.com/en/theme-hooks-and-output-filters) first.
- One handler per theme for `filter:template.variables`. Only the last returned value survives.
- Nothing to switch on; the cache setting is read inside the helper.

## Structure

### Where Work Belongs

| Need | Where it belongs | How often it runs |
| --- | --- | --- |
| Data for one page (this service's invoices) | The controller, with the page data | Once, on that page only. |
| Data every page needs (footer categories, a currency strip) | The theme's `hooks.php`, through the variables filter | Every website request. The only per-request cost a theme creates. |
| Presentation (loops, conditions, formatting) | The template | Every page, and it must stay free. A template that queries has no cache at all. |

### What Belongs in the Key

The key is the whole contract. Anything the value depends on has to appear in it.

| Fragment | Read from | Leaving it out produces |
| --- | --- | --- |
| Currency | `Money::getUCID()` | Prices formatted for whoever warmed the cache, with their symbol. |
| Language | `Language::selected()` | Titles and generated links in one language everywhere. Links are built through the selected language. |
| Scope of the query | The category, product or menu id you passed in | One scope's rows served for every scope. The fragment forgotten when a parameter is added later. |
| Visitor identity | Nothing. It never belongs in a shared cache. | A key per visitor is a leak, not a cache. Per-visitor values are not cached at all. |

## Walkthrough

### 1. Find the Repeated Work

1. Look at what your handler calls, not what it returns. A plain-looking getter can issue one query per node.
2. Ask how the value changes. Catalogue data and menus tolerate an hour; stock and carts do not.
3. Ask who the value belongs to. If the answer names a person, no key makes it cacheable.

### 2. Wrap the Producer

1. Put the expensive work in a closure. Computing first and caching afterwards runs it every time.
2. Build the key from the fragments above, in a fixed order, with a prefix unique to your theme.
3. Choose a lifetime: an hour for operator-edited data, a day for reference data. Zero never expires.
4. Add a static guard when one request asks for the same value several times.

```php
$ucid = (int) Money::getUCID();

$rows = Cache::remember('website', 'acme_footer_' . $ucid . '_' . Language::selected(), 3600,
    fn (): array => Products::group_cards($ucid));
```

### 3. Keep Hooks Outside the Cached Region

1. Cache the data, then filter the result. Inside the producer, a module's listener freezes into the cached copy.
2. Lift the context into local variables first: the by-reference runner turns a literal into a fatal error.
3. Follow the shipped precedent: the software store list caches its rows, then filters them.

```php
$out = Cache::remember('website', 'software_store_' . $categoryId . '_' . $ucid . '_' . Language::selected(), 3600,
    function () use ($ucid, $categoryId): array {
        // one query, many rows, shaped for the template
        return $this->build($categoryId, $ucid);
    });

// OUTSIDE the producer: this must run on every request, cached or not.
$ctx = ['currency' => $ucid, 'category' => $categoryId];
Hook::runRefs('filter:product.software_list', $out, $ctx);

return $out;
```

## Reference

### The Canonical Call

```php
// coremio/classes/Cache.php
// Returns whatever the producer returns: array, string, int, object.
public static function remember(string $name, string $key, int $ttl, callable $producer);
```

- **$name**: The bucket. One file per bucket, loaded and cleared together. Website data uses `website`; menus use `menus`.
- **$key**: The entry, and the whole correctness contract. Prefix it with your theme, then append every fragment.
- **$ttl**: Lifetime in seconds. 3600 for operator-edited data, 86400 for reference data. Zero disables expiry.
- **$producer**: Any callable. Runs on a miss, and on every call when caching is switched off. Never add your own check.

### The Instance API

Rarely needed. Reach for these to inspect or remove a single entry.

```php
// coremio/classes/Cache.php
public static function getInstance(): self;

public function store($key, $data, $expiration = 86400): bool;   // $expiration = 0 never expires
public function retrieve($key, $timestamp = false);              // null when missing or unreadable
public function isCached($key): bool;                            // also drops the entry if expired
public function erase($key): self;                               // drops one entry
public function eraseAll(): self;                                // empties the bucket this instance points at
public function clear($keys = []): void;                         // named buckets, or no argument = every bucket
```

There is no `get()` and no `set()`. A missing or corrupt entry reads back as null, treated as a miss.

### Invalidation

| Trigger | Call | What it removes |
| --- | --- | --- |
| An operator saved something in the panel | Already done by the operation that saved it | Everything. Most save operations clear the whole store. |
| You cached something the panel does not know about | `Cache::getInstance()->clear(['bucket'])` | The named buckets only. Use this when your theme owns both the write and the read. |
| Time passing | Nothing | The entry, on the first read after its lifetime. Expiry is checked on read. |

> **Repeats inside one request are already handled**
> 
> The bucket file is read once per request, so three keys cost one file read. A static variable also skips the key building.

## Example

Product group cards in a footer. The producer is cached, the filter runs outside it, the template only loops.

```php
/*
 * ONE handler per theme: the view layer keeps only the last returned value, so a second
 * registration would drop every key set here. Add keys, do not add a second Hook::add.
 */
Hook::add("filter:template.variables", 1, function ($template, $data) {
    $data["footer_groups"] = acme_footer_groups();

    return $data;
});

if (!function_exists('acme_footer_groups')) {
    /**
     * Product group cards for the footer strip. Runs on every website page, so the query
     * behind it is cached; the value is shared by every visitor, which is exactly why the
     * key has to carry the two things that make it visitor-specific.
     */
    function acme_footer_groups(): array
    {
        // Same request, several partials: skip even the key building.
        static $rows = null;
        if ($rows !== null) return $rows;

        // Currency comes from the visitor, links come from the selected language.
        // Both go in the key, or the first visitor's version is served to everyone.
        $ucid = (int) Money::getUCID();

        $rows = Cache::remember('website', 'acme_footer_groups_' . $ucid . '_' . Language::selected(), 3600,
            fn (): array => Products::group_cards($ucid));

        return $rows;
    }
}
```

```smarty
{if $footer_groups}
    <ul class="footer-links">
        {foreach $footer_groups as $g}
            <li><a href="{$g.link}">{$g.title}</a> <span class="text-body-secondary">{$g.price}</span></li>
        {/foreach}
    </ul>
{/if}
```

The template does no lookup and no formatting: the price arrived formatted.

- **Cache::remember()**: The only cache call a theme needs. Falls through to the producer when caching is off.
- **Money::getUCID()**: The visitor's currency id. Belongs in the key of anything that produces a formatted amount.
- **Language::selected()**: The selected language code. Belongs in the key of anything that produces text or a generated link.
- **Products::group_cards()**: The shipped producer used above; it caches internally with the same two fragments.

## Pitfalls

> **A key without the currency is wrong on a machine you never test on**
> 
> The cache is warmed by whoever loaded the page first; locally that is always you. The same applies to the language.

> **A hook inside the producer runs once an hour instead of once a request**
> 
> A filter inside the closure is applied only when the entry is rebuilt. The module works right after a clear and stops a minute later.

> **The by-reference hook runner takes every argument by reference**
> 
> That includes the context after the filtered value: a literal or a cast is a fatal error.

> **Never cache what belongs to one visitor**
> 
> Stock levels, cart contents, anything behind a login: none of it goes through a shared store. Wrong for the second reader means data exposure.

> **Register the variables filter once per theme**
> 
> Only the last returned value survives, so a second registration drops every key the first one set. The symptom is template variables going empty at once.

## Related Articles

- [Caching](https://dev.wisecp.com/en/caching)
- [Theme Hooks and Output Filters](https://dev.wisecp.com/en/theme-hooks-and-output-filters)
- [Template Variables](https://dev.wisecp.com/en/template-variables)
- [Menus and Navigation](https://dev.wisecp.com/en/menus-and-navigation)

# Responsive and Accessible Markup

https://dev.wisecp.com/en/responsive-and-accessible-markup

The measurable contract a theme's markup must satisfy, and how each rule is proven.

## Overview

Responsiveness and accessibility are not a review pass at the end. They are a few shell declarations plus rules you can measure on a live page.

Two obligations belong to the theme alone: the viewport declaration in every layout, and the root language and direction attributes.

## Prerequisites

- A theme with at least one layout: any template that opens its own document head.
- Bootstrap 5 semantics. A version 4 class name is not an error: it is silently unstyled.
- A browser you can measure in.

## Structure

### Layout Declarations

Three declarations per layout; the engine adds none.

1. The viewport meta, exactly `width=device-width, initial-scale=1`. Without it a phone lays out at desktop width.
2. The root language and writing direction, from the engine's variables.
3. The direction-aware stylesheet choice before first paint, inside the head guard.

Audit, per theme: `grep -L 'name="viewport"' templates/website/<Theme>/layouts/*.tpl` must print nothing.

### Engine Variables

| Variable | Shape | Resolved from |
| --- | --- | --- |
| `$ui_lang` | A language code, such as `en` or `tr`. | The selected language, resolved again on every request. |
| `$ui_dir` | Exactly `ltr` or `rtl`, never anything else. | The language pack's own direction flag. |
| `$setting` | Every manifest field, merged with saved values. | The settings schema; operator-owned layout switches only. |

## Reference

### Theme API

```php
// coremio/classes/Theme.php
// Behind {asset path='...'}. Appends ?v= with the file's modification time for css and js ONLY.
// Fonts and images stay query-less on purpose: a versioned font preload would never match the
// query-less url() inside the font stylesheet, the preload is wasted and the face misses first paint.
public function assetUrl(string $path = ''): string;

// Behind {lang key='...' var='...'}. Every named argument except 'key' and 'g' becomes a
// {var} replacement in the value, which is how an accessible name carries a real number.
// Smarty only: the Twig function takes the key alone and forwards no replacements, so a
// Twig theme substitutes outside the call rather than shipping a literal {count} to a reader.
public function lang(string $key, array $vars = []): string;

// True when views/<view> exists in this theme, used to gate a shell before rendering into it.
public function viewExists(string $view): bool;
```

```php
// coremio/classes/Language.php
public static function selected(): string;
public static function g($key = '', $replaces = [], $slang = ''): array|string|int|bool;
```

```php
// The template only prints the result; it never resolves the direction itself.
$ui_lang = Language::selected();
$ui_dir  = Language::g("package/rtl") ? 'rtl' : 'ltr';
```

- **Theme::assetUrl()**: Resolves and versions a path inside the theme's assets. A hand-written link has no cache buster.
- **Theme::lang()**: Every visible string, and every invisible one: aria labels are text too.
- **Language::selected()**: The code in the root language attribute. Screen readers read pronunciation from it.

### Breakpoints

A fourth query is a decision, not a detail.

| Query | Tier | What belongs here |
| --- | --- | --- |
| `max-width: 575.98px` | Below the small tier | Phone compaction: hiding a label, collapsing a toolbar. |
| `max-width: 767.98px` | Below the medium tier | Layout changes that outlive the phone. |
| `min-width: 992px` | Large and up | Desktop-only chrome. Written as a minimum, so the small screen is the default. |
| `print` | Print | Chrome that has no meaning on paper. One block. |

### Type Scale

A new size is never invented next to an existing one.

| Class | Value | At a 16px root | Use for |
| --- | --- | --- | --- |
| `fs-7` | 0.85rem | 13.6px | Secondary text: card bodies, list rows, table cells. |
| `fs-8` | 0.8rem | 12.8px | Hints and helper lines under a control. |
| `fs-9` | 0.7rem | 11.2px | The floor. Badges and micro labels only. |
| Headings | Bootstrap scale | The fifth heading step is 20px | Section titles. A helper line is never larger than its label. |

### Motion and Focus

| Preference | What the theme does | How to prove it |
| --- | --- | --- |
| Reduced motion | A global block clamps every animation to a hundredth of a millisecond and one iteration. | Count elements still at zero opacity; the answer must be zero. |
| Keyboard focus | Rings appear for keyboard traversal only. | Tab through: every stop must be visible. Then click: no ring may remain. |

### Off-Screen Text

| Need | Correct markup | What the wrong one does |
| --- | --- | --- |
| A control with no visible text | An aria label from the language file. | Hard-coded text stays English everywhere else. |
| Text for screen readers only | `visually-hidden` | `sr-only` is the Bootstrap 4 name; Bootstrap 5 does not define it. A bundled icon library keeps it alive. |
| A landmark for keyboard users | A nav element with an aria label, plus the current page marked. | Unnamed navigation landmarks are indistinguishable in a landmark list. |

### Wide Content

| Content | Wrapper | Without it |
| --- | --- | --- |
| A table with more columns than a phone can show | A responsive table wrapper around the table element. | The page itself scrolls sideways, and every surface on it inherits that. |
| A long unbroken string (a key, a token, a domain) | Wrapping on the cell, not a width removal on the container. | Widening the container pushes the last columns past the edge. |
| A code block or payload | Its own scroll container. | The same page-level sideways scroll, on pages with a long line. |

Wide content scrolls inside its own box; the page body never does.

## Example

The obligations, in layout-head order.

```html
<html lang="{$ui_lang|default:'en'}" dir="{$ui_dir|default:'ltr'}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    {* Before first paint: resolve the stored direction and write the matching Bootstrap file,
       so the first painted frame is already correct instead of flipping a frame later. *}
    <script>
      var storedDir = localStorage.getItem('acme-dir');
      var rtl = storedDir ? storedDir === 'rtl' : document.documentElement.getAttribute('dir') === 'rtl';
      document.documentElement.setAttribute('dir', rtl ? 'rtl' : 'ltr');
      document.write('<link rel="stylesheet" href="' +
        (rtl ? '{asset path="css/bootstrap.rtl.min.css"}' : '{asset path="css/bootstrap.min.css"}') + '">');
    </script>

    <link rel="stylesheet" href="{asset path='css/default.css'}">
</head>
```

The two markup patterns that carry most of the accessibility work.

```smarty
<nav class="client-subnav" aria-label="{lang key='website/index/subnav-aria'}">
    <ul>
        <li>
            <a class="client-subnav-link{if $subnav == 'services'} active{/if}"
               {if $subnav == 'services'} aria-current="page"{/if}
               href="{link route='services'}">
                <i class="bi bi-hdd-stack"></i>{lang key='website/index/subnav-services'}
                {if $client_badges.services > 0}
                    {* The number alone is meaningless out of context, so the badge names itself. *}
                    <span class="client-subnav-badge"
                          aria-label="{lang key='website/index/subnav-badge-services' count=$client_badges.services}">
                        {$client_badges_text.services}
                    </span>
                {/if}
            </a>
        </li>
    </ul>
</nav>

{* Bootstrap 5 name. 'sr-only' is not a Bootstrap 5 class; do not rely on it. *}
<span class="visually-hidden">{lang key='website/index/footer-payment-methods'}</span>
```

Those two variables arrive with the client page data, not from the engine.

- **$client_badges**: Integers keyed `services`, `domains`, `invoices`, `support`. A key is zero rather than absent when the section is off.
- **$client_badges_text**: The same keys, as display strings; past two digits it becomes a plus form. Print this inside the badge, the raw one inside the label.

The acceptance list, measured on a live page.

- **No sideways page scroll**: At 320 CSS pixels wide, document scroll width must not exceed window inner width.
- **Tap targets**: At least 44 CSS pixels on the shorter axis. Measure the box, not the icon.
- **Focus visible on keyboard, absent on pointer**: Tab through: every stop is visible. Click: no ring survives the press.
- **Reduced motion leaves nothing hidden**: With the preference on, no element stays at zero opacity.
- **Contrast**: 4.5 to 1 for body text, 3 to 1 for large text. Measure in light and in dark.
- **Right to left**: Flip the root direction attribute and re-walk. Nothing overlaps, nothing is clipped.

## Pitfalls

> **A Bootstrap 4 class name is silently unstyled**
> 
> Copied markup keeps working until the missing class was the one doing the work. Find the rule that styles it first.

> **One layout without the viewport meta breaks a whole flow**
> 
> The obligation is per layout, not per theme. A checkout shell without it puts the purchase flow at desktop width.

> **Widening the container hides more than it shows**
> 
> Removing a cell's width limit pushes the last columns off the edge. Let the value wrap instead.

> **Invisible text is still text**
> 
> Aria labels are the easiest strings to hard-code, because nobody sees them in review.

> **A fix in one theme is a fix in all of them**
> 
> Everything here is behaviour, not visual identity. A fix in one theme only leaves the same defect elsewhere.

## Related Articles

- [Theme Anatomy](https://dev.wisecp.com/en/theme-anatomy)
- [Theme Assets](https://dev.wisecp.com/en/theme-assets)
- [Translating a Theme](https://dev.wisecp.com/en/translating-a-theme)
- [Page Surfaces](https://dev.wisecp.com/en/page-surfaces)

# Tables and Record Lists

https://dev.wisecp.com/en/tables-and-record-lists

A theme never ships its own grid library: record lists run on the core Table component, and everything else is plain markup.

## Overview

Two paths cover every table a theme needs. The question is not how many rows you have, but **what the rows are**.

- **Record list**: Services, invoices, domains, tickets, ledger entries, messages, activity. The visitor searches, filters and pages through them, and the set can be empty. Use `Table` with `'renderer' => 'list'`.
- **Document table**: Invoice line items, a feature matrix, DNS records, an SMS report. It is read, not navigated, and it belongs to the page. Write plain markup in the template.

## Prerequisites

- Your theme is active, so `Theme::active()` resolves to its directory.
- You know the component itself; this article only covers the client-side list mode.
- A `tables/` directory exists in your theme root.

## Structure

Four layers, and each one owns a different file. The core controller already exists for the pages listed above, so a theme usually writes only the second and third.

- **Data and paging**: Core controller. Builds the table and hands over the two model closures.
- **Row markup**: Your preset, at `templates/website/{Theme}/tables/{name}.php`.
- **Container, toolbar, pager**: Your view, at `views/account/{name}.tpl`.
- **Interaction**: Your `list.js`: client search, filter, paging, counter text, empty state.

## Step by Step

### Adding a record list

1. Copy the preset for the page from another theme into your own `tables/` directory. The file name matches the table name the controller passes.
2. Rewrite the markup inside `setRowRender()` in your own design language, and keep the data and the order untouched.
3. Mirror that markup in the view's `{foreach}` branch, so the first page and the later pages look the same.
4. Give the container its three text bridges, then load the page and page through it. The counter now reads in your language and the empty state appears when a filter matches nothing.

## Reference

The list mode changes three things about the component.

- **Table::preset()**: Reads from two different roots. In the panel it is `templates/admin/tables/`; on the site it is `Theme::active()->dir()."tables"`. A preset placed in the admin directory is never found.
- **setRowRender(callable)**: Your callback returns the whole row in `$row["html"]`. The column matrix under `$row["data"]` is unused in this mode.
- **buildList(string $format = ''): string**: Concatenates the rows and returns them. No columns, no slicing, and `justBody` returns the same thing as the full call.
- **Sorting**: There are no column headers to click, so the order comes from the model's allowlist.
- **Download**: Never offered here. The privilege check returns false for a customer session.

The controller finishes the work the template cannot do, because a theme template may not call the component under the sandbox:

- **build('justBody')**: Builds the rows ahead of time and hands them to the view as a ready string.
- **getAjax()**: The data address, and it is filled **only** when the account crossed the row threshold. The view writes it on the container as `data-ajax`.
- **ajaxControl(['baseLink' => …])**: Answers a data request and returns a non-empty string when it did, so the controller returns that value untouched.

> **Small accounts never reach the server**
> 
> Below the threshold every row is already in the first answer, `data-ajax` is absent and searching, filtering and paging all happen in the browser. Above it the same address serves one page at a time. Your markup is identical in both, which is why the row attributes below carry their own values.

The container carries the wording the interaction script needs:

- **data-noun**: The key inside the script, in English, and it is never shown.
- **data-txt-count · data-txt-nores · data-txt-noun**: The translated counter sentence, the no-match sentence and the word for the record. Leave one out and that text falls back to English.

## Example

```php
/** @var \WISECP\Components\Table $table */
if (!isset($table)) return;

$table->setRowRender(function ($row) {
    $service = $row["model"];
    $name    = htmlspecialchars($service["name"]);
    $status  = $service["status"];
    $danger  = in_array($status, ['suspended', 'expired'], true) ? ' is-danger' : '';
    $link    = LinkGenerator::client("service-detail", [$service["id"]]);

    // The whole row, in your own markup. No <td> anywhere.
    $row["html"] = '<article class="list-item' . $danger . '">'
        . '<a href="' . $link . '">' . $name . '</a>'
        . '<span class="badge">' . $status . '</span>'
        . '</article>';

    return $row;
});
```

```html
<div class="list-rows" data-noun="services"
     data-txt-count="{lang key='website/index/list-count'}"
     data-txt-nores="{lang key='website/index/list-nores'}"
     data-txt-noun="{lang key='website/services/noun'}">
    {$rows nofilter}
</div>
```

## Pitfalls

> **A preset in the wrong directory fails silently**
> 
> The lookup asks whether the file exists and moves on when it does not. Nothing is logged, no error appears, and the list comes back empty. If your rows are missing, check the path before you check your code.

> **The first page and the later pages must match**
> 
> The view builds page one, and the preset builds every page after it. When the two drift apart, the rows change shape as soon as the visitor pages forward.

> **Two empty states, not one**
> 
> No records at all replaces the toolbar, the list and the pager together, and it may offer a way to create the first record. A filter that matches nothing lives inside the container and offers a way to clear the filter. Writing only one of them leaves the visitor stuck.

> **Every theme needs the file**
> 
> Presets live in the theme, so each theme carries its own copy of all seven. A missing one costs that theme an empty list.

## Related Articles

- [Interface Components](https://dev.wisecp.com/en/interface-components)
- [The Client Area](https://dev.wisecp.com/en/the-client-area)

# Client List Interaction

https://dev.wisecp.com/en/client-list-interaction

The rows your preset writes and the script that searches, sorts and pages them meet through a fixed set of attributes.

## Overview

A record list is served in one of two modes, and the theme markup is the same in both. The mode is decided by the row count, not by you.

- **Client mode**: Below the threshold every row arrives in the first answer and the container has no `data-ajax`. Search, filter, sort and paging all run in the browser, against the attributes on each row.
- **Server mode**: Above it the container carries `data-ajax` and the script asks for one page at a time. The totals come from the answer instead of being counted locally.

> **Attributes are not optional in either mode**
> 
> Client mode reads them to decide what a filter matches. Server mode still reads them for sorting and for the visual meters. A row that omits one is not rejected and no error appears. It stops matching the filter it belongs to.

## Prerequisites

- Your preset already returns the row in `$row["html"]`.
- Your view has the `.list-rows` container with its three text attributes.
- Your theme has a `list.js`; copying another theme's is the usual start.

## Structure

Three nodes matter to the script, and it finds them by class or role, never by position.

- **.list-rows**: The container. Carries the data address in server mode and the wording in both.
- **.list-item**: One row, written by your preset, carrying the attributes below.
- **[data-role="empty"]**: The no-match block. It stays in the container and the script shows it. The zero-records block is a different thing, and it replaces the whole list.

## Reference

### Row attributes

Your preset writes these on the row element. Each one has a single reader, so leaving one out disables exactly that feature.

| Attribute | Value | What stops working without it |
| --- | --- | --- |
| `data-status` | status key | Status filter and the default sort order |
| `data-group` | type key | Type filter; the value has to match the option the toolbar offers |
| `data-flag` | segment key | The tile row above the list |
| `data-name` | visible name | Search, which reads this first |
| `data-search` | extra words | Search on anything not in the name, such as a domain or a number |
| `data-price` | raw amount | Sorting by price |
| `data-due` | timestamp | Sorting by due date and the remaining-term meter |
| `data-start` | timestamp | Sorting by newest and the term meter, which needs both ends |

Timestamps are seconds, not formatted dates: the script compares them as numbers and a formatted value reads as zero.

### Container attributes

- **data-ajax**: Written by the view from the table's own address, and present only in server mode. Its absence is the signal for client mode, so never write a placeholder value.
- **data-noun**: The key inside the script, in English, never shown to anyone.
- **data-txt-count · data-txt-nores · data-txt-noun**: The counter sentence, the no-match sentence and the word for the record. The first two hold the placeholders below.

The sentences are filled in, not concatenated, so the wording stays in the translator's hands:

- **_START_ · _END_ · _TOTAL_ · _NOUN_**: First row on the page, last row on the page, the filtered total, and the word from `data-txt-noun`. Every one of them may appear more than once.

### The server answer

In server mode the script asks the table's own address and expects the shape the panel uses:

```json
{
  "type": "partial",
  "total": 240,
  "total_filter": 18,
  "body": "<article class=\"list-item\" ...>...</article>..."
}
```

The request carries `page`, `perPage`, `search`, `filter[type]`, `filter[status]`, `order` and `direction`. The component builds all of them. A filter you add to the toolbar reaches the query only when the controller declared it.

## Example

```php
$row["html"] = '<article class="list-item"'
    . ' data-status="' . $status . '"'
    . ' data-group="'  . $type . '"'
    . ' data-flag="'   . $segment . '"'
    . ' data-name="'   . htmlspecialchars($name) . '"'
    . ' data-search="' . htmlspecialchars($domain . ' ' . $service["id"]) . '"'
    . ' data-price="'  . $amount . '"'
    . ' data-due="'    . strtotime($service["duedate"]) . '"'
    . ' data-start="'  . strtotime($service["cdate"]) . '">'
    . $body
    . '</article>';
```

## Pitfalls

> **A filter value has two ends**
> 
> The option in the toolbar and the attribute on the row have to hold the same value. When they drift apart the filter selects nothing. An empty result is a legitimate answer, so nobody is told that a mistake happened.

> **The two empty blocks are different**
> 
> The no-match block lives inside the container and the script shows it. The zero-records block replaces the toolbar, the list and the pager together, and the view decides that before the script ever starts. Writing only one leaves the visitor with an empty page and no way out.

> **Typing cancels the previous request**
> 
> In server mode each keystroke supersedes the one before it, and the older request is dropped on purpose. Do not add your own timer on top: two of them make the list flicker between answers.

> **Formatted values break sorting**
> 
> Dates go in as seconds and amounts as raw numbers. A value with a currency symbol or a thousands separator reads as zero and sinks to the bottom of every sort.

## Related Articles

- [Tables and Record Lists](https://dev.wisecp.com/en/tables-and-record-lists)
- [The Client Area](https://dev.wisecp.com/en/the-client-area)
- [Theme Translations](https://dev.wisecp.com/en/translating-a-theme)

# How Notification Templates Work

https://dev.wisecp.com/en/how-notification-templates-work

Every mail and text message the platform sends is built from three files on disk, one set per event and language. A template author owns those files the same way a theme author owns a view.

## Overview

A notification is not written in code. Code decides that something happened and hands over the facts; the wording, the markup and the subject line live under `templates/notifications`. An installation can change what a customer reads without a developer, and a module can ship its own wording.

The unit is the **event**, written as `group/name`. Each event owns three files per installed language: the mail body, the text message body and a small file carrying the subject. The mail body is a fragment, not a document.

Two things reach that template. The **resolver** for the group turns what the caller passed (an invoice row, a service id, a ticket) into named variables. The platform then adds the installation's own values: logos, colours, company details, the recipient's profile. The template only prints.

## Structure

### The File Layout

Three roots, each answering a different question. A subject line and a text message read the same whatever the mail looks like, so they live outside the designs entirely. The mail body belongs to the design that produces it.

```bash
templates/notifications/
├── content/                     # SHARED TEXT — one copy serves every design
│   └── en/invoice/
│       ├── invoice-created.json   # {"subject": "..."}
│       └── invoice-created.txt    # the text message body, no shell at all
├── themes/                      # DESIGN — one directory per installed design
│   ├── aurora/                  # the design that ships with the product
│   │   ├── header.html            # the outermost shell
│   │   ├── content.html           # the body slot, a single {$notifi_body} placeholder
│   │   ├── footer.html            # the closing shell
│   │   └── en/invoice/
│   │       └── invoice-created.html   # the mail body, a FRAGMENT of the shell
│   └── ledger/                  # a second design; may carry only some events
├── custom/                      # the operator's own edits, one tree per design
│   └── ledger/en/invoice/invoice-created.html
└── .htaccess                    # nothing in this tree is served over HTTP
```

Never build one of these paths by hand; the helpers below are the only place that knows the order. A design carrying no file for an event still sends the mail. The body comes from the base design; the shell stays its own.

- **View::notification_file()**: The mail body: `custom/{active}` → `themes/{active}` → `themes/aurora`. The first file that exists wins.
- **View::notification_content_file()**: The subject and the text message, from `content/{lang}` — no design takes part.
- **View::notification_write_path()**: Where an edit is saved: `custom/{active}` for any design other than the base one. Deleting that file restores the default.

### The Shell

A mail body is never sent on its own. The platform concatenates three files into one shell. It assigns the finished event body to `notifi_body` and builds the shell with the same variables. The shell resolves through the designs in the same order as the body. A design that ships none frames its mails with the base one.

- **header + content + footer**: Concatenated in that order and cached per language for the life of the process. The shipped `content.html` in each language is a single placeholder, so the frame is really the header and the footer.
- **the event body is a fragment**: The header opens a table and leaves it open; the event body continues it and the footer closes it. An event body that opens its own document produces markup no mail client will lay out.
- **text messages get no shell**: The concatenation is skipped for the text channel. A `.txt` body is delivered exactly as written, with no logo, footer or contact block.
- **the subject goes through the engine too**: The `.json` file's `subject` goes through the same engine and the same variables, so it can carry placeholders. It is read for the mail channel only.

### Groups and Resolvers

A group is a directory and a section of the configuration file at once. It decides which resolver builds the variables and which notification category the recipient's preferences are checked against.

| Group | Events shipped | Resolver | Preference category |
| --- | --- | --- | --- |
| `invoice` | 10 | `resolve_invoice_context` | invoices |
| `service` | 11 | `resolve_service_context` | product |
| `order` | 4 | `resolve_order_context` | product |
| `domain` | 13 | `resolve_domain_context` | domain |
| `user` | 36 | `resolve_user_context` | general |
| `user-tickets` | 8 | `resolve_ticket_context` | support |
| `admin-tickets` | 4 | `resolve_ticket_context` | support |
| `admin-messages` | 18 | `resolve_admin_message_context` | general |
| `sms-intl` | 3 | `resolve_sms_intl_context` | product |
| `newsletter` | 2 | none | not dispatched, see below |
| `license-transfer` | 3 | none | not registered, see below |

The last two rows are the two ways an event can exist outside the normal path. `newsletter` has configuration entries but no resolver, so a dispatch would answer `error`. Its templates are built directly and pushed onto the queue by the newsletter code, which supplies the variables itself. `license-transfer` is the opposite: three events sit on disk with no configuration section at all. They are sent through the low-level entry point, which falls back to mail-and-text-on when it finds no settings.

### The Delivery Path

From a single call to an actual message the chain is fixed, and every step can end it.

```bash
Notification::dispatch('invoice', 'invoice-created', ['entity' => $invoice])
  1. gate:notification.dispatch          a listener may veto      -> blocked
  2. read notifications/invoice/invoice-created                   -> disabled if absent or off
  3. run the group's resolver: entity -> variables + user_id      -> error if it returns nothing
  4. check the recipient's preference bitmask for the category    -> opted_out
  5. build the recipient list (owner, extra addresses, admins)    -> no_recipients if empty
  6. render per recipient, in THAT recipient's language and channel
  7. deliver now if the event is in the sync list, otherwise queue -> sent | queued
  8. write the in-app rows: the recipient's, and the admin copy
  9. action:notification.dispatched
```

Step 6 reads the template once per recipient, not once per dispatch. Two recipients of the same event can be reading in different languages and on different channels. The synchronous list at step 7 is short and holds security-critical events; everything else waits for the queue.

## Reference

### The Two Entry Points

```php
public static function dispatch(string $group, string $name, array $context = []): array;
public static function send(array $params): array|string|bool;
public static function get_recipients(string $group, string $name, array $context = []): array;
```

- **Notification::dispatch()**: The one to use. It honours the on/off switch, the recipient's preferences and the gate hook, and returns an array whose `status` says what happened.
- **Notification::send()**: The low-level one. It takes a single array and skips the resolver entirely, which means **you** supply the variables. Legitimate for a body with no template, a raw address list, or the queue worker delivering a row that is already built.
- **Notification::get_recipients()**: Runs the first half of a dispatch and stops, on the same arguments: returns who would be written to instead of writing. Useful while wiring up a new event.

### What a Dispatch Returns

Never a boolean. The array always carries `status`, and a successful one also carries `batch_id` and one entry per recipient under `items`.

| status | Meaning | Where it is decided |
| --- | --- | --- |
| `queued` | Rows written to the queue, delivery follows within a minute. | normal path |
| `sent` | Delivered inline, because the event is in the synchronous list or the caller forced it. | normal path |
| `blocked` | A listener vetoed the event. | the dispatch gate hook |
| `disabled` | The event has no configuration entry, or its switch is off. This is what an unregistered event answers. | the configuration file |
| `error` | The group has no resolver, or the resolver returned nothing (unknown invoice, deleted user). | the resolver map |
| `opted_out` | The recipient's preference bitmask excludes the group's category. | the recipient's profile |
| `no_recipients` | Nobody left after the channel switches and per-channel preferences were applied. | the recipient builder |

### The Template Engine

Which engine parses the files is an installation-wide setting, read from `options/notification-template-engine`. It ships as Smarty, and every template in the tree is written for it. Two other values exist: Twig, and a plain string replacement mode that understands nothing but `{name}`.

It is not the theme sandbox and does not behave like one:

- **no automatic escaping**: Values print exactly as they arrive, markup included. A theme escapes by default; this one does not. Use the escape modifier on anything a customer typed.
- **a restricted policy**: Thirteen platform classes are callable from a template and thirty nine language functions are allowed. Anything else is a compile error, and a compile error has consequences: see the pitfalls.

### The Queue

- **NotificationQueue::add()**: Takes one row that is already built: channel, recipient, subject, body, attachments, priority, batch id, optional schedule. Returns its id. The dispatch writes one row per recipient.
- **NotificationQueue::process()**: Delivers one row through the mail or text module. The panel's manual retry uses this same path, so a row that fails by hand fails the same way in the background.
- **the scheduled drain**: Runs every minute, reclaims rows stranded by a crash, and hands out at most two hundred per tick. Because the body was built at dispatch time, editing a template does not change a message already waiting in the queue.

## Pitfalls

> **A missing language file is silence, not an error**
> 
> The path is built from the recipient's language with no fallback. If the file is not there nothing is produced and the recipient is skipped. The dispatch still reports success for everyone else. Measured in the shipped tree: eight events exist in English and are absent in German.

> **One bad tag leaves the whole file unrendered**
> 
> Compilation is all or nothing, and a failure is caught and answered with the raw source. A stray placeholder does not fail on its own line. Every other placeholder in the file ships unrendered too, and the customer receives a message full of visible braces. The warning goes to the log, not to the operator.

> **Under the base design there is no override layer**
> 
> The base design ships with the product. An edit made while it is active is written straight over the shipped file, and the next update replaces it. Any other design saves into `custom/`, which an update cannot touch.

> **Files alone are not an event**
> 
> Adding three files gives you nothing. Without a configuration entry the switch cannot be found and the dispatch answers `disabled`. Without a resolver case the variables never exist, and without a label the panel lists the raw key. The trio is one of several places that have to agree.

## Related Articles

- [Writing an Email Template](https://dev.wisecp.com/en/writing-an-email-template)
- [Writing an SMS Template](https://dev.wisecp.com/en/writing-an-sms-template)
- [Notification Template Variables](https://dev.wisecp.com/en/notification-template-variables)
- [Writing a Mail Module](https://dev.wisecp.com/en/writing-a-mail-module)
- [Writing an SMS Module](https://dev.wisecp.com/en/writing-an-sms-module)
- [Writing a Hook Listener](https://dev.wisecp.com/en/writing-a-hook-listener)

# Writing an Email Template

https://dev.wisecp.com/en/writing-an-email-template

Add a new mail to the platform by writing three small files and registering them in four other places. None of those steps can be skipped.

## Overview

What you author for a mail is a **fragment** and a subject line. The document around it, the logo, the footer, the contact block and the colours all come from the shared shell. A template that opens its own document tag fights the frame.

The rest of the work is registration. Without a configuration entry the event has no switch and is reported as disabled. Without a resolver case the variables the body prints do not exist, and without a label the panel lists the raw key.

## Prerequisites

- **a group that has a resolver**: Put the event in an existing group whenever you can. A new group needs a resolver and an entry in the resolver map. A group without one answers every dispatch with an error.
- **the engine setting**: Templates are written for the configured engine, and the shipped tree is Smarty. Check `options/notification-template-engine` before you copy placeholder syntax from somewhere else.
- **every installed language**: There is no fallback between languages. Ship the trio in each language the installation has, or the recipients reading in the missing one get nothing at all.
- **a way to read what was sent**: A sandbox mail module writes each outgoing message to disk instead of opening a connection. It is the fastest way to inspect a body, and the only safe one on an installation with real addresses in it.

## Structure

### The Fragment Contract

The shell's header opens the outer table and leaves it open; the footer closes it. Your body sits between the two, which fixes what it may begin and end with.

```bash
{lang}/header.html   opens the page table, prints the logo and the site title, leaves it OPEN
      ↓
{lang}/content.html  a single {$notifi_body} placeholder
      ↓
   YOUR FILE          one or more table rows: it continues the open table
      ↓
{lang}/footer.html   the contact block, the links, the company details, then CLOSES everything
```

- **rows, not a page**: Begin with a table row and end with one. No document tag, no head, no body tag, and no stylesheet: the frame already opened all of them.
- **presentation goes inline**: Mail clients drop stylesheets, so every shipped body carries its styling as an attribute on each element. Tables are used for layout for the same reason.
- **colours come from the installation**: Three are injected: `theme_color1`, `theme_color2` and `theme_text_color`. Each holds the digits **without** the leading marker, so a template writes the marker itself and the value after it: `bgcolor="#{$theme_color1}"`. Never hard-code a brand colour.
- **logic can hide in comments**: The shipped templates wrap loops and conditions in HTML comments so a visual editor does not mangle them. The engine still executes them; only the comment markers survive into the output.

### The Three Files

| File | Read for | Contains |
| --- | --- | --- |
| `{name}.html` | mail | The body fragment. Wrapped in the shell before sending. |
| `{name}.json` | mail | `{"subject": "..."}`. Built with the same variables, so it may carry placeholders. |
| `{name}.txt` | text message | The text body, sent with no shell. Required even if the event never uses the text channel, because the file is what the panel edits. |

## Walkthrough

### 1. Write the Trio

1. Pick the group and a lower-case, dash-separated event name. The pair is the identity of the event everywhere else.
2. Create `{name}.html`, `{name}.json` and `{name}.txt` under every installed language directory.
3. Copy the closest shipped body rather than writing table markup from scratch. The spacing and the colour variables are already correct there.
4. Keep the placeholders in the engine's syntax. A placeholder written in another engine's syntax is not an error you will see. It is a compile failure that leaves the entire file unrendered.

### 2. Register the Event

1. Add an entry under the group in `coremio/configuration/notifications.php`. The keys are listed in the reference below.
2. Set `status` to 1 and turn on the channels the event actually uses. Everything left at 0 stays silent.
3. Reload and confirm the event now appears in the panel's template list. If it does not, the entry is under the wrong group key.

### 3. Feed the Variables

1. Open the group's resolver in `coremio/helpers/notification.php` and add a case for the event name.
2. Set only what the group's baseline does not already provide. The invoice, service, order, domain and ticket resolvers each build a full set before the switch runs.
3. If the mail must reach an address that is not the account's, override the recipient inside the case. Doing it there rather than at the call site keeps every existing caller behaving as before.
4. Call the dispatch and read the returned `status`. An `error` here means the resolver returned nothing, usually because the entity could not be loaded.

### 4. Decide When It Goes

1. By default the message is queued and leaves within a minute.
2. An event the customer is *waiting on* (a code, a link, an invitation) is delivered inline instead. Add its name to the `$sync_notifications` list, in the same file as the resolver.
3. A single call can override that list without changing it: pass `'_sync' => true` in the context. Use the list for an event that is always urgent, the context key for one caller that is.
4. Inline delivery happens in the request, so it also fails in the request. Only put an event there when the delay is worse than the risk.

### 5. Make It Previewable and Named

1. Add a human label for the event key to the admin notification language files, in every language. Without it the panel prints the raw key in its list.
2. Add sample values for the event's own variables to the preview operation in `coremio/operations/AdminNotifications.php`. The preview does not run the resolver, so anything it is not given comes out as an empty string.
3. Open the preview. It is also the only place a compile error is shown rather than logged.

### 6. Verify It

1. Trigger the real flow, not the preview.
2. Read the delivered message from the sandbox mail module's output directory and check that no placeholder survived into the text.
3. Repeat with an account in the other language. That catches a trio you created in only one language directory.

## Reference

### The Configuration Entry

One array per event, under its group. Only `status` and the four channel switches decide whether anything is sent. The rest shape who receives it and what the panel shows.

```php
return [
    'notifications' => [
        'invoice' => [
            'invoice-created' => [
                'variables'   => '{invoice_idn},{invoice_total},{invoice_payment_link},{items}',
                'emails'      => '',
                'phones'      => null,
                'departments' => ['4'],
                'status'      => 1,
                'user-mail'   => 1,
                'admin-mail'  => 0,
                'user-sms'    => 1,
                'admin-sms'   => 0,
            ],
        ],
    ],
];
```

- **status**: The master switch. 0, or a missing entry, makes every dispatch answer `disabled` without touching the template at all.
- **user-mail · user-sms**: Whether the account holder is written to on that channel. A text message also needs a mobile number on the account; without one the recipient drops out of the list.
- **admin-mail · admin-sms**: Whether staff are written to as well. The recipients are resolved from the departments below, plus the addresses in the two fields after them. They fall back to the root administrator when the switch is on but nothing resolved.
- **emails · phones**: Comma separated extra staff recipients, added on top of the departments. Read only when the matching admin switch is on.
- **departments**: Support department ids whose staff receive the staff copy. An array of id strings, and the usual way to route an event to the right team instead of to everyone.
- **user-notification**: Whether the recipient also gets an in-app row. Absent means on, which is why existing events keep their bell after a new key is introduced.
- **admin-notification**: Whether staff get an in-app copy. When the key is absent the answer is derived. It is on for a short list of events staff must act on, and otherwise follows `admin-mail`.
- **send-pdf**: Invoice group only. Absent means on, so an invoice mail attaches the generated document unless the entry says otherwise.
- **variables**: Editor metadata, nothing more. It fills the badge list the operator sees while editing and is never consulted when a message is built. An out-of-date list misleads the operator but breaks nothing.

### Building a Message Directly

The dispatch calls this once per recipient. Call it yourself only when there is no dispatch to make. That means a raw address list, or anything where you already hold the variables.

```php
public static function notifications(
    $type = 'mail',            // 'mail' | 'email' (alias) | 'sms' — picks .html or .txt
    $template_name = '',       // "group/name", no extension
    $content = '',             // pass a body to render THAT instead of reading the file
    $variables = [],           // your variables; the platform's are added on top
    $lang = '',                // empty falls back to the currently selected language
    $user = 0                  // a user id fills every user_* variable from the account
): array;                      // ['content' => ..., 'subject' => ...]; EMPTY array on failure
```

- **the return is the failure signal**: An empty array means the file was not found or was empty. There is no exception and no log line, so a caller that does not check it sends nothing and reports success.
- **what gets added on top**: Logos, colours, site title, company details, contact link, current year, and the whole recipient profile when a user id was passed. Your own variables win over none of these, so do not reuse their names.
- **building is not sending**: The call returns a finished body; nothing has left the installation. Hand the result to the queue, or to the low-level sender, depending on whether it may wait.

## Example

A complete event, in the order the platform reads it.

```json
{"subject":"{$service_name} has reached {$quota_percent}% of its quota"}
```

```smarty
<!-- the shell left the outer table open: continue it, do not reopen it -->
<tbody>
<tr>
  <td>
    <p>Dear <strong>{$user_greeting_name}</strong>,</p>
    <p>{$service_name} has used {$quota_percent}% of its allowance on {$service_domain}.</p>
  </td>
</tr>

<!-- Conditions and loops are wrapped in comments so a visual editor leaves them alone.
     The engine executes them anyway; only the comment markers reach the recipient. -->
<!-- {if $quota_percent >= 100} -->
<tr>
  <td><p>New uploads are refused until the allowance is raised.</p></td>
</tr>
<!-- {/if} -->

<tr>
  <td>
    <!-- theme_color1 holds the digits only, so the marker is written here.
         A button is a nested table with bgcolor: mail clients drop CSS backgrounds. -->
    <table border="0" cellpadding="0" cellspacing="0" bgcolor="#{$theme_color1}">
      <tbody><tr>
        <td align="center"><a href="{$service_detail_link}">Open the service</a></td>
      </tr></tbody>
    </table>
  </td>
</tr>
</tbody>
```

```php
// coremio/helpers/notification.php, inside resolve_service_context().
// service_name, service_domain and service_detail_link are already built above
// the switch; only what is specific to this event belongs inside it.
switch ($name) {
    case 'acme-quota-reached':
        $variables['quota_percent'] = (int) ($context['percent'] ?? 0);
        break;
}
```

```php
// 'entity' is what every resolver accepts; each group also takes its own alias
// ('invoice', 'service', 'order', 'ticket'). A row or an id both work.
$result = \Notification::dispatch('service', 'acme-quota-reached', [
    'entity'  => $service,
    'percent' => 92,
]);

// Never treat the return as a boolean. 'disabled' means the operator turned the
// event off and is not a failure; 'error' means the resolver could not build it.
if (($result['status'] ?? '') === 'error')
    Logger::getInstance()->warning('quota notice not built', [
        'service_id' => $service['id'],
        'message'    => $result['message'] ?? '',
    ]);
```

## Pitfalls

> **Nothing is escaped for you**
> 
> This engine prints values exactly as they arrive, markup included, unlike the theme engine which escapes by default. Anything a customer or a third party typed needs the escape modifier before it reaches a mail body. A ticket message above all.

> **Some variables are real credentials**
> 
> The service variable set includes the decrypted account password and the server login. Printing it puts a working credential in an inbox and in the mail log forever. Link to the service page instead, and reserve the credential for the one message whose entire purpose is delivering it.

> **A preview that looks right proves less than it seems**
> 
> The preview builds its own sample values and never calls the resolver. A body full of variables nobody supplies still looks perfect there. Only a real dispatch proves the wiring.

> **One event, several bodies**
> 
> The template is read once per recipient, in that recipient's own language. A staff copy of a customer event comes from the same file with the same wording. Avoid a sentence that only makes sense addressed to the customer.

## Related Articles

- [How Notification Templates Work](https://dev.wisecp.com/en/how-notification-templates-work)
- [Notification Template Variables](https://dev.wisecp.com/en/notification-template-variables)
- [Writing an SMS Template](https://dev.wisecp.com/en/writing-an-sms-template)
- [Writing a Mail Module](https://dev.wisecp.com/en/writing-a-mail-module)
- [Translations and Language Files](https://dev.wisecp.com/en/translations-and-language-files)

# Writing an SMS Template

https://dev.wisecp.com/en/writing-an-sms-template

The text message body of an event is a single plain file with no shell around it. That makes it the shortest template to write, and the easiest one to fill with markup by accident.

## Overview

Every event carries a `.txt` alongside its mail body, and that file is the whole message: nothing is prepended or appended. Whatever the file produces is what arrives on the handset.

It is the same engine, the same variables and the same event as the mail. The differences all follow from the channel. There is no frame to inherit context from and no markup to lean on. An account without a mobile number is not a recipient at all.

## Prerequisites

- **an event that already exists**: A text body is one third of an event, not an event of its own. The configuration entry, the resolver case and the panel label are shared with the mail, so wire those first.
- **the text channel switched on**: The file is read only when `user-sms` or `admin-sms` is set in the event's configuration entry. With both at 0 the file is inert no matter what it contains.
- **a mobile number on the account**: The recipient's number and its dialling code come from the account profile.
- **a sandbox text module**: A driver writes each message to disk instead of calling a gateway. That lets you read the exact delivered characters, including the ones a browser would have hidden.

## Structure

### No Shell, No Markup

The mail path concatenates a header, a body slot and a footer first. The text path skips that entirely, so the file stands alone.

```smarty
Your service has been activated.

Service Information
------------------------------------
{$service_name} - {$service_amount}

Service Details
------------------------------------
{$service_detail_link}
```

- **say who you are**: The only branding is the sender name the gateway shows and whatever the text says. A message that opens with "your service" and never names the site reads like a stranger's.
- **plain text, and nothing escapes it**: The engine does not escape and does not strip. A variable holding rich text prints its tags verbatim into a message that cannot display them.
- **links are long**: Detail and payment links are absolute and carry tokens, so one of them can be most of the message. Put it last, on its own line, and do not wrap it in punctuation a handset will swallow into the address.

### Which File the Channel Reads

| Channel | Body file | Shell | Subject |
| --- | --- | --- | --- |
| mail | `{name}.html` | header, content slot, footer | `subject` from the `.json`, with variables filled in |
| text message | `{name}.txt` | none | none: the `.json` is not read for this channel |

The engine understands a `title` key in the `.json` and would return it for the text channel. The dispatcher does not carry it further, and no shipped template declares one.

## Walkthrough

### 1. Write the Body

1. Open the event's `.txt` in every installed language directory. It was created with the trio; if it is missing, the text channel has nothing to read.
2. Write the message as a customer would want to receive it: what happened, to which service, and where to look. One or two short lines and a link.
3. Use the same placeholder syntax as the mail body. The engine is the same, so a syntax error fails the same way and leaves the whole file unrendered.
4. Prefer the short variables. A description field or a ticket message is not sized for this channel even when it fits.

### 2. Enable the Channel

1. Set `user-sms` to 1 in the event's configuration entry for a message to the account holder. Set `admin-sms` for a copy to staff.
2. Staff numbers resolve from the event's departments plus the entry's own phone list. They fall back to the root administrator when the switch is on and nothing else resolved.
3. Leave both at 0 for any event whose text version is not worth a charge. Most events ship that way on purpose.

### 3. Keep It Plain

1. Strip anything that might carry markup at the point of printing: a ticket reply, a description, an operator-written note.
2. Watch the alphabet: one character outside the basic set changes how the whole message is counted.
3. Keep the wording independent of the mail. The text version is the message for a reader who will never open the mail.

### 4. Verify the Send

1. Trigger the real flow with an account that actually has a mobile number, and check the returned status. A `no_recipients` answer with the mail arriving normally means the number is what is missing.
2. Read the delivered file from the sandbox module's output directory. It records the sender name, the destination and the exact body, so trailing whitespace and stray tags become visible.
3. Repeat in the other language. A text body is small enough that a missing translation is easy to overlook and produces silence rather than an error.

## Reference

### Length and Encoding

The notification path does not measure, split or truncate a text body. It reads the file and hands the string to the module. Any limit is the gateway's, and any cost is per message part.

That arithmetic exists in the platform, in the credit-funded international messaging panel rather than in notification templates. It is still the right model to size a template against.

| Encoding | When it applies | Single message | Per part once split |
| --- | --- | --- | --- |
| basic alphabet | every character is in the standard messaging alphabet | 160 | 153 |
| wide | one single character outside it, anywhere in the message | 70 | 67 |

- **Sms::analyze_message()**: Returns the encoding, the counted length, the number of parts and whether it had to be cut. Called by the panel's quoting path, never by a notification, so nothing here trims a template for you.
- **some characters count twice**: In the basic alphabet a handful of symbols are transmitted as two units. In the wide encoding an emoji counts as two. A body sized by eye is routinely one unit over.
- **the part ceiling is configurable**: The panel refuses to go beyond a configured number of parts, six by default. It cuts the text rather than letting the gateway drop it. A notification has no such guard.

### What the Module Receives

A text module is handed a body, a destination and a dialling code, and is asked to submit. The notification path uses only this much of its surface.

```php
$sms = new $smsModule();

// The already-rendered body. Passing a template name here instead is the module's
// own shortcut and is what the low-level sender uses; the dispatch never does.
$sms->body($item['body']);

// One recipient, or a list. The dialling code is a separate argument because the
// account stores it separately from the number.
$sms->addNumber($item['recipient'], $item['recipient_cc']);

$sent = $sms->submit();
if (!$sent) $error = $sms->getError();
```

- **the sender name is the module's**: It is set from the module's own configuration when the module is constructed, and the delivery path never overrides it. A template cannot change who the message appears to be from.
- **the dialling code travels separately**: Number and code are two fields on the account and two arguments here. A module that concatenates them itself decides its own format. That is why a number that works on one gateway can fail on another.
- **a successful send is logged with its body**: The message log keeps the sender name, the text and the destinations. Useful for support, and a reason not to print anything secret into a text body.

## Example

The text half of an event, next to the mail half it shares everything else with.

```smarty
{$company_name}: {$service_name} has used {$quota_percent}% of its quota.

{* A condition costs nothing in the output, so the message stays one line longer
   only when it has to. Comments like this one are stripped entirely. *}
{if $quota_percent >= 100}New uploads are refused until the allowance is raised.
{/if}
{$service_detail_link}
```

```smarty
{* {$admin_reply} is the reply as it was stored: line breaks were turned into
   newlines, but every other tag the editor produced is still in there. *}
Ticket {$ticket_num} has been answered.

{$admin_reply|strip_tags|truncate:120}

{$ticket_link}
```

```php
// One call. Which files are read depends on the recipient's channel, and both
// channels can be produced for the same person in the same dispatch.
$result = \Notification::dispatch('service', 'acme-quota-reached', [
    'entity'  => $service,
    'percent' => 100,
]);

// Every recipient row says which channel it was written for, so a missing text
// message is visible here rather than only in the gateway's report.
foreach ($result['items'] ?? [] as $item)
    if (($item['channel'] ?? '') === 'sms')
        Logger::getInstance()->info('quota notice queued as text', [
            'recipient' => $item['recipient'] ?? '',
            'queue_id'  => $item['queue_id'] ?? 0,
        ]);
```

```php
// Overrides the four channel switches for this call only and drops the staff copy.
// It also bypasses the recipient's category preference, so reserve it for a message
// the account explicitly asked for, such as a verification code.
$result = \Notification::dispatch('user', 'gsm-activation', [
    'entity'          => $userId,
    'code'            => $code,
    '_force_channels' => ['sms'],
    '_sync'           => true,   // deliver inline: the customer is waiting on this
]);
```

## Pitfalls

> **Rich text reaches the text channel intact**
> 
> Ticket messages are stored as the editor produced them; only the line breaks are normalised on the way out. Printing one into a text body sends the tags along, and the recipient pays for the characters. Strip at the point of printing rather than trusting the variable.

> **No number, no recipient, no error**
> 
> An account without a mobile number never enters the recipient list. The mail still goes out, the dispatch still reports success, and the text message that was never sent leaves no trace. When the text half seems missing, check the account before the template.

> **One accented letter more than halves the room**
> 
> The counting switches to the wide encoding as soon as a single character falls outside the basic alphabet. The limit drops from a hundred and sixty to seventy. A translation that reads as long as its English original routinely costs twice as many parts.

> **An empty body is a decision, so make it one**
> 
> A file that produces nothing is skipped silently. That is exactly right when the event has no text version, and indistinguishable from a mistake when it does. If an event should not send text, turn the channel off in its configuration entry. An empty file is not a way to say it.

## Related Articles

- [How Notification Templates Work](https://dev.wisecp.com/en/how-notification-templates-work)
- [Writing an Email Template](https://dev.wisecp.com/en/writing-an-email-template)
- [Notification Template Variables](https://dev.wisecp.com/en/notification-template-variables)
- [Writing an SMS Module](https://dev.wisecp.com/en/writing-an-sms-module)

# Notification Template Variables

https://dev.wisecp.com/en/notification-template-variables

Exactly what a notification template can print, and which of the three layers each name comes from. Also which of them silently overwrite anything you set yourself.

## Overview

A template receives one flat set of names. It is assembled from three sources in a fixed order. Knowing which source a name comes from answers the two questions that actually come up. Why did a placeholder come out empty, and why was the value you passed ignored?

- **the installation layer**: Logos, colours, company details, contact links, the current year. Added to every template of every event, whether the event asked for them or not.
- **the recipient layer**: The account the message is being built for. Present only when a user id reached the build call. That is why the same name is filled in one flow and empty in another.
- **the event layer**: What the group's resolver built from the entity, plus whatever the resolver's case for that specific event added. This is the layer you extend when you add an event.

## Structure

### Assembly Order

The event layer is assembled first and the other two are laid over it, not under it. That is the source of nearly every surprise in this article.

```bash
the resolver          entity -> the event's variables      (yours)
      ↓
View::notifications   adds template_name and template_type
      ↓
variables_handler     adds the recipient block  (user id > 0 only)
                      adds the installation block  (always)
      ↓                  ^ both OVERWRITE what is already there, with four exceptions
the engine            renders the body with the merged set
      ↓
                      the rendered body becomes notifi_body
      ↓
the engine            renders the shell, then the subject, with the same set
```

- **four names defer to you**: The recipient's display names are only filled when the resolver did not already set them. Those four are the full name, the first name, the surname and the greeting name. Every other name in both platform layers is written unconditionally.
- **no user id, no recipient block**: The whole recipient layer is skipped when the build is called with a user id of zero. A message aimed at an address that is not an account leaves every one of those names empty.
- **the body is a variable too**: The finished event body is assigned as a variable and the shell prints it. That name is meaningful in the shell files only. Printing it inside an event body prints nothing, because it does not exist yet.

### Declared Versus Injected

Two lists exist and they are not the same list. One drives the badges an operator sees while editing a template; the other is what actually arrives. Neither is a filter: a name absent from both still prints if the resolver set it.

| List | Built by | Used for | Effect at build time |
| --- | --- | --- | --- |
| the platform sets | `Notification::variables()` | the editor's badge list | none |
| the group baseline | `Notification::group_variables()` | the editor's badge list | none |
| the entry's own list | the `variables` key in the configuration entry | the editor's badge list | none |
| what is actually injected | the resolver, then the build call | building the message | everything |

Because the first three are documentation rather than mechanism, they drift. Measured against the shipped code: the recipient set declares seventeen names while nineteen are injected. The installation set declares one name that the build call adds separately. The gaps are listed with the tables below.

## Reference

### The Signatures

```php
// coremio/helpers/notification.php
// $type is 'system' | 'user'. Anything else returns $variables with the braces
// stripped, which is how the configuration entry's CSV becomes a badge list.
public static function variables(string $type = '', array $variables = []): array;

// The group baseline the editor shows: 'invoice' | 'order' | 'service' | 'domain'
// | 'user-tickets' | 'admin-tickets'. Every other group returns an empty array.
public static function group_variables(string $group = ''): array;

// coremio/classes/View.php
public static function notifications($type = 'mail', $template_name = '', $content = '',
                                     $variables = [], $lang = '', $user = 0): array;

// Merges the two platform layers into $variables, then renders $str IN PLACE.
// $str is by reference: it is both the template source and the result.
public static function variables_handler($type, $user_id = 0, $variables = [], &$str = '', $lang = ''): void;

// coremio/classes/TemplateEngine.php
// $engine is 'smarty' | 'twig' | 'none'. Returns the ORIGINAL string on any
// compile error, so a failure looks like a template that did nothing.
public static function render_notification($engine, $content, $variables = []): string;
```

- **Notification::variables()**: The two platform sets by name. With any other argument it becomes a small utility that strips the braces off a list. That is how the configuration entry's comma separated value turns into badges.
- **Notification::group_variables()**: The group's baseline by name. The answer for a group without one is an empty array, not an error. An unbaselined group shows a short badge list rather than a broken editor.
- **View::notifications()**: Reads the files, merges the layers, builds body, shell and subject. Everything in this article happens inside one call to it.
- **View::variables_handler()**: Where both platform layers are actually written, and where the overwrite rule lives. It works in place through its by-reference argument and returns nothing.
- **filter:notification.render_variables**: Runs once per recipient, immediately before that recipient's body is built. The way to add a name without touching a resolver, and the only one that can vary the value per recipient. The variable set arrives by reference: write into it, because the return value is ignored.
- **filter:notification.template_merge_fields**: Editor time only. Adds a name to the badge list an operator sees while editing, and has no effect on the message whatsoever. The list arrives by reference too; append to it, and the return value is ignored.

### The Installation Layer

Twenty two names are declared and all of them are injected every time. One more is added by the build call and never declared.

| Name | Holds | Worth knowing |
| --- | --- | --- |
| `website_url` | The installation address. | Rewritten to a secure scheme when the installation forces one. |
| `website_domain` | The host on its own, with no scheme. | For prose, not for building a link. |
| `website_title` | The site title, in the message language. | Comes from the website translations, not from the company details. |
| `company_name` | The legal name. | Falls back to the first line of the information block when unset. |
| `website_infos` | The multi-line information block. | Line breaks are converted to markup for mail and left alone for text messages. |
| `website_address` | The postal address, per language. | A language specific address overrides the general one. |
| `website_emails` | Public addresses, joined into one string. | Already a string, not a list: it cannot be looped. |
| `website_phones` | Public numbers, joined into one string. | Same shape as the addresses above. |
| `website_contact_url` | The contact page, in the message language. | Localised per recipient, so it differs between two rows of one dispatch. |
| `support_link` | The ticket creation page. | Pair it with the flag below before printing it. |
| `is_enable_support` | Whether ticketing is on. | A boolean, for a condition. The shipped footer hides its whole contact block behind it. |
| `website_header_logo` | The site's light logo. | An absolute address. |
| `website_footer_logo` | The site's dark logo. | An absolute address. |
| `notifi_header_logo` | The mail specific logo. | Falls back to the site logo. A vector file is swapped for a raster one when it exists, because mail clients cannot draw vectors. |
| `notifi_footer_logo` | The mail specific dark logo. | Same fallback and the same substitution. |
| `theme_color1` | The primary colour. | Digits only, with no leading marker: the template writes the marker itself. |
| `theme_color2` | The secondary colour. | Same shape. |
| `theme_text_color` | The body text colour. | Same shape. |
| `social_links` | A list of social profiles. | A list of rows, see the shapes below. Empty when none are configured, so guard the loop. |
| `current_year` | The year, four digits. | For a copyright line, so it never goes stale. |
| `template_name` | The event, as `group/name`. | Set by the build call itself. |
| `notifi_body` | The finished event body. | Declared here, but it only exists once the body is built. Usable in the shell, empty in an event body. |
| `template_type` | The channel: mail or sms. | **Injected but not declared**, so it never appears in the editor's badge list. Lets one shared partial branch on the channel. |

### The Recipient Layer

Present only when the build was given a user id. Seventeen names are declared; two more are injected without being declared.

| Name | Holds | Worth knowing |
| --- | --- | --- |
| `user_greeting_name` | The company name if there is one, otherwise the full name. | The right one to open a message with. **Deferred**: a resolver that already set it wins. |
| `user_full_name` | First and last name. | **Deferred** to the resolver. |
| `user_name` | First name. | **Deferred** to the resolver. |
| `user_surname` | Last name. | **Deferred** to the resolver. |
| `user_company_name` | The company name, empty for an individual. | Overwritten unconditionally. |
| `user_email` | The account address. | The address on the account, not necessarily the one this copy is going to. |
| `user_phone` | The phone, prefixed when present. | Null rather than empty when the account has none. |
| `user_id` | The account id. | Useful in a reference line, meaningless to a customer on its own. |
| `user_group` | The customer group name. | Empty when the account is in no group. |
| `user_country` | Country name from the primary address. | Null when there is no address on file. |
| `user_city` | City. | Same source and same caveat. |
| `user_state` | State or province. | Same source and same caveat. |
| `user_address` | Street address. | Same source and same caveat. |
| `user_zipcode` | Postal code. | Same source and same caveat. |
| `user_ip` | The address recorded on the account. | Registration time, not the address of whatever triggered this message. |
| `user_login_link` | The sign-in page, in the message language. | Localised per recipient. |
| `user` | The whole account row plus its address. | A collection, see the shapes below. Reach for a named variable first. |
| `admin_login_link` | The panel sign-in page. | **Injected but not declared.** For staff copies; do not print it in a customer facing body. |
| `user_pass` | Five asterisks. | **Injected but not declared**, and a mask rather than a value. It exists so an older template that prints it shows a mask instead of a blank. |

### The Event Layer, by Group

What the group's resolver builds before it looks at the event name. Only six groups declare a baseline; the rest build their set entirely inside the event's own case.

- **invoice**: `invoice, invoice_idn, invoice_payment_link, invoice_subtotal, invoice_total, invoice_tax_rate, invoice_tax, invoice_date_created, invoice_date_due, invoice_date_paid, invoice_date_taxed, invoice_payment_method, invoice_remaining_day, invoice_delayed_day, invoice_refund_date, invoice_cancelled_date, legal_invoice_download_link, items`. Amounts arrive already formatted with their currency symbol, so do not format them again. The last four are set by their own events only.
- **order**: `order, order_id, order_number, order_name, order_amount, order_currency, order_payment_method, order_status, order_detail_link, order_date_created, order_date_start, order_date_end, order_period, order_period_unit, order_group_name, order_category_name, order_services_summary, items`. The item collection has a different shape from the invoice one: one row per purchased product, with its add-ons nested.
- **service**: `service, service_id, service_order_id, service_name, service_type, service_module, service_subscription_identifier, service_period, service_period_unit, service_period_time, service_cycle, service_amount, service_date_created, service_date_start, service_date_end, service_detail_link, service_group_name, service_category_name, service_domain, service_ip, service_requirements, service_addons, service_server_ip, service_server_hostname, service_server_port, service_ns1, service_ns2, service_ns3, service_ns4, service_username, service_password, service_assigned_ips`. The last group of names are live access details, including a decrypted password: see the pitfalls.
- **domain**: The entire service set above, plus `domain, day, grace_days, redemption_days, redemption_fee, days_past_due, domain_transfer_code, reason`. A domain is a service, so its templates can print any service name as well.
- **user-tickets and admin-tickets**: `ticket, ticket_id, ticket_num, ticket_link, ticket_subject, ticket_department, ticket_service, ticket_status, ticket_priority, ticket_admin_name, ticket_date, ticket_last_reply_date, ticket_assigned_by_admin, user_last_message, admin_last_message, user_reply, admin_reply`, plus the entire service set when the ticket is attached to one. The link differs by group: the staff set points into the panel, the customer set into the portal.
- **user, admin-messages, sms-intl**: No baseline at all. Each event's case builds exactly what its template needs. Two events in the same group can share almost no names.

### Collections and Their Keys

Four of the names are not strings. Printing one directly shows nothing useful; these are for a loop or for a keyed read.

| Name | Shape | Keys on each row |
| --- | --- | --- |
| `items` (invoice) | rows, one per invoice line | `id, owner_id, user_id, user_pid, description, quantity, amount, total_amount, currency, rank, amountF, service_id, service_domain, service_ip, service_group_name, service_category_name`. A renewal line also carries `service_type, service_old_duedate, service_new_duedate`. |
| `social_links` | list of rows | `name, url, icon`. The shipped shell builds its image file name from the lower-cased name. |
| `user` | one row | The account columns, plus `address` holding the primary address. Every field worth printing already has a named variable. |
| `service_addons` and `service_requirements` | lists of rows | The stored rows as they are, unformatted. Loop them only when the template really has to itemise; there is no ready formatting behind them. |

A formatted amount ends in a capital F on the invoice item rows. The bare key is the raw number; the one ending in F is the string to print. Getting that pair the wrong way round prints an unformatted figure with no currency on it.

### Per Event Extras

Beyond the baseline, a resolver's case adds what only that event needs. A representative sample, with what the caller has to pass for each.

| Event | Adds | Fed by the context key |
| --- | --- | --- |
| `user/two-factor-verification` | `code` | `code` |
| `user/email-activation` | `activation_code, activation_link` | `code`, `activation_link`, optional `to_email` to redirect the message |
| `user/email-changed` | `old_email, new_email` plus the device block | `old`, `new`, optional `to_email` |
| `user/password-changed` | `reset_password_link` plus the device block | `reset_link` |
| `invoice/invoice-reminder` | `invoice_remaining_day` | `remaining_day` |
| `invoice/invoice-overdue` | `invoice_delayed_day` | `delayed_day` |
| `invoice/invoice-auto-payment-failed` | `error_message, card_ln4` | `error_message`, `card_ln4` |
| `admin-messages/backup-completed` | whatever the caller built | `variables`, forwarded verbatim |

The device block referred to above is a small fixed set added to the security events. It carries `browser`, `platform`, `ip`, `location_country`, `location_city` and `date`. The two location names are placeholders and are always empty, so a template that prints them prints nothing.

## Example

All three layers in one fragment, and then the two supported ways to add a name of your own.

```smarty
{* recipient layer *}
Dear {$user_greeting_name},

{* event layer: already formatted with its currency, so it is printed as-is *}
Invoice {$invoice_idn} for {$invoice_total} is due on {$invoice_date_due}.

{* event layer, a collection: amountF is the formatted string, amount is the number *}
{foreach from=$items item=item}
- {$item.description} {$item.amountF}
  {if $item.service_domain != ""}({$item.service_domain}){/if}
{/foreach}

{* installation layer, guarded because the list can be empty *}
{if $is_enable_support}Questions: {$support_link}{/if}
{$company_name} {$current_year}
```

```php
// coremio/helpers/notification.php, in the group's resolver.
// Do not reuse a platform name: user_email and website_url are written after
// this runs and would overwrite whatever you put there.
switch ($name) {
    case 'acme-quota-reached':
        $variables['quota_percent'] = (int) ($context['percent'] ?? 0);
        $variables['quota_limit']   = Money::formatter_symbol(
            (float) ($context['limit'] ?? 0),
            (int) ($service['amount_cid'] ?? 0),
        );
        break;
}
```

```php
// Runs once per recipient, right before that recipient's body is rendered, so it
// can also vary the value per recipient. The first argument is by reference.
Hook::add('filter:notification.render_variables', 1, function (&$variables, $group, $name, $recipient) {
    if ($group !== 'service') return;

    $variables['acme_portal_link'] = 'https://portal.example.com/s/' . (int) ($variables['service_id'] ?? 0);
});

// Editor only: puts the name in the badge list the operator sees while editing.
// It changes nothing at render time, so both listeners are needed for a name that
// is meant to be discoverable as well as printable.
Hook::add('filter:notification.template_merge_fields', 1, function (&$fields, $group, $name) {
    if ($group === 'service') $fields[] = 'acme_portal_link';
});
```

## Pitfalls

> **One of these names is a working password**
> 
> The service set carries the stored credential decrypted, next to the server address, the port and the username. Printing it puts a live login into an inbox and into the message log permanently. Link to the service page instead, and keep the credential for the single message whose whole purpose is to deliver it.

> **Reusing a platform name loses your value**
> 
> Both platform layers are laid over the resolver's set, and only the four recipient display names defer to what is already there. Set an address, a link or a colour under a platform name and it is overwritten before the template ever sees it. Nothing is logged.

> **A name that does not exist prints as nothing**
> 
> There is no warning and no marker in the output. A misspelt placeholder looks exactly like a value that happened to be empty. When a line disappears from a message, check the spelling against the tables here before looking at the resolver.

> **The declared list is documentation, not a contract**
> 
> The badge list an operator sees is assembled from three static declarations that nothing verifies against the resolver. A name can be missing from it and still print, and it can be listed and never arrive. Trust what the resolver sets, and update the declarations so the operator can trust them too.

## Related Articles

- [How Notification Templates Work](https://dev.wisecp.com/en/how-notification-templates-work)
- [Writing an Email Template](https://dev.wisecp.com/en/writing-an-email-template)
- [Writing an SMS Template](https://dev.wisecp.com/en/writing-an-sms-template)
- [Writing a Hook Listener](https://dev.wisecp.com/en/writing-a-hook-listener)
- [Domain Helpers](https://dev.wisecp.com/en/domain-helpers)

# Shipping Templates with a Module

https://dev.wisecp.com/en/shipping-templates-with-a-module

Ship a notification with your module, so enabling it is all the operator has to do.

## Overview

A module that sends its own mail installs two things. The **registration** puts the template on the Notification Templates screen, where the operator turns it on or off. The **files** put the subject, the body and the text message where they are read from.

Skip the first and `dispatch()` returns `disabled`. Skip the second and the mail goes out with no body and no subject. One call does both.

- **Notification::seed_templates()**: Adds the missing registration and writes the parts to their roots. An existing file is never overwritten, so an operator's edit survives every re-enable.
- **Notification::dispatch()**: Sends it afterwards. A seeded template is dispatched like a core one.

## Prerequisites

- A module with an `enable()` path.
- A group name. Reuse a core group when the mail belongs to that domain, or use your own.
- Read [How Notification Templates Work](https://dev.wisecp.com/en/how-notification-templates-work) first. It explains the roots this call writes into.

## Structure

Keep the templates inside the module, so they travel and are removed with it. Two source shapes are read; use whichever suits you.

```bash
coremio/modules/Servers/Acme/
├── Acme.php
└── notifications/                       # flat form: one file per part
    ├── en/
    │   ├── acme-quota-reached.json        # {"subject": "..."}
    │   ├── acme-quota-reached.html        # the mail body
    │   └── acme-quota-reached.txt         # the text message
    └── tr/ ...

coremio/modules/Servers/Acme/notifications/    # folder form: adds a per-design body
└── en/acme-quota-reached/
    ├── content.json
    ├── content.txt
    ├── content.html                       # the base design's body
    └── ledger.html                        # the Ledger design's own body
```

The folder form is what a release package uses, so a module and a release describe a template alike.

## Walkthrough

1. Write the mail body as a **fragment**. The shell around it comes from the active design.
2. Write one file per language you support. A language you skip has no message; the operator can fill it in.
3. Use Smarty placeholders (`{$service_name}`) and declare them in `settings`, so the panel offers them.
4. Call the seeder from `enable()`. Add a guard that also runs on update: an installation enabled long ago never toggles the module to receive a new template.
5. Send with `dispatch()` and read the returned `status`. `disabled` means the registration is missing or the operator turned it off.

## Reference

```php
static function seed_templates(string $group, array $templates, array $options = []): array
{
    // ...
}
```

- **$templates[key]['settings']**: The config entry, written only when the key is absent. Anything you omit is defaulted: `status` 1, `user-mail` 1, `admin-mail`/`user-sms`/`admin-sms` 0, empty `emails`/`phones`/`departments`. **Leave the key out entirely** and the config file is not touched at all — for a module that maintains its own entry.
- **$templates[key]['source']**: Directory holding the files. Both shapes above are read; the folder form is tried first because it is the one that can carry a per-design body.
- **$templates[key]['text']**: Inline alternative to `source`: `[lang => ['subject' => …, 'html' => …, 'sms' => …, 'themes' => [design => html]]]`. A part you leave out is not written.
- **$options['themes']**: `'base'` (default) writes the body to the design that ships with the product and lets every other design fall back to it. `'all'` copies that body into each installed design — see the pitfall below before choosing it.
- **return**: `['written' => string[], 'skipped' => int, 'registered' => string[]]` — paths written this run, files left alone because they already existed, and the `group/key` pairs added to the config.

## Example

```php
private function ensure_notification_template(): void
{
    static $checked = false;
    if ($checked) return;
    $checked = true;

    \Notification::seed_templates('service', [
        'acme-quota-reached' => [
            'settings' => [
                'variables' => '{service_id},{service_name},{quota_usage}',
                'status'    => 1,
                'user-mail' => 1,
            ],
            'source' => __DIR__ . DS . 'notifications',
        ],
    ]);
}
```

```php
$this->ensure_notification_template();

$result = \Notification::dispatch('service', 'acme-quota-reached', [
    'user_id'   => (int) $service['owner_id'],
    'variables' => [
        '{service_id}'   => $service['id'],
        '{service_name}' => $service['name'],
        '{quota_usage}'  => $usage . '%',
    ],
]);

// 'disabled' is a decision, not a failure: the operator turned this mail off.
if (($result['status'] ?? '') === 'error') \Logger::error('Acme quota mail failed');
```

## Pitfalls

> **A design you do not ship is not a gap**
> 
> A design with no file for the event shows the base body inside *its own* shell. Copying the body into every design is not the same thing. The copy outranks the fallback, so that theme's author can never ship a design for the mail afterwards.

> **Registration alone is not enough**
> 
> A config entry with no files sends an empty mail. The first sign of it is a customer complaint. If you maintain the entry yourself, still call the seeder for the files. Pass no `settings` key and your entry is left alone.

> **Never build the path yourself**
> 
> The parts live in different roots, and the body's root depends on the active design. Code that joins `templates/notifications/` to a language folder writes where nothing reads. The write succeeds and the mail stays empty.

## Related Articles

- [How Notification Templates Work](https://dev.wisecp.com/en/how-notification-templates-work)
- [Writing an Email Template](https://dev.wisecp.com/en/writing-an-email-template)
- [Notification Template Variables](https://dev.wisecp.com/en/notification-template-variables)

