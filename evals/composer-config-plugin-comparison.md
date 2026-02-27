# Comparison: composer-config-plugin vs. Alternatives

## The Core Problem

How does a package ship its own configuration and have it automatically integrated into the host application? This is what composer-config-plugin solves.

---

## PHP Ecosystem

| Approach | Auto-Discovery | Config Layering | Merge Strategy | Override Ease |
|----------|---------------|-----------------|----------------|---------------|
| **composer-config-plugin** (hiqdev/yiisoft) | `composer.json` extra | 3 layers: vendor/root/env | Deep recursive merge | Root layer overrides all |
| **Symfony Flex** | Centralized recipe repos | Copies files on install | None — files are yours after copy | Edit copied files |
| **Laravel** | `composer.json` extra | `mergeConfigFrom()` + publish | Shallow merge (1 level only) | Publish and edit |
| **WordPress** | File detection in plugins dir | None (DB-stored) | None | Admin UI / filters |
| **Drupal** | Module YAML + Recipes | Config API + settings.php overrides | Import/export YAML | Admin UI / config files |

### Key differences

- **Symfony Flex** — recipes copy config files into your project once. After that, they're yours. No ongoing merge. More comprehensive initial setup (handles Docker, Makefiles, .gitignore too), but config drift from upstream is your problem.
- **Laravel** — `mergeConfigFrom()` is elegant but only does shallow (1-level) merge. For deeply nested configs (like Yii2 components), this is insufficient. Laravel 11+ hides config by default — publish only what you override.
- **composer-config-plugin** — the only PHP solution with true deep recursive merge across the full dependency tree. The 3-layer system (vendor → root → environment) gives the clearest override semantics.

---

## Other Ecosystems

| System | Mechanism | Strength |
|--------|-----------|----------|
| **Spring Boot** (Java) | `@ConditionalOnMissingBean` + classpath scanning | Gold standard — "provide defaults, back off if user defines their own" |
| **Rails Engines** (Ruby) | Railtie auto-registration + initializer ordering | Full app-in-app encapsulation with convention-over-configuration |
| **NestJS** (Node.js) | `forRoot()`/`forFeature()` explicit imports | Strong TypeScript typing, but no auto-discovery |
| **Django** (Python) | Manual `INSTALLED_APPS` + settings.py | Maximum flexibility (it's Python code), but no automatic package config |

**Spring Boot** is the most sophisticated: conditional annotations let a starter provide beans only when the user hasn't defined their own. composer-config-plugin achieves similar results through merge order — the root package always wins — but Spring's approach is more granular (per-bean vs per-config-key).

---

## Strengths of the composer-config-plugin Approach

1. **True deep merge** — unlike Laravel's shallow merge or Symfony's copy-once, configs are deeply merged across the entire dependency tree
2. **Declarative** — config contributions declared in `composer.json`, not in runtime code (no bootstrap performance hit)
3. **Pre-built at install time** — assembled configs are written to disk during `composer install`, zero runtime overhead
4. **Clear override hierarchy** — dependency order guarantees: root > project > base > framework
5. **Framework-agnostic** — works with any PHP project using Composer, not tied to Yii
6. **Params flow** — the dotenv → defines → params → configs pipeline lets values cascade cleanly

---

## Weaknesses / Trade-offs

1. **Smaller ecosystem** — Symfony Flex and Laravel have vastly more community adoption and tooling
2. **Debugging opacity** — when 20 packages contribute to one merged config, tracing where a value came from is hard. Spring Boot has `--debug` and Actuator for this; composer-config-plugin has the output files but no equivalent tooling
3. **No conditional inclusion** — Spring Boot can say "only configure X if class Y exists on classpath." composer-config-plugin always merges everything that's required — the only way to exclude is to not require the package
4. **Merge conflicts** — deep recursive merge can produce unexpected results when multiple packages define the same nested keys. No conflict detection or warnings
5. **No post-install sync** — like Symfony Flex, changes to a plugin's config require `composer update` to propagate. No hot-reload

---

## Modern Best Practices Alignment

| Practice | composer-config-plugin | Assessment |
|----------|----------------------|------------|
| **12-Factor (config in env)** | Supports `.env` → params pipeline | Good — env vars flow into configs |
| **Convention over configuration** | Packages declare configs, app just `require`s | Good — zero boilerplate for consumers |
| **Secrets management** | `.env` files, `params-local.php` | Adequate for small teams; lacks encryption (vs Rails credentials, Vault) |
| **Config as code** | All config in PHP files in repos | Good — version-controlled, reviewable |
| **Environment separation** | Supports env-specific config layers | Good |
| **Dynamic config / feature flags** | Not addressed | Gap — no runtime config changes without redeploy |

---

## Bottom Line

The composer-config-plugin approach was **ahead of its time** in the PHP ecosystem (2016) — solving distributed config assembly before Symfony Flex (2017) and Laravel auto-discovery (2017). Its deep merge across the dependency tree remains **unique in PHP** and is the most complete solution when projects are organized as plugin hierarchies.

The main trade-off is **ecosystem size and tooling** — Symfony and Laravel have invested heavily in developer experience (debug toolbars, config caching, IDE integration) around their approaches. For projects already using this plugin system, the architecture is sound and aligns well with modern practices. The biggest gap vs. current best practices is **secrets management** (consider adding encrypted credentials) and **observability** (tooling to trace config provenance).
