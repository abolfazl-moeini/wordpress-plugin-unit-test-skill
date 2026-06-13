# Test authoring examples

## @dataProvider and fixtures

When input varies, use a **provider method** that returns rows of arguments (fixtures). Keep rows in the provider or in a private helper — no separate folder required unless the project already uses one.

```php
/**
 * @test
 * @dataProvider invalidUrlsProvider
 */
public function itShouldRejectInvalidDevelopmentUrls( string $url ): void {
    $this->assertFalse(
        URLValidator::is_dev_domain( $url ),
        sprintf( 'URL should be invalid: %s', $url )
    );
}

public function invalidUrlsProvider(): array {
    return [
        'empty'     => [ '' ],
        'no scheme' => [ 'not-a-url' ],
        'ftp'       => [ 'ftp://example.com' ],
    ];
}
```

See the `@dataProvider` example above; bundled package tests: `packages/plugin-core-test/tests/unit/`.

## login( '{cap}' )

Team convention: pass a **WordPress role slug** as `{cap}`:

```php
$admin_id = $this->login( 'administrator' );
$sub_id   = $this->login( 'subscriber' );
```

REST: `dispatch( $params, true )` calls `login()` with default **subscriber**. Set role explicitly when caps matter:

```php
$this->login( 'administrator' );
$response = $this->dispatch( $params );
```

## Admin: set_current_screen in setUp

Before code that uses `get_current_screen()` or admin-only hooks, set the screen in **`setUp()`** (or use plugin `AdminTestCase`):

```php
protected function setUp(): void {
    parent::setUp();
    set_current_screen( 'edit' );
    MyPlugin\Admin\MetaBoxSetup::init();
}
```

## URIs and rewrite rules

`go_to()` sets permalink structure to `/%postname%/` then loads the URL:

```php
$post_id = $this->factory()->post->create( [ 'post_name' => 'sample-post' ] );
$this->go_to( get_permalink( $post_id ) );
```

Register custom rewrites in `setUp()` / `init()` when needed. Use `flush_rewrite_rules()` only for activation/rewrite tests — not every test.

## method_call and captured output

```php
$output = $this->method_call( $handler, 'renderNotice', [], true );
$this->assertStringContainsString( 'Success', $output, 'Notice was not rendered' );
```

## tearDown: scripts and filters

```php
private string $script_handle = 'my-plugin-admin';

protected function tearDown(): void {
    wp_dequeue_script( $this->script_handle );
    wp_deregister_script( $this->script_handle );
    parent::tearDown();
}
```

`LibraryTestCase` calls `wp_deregister_script( $this->handle_name() )` in `tearDown()` automatically.

## expectFilter

`expectFilter( $filter, ...$arguments )` expects the filter callback to receive those arguments; returns a mock for `willReturn()`:

```php
$value = 'original';
$mock  = $this->expectFilter( 'my_plugin_filter', $value, 'context' );
$mock->willReturn( 'filtered' );

$this->assertSame( 'filtered', apply_filters( 'my_plugin_filter', $value, 'context' ) );
```
