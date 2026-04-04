---
domain: django-apps
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - django/apps/config.py
  - django/apps/registry.py
---

# Django-apps

## What this domain does
The django-apps domain implements Django's application registry — the system that discovers, loads, and manages installed Django applications at startup. It provides two core abstractions: `AppConfig` (metadata + lifecycle hooks for a single app) and `Apps` (the global registry coordinating all installed apps). Together they enforce a strict three-phase boot sequence: (1) import app modules and create AppConfig instances, (2) import model modules, (3) run `ready()` hooks. This ordering is intentional — models must be importable before `ready()` runs so that app code can safely reference... `[llm-analyzed]`

## Key behaviors
- **Auto-discovery of AppConfig subclasses from `apps.py`**: When an INSTALLED_APPS entry is a plain module path (e.g. `'myapp'`), `AppConfig.create()` checks for a `myapp.apps` submodule and inspects it for AppConfig subclasses. If exactly one subclass exists (excluding those with `default = False`), it is used automatically. If multiple exist, one must explicitly set `default = True` or Django raises RuntimeError. This means adding a second AppConfig to `apps.py` is a silent breaking change unless `default` is managed. `[llm-analyzed]`
- **Three-phase registry population with readiness guards**: `Apps.populate()` runs in three sequential phases protected by an RLock: phase 1 (app configs), phase 2 (models), phase 3 (`ready()` methods). `apps_ready` and `models_ready` flags are set between phases. Any code that calls `get_app_configs()` or `get_models()` before the corresponding phase completes raises `AppRegistryNotReady`. This prevents partial-state bugs where models are referenced before they exist. `[llm-analyzed]`
- **Idempotent but non-reentrant population**: `populate()` is safe to call from multiple threads simultaneously (guarded by RLock, double-checked `self.ready`), but it explicitly raises `RuntimeError` on reentrant calls within the same thread. This guards against AppConfig.ready() accidentally triggering another populate(), which would run ready() methods twice. `[llm-analyzed]`
- **`all_models` is never reset — all imported models are tracked regardless of INSTALLED_APPS**: Every time a model class is created (via `ModelBase.__new__`), it registers itself in `Apps.all_models` keyed by (app_label, model_name). This happens even for models in uninstalled apps or before `populate()` runs. `all_models` is never cleared or overridden because re-importing a module is unsafe (initialization code could re-execute). This means stale model registrations can accumulate across test runs unless `set_installed_apps()` is used. `[llm-analyzed]`
- **App label must be unique and a valid Python identifier**: The label defaults to the last component of the app's dotted path. It is validated as a Python identifier and must be globally unique across the project. Using the same label twice in INSTALLED_APPS raises `ImproperlyConfigured` immediately during phase 1, before any models are imported. `[llm-analyzed]`

## Domain interactions
- **Conf**: AppConfig.default_auto_field reads `settings.DEFAULT_AUTO_FIELD` lazily (cached_property). The registry's `check_apps_ready()` also accesses `settings.INSTALLED_APPS` to surface a more helpful ImproperlyConfigured error when settings are not yet configured. `[llm-analyzed]`
- **Core**: Raises `AppRegistryNotReady` and `ImproperlyConfigured` from `django.core.exceptions`. These are the primary error types callers should catch when accessing the registry before it is fully initialized. `[llm-analyzed]`
- **Utils**: Uses `django.utils.module_loading.import_string` and `module_has_submodule` to safely inspect app modules without triggering import errors. Uses `django.utils.functional.cached_property` for lazy settings access on AppConfig instances. `[llm-analyzed]`
- **Models**: The registry is the authority for model lookups. ModelBase (in the models domain) calls `apps.register_model()` on class creation, populating `all_models`. The registry then exposes `get_model()` and `get_models()` back to the models layer for reverse relation resolution and ORM queries. `[llm-analyzed]`
- **Django-views / templates / templatetags**: These domains query the registry via `apps.get_app_config()` or `apps.get_models()` to discover installed apps at runtime — for template tag registration, static file serving, and admin autodiscovery triggered from AppConfig.ready(). `[llm-analyzed]`

## Gotchas and edge cases
- Adding a second AppConfig subclass to `apps.py` without setting `default = False` on the old one (or `default = True` on the new one) raises RuntimeError at startup — not a helpful import error. `[llm-analyzed]`
- INSTALLED_APPS entries that look like class paths (last component starts uppercase) trigger a different error path with a 'Did you mean?' hint. Lowercase entries re-raise the original ImportError. This means a typo like `'myapp.Apps'` gives a better error than `'myApps'`. `[llm-analyzed]`
- `all_models` accumulates models from uninstalled apps. Test suites that import models from test-only apps may see them in the registry even when those apps are not in INSTALLED_APPS. `[llm-analyzed]`
- The `Apps` singleton for the main Django registry is created with `installed_apps=None` and populated lazily. Any other `Apps` instance (e.g., for isolated test setups) must pass `installed_apps` explicitly and is populated immediately in `__init__`. `[llm-analyzed]`
- `set_available_apps()` and `set_installed_apps()` use a stack (`stored_app_configs`) to save/restore state — these are meant for test isolation and must always be paired with their restore calls or the registry will be left in a broken state. `[llm-analyzed]`

## Dependencies
- [Conf](../conf/_overview.md)
- [Core](../core/_overview.md)
- [Django](../django/_overview.md)
- [Utils](../utils/_overview.md)

## Referenced by
- [Apps](../apps/_overview.md)
- [Base](../base/_overview.md)
- [Conf](../conf/_overview.md)
- [Contrib](../contrib/_overview.md)
- [Core](../core/_overview.md)
- [Db](../db/_overview.md)
- [Django](../django/_overview.md)
- [Django-views](../django-views/_overview.md)
- [Models](../models/_overview.md)
- [Namespace-package-base](../namespace-package-base/_overview.md)
- [One-config-app](../one-config-app/_overview.md)
- [Query-performing-app](../query-performing-app/_overview.md)
- [Templates](../templates/_overview.md)
- [Templatetags](../templatetags/_overview.md)
- [Tests](../tests/_overview.md)
- [Two-configs-app](../two-configs-app/_overview.md)
- [Two-configs-one-default-app](../two-configs-one-default-app/_overview.md)
- [Two-default-configs-app](../two-default-configs-app/_overview.md)
- [Utils](../utils/_overview.md)