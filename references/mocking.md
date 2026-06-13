# Mocking

Use **PHPUnit mocks** and **`WPDevTest\Proxy\ObjectProxy`**. Do **not** use `createTestProxy` (external / not in `plugin-core-test`).

## PHPUnit createMock (interaction tests)

Assert a dependency method is called with expected arguments:

```php
$widget = $this->createMock( SampleWidget::class );

$widget
    ->expects( $this->once() )
    ->method( 'add_control' )
    ->with(
        $this->equalTo( 'control_id' ),
        $this->equalTo( [ 'type' => 'select', 'label' => 'Choose' ] )
    );

$transformer->apply( $widget, $panel );
```

Reference (PHPUnit only — ignore `createTestProxy` in base classes):

- `sample-lib/elementor-integration/tests/unit/WidgetFieldsTransformer/ControlsSuite/TestChooseControlsTransform.php`
- More `createMock()` patterns: browse `sample-lib/elementor-integration/tests/unit/` (PHPUnit only)

## getMockBuilder for hook callbacks

```php
$mock = $this->getMockBuilder( \stdClass::class )
    ->setMethods( [ 'hook' ] )
    ->getMock();

$mock->expects( $this->once() )
    ->method( 'hook' )
    ->with( $this->equalTo( false ), $this->equalTo( 123 ), $this->equalTo( $options ) )
    ->willReturn( [] );

add_filter( 'WPDev/License/GetBy/sample', [ $mock, 'hook' ], 9, 3 );

LicenseModel::get_by( 'sample', 123, $options );
```

Reference: `packages/plugin-core-test/tests/unit/TestCases/TestBaseTestCase.php` (`expectAddAction` / `expectFilter` patterns).

## Simple filter stub

```php
add_filter( 'WPDev/License/GetBy/my', '__return_empty_array' );

$this->assertEquals( [], LicenseModel::get_by( 'my', 123 ) );
```

## Hook registration assertions

From `WPDevTest\TestCases\BaseTestCase`:

```php
$this->expectAddAction( 'my_plugin_init', [ MySetup::class, 'init' ], 10 );
MySetup::register();
```

For **`expectFilter()`** — pass the filter name and the exact arguments the callback receives; use the returned mock for `willReturn()`. See [test-authoring-examples.md](test-authoring-examples.md).

## ObjectProxy

`WPDevTest\Proxy\ObjectProxy` wraps a real object. Overrides live in stacks; unmatched calls/properties fall through to the wrapped instance.

### When to use

| Tool | Use when |
|------|----------|
| `ObjectProxy` | Substitute behavior in production code paths (fake method return, inject dependency) |
| `method_call()` / `property_get()` | One-off access to private/protected API on the real class |
| `createMock()` | Strict interaction / call-count assertions |

### API

```php
use WPDevTest\Proxy\ObjectProxy;

$real  = new MyService( $dependency );
$proxy = new ObjectProxy( $real );

// Fixed method return (not callable — stored value):
$proxy->method_set( 'fetchRemote', [ 'id' => 1, 'name' => 'stub' ] );
$data = $proxy->fetchRemote();  // returns stub array; real method not called

// Property override:
$proxy->property_set( 'apiKey', 'test-key' );
// or: $proxy->apiKey = 'test-key';

// Pass proxy where production expects MyService:
$handler = new Handler( $proxy );
```

| Method | Behavior |
|--------|----------|
| `method_set( $method, $value )` | Return `$value` on `$method()` (uses `isset` — **cannot** override with `null`) |
| `method_get( $method )` | Read stacked override |
| `property_set( $property, $value )` | Override `__get` |
| `property_get( $property )` | Read property stack |
| `__call` | Stack override, else delegate to `$instance` |
| `__get` / `__set` | Stack override, else delegate |

### Limitations

- `method_set` with `null` does not work (stack uses `isset`).
- `__isset` only checks the property stack, not the real object.

Tests: `packages/plugin-core-test/tests/unit/Proxy/TestObjectProxy.php`.

## External HTTP

See [patterns-rest-ajax-http.md](patterns-rest-ajax-http.md) — `pre_http_request` with **`remove_filter` in `tearDown()`**.

## tearDown cleanup checklist

Non-DB side effects to undo:

- [ ] `remove_filter` / `remove_action` for hooks added in test
- [ ] `wp_deregister_script` / `wp_dequeue_script` for registered handles
- [ ] `delete_transient` for transients created in test
- [ ] Reset globals or statics if mutated

Database changes roll back automatically between tests.
