# Comparison: composer-config-plugin vs. Alternatives

## The Core Problem

How does a package ship its own configuration and have it automatically integrated into the host application? This is what composer-config-plugin solves.

---

## PHP Ecosystem

| Approach | Auto-Discovery | Config Layering | Merge Strategy | Override Ease |
|----------|---------------|-----------------|----------------|---------------|
| **hiqdev/composer-config-plugin** | `composer.json` extra | 2 layers: vendor + root, ordered by dependency depth | Always deep recursive merge | Root package overrides all |
| **yiisoft/config** (Yii3) | `composer.json` extra | 4 layers: vendor / vendor-override / application / environment | Configurable per-group (recursive, reverse, removal) | Application + environment layers override |
| **Symfony Flex** | Centralized recipe repos | Copies files on install | None — files are yours after copy | Edit copied files |
| **Laravel** | `composer.json` extra | `mergeConfigFrom()` + publish | Shallow merge (1 level only) | Publish and edit |
| **WordPress** | File detection in plugins dir | None (DB-stored) | None | Admin UI / filters |
| **Drupal** | Module YAML + Recipes | Config API + settings.php overrides | Import/export YAML | Admin UI / config files |

### Key differences

- **Symfony Flex** — recipes copy config files into your project once. After that, they're yours. No ongoing merge. More comprehensive initial setup (handles Docker, Makefiles, .gitignore too), but config drift from upstream is your problem.
- **Laravel** — `mergeConfigFrom()` is elegant but only does shallow (1-level) merge. For deeply nested configs (like Yii2 components), this is insufficient. Laravel 11+ hides config by default — publish only what you override.
- **hiqdev/composer-config-plugin** — deep recursive merge across the full dependency tree. Simple 2-layer system with dependency-depth ordering.
- **yiisoft/config** — the evolution of the above, adding configurable merge strategies, 4-layer system, vendor-override layer, and environment support.

---

## hiqdev/composer-config-plugin vs. yiisoft/config (Yii3)

`yiisoft/config` is a complete rewrite of the original `hiqdev/composer-config-plugin`. The core idea is the same — packages declare configs in `composer.json` and they are automatically assembled — but the architecture differs significantly.

### Architecture: Build-Time vs. Runtime Assembly

| Aspect | hiqdev/composer-config-plugin | yiisoft/config |
|--------|-------------------------------|----------------|
| **Assembly timing** | Build-time (full merge on `composer update`) | Runtime (lazy assembly on `Config::get()`) |
| **Build output** | Pre-merged PHP files in `vendor/.../output/` | Only a merge plan manifest (`.merge-plan.php`) |
| **Config loading** | `require Builder::path('web')` — single file include | `$config->get('web')` — programmatic assembly |
| **Runtime overhead** | Zero — just a `require` | Config files are `require`d and merged per-request (cached within request) |

**hiqdev** approach: Composer plugin does the heavy lifting — scans packages, reads all config files, merges arrays with `ArrayHelper::merge`, writes pre-built PHP files. Application just `require`s the output.

**yiisoft** approach: Composer plugin only produces a manifest listing which files go where. All actual merging happens at runtime via `Config::get()`, enabling dynamic modifiers and lazy loading.

### Layer System

| Aspect | hiqdev | yiisoft |
|--------|--------|---------|
| **Layers** | 2: vendor packages + root package | 4: vendor → vendor-override → application → environment |
| **Ordering** | Dependency-depth traversal (children before parents) | Composer's package order, then override, then root |
| **Vendor-override** | Not available | Configurable — selected packages get an intermediate override layer |
| **Environments** | Via optional files (`?params-local.php`) | First-class: `config-plugin-environments` with named environments (dev/test/prod) |

### Merge Control

| Aspect | hiqdev | yiisoft |
|--------|--------|---------|
| **Default merge** | Always deep recursive (`ArrayHelper::merge`) | Strict: duplicate string keys throw `ErrorException` |
| **Recursive merge** | Always on | Opt-in per group: `RecursiveMerge::groups('params', 'events')` |
| **Reverse merge** | Not available | `ReverseMerge::groups('events')` — application handlers first |
| **Key removal** | Not available | `RemoveFromVendor::keys(...)` — selectively remove vendor config keys |
| **Duplicate detection** | Silent override (last wins) | Strict error with file paths listed |

### Features Comparison

| Feature | hiqdev | yiisoft |
|---------|--------|---------|
| **Special config types** | `dotenv`, `defines`, `params`, arbitrary | `params` (special), all others arbitrary |
| **File formats** | PHP, JSON, YAML, .env | PHP only |
| **File markers** | `?` (optional), `$` (reference) | `?` (optional), `$` (reference), `*` (glob wildcard) |
| **Sub-config access** | Not available | `$config->get('routes')` within config files |
| **Config declaration** | Only via `composer.json` extra | Also via PHP file (`config-plugin-file`) |
| **Debugging** | Read output files in output dir | `composer yii-config-info` command + readable merge plan |
| **Rebuild** | `composer dump-autoload` | `composer yii-config-rebuild` |
| **Config copy** | Not available | `composer yii-config-copy vendor/pkg` |

### Yii3 Config Groups Convention

The Yii3 app template establishes a convention of config groups with inheritance via `$` references:

