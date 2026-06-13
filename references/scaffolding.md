# Test scaffolding

**Before `composer install`:** set up private wpdev packages — see [private-packages.md](private-packages.md).

## Always create

### composer.json

```json
{
  "repositories": [
    {
      "type": "path",
      "url": "packages/*",
      "options": {
        "monorepo": true,
        "symlink": false
      }
    }
  ],
  "autoload": {
    "psr-4": {
      "MyVendor\\MyPlugin\\": "./src/"
    }
  },
  "autoload-dev": {
    "psr-4": {
      "MyVendorTest\\MyPlugin\\": "./tests/unit-tests/"
    }
  },
  "require-dev": {
    "wpdev/plugin-core-test": "^1.2"
  },
  "scripts": {
    "tests": "phpunit",
    "tests:pre": "@php ./tests/patch/apply-patches.php"
  }
}
```

**Namespace rule:** append `Test` to the first segment (`WPDev\Core\Rest` → `WPDevTest\Core\Rest`).

**Optional require-dev** (add only when the package was copied to `packages/`):

- `wpdev/wc-core-test` — WooCommerce test helpers (also add `woocommerce/woocommerce.php` to `tests/plugins-list.php`)

**Optional:** add `"files": [ "tests/unit-tests/functions.php" ]` under `autoload-dev` for global function stubs (e.g. `bf_asset_info`) loaded in every test run — only when tests need them.

For merging into an existing `composer.json`, see [private-packages.md](private-packages.md).

### phpunit.xml.dist

Copy this structure; replace plugin-specific placeholders:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit
        bootstrap="tests/unit-tests/bootstrap.php"
        backupGlobals="false"
        colors="true"
        convertErrorsToExceptions="true"
        convertNoticesToExceptions="true"
        convertWarningsToExceptions="true"
        verbose="true"
>
    <testsuites>
        <testsuite name="plugin">
            <directory suffix=".php">./tests/unit-tests/</directory>
            <exclude>./tests/unit-tests/vendor</exclude>
        </testsuite>
    </testsuites>

    <php>
        <includePath>.</includePath>
        <env name="stage" value="development"/>
        <env name="db" value="wpdev-test"/>

        <env name="WP_TESTS_DIR" value="/Users/moeini/Dev/wordpress-develop/tests/phpunit"/>
        <env name="BOOTSTRAP_FILE"
             value="{wp-root}/wp-content/plugins/{plugin-slug}/{main-file}.php"/>
        <env name="PLUGIN_ROOT" value="{wp-root}/wp-content/plugins/{plugin-slug}"/>

        <!-- Only when tests need other plugins: -->
        <!-- <env name="ACTIVE_PLUGINS" value="tests/plugins-list.php"/> -->
        <!-- <env name="WPML_ENABLED" value="1"/> -->
    </php>

    <filter>
        <whitelist processUncoveredFilesFromWhitelist="true">
            <directory suffix=".php">src</directory>
        </whitelist>
    </filter>
</phpunit>
```

Replace `{wp-root}` with your WordPress root (default: `/Users/moeini/Dev/wordpress-develop`).

### Env vars → bootstrap wiring

`WPDevTest\Setup::setup()` reads `<env>` values from phpunit at runtime:

| Variable | Purpose | Default |
|----------|---------|---------|
| `WP_TESTS_DIR` | WordPress PHPUnit includes (required) | `/Users/moeini/Dev/wordpress-develop/tests/phpunit` |
| `BOOTSTRAP_FILE` | Main plugin PHP file to load (required) | `{wp-root}/wp-content/plugins/{plugin-slug}/{main-file}.php` |
| `PLUGIN_ROOT` | Plugin root directory (required) | `{wp-root}/wp-content/plugins/{plugin-slug}` |
| `ACTIVE_PLUGINS` | Path to `tests/plugins-list.php` (conditional) | — |
| `WPML_ENABLED` | Load WPML mock via `WPDevTest\WPML::init()` (optional) | — |
| `stage`, `db` | Project-specific; used by some plugins | `development`, `wpdev-test` |

**WP root default:** `/Users/moeini/Dev/wordpress-develop`

Do **not** hardcode these paths in `bootstrap.php` — configure them in `phpunit.xml.dist`.

### tests/unit-tests/bootstrap.php

**Fixed skeleton:**

```php
<?php

declare( strict_types=1 );

require dirname( __DIR__, 2 ) . '/vendor/autoload.php';

WPDevTest\Setup::setup( static function () {
    // Plugin-specific setup (optional) — see below
} );
```

**Optional blocks** (before or inside the callback):

```php
// When uninstall / WC data cleanup matters in tests:
define( 'WP_UNINSTALL_PLUGIN', true );
define( 'WC_REMOVE_ALL_DATA', true );

// Inside Setup::setup callback:
add_filter( 'WPDev/Core/RequestMock', '__return_empty_array', 1 );

// Custom DB tables — install fresh each run:
foreach ( [ MyPlugin\Model\FooModel::class ] as $model_class ) {
    $model = new $model_class();
    $model->uninstall();
    $model->install();
}
```

## Create only when needed

### tests/plugins-list.php

Return an array of plugin paths relative to `wp-content/plugins/`:

```php
<?php

return [
    'woocommerce/woocommerce.php',
    'user-switching/user-switching.php',
];
```

Reference in phpunit: `<env name="ACTIVE_PLUGINS" value="tests/plugins-list.php"/>`.

Do **not** create this file for plugins with no external plugin dependencies.

## Running tests

```bash
# 1. Copy packages from skill — see private-packages.md
# 2. composer install
# 3. Set BOOTSTRAP_FILE and PLUGIN_ROOT in phpunit.xml.dist
composer tests:pre   # optional — patches (e.g. WooCommerce)
composer tests
```
