# Bundled wpdev packages

This folder is the **canonical source** inside the skill. When scaffolding unit tests for a target plugin, copy packages from here into `{plugin-root}/packages/`.

Do **not** copy from external paths or GitLab at runtime — only from this skill bundle.

## Bundled packages

| Directory | Composer name | Version | When to copy |
|-----------|---------------|---------|--------------|
| `plugin-core-test/` | `wpdev/plugin-core-test` | 1.2.1 | Always (required) |
| `wc-core-test/` | `wpdev/wc-core-test` | — | WooCommerce tests only |

## Copy rules

1. **Always** copy `plugin-core-test/` to the target plugin.
2. Copy `wc-core-test/` only when tests need WooCommerce helpers.
3. Skip copy if the target already has the package (do not overwrite unless the user asks).
4. After copy, add the path repository to the target `composer.json` — see [private-packages.md](../references/private-packages.md).

## Not bundled

`wpdev/wp-test-tools` is **not** supported — it depends on `wpdev/console`, which is outside this skill.

## Contents per package

Each package includes `src/`, `composer.json`, and (for `plugin-core-test`) `tests/` for agent reference.