```
params         → base params
params-web     → $params + web params
params-console → $params + console params

di             → base DI definitions (common/di/*.php)
di-web         → $di + web DI definitions
di-console     → $di

events         → base events
events-web     → $events + web events
events-console → $events + console events

routes, bootstrap, di-delegates, di-providers...
```

Each vendor package contributes to these groups. The DI container is built from the assembled arrays:
```php
$container = new Container(
    ContainerConfig::create()
        ->withDefinitions($config->get('di-web'))
        ->withProviders($config->get('di-providers-web'))
        ->withDelegates($config->get('di-delegates-web'))
);
```

### Assessment

The yiisoft/config rewrite addresses several weaknesses of the original:
- **Strict duplicate detection** prevents silent config conflicts
- **Configurable merge strategies** replace the one-size-fits-all deep merge
- **Vendor-override layer** supports intermediate "platform" packages
- **First-class environments** replace ad-hoc `?local.php` files
- **Debugging tooling** (`yii-config-info`, `yii-config-copy`) improves observability

The trade-off is **runtime overhead** (configs assembled per-request instead of pre-built) and **PHP-only** format support (dropped JSON, YAML, .env parsing).

---

## Other Ecosystems

| System | Mechanism | Strength |
|--------|-----------|----------|
| **Spring Boot** (Java) | `@ConditionalOnMissingBean` + classpath scanning | Gold standard — "provide defaults, back off if user defines their own" |
| **Rails Engines** (Ruby) | Railtie auto-registration + initializer ordering | Full app-in-app encapsulation with convention-over-configuration |
| **NestJS** (Node.js) | `forRoot()`/`forFeature()` explicit imports | Strong TypeScript typing, but no auto-discovery |
| **Django** (Python) | Manual `INSTALLED_APPS` + settings.py | Maximum flexibility (it's Python code), but no automatic package config |

**Spring Boot** is the most sophisticated: conditional annotations let a starter provide beans only when the user hasn't defined their own. Both composer-config-plugin variants achieve similar results through merge order — the root/application package always wins — but Spring's approach is more granular (per-bean conditional logic vs per-config-key override).

---

## Strengths of the composer-config-plugin Approach

1. **True deep merge** — unlike Laravel's shallow merge or Symfony's copy-once, configs are deeply merged across the entire dependency tree
2. **Declarative** — config contributions declared in `composer.json`, not in runtime code (no bootstrap performance hit)
3. **Clear override hierarchy** — outer packages override inner ones; the root has full control
4. **Framework-agnostic** — works with any PHP project using Composer, not tied to Yii
5. **Params flow** — the dotenv → defines → params → configs pipeline lets values cascade cleanly
6. **Zero-boilerplate plugin installation** — `composer require` a package and its config is automatically integrated

**hiqdev-specific strength:** Pre-built at install time — assembled configs are written to disk, zero runtime overhead.

**yiisoft-specific strengths:** Configurable merge strategies, strict duplicate detection, vendor-override layer, first-class environments, debugging tooling.

---

## Weaknesses / Trade-offs

1. **Smaller ecosystem** — Symfony Flex and Laravel have vastly more community adoption and tooling
2. **No conditional inclusion** — Spring Boot can say "only configure X if class Y exists on classpath." composer-config-plugin always merges everything that's required — the only way to exclude is to not require the package
3. **Debugging opacity** — when many packages contribute to one merged config, tracing where a value came from is hard (yiisoft/config improves this with `yii-config-info` and strict duplicate errors)
4. **Merge conflicts** — deep recursive merge can produce unexpected results when multiple packages define the same nested keys (yiisoft/config addresses this with strict-by-default + opt-in recursion)

---

## Modern Best Practices Alignment

| Practice | composer-config-plugin | Assessment |
|----------|----------------------|------------|
| **12-Factor (config in env)** | Supports `.env` → params pipeline | Good — env vars flow into configs |
| **Convention over configuration** | Packages declare configs, app just `require`s | Good — zero boilerplate for consumers |
| **Secrets management** | `.env` files, `params-local.php` | Adequate for small teams; lacks encryption (vs Rails credentials, Vault) |
| **Config as code** | All config in PHP files in repos | Good — version-controlled, reviewable |
| **Environment separation** | hiqdev: optional files; yiisoft: first-class environments | Good (yiisoft better) |
| **Dynamic config / feature flags** | Not addressed | Gap — no runtime config changes without redeploy |

---

## Bottom Line

The composer-config-plugin approach was **ahead of its time** in the PHP ecosystem (2016) — solving distributed config assembly before Symfony Flex (2017) and Laravel auto-discovery (2017). Its deep merge across the dependency tree remains **unique in PHP** and is the most complete solution when projects are organized as plugin hierarchies.

**yiisoft/config** validated the approach by adopting and extending it, adding configurable merge strategies, strict safety checks, and better tooling. The architectural shift from build-time to runtime assembly trades zero-overhead loading for greater flexibility.

The main trade-off vs. mainstream frameworks is **ecosystem size and tooling** — Symfony and Laravel have invested heavily in developer experience around their approaches. For projects using this plugin system, the architecture is sound and aligns well with modern practices. The biggest gaps vs. current best practices are **secrets management** (consider encrypted credentials) and **observability** (tooling to trace config provenance — partially addressed in yiisoft/config).
