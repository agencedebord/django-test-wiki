---
domain: conf
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - django/conf/__init__.py
  - django/conf/global_settings.py
  - django/conf/app_template/__init__.py-tpl
  - django/conf/app_template/views.py-tpl
  - django/conf/app_template/admin.py-tpl
  - django/conf/app_template/apps.py-tpl
  - django/conf/app_template/models.py-tpl
  - django/conf/app_template/tests.py-tpl
  - django/conf/project_template/project_name/__init__.py-tpl
  - django/conf/project_template/project_name/urls.py-tpl
  - django/conf/project_template/project_name/asgi.py-tpl
  - django/conf/project_template/project_name/wsgi.py-tpl
  - django/conf/project_template/project_name/settings.py-tpl
  - django/conf/project_template/manage.py-tpl
  - django/conf/urls/static.py
  - django/conf/urls/i18n.py
---

# Conf

## What this domain does
The `conf` domain is Django's settings infrastructure. It solves two core problems: deferring settings access until first use (to avoid circular imports at startup), and merging user-defined settings with framework defaults transparently. Almost every other domain depends on it via `from django.conf import settings`, making it the lowest-level shared dependency in the framework. `[llm-analyzed]`

## Key behaviors
- **Settings are loaded lazily, not at import time**: The `settings` object exported from `django.conf` is a `LazySettings` instance wrapping a `LazyObject`. Actual loading only happens on first attribute access. This is essential because many settings.py files themselves import Django modules, which would cause circular imports if settings were loaded eagerly at `import django.conf`. `[llm-analyzed]`
- **Attribute values are cached in __dict__ after first access**: `LazySettings.__getattr__` stores the resolved value directly into `self.__dict__` after first retrieval. Subsequent accesses bypass `__getattr__` entirely (Python's attribute lookup checks `__dict__` first). This is why `@override_settings` must replace `_wrapped` entirely — it triggers `__setattr__` with `name == '_wrapped'`, which calls `self.__dict__.clear()` to bust the full cache. `[llm-analyzed]`
- **MEDIA_URL and STATIC_URL receive SCRIPT_NAME prefix automatically**: In `LazySettings.__getattr__`, accessing `MEDIA_URL` or `STATIC_URL` triggers `_add_script_prefix()` before caching. This adds the Django SCRIPT_NAME prefix to relative paths (e.g., `static/` becomes `/myapp/static/`). Absolute paths and full URLs (starting with `http://`, `https://`, or `/`) are returned unchanged. This is intentional for subpath deployments where setting the full path in settings.py is inconvenient. `[llm-analyzed]`
- **SECRET_KEY emptiness is checked at access time, not at load time**: `LazySettings.__getattr__` raises `ImproperlyConfigured` if `SECRET_KEY` is falsy when accessed. The error is deferred to first use rather than thrown at startup, which allows tools that don't use cryptographic functions to operate without a SECRET_KEY. `[llm-analyzed]`
- **TIME_ZONE is validated against the OS zoneinfo database and applied to the process environment**: During `Settings.__init__`, if `time.tzset` is available (Unix only, excluded on Windows) and `/usr/share/zoneinfo` exists, the timezone is validated by checking for the corresponding file. If invalid, a `ValueError` is raised immediately at settings load. Valid timezones are written to `os.environ['TZ']` and applied via `time.tzset()`, making them effective process-wide. `[llm-analyzed]`

## Domain interactions
- **Utils (functional)**: LazySettings inherits from `LazyObject` (defined in `django.utils.functional`), which provides the `_wrapped`/`empty` lazy initialization protocol. The caching-in-`__dict__` pattern is an extension of this base class behavior. `[llm-analyzed]`
- **Django-urls**: `conf.urls.i18n` wraps URL patterns with `URLResolver`/`LocalePrefixPattern` from `django.urls`. It also calls `get_script_prefix()` from `django.urls` inside `_add_script_prefix` for MEDIA_URL/STATIC_URL handling. `[llm-analyzed]`
- **Core (exceptions)**: Raises `ImproperlyConfigured` from `django.core.exceptions` for missing DJANGO_SETTINGS_MODULE, empty SECRET_KEY, invalid tuple settings, and empty static prefix. This is the primary error type for misconfiguration. `[llm-analyzed]`
- **All other domains**: Nearly every domain imports `from django.conf import settings` and accesses settings as needed. The `conf` domain makes no assumptions about which settings will be accessed — it serves as a passive registry. It does not validate setting semantics beyond the explicit type checks (tuple_settings) and value checks (SECRET_KEY, TIME_ZONE). `[llm-analyzed]`

## Gotchas and edge cases
- Accessing any setting before `DJANGO_SETTINGS_MODULE` is set (and without calling `settings.configure()`) raises `ImproperlyConfigured`. The error message identifies which setting was accessed, which is useful for debugging startup order issues. `[llm-analyzed]`
- Mutating a list or dict setting returned from `settings` mutates the cached value in `__dict__`, not the underlying module. Code that does `settings.MIDDLEWARE.append(...)` will see the change for the rest of the process, but only because the cached copy was mutated. `[llm-analyzed]`
- `is_language_prefix_patterns_used()` is decorated with `@functools.cache`, so it caches results per URLconf string. If the URLconf is dynamically changed (e.g., in tests via `override_settings`), the cache may return stale results unless explicitly cleared. `[llm-analyzed]`
- The `static()` helper checks `settings.DEBUG` at call time (usually during URLconf loading), not per-request. If DEBUG changes after URLconf is loaded (unusual but possible in tests), the URL patterns are already fixed. `[llm-analyzed]`
- On Windows, TIME_ZONE validation and `os.environ['TZ']` / `time.tzset()` are skipped entirely. Timezone handling on Windows relies on Python's own zoneinfo support without process-level enforcement. `[llm-analyzed]`

## Dependencies
- [Base](../base/_overview.md)
- [Constraints](../constraints/_overview.md)
- [Contrib](../contrib/_overview.md)
- [Core](../core/_overview.md)
- [Db](../db/_overview.md)
- [Dispatch](../dispatch/_overview.md)
- [Django](../django/_overview.md)
- [Django-apps](../django-apps/_overview.md)
- [Django-urls](../django-urls/_overview.md)
- [Django-views](../django-views/_overview.md)
- [Expressions](../expressions/_overview.md)
- [Fields](../fields/_overview.md)
- [Forms](../forms/_overview.md)
- [Http](../http/_overview.md)
- [Indexes](../indexes/_overview.md)
- [Lookups](../lookups/_overview.md)
- [Middleware](../middleware/_overview.md)
- [Options](../options/_overview.md)
- [Proxy](../proxy/_overview.md)
- [Tasks](../tasks/_overview.md)
- [Templates](../templates/_overview.md)
- [Templatetags](../templatetags/_overview.md)
- [Utils](../utils/_overview.md)

## Referenced by
- [Base](../base/_overview.md)
- [Contrib](../contrib/_overview.md)
- [Core](../core/_overview.md)
- [Db](../db/_overview.md)
- [Dispatch](../dispatch/_overview.md)
- [Django](../django/_overview.md)
- [Django-apps](../django-apps/_overview.md)
- [Django-urls](../django-urls/_overview.md)
- [Django-views](../django-views/_overview.md)
- [Forms](../forms/_overview.md)
- [Http](../http/_overview.md)
- [Middleware](../middleware/_overview.md)
- [Tasks](../tasks/_overview.md)
- [Templates](../templates/_overview.md)
- [Templatetags](../templatetags/_overview.md)
- [Tests](../tests/_overview.md)
- [Utils](../utils/_overview.md)