# Factories

## WordPress core factory

Access via `$this->factory()` on `TestCase` and `BaseAjaxTestCase`:

```php
$user_id = $this->factory()->user->create( [
    'user_login' => 'wpdev',
    'role'       => 'subscriber',
] );

$post_id = $this->factory()->post->create( [
    'post_title'  => 'Test post',
    'post_status' => 'publish',
] );

$term_id = $this->factory()->term->create( [
    'name'     => 'Category',
    'taxonomy' => 'category',
] );
```

Full factory API: **`wp-docs/factory.md`** at project root (`user`, `post`, `comment`, `term`, `attachment`, etc.) — external, outside this skill.

## Custom domain factories

Register on the `WPDevTest/Factory` filter from `PluginBaseTestCase::setUp()`:

```php
function setUp(): void {
    tests_add_filter( 'WPDevTest/Factory', [ static::class, 'setup_update' ] );
    parent::setUp();
}

public static function setup_update( $factory ): void {
    $factory->license = new Factory\LicenseFactory( $factory );
    $factory->site    = new Factory\SiteFactory( $factory );
}
```

### Factory class

Extend `WP_UnitTest_Factory_For_Thing`:

```php
class LicenseFactory extends \WP_UnitTest_Factory_For_Thing {
    public function __construct( $factory = null ) {
        parent::__construct( $factory );
        $this->default_generation_definitions = [
            'order_id' => 1,
            'user_id'  => 0,
            'state_id' => 'active',
        ];
    }

    public function create_object( $args ) {
        $model = new LicenseModel();
        return $model->insert( wp_parse_args( $args, $this->default_generation_definitions ) );
    }

    public function update_object( $license_id, $fields ) {
        // ...
    }
}
```

### Usage in tests

```php
$license_id = $this->factory()->license->create( [
    'user_id'  => $user_id,
    'state_id' => 'active',
] );

$site_id = $this->factory()->site->create( [
    'license_id' => $license_id,
] );
```

## Ajax tests and factories

Ajax tests do not extend `PluginBaseTestCase` by default. Use a plugin **`BaseAjaxTestCase` wrapper** that registers the same `WPDevTest/Factory` hook in `setUp()` so `$this->factory()->license` works in Ajax tests.

Reference: `packages/plugin-core-test/src/TestCases/BaseAjaxTestCase.php`.
