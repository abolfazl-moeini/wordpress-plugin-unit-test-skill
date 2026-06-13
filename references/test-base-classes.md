# Test base classes

All plugin unit tests use **`wpdev/plugin-core-test`** (`WPDevTest\TestCases\*`). The package extends WordPress `WP_UnitTestCase` (or `WP_Ajax_UnitTestCase` for Ajax).

## Default rule

If `tests/unit-tests/TestCases/PluginBaseTestCase.php` exists:

- **Domain / feature tests** → extend `PluginBaseTestCase` or its children.
- **Pure plugin bootstrap** (hooks, activation, no factories) → extend `WPDevTest\TestCases\TestCase` directly.

## Hierarchy (create per plugin)

```
WPDevTest\TestCases\TestCase
└── {Plugin}Test\TestCases\PluginBaseTestCase
    ├── RestTestCase
    ├── AdminTestCase       ← extends PluginBaseTestCase, NOT package AdminTestCase
    ├── PurchaseTestCase
    └── …

WPDevTest\TestCases\BaseAjaxTestCase    ← separate branch (WP_Ajax_UnitTestCase)
└── {Plugin}Test\TestCases\BaseAjaxTestCase   ← thin wrapper; registers factories
```

## Package classes

| Class | Extends | Use when |
|-------|---------|----------|
| `TestCase` | `BaseTestCase` | Default; login, go_to, reflection helpers, factory |
| `BaseTestCase` | `WP_UnitTestCase` | `expectFilter()`, `expectAddAction()` only |
| `AdminTestCase` | `TestCase` | Package-only admin tests; sets `edit` screen + `init()` |
| `BaseAjaxTestCase` | `WP_Ajax_UnitTestCase` | `wp_ajax_*` handlers; `request()`, `ajax_action()` |
| `DBTestCase` | `TestCase` | Raw `$wpdb` / schema checks |
| `MultiLangTestCase` | `TestCase` | `wpml_object_id` without real WPML |
| `LibraryTestCase` | `TestCase` | Script enqueue; implement `handle_name()`, `version_parser()` |

## Plugin PluginBaseTestCase

Register custom factories on `WPDevTest/Factory` in `setUp()`:

```php
function setUp(): void {
    tests_add_filter( 'WPDevTest/Factory', [ static::class, 'setup_update' ] );
    parent::setUp();
}

public static function setup_update( $factory ): void {
    $factory->license = new Factory\LicenseFactory( $factory );
}
```

Pattern above is the canonical `PluginBaseTestCase` shape. More examples: [test-authoring-examples.md](test-authoring-examples.md). Package source: `packages/plugin-core-test/src/TestCases/TestCase.php`.

## Plugin AdminTestCase

```php
abstract class AdminTestCase extends PluginBaseTestCase {
    function setUp(): void {
        parent::setUp();
        set_current_screen( 'edit' );
        $this->init();
    }
    abstract public function init();
}
```

Override screen in `init()`:

```php
public function init(): void {
    set_current_screen( 'post' );
    MyPlugin\Admin\Setup::setup();
}
```

## Plugin RestTestCase

Extend `PluginBaseTestCase`. Implement:

- `rest_handler_instance()` — handler singleton
- `init()` — boot related `Setup::instance()->init()`

`setUp()` builds `Spy_REST_Server`, runs `rest_api_init`. Use `$this->dispatch( $params, $login )` and `$this->request()`.

See [patterns-rest-ajax-http.md](patterns-rest-ajax-http.md).

## Plugin BaseAjaxTestCase wrapper

Root is always `WPDevTest\TestCases\BaseAjaxTestCase`. Add a thin plugin wrapper when Ajax tests need domain factories:

```php
abstract class BaseAjaxTestCase extends \WPDevTest\TestCases\BaseAjaxTestCase {
    function setUp(): void {
        tests_add_filter( 'WPDevTest/Factory', [ PluginBaseTestCase::class, 'setup_update' ] );
        parent::setUp();
    }
}
```

Test classes implement `ajax_action(): string` and call `$this->request( $params )`.

**Note:** Ajax base does **not** extend `TestCase` — no `method_call()` / `property_get()` unless composed manually.

## Auth helpers

### login( '{cap}' )

Team convention: `{cap}` is the **WordPress role slug** (`subscriber`, `administrator`, `editor`, …).

| Base class | Default role |
|------------|--------------|
| `TestCase` | `subscriber` |
| `BaseAjaxTestCase` | `administrator` |

```php
$user_id = $this->login( 'editor' );
```

Custom capability: register on a role before `login()`, then `$this->flush_capabilities()`.

### switch_user( $user_id )

Requires **user-switching** plugin in `tests/plugins-list.php`. Skips test if plugin inactive.

## Reflection helpers (TestCase only)

| Method | Purpose |
|--------|---------|
| `method_call( $object, $method, $args, $get_echo )` | Invoke private/protected/static method |
| `property_set( $object, $property, $value )` | Set private/protected property |
| `property_get( $object, $property, &$value )` | Read private/protected property |

### Capturing echoed output

`method_call( $object, $method, $arguments, true )` — fourth argument `true` returns buffered output instead of the return value:

```php
$output = $this->method_call( $handler, 'renderNotice', [], true );
$this->assertStringContainsString( 'Success', $output, 'Notice was not rendered' );
```

## Assertion message helpers

```php
$this->assertTrue(
    $ok,
    $this->method_working_notice( MyClass::class, 'save', 42 )
);

$this->assertFileExists(
    $path,
    $this->template_working_notice( 'panel.php', 'Admin' )
);
```

## URIs

```php
$this->go_to( get_permalink( $post_id ) );
```

Sets permalink structure to `/%postname%/` then simulates the main query.
