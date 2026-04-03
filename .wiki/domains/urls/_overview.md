---
domain: urls
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - django/urls/__init__.py
  - django/urls/base.py
  - django/urls/conf.py
  - django/urls/converters.py
  - django/urls/exceptions.py
  - django/urls/resolvers.py
  - django/urls/utils.py
---

# Urls

## What this domain does

The `django/urls/` package is Django's URL routing engine. It is responsible for two complementary operations:

1. **Resolving** -- mapping an incoming request path to a view callable, extracting positional/keyword arguments along the way.
2. **Reversing** -- generating a URL string from a view name (or callable) and arguments, which is the inverse of resolving.

The package provides the public API used in every Django project's `urls.py` files (`path()`, `re_path()`, `include()`) and the lower-level resolver machinery that the request handler invokes on every request.

## Key behaviors

- **`path()` and `re_path()`** (defined in `conf.py`) are the two ways to register URL patterns. `path()` uses `RoutePattern` with angle-bracket converters (`<int:pk>`); `re_path()` uses `RegexPattern` with raw regexes. Both produce either a `URLPattern` (leaf endpoint) or a `URLResolver` (when wrapping an `include()`).

- **Path converters** (`converters.py`) handle type coercion between URL segments and Python values. Five built-in converters exist: `int`, `str`, `slug`, `path`, `uuid`. Custom converters can be registered via `register_converter()`, which invalidates the `_route_to_regex` and `get_converters` caches.

- **`URLResolver._populate()`** builds three per-language lookup dictionaries (`_reverse_dict`, `_namespace_dict`, `_app_dict`) by walking the URL tree in reverse order. These are populated lazily on first access and cached per language code, enabling i18n URL patterns.

- **`reverse()`** (`base.py`) walks namespace segments separated by colons, resolves them against `namespace_dict` and `app_dict`, then delegates to `_reverse_with_prefix()`. It supports `query` (dict or `QueryDict`) and `fragment` keyword-only arguments for building full URLs. `reverse_lazy` is a lazy wrapper for use at module level.

- **Thread/async-local state** (`base.py`) -- `_prefixes` (script prefix) and `_urlconfs` (overridden URLconf) are stored on `asgiref.Local()` instances, making them safe for both threaded WSGI and async ASGI contexts.

- **`translate_url()`** resolves a URL, then re-reverses it under a different language to produce an i18n-translated URL.

- **Caching** -- Three `functools.cache`/`lru_cache` layers exist: `_get_cached_resolver`, `get_ns_resolver`, and `get_callable`. All are cleared together by `clear_url_caches()`.

- **System checks** -- `RegexPattern`, `RoutePattern`, and `URLPattern` implement `check()` methods that emit warnings (`urls.W001` through `urls.W010`) and errors (`urls.E009`) for common misconfigurations (trailing `$` in includes, leading `/`, colons in names, unmatched angle brackets, class-based views passed without `.as_view()`).

- **`ResolverMatch`** is the value object returned by resolution. It carries `func`, `args`, `kwargs`, namespace chain, route string, `captured_kwargs`, and `extra_kwargs`. It is explicitly non-picklable.

## Domain interactions

- **`django.core.handlers`** (`base.py`, `wsgi.py`, `asgi.py`) -- The request handlers call `set_urlconf()`, `set_script_prefix()`, and `resolve()` on every request to dispatch to a view.
- **`django.conf.urls`** -- Re-exports `include` and provides `i18n_patterns` via `LocalePrefixPattern` and `URLResolver`.
- **`django.core.checks.urls`** -- Invokes `check_resolver()` on the root URL resolver during `manage.py check`.
- **`django.template.defaulttags`** -- The `{% url %}` template tag calls `reverse()`.
- **`django.shortcuts`** -- `redirect()` uses `reverse()` internally.
- **`django.middleware.common`** -- `CommonMiddleware` uses `is_valid_path()` for `APPEND_SLASH` redirect logic.
- **`django.middleware.locale`** -- Uses `translate_url()` and `is_valid_path()` for locale-aware URL handling.
- **`django.test.utils`** -- `override_settings(ROOT_URLCONF=...)` triggers `clear_url_caches()` via signal.
- **`django.contrib.admin`**, **`django.contrib.auth`**, **`django.contrib.admindocs`**, **`django.contrib.flatpages`**, **`django.contrib.sitemaps`** -- All define their own `urls.py` using `path()`/`re_path()` and `include()`.

## Gotchas and edge cases

- **Mixing `*args` and `**kwargs` in `reverse()`** raises `ValueError`. This is enforced in `_reverse_with_prefix()`.
- **`include()` with `i18n_patterns`** is explicitly forbidden and raises `ImproperlyConfigured` -- `LocalePrefixPattern` can only appear at the root level.
- **Namespace without `app_name`** -- Passing `namespace` to `include()` without an `app_name` on the included module raises `ImproperlyConfigured`.
- **`_populate()` recursion guard** -- Uses a thread-local `populating` flag to prevent infinite recursion when URL patterns have circular references, but concurrent threads may populate simultaneously.
- **Converter registration is one-shot** -- `register_converter()` raises `ValueError` if the type name is already registered; there is no `unregister`.
- **`RoutePattern.match()` fast path** -- When there are no converters, `RoutePattern` bypasses regex entirely and uses direct string comparison (`==` for endpoints, `startswith` for prefixes), which is a significant performance optimization.
- **`reverse()` with `query={}` or `query=QueryDict()`** that evaluates to an empty string does not append a `?` to the URL -- the `if query_string:` guard prevents it.
- **Lazy-translated routes** -- Both `RegexPattern` and `RoutePattern` support lazily-translated strings, compiling per-language regex variants cached in `_regex_dict`.

## Dependencies

- `asgiref.local.Local` -- Thread/async-local storage for prefix and urlconf overrides.
- `django.conf.settings` -- Reads `ROOT_URLCONF`, `APPEND_SLASH`, `LANGUAGE_CODE`.
- `django.http.QueryDict` -- Used by `reverse()` for query string building.
- `django.utils.functional` -- `lazy()` for `reverse_lazy`, `cached_property` on `URLResolver`.
- `django.utils.translation` -- `get_language()` and `override()` for i18n URL support.
- `django.utils.regex_helper` -- `normalize()` and `_lazy_re_compile()` for regex processing.
- `django.utils.http` -- `RFC3986_SUBDELIMS` and `escape_leading_slashes` for safe URL encoding.
- `django.core.checks` -- `Error` and `Warning` classes for the system check framework.
- `django.core.exceptions` -- `ImproperlyConfigured`, `ViewDoesNotExist`.

## Referenced by

- `django.core.handlers.base` / `wsgi` / `asgi` -- URL resolution on every request.
- `django.template.defaulttags` -- `{% url %}` tag.
- `django.shortcuts` -- `redirect()`.
- `django.middleware.common` -- `APPEND_SLASH` logic via `is_valid_path()`.
- `django.middleware.locale` -- Locale-aware redirects via `translate_url()`.
- `django.test.utils` / `django.test.client` -- URL overrides and test request dispatching.
- `django.contrib.admin` -- Admin URL registration and reversing.
- `django.contrib.auth` -- Auth URL patterns.
- `django.contrib.admindocs` -- Documentation URL introspection.
- `django.contrib.sitemaps` -- Sitemap URL generation.
- `django.conf.urls` -- Re-exports and `i18n_patterns`.
- `django.views.i18n` -- Language switching view uses `translate_url`.
- `django.views.debug` -- Technical 404 page inspects resolver data.
