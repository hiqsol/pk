# Composer Config Plugin: Plugin System for Yii2 Projects

## Core Concept

Instead of copy-pasting extension configs into the application config, each Composer package declares its own configuration in `composer.json`. A Composer plugin (`hiqdev/composer-config-plugin`) automatically collects and merges all configs from all dependencies into ready-to-use config files. This turns Composer packages into **plugins** — self-contained units of code + configuration.

## How It Works

1. Each package declares configs in `composer.json` `extra` section:
```json
"extra": {
    "config-plugin": {
        "params": "src/config/params.php",
        "web": "src/config/web.php"
    }
}
```

2. On `composer install/update/dump-autoload`, the plugin scans all dependencies, merges their configs respecting the dependency hierarchy, and writes assembled configs to `vendor/hiqdev/composer-config-plugin-output/`.

3. The application loads the assembled config:
```php
$config = require hiqdev\composer\config\Builder::path('web');
(new yii\web\Application($config))->run();
```

## Config Types and Processing Order

Configs are processed in this specific order so that earlier values are available in later configs:

1. **dotenv** — environment variables (`.env` files)
2. **defines** — PHP constants
3. **params** — application parameters (available as `$params` in other configs)
4. **any other configs** — e.g. `common`, `web`, `console`

## Merging Rules

- Configs are merged in dependency order (deepest dependency first)
- **Outer packages override inner ones** — the root project has full control
- Plugins only set default values; the root project can override anything
- Merging uses deep array merge (like `ArrayHelper::merge`)
- `?` prefix marks optional files: `"?config/params-local.php"`
- `$` references other config groups: `"web": ["$common", "config/web.php"]`

## Project Hierarchy (Package Composition)

Projects are organized as a hierarchy of plugins:

```
root (deployment-specific: .env, composer.json, params.php)
  └── main project (reusable project logic)
        ├── project plugins (feature modules)
        ├── third-party plugins
        └── base project (e.g. hisite — like yii2-app-basic as a plugin)
              └── framework (yii2)
```

- **Root** — minimal deployment-specific files: `.env`, `composer.json`, `src/config/params.php`. Contains credentials, environment-specific settings, and the list of plugins to include.
- **Main project** — reusable project code and config, pulled in via Composer.
- **Base project** — shared application foundation (replaces app templates like `yii2-app-basic`).

## Key Principles for AI Agents

- **Config lives with code**: each package/plugin carries its own config. Don't look for all config in one place — check each package's `composer.json` `extra.config-plugin` section and referenced config files.
- **Assembled output**: the actual merged config used at runtime is in `vendor/hiqdev/composer-config-plugin-output/`. Useful for debugging what the final config looks like.
- **Override direction**: root overrides project, project overrides base, base overrides framework defaults. To change a value, override it in the appropriate level.
- **Params flow down**: `params.php` values from all packages merge into `$params`, which is available in all other config files.
- **Adding features = adding plugins**: new functionality is added by requiring a new Composer package that carries its own config. No need to modify existing application code or config.
- **Removing features = removing plugins**: unrequire the package, run `composer update`, the config disappears automatically.
