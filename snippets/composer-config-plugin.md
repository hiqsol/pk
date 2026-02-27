# composer-config-plugin: Agent Reference

Quick reference for AI agents working on projects that use
`hiqdev/composer-config-plugin`.

## Where config lives

- **Per-package**: `composer.json` → `extra.config-plugin` section + referenced PHP files
- **Assembled output**: `vendor/hiqdev/composer-config-plugin-output/`
- **Root project** overrides everything

## Override direction

```
root > main project > plugins > base project > framework
```

Outer packages override inner ones. Plugins only set defaults; root has full control.

## Config processing order

1. `dotenv` — environment variables (`.env` files)
2. `defines` — PHP constants
3. `params` — available as `$params` in later configs
4. Other configs: `common`, `web`, `console`, etc.

## Merging

- Deep recursive merge (`ArrayHelper::merge`)
- `?` prefix = optional file: `"?config/params-local.php"`
- `$` prefix = reference: `"web": ["$common", "web.php"]`

## Adding / removing features

- **Add**: `composer require vendor/plugin` → config auto-merges
- **Remove**: `composer remove vendor/plugin` → config disappears
- No manual config editing needed

## Key lookups

To find what a package configures: check its `composer.json` `extra.config-plugin`
section, then read the referenced config files.

To debug final merged config: read files in
`vendor/hiqdev/composer-config-plugin-output/`.
