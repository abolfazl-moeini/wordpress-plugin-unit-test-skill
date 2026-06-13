# REST, Ajax, HTTP, and admin patterns

## REST API tests

Pattern is **evolving** — treat sample `RestTestCase` as reference; improvements (POST body, cleaner dispatch) are allowed.

### Plugin RestTestCase

```php
abstract class RestTestCase extends PluginBaseTestCase {
    protected $rest;

    abstract public function rest_handler_instance();
    abstract public function init();

    public function setUp(): void {
        global $wp_rest_server;
        parent::setUp();
        $this->init();
        $this->rest = $this->rest_handler_instance();
        $wp_rest_server = new \Spy_REST_Server;
        do_action( 'rest_api_init', $wp_rest_server );
    }

    protected function dispatch( array $params = [], $login = false ): \WP_REST_Response {
        return rest_get_server()->dispatch( $this->request( $params, $login ) );
    }

    protected function request( array $params = [], $login = false ): \WP_REST_Request {
        $login && $this->login();
        $request = new \WP_REST_Request( $this->rest->methods(), $this->path() );
        $request->set_header( 'content-type', 'application/json' );
        $request->set_query_params( array_merge( [
            'length'   => 10,
            '_wpnonce' => wp_create_nonce( 'wp_rest' ),
        ], $params ) );
        return $request;
    }

    protected function path(): string {
        return '/' . Rest\RestSetup::NAMESPACE . '/' . $this->rest->rest_end_point();
    }
}
```

### Test class

Mirror handler under a subfolder:

`src/.../Delete/DeleteRestHandler.php` → `tests/.../DeleteRestHandler/DeleteTest.php`

```php
class DeleteTest extends RestTestCase {
    public function rest_handler_instance() {
        return Delete\DeleteRestHandler::instance();
    }

    public function init() {
        LicenseManager\Setup::instance()->init();
    }

    /**
     * @test
     */
    public function itShouldSoftDeleteSiteItem() {
        $user_id = $this->login();
        // arrange with $this->factory()->license->create([...])
        $response = $this->dispatch( [
            'license_id' => $license_id,
            'id'         => $site_id,
            'token'      => $this->token( $site_id ),
        ] )->get_data();

        $this->assertTrue( $response['success'], $response['message'] ?? '' );
        $this->assertEquals( 'deleted', $site->state_id, 'Site was not soft-deleted' );
    }
}
```

- `dispatch( $params, $login = false )` — `$login = true` calls `login()` with default **subscriber**. For admin-only endpoints, call `$this->login( 'administrator' )` (or another role) before `dispatch()`.
- Assert `$response['success']`, `$response['code']`, and DB/state as needed.

## Ajax tests

Always extend **`WPDevTest\TestCases\BaseAjaxTestCase`** (or plugin wrapper).

```php
class SaveActionTest extends BaseAjaxTestCase {
    public function ajax_action(): string {
        return 'my_plugin_save';
    }

    /**
     * @test
     */
    public function itShouldReturnSuccessForValidPayload() {
        $this->login( 'administrator' );
        $result = $this->request( [ 'foo' => 'bar' ] );
        $this->assertTrue( $result['success'], 'Ajax save did not succeed' );
    }
}
```

- `request( $params )` sets `$_GET`, runs `_handleAjax`, returns decoded JSON.
- Default `login()` role on Ajax base is `administrator`.

## Admin screen simulation

Admin-only code (meta boxes, list tables, `current_screen` checks) requires **`set_current_screen()` before the code under test runs** — typically in `setUp()`.

**With plugin AdminTestCase:** base `setUp()` calls `set_current_screen( 'edit' )` then `$this->init()`; override screen in `init()` if needed.

**Without AdminTestCase:** in every test class `setUp()`:

```php
protected function setUp(): void {
    parent::setUp();
    set_current_screen( 'edit' );  // post list; 'post' = editor, 'dashboard', etc.
    MyPlugin\Admin\Setup::init();
}
```

Common slugs: `edit`, `post`, `dashboard`, `options-general`.

## Front-end URL simulation

```php
$post_id = $this->factory()->post->create( [ 'post_name' => 'sample-post' ] );
$this->go_to( get_permalink( $post_id ) );
// assert query, template, hooks
```

### Rewrite rules

`go_to()` sets permalink structure to `/%postname%/` and simulates the main query. If the plugin registers custom rewrite rules, boot the same `Setup::init()` (or equivalent) in `setUp()` so rules exist before `go_to()`. Call `flush_rewrite_rules()` only in tests that verify activation or rewrite registration — not in every test (slow).

## External HTTP (pre_http_request)

```php
private $http_callback;

protected function setUp(): void {
    parent::setUp();
    $this->http_callback = function ( $preempt, $args, $url ) {
        if ( 'https://api.example.com/data' === $url ) {
            return [
                'response' => [ 'code' => 200, 'message' => 'OK' ],
                'body'     => wp_json_encode( [ 'status' => 'success' ] ),
            ];
        }
        return $preempt;
    };
    add_filter( 'pre_http_request', $this->http_callback, 10, 3 );
}

protected function tearDown(): void {
    remove_filter( 'pre_http_request', $this->http_callback, 10 );
    parent::tearDown();
}
```

Always **remove** filters added in the test during `tearDown()`.
