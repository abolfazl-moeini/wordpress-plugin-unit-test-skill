---
name: wordpress-plugin-unit-tests
description: >-
  Scaffolds and writes WordPress plugin unit tests using self-contained local
  path packages (wpdev/plugin-core-test), PHPUnit, mirrored tests/unit-tests
  layout, TDD workflow, REST/Ajax patterns, factories, and mocks. Use when
  adding unit tests, phpunit.xml, bootstrap.php, PluginBaseTestCase,
  RestTestCase, BaseAjaxTestCase, or test-driven plugin work.
---

# WordPress Plugin Unit Tests

Unit-test guidance for WordPress plugins using **`wpdev/plugin-core-test`** (`WPDevTest\`). Extend the package **`TestCase`** hierarchy (or **`BaseAjaxTestCase`** for Ajax — separate branch, not a child of `TestCase`). Never extend `WP_UnitTestCase` directly in plugin tests.

**Complements:** [wordpress-plugin](../wordpress-plugin/SKILL.md) (implementation); this skill covers **how to test**.

**Reference implementation:** Package API in [`packages/plugin-core-test/src/TestCases/`](packages/plugin-core-test/src/TestCases/). Sample tests in [`packages/plugin-core-test/tests/unit/`](packages/plugin-core-test/tests/unit/).

**Scope:** unit tests only (`tests/unit-tests/`). Acceptance/integration tests are out of scope.

## Quick workflow

0. Set up private packages — see [private-packages.md](references/private-packages.md).
1. Ensure scaffolding exists — see [scaffolding.md](references/scaffolding.md).
2. Mirror `src/` under `tests/unit-tests/` (rules below).
3. Pick a base class — see [test-base-classes.md](references/test-base-classes.md).
4. Write the test (`@test`, `itShould…`, assertion messages).
5. Run `composer tests` (after configuring `phpunit.xml.dist` env vars).

## TDD loop

When the user asks for TDD or feature + tests together:

1. **Red** — Add mirrored `*Test.php` with `@test` method `itShould…()`; run `phpunit`; expect failure.
2. **Green** — Implement minimal code in mirrored `src/`.
3. **Refactor** — Clean production code; keep tests green.
4. **Scope** — One behavior per method; shared arrange in `setUp()`, `@dataProvider`, or fixtures.

## Directory layout

```
plugin-root/
├── packages/                     # copied from skill/packages/
│   ├── plugin-core-test/         # always
│   └── wc-core-test/             # optional (WooCommerce)
├── src/                          # plugin source
├── tests/
│   ├── unit-tests/               # all unit tests
│   │   ├── bootstrap.php         # always
│   │   ├── TestCases/            # plugin base classes (no src mirror)
│   │   └── …                     # mirrors src/
│   └── plugins-list.php          # only when other plugins required
├── phpunit.xml.dist              # always
└── composer.json                 # path repositories
```

## File mirroring

| Source | Test |
|--------|------|
| `src/Model.php` | `tests/unit-tests/ModelTest.php` |
| `src/DoComplex.php` (many tests) | `tests/unit-tests/DoComplex/ApiTest.php`, `ParseTest.php`, … |
| `src/.../DeleteRestHandler.php` | `tests/.../DeleteRestHandler/DeleteTest.php` |

- Append **`Test`** to class and file names.
- Split into a **subfolder** when one source file needs many test classes.
- **REST handlers:** subfolder named after handler class; test file named by behavior.

Plugin bases (`PluginBaseTestCase`, `RestTestCase`, `AdminTestCase`) live in `tests/unit-tests/TestCases/` — not mirrored from `src/`.

## Namespace (composer autoload-dev)

Append `Test` to the **first** PSR-4 segment; keep the rest:

| Production | Tests |
|------------|-------|
| `WPDev\Core\Rest` | `WPDevTest\Core\Rest` |
| `WPDevSale\` | `WPDevSaleTest\` |

Map test namespace to `./tests/unit-tests/`.

## Base class picker

**Default:** extend plugin **`PluginBaseTestCase`** when it exists. Use `WPDevTest\TestCases\TestCase` only for pure bootstrap/setup tests (no domain factories).

| Need | Extend |
|------|--------|
| Domain / feature logic | Plugin `PluginBaseTestCase` (or child) |
| Plugin hooks / activation only | `WPDevTest\TestCases\TestCase` |
| Admin UI | Plugin `AdminTestCase extends PluginBaseTestCase` |
| REST endpoint | Plugin `RestTestCase` — `rest_handler_instance()` + `init()` |
| `wp_ajax_*` | `WPDevTest\TestCases\BaseAjaxTestCase` or plugin wrapper |
| Raw SQL | `WPDevTest\TestCases\DBTestCase` |
| WPML mock | `MultiLangTestCase` + `WPML_ENABLED=1` in phpunit |
| Script enqueue | `LibraryTestCase` |

Details: [test-base-classes.md](references/test-base-classes.md).

## Test authoring rules

| Rule | Practice |
|------|----------|
| Header | `declare(strict_types=1);` on every new test file |
| Discovery | `@test` on public methods (no `test*` prefix required) |
| Naming | `itShouldVerbExpectedOutcome()` |
| Assertions | Always pass a message: `$this->assertTrue( isset( $panel['something'] ), 'Admin Panel Was not Registered' );` |
| Helpers | `$this->method_working_notice($class, $method, $id)`, `$this->template_working_notice(...)` |
| Data | `@dataProvider` + provider methods — see [test-authoring-examples.md](references/test-authoring-examples.md) |
| Cleanup | DB auto-rollback; in `tearDown()` `remove_filter` (e.g. `pre_http_request`), `wp_dequeue_script` / `wp_deregister_script`, transients |
| Private API | `method_call()`, `property_set()`, `property_get()` — not raw Reflection; `method_call( …, true )` captures echo output |

Examples (dataProvider, admin `setUp`, tearDown, `expectFilter`): [test-authoring-examples.md](references/test-authoring-examples.md).

## Auth & URLs

```php
$user_id = $this->login( '{cap}' );       // e.g. 'subscriber', 'administrator', 'editor'
$this->switch_user( $other_user_id );     // needs user-switching in plugins-list.php

$this->go_to( get_permalink( $post_id ) ); // front-end / rewrite query (permalink structure set by go_to)
```

`{cap}` is the **WordPress role slug** passed to `user->create( [ 'role' => … ] )`. Custom capabilities: register role/cap **before** `login()`, then `flush_capabilities()`.

## Patterns (by topic)

| Topic | Reference |
|-------|-----------|
| Private packages (path repo) | [private-packages.md](references/private-packages.md) |
| composer, phpunit, bootstrap | [scaffolding.md](references/scaffolding.md) |
| Base classes & hierarchy | [test-base-classes.md](references/test-base-classes.md) |
| REST, Ajax, HTTP, admin screen, `go_to` | [patterns-rest-ajax-http.md](references/patterns-rest-ajax-http.md) |
| Core + custom factories | [factories.md](references/factories.md) |
| PHPUnit mocks, ObjectProxy | [mocking.md](references/mocking.md) |
| dataProvider, tearDown, expectFilter | [test-authoring-examples.md](references/test-authoring-examples.md) |
| Factory API (WordPress) | `wp-docs/factory.md` (external — outside this skill) |

## Agent checklist

When adding or fixing unit tests:

1. Set up `packages/` and path repo — [private-packages.md](references/private-packages.md).
2. Create missing scaffolding (`composer.json`, `phpunit.xml.dist`, `bootstrap.php`; `plugins-list.php` only if needed).
3. Place test file mirroring `src/` with correct namespace.
4. Extend the right base class (`PluginBaseTestCase` by default).
5. Use `@test`, descriptive method name, assertion messages.
6. Register custom factories on `WPDevTest/Factory` when domain data is needed.
7. Roll back non-DB side effects in `tearDown()` (filters, enqueued scripts).
8. Run `composer tests` when env vars are configured.

## Out of scope

- Acceptance / Behat (`tests/acceptance/`)
- Integration test suite (future skill)
- `createTestProxy` / BetterStudioTest
- Modifying **bundled** packages in this skill
- `wpdev/wp-test-tools` (dependencies outside this skill)
- `sample plugin/` and `wp-docs/` references (external until a future skill)
