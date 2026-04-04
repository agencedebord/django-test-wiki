---
domain: django-urls
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - django/urls/resolvers.py
  - django/urls/conf.py
  - django/urls/__init__.py
  - django/urls/utils.py
  - django/urls/exceptions.py
  - django/urls/converters.py
  - django/urls/base.py
---

# Django-urls

## What this domain does
The django-urls domain is Django's URL routing engine. It handles two core problems: resolving incoming URL paths to view functions (resolve), and generating URLs from view names and arguments (reverse). It is the central nervous system connecting HTTP requests to application logic, and is imported by nearly every other domain in the project. `[llm-analyzed]`

## Key behaviors
- **URL resolution: path → view function**: resolve() takes a URL path string and walks the URLResolver tree (built from ROOT_URLCONF) to find a matching URLPattern. The result is a ResolverMatch object carrying the matched view callable, positional args, keyword kwargs, url_name, app_names, namespaces, and the matched route string. Resolution is thread-local and urlconf-overridable via set_urlconf(). `[llm-analyzed]`
- **URL reversal: view name → URL string**: reverse() reconstructs a URL from a view name (dotted path or named URL), optional args/kwargs, and an optional namespace chain separated by colons (e.g. 'admin:auth:login'). It navigates the namespace_dict of resolvers using current_app context to select the right namespace instance when multiple app instances exist. Returns a string with optional ?query and #fragment appended. reverse_lazy() wraps this in a lazy evaluation proxy for use at import time. `[llm-analyzed]`
- **Two pattern types: path() vs re_path()**: path() uses RoutePattern which accepts Django's <type:name> converter syntax and produces human-readable patterns. re_path() uses RegexPattern which accepts raw regular expressions. Both share the same _path() factory; they differ only in the Pattern class injected via functools.partial. RoutePattern patterns are converted to regex internally via _route_to_regex (cached, cache cleared on register_converter). `[llm-analyzed]`
- **Type-converting URL parameters via converters**: Converters (int, str, slug, uuid, path) define a regex for matching, to_python() for converting the captured string to a Python type, and to_url() for the reverse direction. UUIDConverter returns uuid.UUID objects, not strings. Custom converters can be registered via register_converter(), but duplicate names raise ValueError immediately. Registering a converter clears both get_converters and _route_to_regex caches. `[llm-analyzed]`
- **Namespace resolution for multi-instance apps**: When reversing a namespaced URL like 'polls:detail', reverse() checks app_dict to see if the namespace is an app identifier with multiple instances. If current_app matches one of those instances, that instance's namespace is preferred. Otherwise it falls back to the first registered instance. This allows the same app installed multiple times under different namespaces to reverse to the correct instance based on request context. `[llm-analyzed]`

## Domain interactions
- **Conf (django.conf.settings)**: Reads ROOT_URLCONF as the default URLconf when no override is set. Also reads APPEND_SLASH to decide whether to warn about patterns starting with '/'. `[llm-analyzed]`
- **Django-views**: Imports django.views.View in _path() solely to detect the 'passed instance instead of .as_view()' mistake and raise a helpful TypeError. No functional dependency beyond this guard. `[llm-analyzed]`
- **Http (django.http)**: Resolver404 inherits from Http404, making unresolved URLs automatically trigger Django's 404 handler when raised during request processing. reverse() accepts QueryDict as the query= argument and calls its urlencode() method. `[llm-analyzed]`
- **Utils (django.utils.*)**: Uses django.utils.functional.lazy for reverse_lazy, django.utils.translation.override and get_language for locale-aware patterns, django.utils.regex_helper.normalize for route-to-regex conversion, and django.utils.http for RFC3986 character escaping during reversal. `[llm-analyzed]`
- **Core (django.core.checks)**: URLResolver.check() integrates with the system check framework via check_resolver, emitting warnings (e.g. urls.W002 for patterns starting with '/') during manage.py check. `[llm-analyzed]`
- **Middleware**: LocaleMiddleware calls set_urlconf() to swap in a locale-specific URLconf per request, and translate_url() is used by the language-switching view. The script prefix API (set_script_prefix) is called by WSGIHandler/ASGIHandler before each request. `[llm-analyzed]`
- **Templates / templatetags**: The {% url %} template tag calls reverse() (or reverse_lazy) internally. NoReverseMatch is caught and re-raised as a template rendering error. `[llm-analyzed]`

## Gotchas and edge cases
- clear_url_caches() must be called in tests after modifying urlpatterns at runtime, otherwise the cached resolver continues matching old patterns. `[llm-analyzed]`
- UUIDConverter.to_python() returns a uuid.UUID object, not a string. View code that expects a string UUID in kwargs will break silently or with an unexpected AttributeError. `[llm-analyzed]`
- reverse() with a namespace requires current_app to be set correctly on the request or passed explicitly; without it, it falls back to the first registered instance of the app, which may not be the intended one. `[llm-analyzed]`
- register_converter() raises ValueError if the type name is already registered (including built-in names like 'int', 'str'). There is no update/replace path — you cannot override a built-in converter. `[llm-analyzed]`
- The script prefix stored by set_script_prefix() is per-thread (asgiref.Local). In async contexts, it flows with the task, but code that spawns new threads/tasks must re-set the prefix. `[llm-analyzed]`

## Dependencies
- [Conf](../conf/_overview.md)
- [Core](../core/_overview.md)
- [Django](../django/_overview.md)
- [Django-views](../django-views/_overview.md)
- [Http](../http/_overview.md)
- [Utils](../utils/_overview.md)

## Referenced by
- [Base](../base/_overview.md)
- [Cond-get-urls](../cond-get-urls/_overview.md)
- [Conf](../conf/_overview.md)
- [Contrib](../contrib/_overview.md)
- [Core](../core/_overview.md)
- [Db](../db/_overview.md)
- [Django](../django/_overview.md)
- [Django-views](../django-views/_overview.md)
- [Extra-urls](../extra-urls/_overview.md)
- [Middleware](../middleware/_overview.md)
- [Templates](../templates/_overview.md)
- [Templatetags](../templatetags/_overview.md)
- [Tests](../tests/_overview.md)
- [Tests-custom-error-handlers](../tests-custom-error-handlers/_overview.md)
- [Urls](../urls/_overview.md)
- [Utils](../utils/_overview.md)